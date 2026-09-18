# 0009. 敏感凭据双重脱敏导出与防原型链污染安全架构

## 状态
已接受 (Accepted)

## 上下文
随着插件增加了 WebDAV、GitHub/Gitee 云端备份以及“导出配置与数据 JSON”功能，用户经常会在社区交流、分享自己的备份文件或截图。在早期版本中，备份包可能会完整序列化整个 `localStorage`，导致用户的 WebDAV 账号密码、Git 个人访问令牌（Personal Access Token）以及 JAVDB 登录 Cookie 凭据被无意中打包公开，存在巨大的凭据泄露风险；同时，导入第三方分享的恶意 JSON 备份文件还可能面临原型链污染（Prototype Pollution）与 XSS 跨站脚本攻击。

## 决策
我们决定**实施严格的凭据导出白名单脱敏审查机制，并在导入反序列化时设立纵深防御（Defense-in-Depth）安全屏障**：
1. **导出时强力脱敏过滤**：
   * 无论通过 `exportJSON` 还是云端同步，默认情况下（`javdb_backup_include_auth !== true`），系统**强制剥离**所有敏感鉴权字段：
     * `jhs_appAuthorization` (移动端与网页端鉴权头)
     * `jb_jdsignature` (私有数字签名)
     * WebDAV 密码与 URL 认证信息
     * GitHub / Gitee 的访问 Token
   * 仅在用户于设置面板中显式勾选“备份包含密码与敏感凭据”并弹窗确认后，才允许携带导出。
2. **导入反序列化彻底免疫原型链污染**：
   * 在 `importJSON`、`restoreAllConfig` 及批量元数据处理时，临时建立的哈希字典必须一律使用 `Object.create(null)`，严禁使用对象字面量 `{}`：
     ```javascript
     const oldById = Object.create(null);
     const decided = Object.create(null);
     const merged = Object.create(null);
     ```
   * 这保证了即使导入的 JSON 文件中含有恶意属性键 `"__proto__"`、`"constructor"` 或 `"prototype"`，也不会被写入 JavaScript 全局 Object 原型中，彻底封死攻击面。
3. **全局动态渲染防御性 XSS 转义**：
   * 对用户输入的笔记（`note.content`）、自定义标题、自定义演员名、外部播放站点模板，必须严格经过 `escapeHtml()` 与 `escapeAttr()` 转义后方可插入 DOM；
   * 对自定义播放站点网址模板进行协议过滤，严禁 `javascript:` 或 `data:text/html` 等危险伪协议注入。

## 原因与权衡
* **保护普通用户的隐私底线**：普通用户往往缺乏安全审计意识，在论坛求助或分享整理好的收藏库时，极易直接把包含个人私密 Token 的文件公开发布。默认脱敏是守护用户隐私最根本的一道防线。
* **反序列化漏洞的隐蔽性**：油猴脚本拥有操作整个网页甚至发起跨域请求的高级权限。一旦发生原型链污染或 XSS 注入，恶意脚本可以窃取用户在 JAVDB 上的全部账号权限与敏感数据。

## 结果与影响
* **收益**：用户可放心分享备份文件；彻底消除恶意 JSON 导入引发的远程代码执行（RCE）与原型污染风险。
* **代价**：当合法用户需要在新设备上完整克隆包含 WebDAV 密码的私密环境时，必须在导出前主动勾选确认。
