# 0014. Rails UJS DELETE 会话生命周期与 Emby 抽屉安全登出管道架构

## 状态
已接受 (Accepted)

## 上下文
在 JAVDB 站点使用 JavdbEmbySkin 时，用户反馈了一个困扰已久的隐蔽异常：
> “脚本无法使用退出账号的功能（https://javdb.com/logout ）。退出包括关闭皮肤之后，去 JAVDB 原生页面退出账号，刷新之后账号又恢复回登陆状态了；打开皮肤之后，再关闭后，就又登陆上了。”

通过使用 Chrome 远程调试协议（CDP）实机挂载、网络抓包分析与源码审查，深入挖掘出以下根本原因：

1. **Rails RESTful UJS 架构与 DELETE 协议强制约束**：
   * JAVDB 服务端基于 Ruby on Rails 体系开发，其用户注销（Logout）严格遵循 RESTful 架构规范。
   * JAVDB 原生导航栏中的登出按钮并非普通的超链接，而是由 Rails UJS 托管的复合节点：
     ```html
     <a data-confirm="確定要退出嗎?" class="navbar-item" rel="nofollow" data-method="delete" href="/logout">登出</a>
     ```
   * 浏览器原生点击该链接时，Rails UJS 会拦截默认事件，动态构造一个带 `_method=delete` 和 CSRF Token (`authenticity_token`) 的隐藏 POST 表单提交至 `/logout`，由服务端控制器执行 Session 销毁与 Cookie 注销。

2. **服务端对 `GET /logout` 的重定向欺骗陷阱**：
   * JAVDB 服务端**坚决拒绝通过 HTTP GET 方式执行注销**。
   * 若直接访问 `https://javdb.com/logout`（即 HTTP GET），服务端**完全不会销毁**会话 Cookie（`_jdb_session`），反而会设置 Cookie `redirect_to=%2Flogout` 并将请求 302 重定向至 `https://javdb.com/login`（页面提示“歡迎登入 / 此內容需要登入才能查看或操作”）。
   * **视觉误导**：这导致用户误以为自己已经退出了账号，但实际上服务端的登录会话毫发无损！当用户回到主页刷新、或者关闭皮肤重载页面时，浏览器请求仍然自动携带有效的 `_jdb_session`，服务端直接判定为已登录并返回用户主页，造成“账号阴魂不散、开关皮肤又复活”的幽灵登录假象。

3. **JavdbEmbySkin 脚本中的微观缺陷**：
   * **属性丢失缺陷**：在 `buildAccountMenuHtml()`（原 6764、6802 行）从原生 navbar 提取账户菜单子项时，仅提取了 `textContent` 和 `href`，彻底遗漏了 `data-method="delete"`、`data-confirm` 与 `rel="nofollow"`。Emby 抽屉输出的仅仅是普通 `<a href="/logout">`，用户在抽屉内点击“登出”直接触发了无效的 `GET /logout`，掉入上述服务端假退出陷阱。
   * **Cookie 粗放正则误判缺陷**：在 `isJavdbUserLoggedIn()`（原 121 行），历史代码采用 `/remember_user_token|_session/i.test(document.cookie)` 判定登录态。然而 JAVDB 即使在未登录的游客状态下，也会下发图形验证码 Cookie `_rucaptcha_session_id`。因其包含 `_session` 子串，导致即使服务端已注销，该函数仍误判当前为“已登录”。

## 决策
坚持第一性原理与手术级精准修复，在不重构任何稳定模块的前提下，建立**双通道安全登出管道（Safe Logout Pipeline）与严格 Cookie 判别体系**：

1. **严格 Cookie 认证正则收敛**：
   * 将 `isJavdbUserLoggedIn()` 中的 Cookie 检测正则表达式精准化：
     ```javascript
     if (document.cookie && /(?:^|;\s*)(?:remember_user_token|_javdb_session)\s*=/i.test(document.cookie)) return true;
     ```
   * 彻底杜绝 `_rucaptcha_session_id` 等非认证类 Cookie 引发的登录态误判。

2. **抽屉账户项完整属性保全与动作类注入**：
   * 在 `buildAccountMenuHtml()` 中克隆提取原生节点时，完整保全 `data-method`、`data-confirm` 和 `rel` 属性，并正确转义渲染：
     ```javascript
     const methodAttr = e.dataMethod ? ' data-method="' + escapeAttr(e.dataMethod) + '"' : '';
     const confirmAttr = e.dataConfirm ? ' data-confirm="' + escapeAttr(e.dataConfirm) + '"' : '';
     const relAttr = e.rel ? ' rel="' + escapeAttr(e.rel) + '"' : '';
     ```
   * 针对登出项自动注入专用标识类 `.emby-logout-action`。

3. **双通道安全登出管道（Safe Logout Pipeline）**：
   * 在 `buildChrome()` 中对 `.emby-logout-action` 注册专属拦截与安全处理通道：
     * **用户确认**：遵循原生交互规范，先执行 `confirm(data-confirm || '確定要退出嗎?')`；
     * **本地凭据清理**：主动清空本地可能残留的前端凭据（`jb_saved_username`、`jb_appAuthorization`、`jhs_appAuthorization`）；
     * **通道一（优先）**：查询隐藏在原生 navbar 中的原生登出节点（`nav.navbar a[data-method="delete"]`），调用 `.click()` 委托给官方 Rails UJS 触发原生注销流程；
     * **通道二（兜底）**：若原生节点未就绪或被移除，动态构造隐藏的标准 Rails Form 表单，注入 `_method=delete` 与当前页面的 CSRF Token (`meta[name="csrf-token"]`) 并执行 `form.submit()`，100% 触发服务端的 DELETE 请求。

## 原因与权衡
* **为什么不直接用 `fetch('/logout', { method: 'DELETE' })`？**
  * 使用 AJAX `fetch` 注销后还需要手动处理 HTTP 302 重定向、Cookie 刷新以及全页面状态清理；而构造标准 Form 提交或委托原生 Rails UJS 链接，能利用浏览器原生整页导航生命周期，让服务端 Set-Cookie 响应头自动生效并顺滑跳转至 JAVDB 官方注销完成页，体验最纯粹且不会残留任何前端脏状态。

## 结果与影响
* **收益**：
  * 用户在 Emby 抽屉点击“登出”能够 100% 发起标准的 Rails DELETE 请求，真实销毁服务端的 `_jdb_session` 会话；
  * 彻底消除了登出后刷新、开关皮肤重新进入时“账号幽灵复活”的 Bug；
  * 杜绝了 `_rucaptcha_session_id` 导致的游客登录态误判；
  * 零 DOM 结构破坏，零多余重构，保持了 25,000+ 行巨型脚本的架构稳定性。
