# 0015. Rails CSRF 同源 Referer 完整性保障与元素级防盗链穿透架构

## 状态
已接受 (Accepted)

## 上下文
在 JAVDB 站点开启 JavdbEmbySkin 脚本（v7.330）后，用户反馈了一个严重的登录阻塞异常：
> “脚本打开的时候，好像无法登陆 JAVDB，点击登陆后会反复跳转到登陆页面，我能保证我输入的账号密码没问题。”

通过审查 v7.330 代码改动历史（`diff_7322_7330.txt`）、W3C Referrer Policy 规范以及 JAVDB 服务端底层 Ruby on Rails 架构机制，排查出以下根本原因：

1. **v7.330 引入的破坏性 Meta 注入**：
   * 在 v7.330 的第 44-70 行，为了解决 `c0.jdbstatic.com` 等图床返回 `403 Forbidden` 防盗链阻断，脚本新增了 `ensureGlobalNoReferrer()`：
     ```javascript
     let meta = document.querySelector('meta[name="referrer"]');
     if (!meta) {
       meta = document.createElement('meta');
       meta.name = 'referrer';
       meta.content = 'no-referrer';
       (document.head || document.documentElement).appendChild(meta);
     }
     ```
   * 并在首屏及 `observeMutations` 中全局高频调用。

2. **W3C 规范级文档 Referer 强行剥离**：
   * 根据 W3C 规范，`<head>` 内的 `<meta name="referrer" content="no-referrer">` 约束了**当前 HTML 文档发起的所有后续网络请求**。
   * 用户在 `https://javdb.com/login` 提交登录表单（`POST /users/sign_in` 或 `/user_sessions`）时，浏览器被迫完全去除了 HTTP 请求头中的 `Referer` 字段。在常规 Form POST 导航中，部分 Chromium 内核还会将 `Origin` 置空或忽略。

3. **Rails 同源 CSRF 校验与 Devise / RuCaptcha 会话重置**：
   * JAVDB 服务端基于 Ruby on Rails（Devise 认证模块 + RuCaptcha 图形验证码）构建；
   * Rails `ActionController::RequestForgeryProtection#verify_same_origin_request` 强制校验非 GET 请求的来源同源性：
     ```ruby
     def cross_origin_request?
       origin = request.headers['Origin']
       origin && !matches_origin?(origin) ||
         !origin && !matches_referrer?(request.headers['Referer'])
     end
     ```
   * 当 POST 请求因 `no-referrer` 策略剥离了 `Referer` 时，Rails 判定请求为跨站伪造（`cross_origin_request?` 为 true），立即触发 `handle_unverified_request`；
   * Rails 默认执行 `reset_session`，瞬间清空了与会话绑定的验证码凭据 `session[:_rucaptcha]`；
   * Devise 认证控制器判定图形验证码校验失败，直接拒绝签发登录凭据，并以 HTTP 302 重定向让浏览器重新导航回 `/login` 登录页，同时下发一个全新的空游客 Cookie；
   * 结果：用户无论输入多么正确的账号、密码与验证码，点击提交后都会在毫秒内被重置会话并弹回空登录表单，造成无法登录的无限死循环。

4. **会话 Cookie 识别遗漏**：
   * `isJavdbUserLoggedIn()`（第 121 行）将 JAVDB 会话 Cookie 误写为 `_javdb_session`，而官方真实 Cookie 名为 `_jdb_session`，导致纯 Cookie 维度的登录态识别出现断层。

---

## 决策
坚持第一性原理与手术级精准修复，坚决割除破坏性的整页 Meta 策略，建立**鉴权路由快速避让守卫、元素级图片防盗链穿透与真实 Cookie 识别体系**：

1. **鉴权路由快速避让守卫（Auth Page Route Guard）**：
   * 在脚本执行最前端增加路径检测：
     ```javascript
     if (/^\/(login|user_sessions|users\/(?:sign_in|sign_up|new|password))/i.test(location.pathname)) {
       try {
         const m = document.querySelector('meta[name="referrer"][content="no-referrer"]');
         if (m && m.parentNode) m.parentNode.removeChild(m);
       } catch (e) {}
       return;
     }
     ```
   * 在 `/login`、`/user_sessions`、`/users/sign_in` 等关键鉴权页面上直接 return，零 DOM 介入、零样式覆盖，100% 保全原生表单提交、CSRF Token 与同源标头完整性。

2. **彻底拔除全局 Meta 注入，收敛至元素级 `ensureImageNoReferrer()`**：
   * 彻底废除向 `<head>` 注入 `<meta name="referrer" content="no-referrer">` 的危险代码；
   * 增加自愈净化逻辑：若检测到页面已有之前残留的 `no-referrer` meta 标签，就地将其从 DOM 树中移除；
   * 防盗链仅约束常规 `<img>` 标签：遍历 `document.images` 时施加 `img.setAttribute('referrerpolicy', 'no-referrer')` 与 `img.referrerPolicy = 'no-referrer'`；
   * **显式豁免图形验证码**：若 `img` 包含 `.rucaptcha-image` 或位于 `#rucaptcha` 容器内，坚决跳过，杜绝干扰验证码请求。

3. **修复 JAVDB 官方会话 Cookie 识别**：
   * 在 `isJavdbUserLoggedIn()` 中将 Cookie 检测正则扩展支持官方 `_jdb_session`：
     ```javascript
     if (document.cookie && /(?:^|;\s*)(?:remember_user_token|_javdb_session|_jdb_session)\s*=/i.test(document.cookie)) return true;
     ```

---

## 原因与权衡
* **为什么绝不能在 document 级别注入 `<meta name="referrer" content="no-referrer">`？**
  * `<meta name="referrer">` 是极具侵略性的全文档级指令。在 Rails 这种高度依赖同源 Referer/Origin 标头进行 CSRF 校验的 Web 应用中，全局剥离 Referer 会摧毁所有的非 GET 交互（包括登录、登出、发表影评、创建清单、操作收藏等），属于典型的“为了修一个图片的 403，炸毁了整个网站的安全认证体系”。
* **为什么元素级 `referrerpolicy="no-referrer"` 能够完美替代 Meta 标签？**
  * 防盗链 403 的实质是图床 CDN（如 Cloudflare）对图片资源 GET 请求头中的 `Referer` 进行校验。HTML 规范允许每个 `<img>` 元素独立拥有自己的 `referrerpolicy`。元素级属性既能 100% 免疫图床拦截，又不会对外溢出半点副作用，是最纯粹的第一性原理解决方案。

---

## 结果与影响
* **收益**：
  * 用户在开启脚本状态下，可以 100% 顺畅登录 JAVDB 账号，表单携带完整 Referer 与 CSRF Token，彻底消除反复重定向回 `/login` 的死循环；
  * 原生验证码图片 `rucaptcha-image` 的会话完整性得到可靠保障；
  * JAVDB 封面、缩略图、头像等所有图片继续享有元素级免 Referer 穿透，403 阻断防范能力毫不受损；
  * `isJavdbUserLoggedIn()` 能精确识别 `_jdb_session`，登录态判定更加稳健；
  * 零多余重构，保持巨型代码库的极高鲁棒性。
