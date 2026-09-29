# ADR-0020: 未登录状态全链路行为拦截与数据库孤儿清洗对称性架构 (Unauthenticated Session Defense & Symmetrical Orphan Purging Architecture)

## 状态 (Status)
**已接受 (Accepted)** - 2026-09-29

## 上下文 (Context)

JavdbEmbySkin 深度集成了 JAVDB 官方账号的“想看/看過（评分）”标记体系、自定义清单以及女优收藏同步功能。随着系统功能的不断扩展，在未登录（游客态）与已登录会话之间出现了一系列逻辑脱节与数据清洗漏洞：

1. **状态快捷标记未登录击穿与幽灵记录生成**：
   - JAVDB 原生站点的 Rails 架构为**所有访客（含未登录游客）**的 `<head>` 均渲染了 `csrf-token`；
   - 卡片快捷标记组件（`attachCoverStatusAndPreview`）、评分模态弹窗（`showCoverRatingModal`）以及后台写入函数（`updateCoverReviewStatus`）最初仅检测 `csrfMeta` 是否存在，完全缺乏登录态校验（`isJavdbUserLoggedIn()`）；
   - 当游客在卡片上点击“想看”或“已看”时，浏览器向 JAVDB 发起 `POST /v/:id/reviews` 请求，由于同源凭证为空，JAVDB 服务端返回 `302 Found` 重定向至 `/login`；
   - 现代浏览器 `fetch` 默认自动跟随 302，最终拿到登录页的 HTML 并返回 `status === 200`（即 `res.ok === true`）；
   - 脚本误判为请求成功，顺理成章地执行了 `FAV.dbPutMovie(m)`，将无云端归属的幽灵记录（Phantom Records）写入了本地 IndexedDB 数据库。

2. **数据库静默安全清洗的非对称性漏洞**：
   - 脚本在 `FAVUI.reload` 时会自动在后台调用 `dbPurgeOrphanMovies` 清洗误入库的非收藏孤儿作品（如 TOP250 残留数据）；
   - 在旧版清洗算法中，设计者试图保护用户资产，设置了如下豁免白名单：
     ```javascript
     if (m.customMeta || (typeof m.userScore === 'number' && m.userScore > 0) || (m.notes && typeof m.notes === 'string' && m.notes.trim())) return false;
     ```
   - 标记“看過”时，必然附带 1~5 星的主观评分（`m.userScore`）；而标记“想看”时，`m.userScore` 为 `undefined`；
   - 当 `dbPurgeOrphanMovies` 运行在未登录态时，未登录标记的“想看”因无 `userScore` 且无清单归属，被正确判定为孤儿清理；但标记的“看過”却因为匹配了 `m.userScore > 0` 被豁免清除，永久残留在数据库中，形成了“想看被清、看过不灭”的严重非对称 Bug；
   - 此外，当已登录用户主动在“数据管理 -> 清理残留”中确认“一并清理无清单归属的想看/看過标记残留作品”时，所有无清单的“看過”作品同样因为该豁免而无法被清除，违背了用户的显式清理意愿。

3. **女优收藏与数据同步的游客态穿透**：
   - 女优悬浮卡片（`.ap-fav-btn`）点击事件未检测登录状态，同样向 JAVDB 发起 POST 请求并跟随 302，导致未登录状态下本地 `FavoriteActressManager` 误记录红心关注；
   - 女优详情页原生 `collectBtn` 监听器在未登录用户点击打开登录框时，延时 600ms 将演员加入了本地收藏集合；
   - 收藏夹各同步入口（`syncReviewStatus`、`runSync`、`syncListMeta`、`autoSync`）在未登录状态下缺乏前置守卫，白白消耗网络请求并抛出异常。

---

## 架构决策 (Architecture Decisions)

为彻底阻断未登录会话产生的脏数据污染，并实现孤儿清洗的绝对逻辑一致性，系统实施以下架构级重构：

### 1. 全链路四层未登录拦截守卫 (Four-Tier Unauthenticated Defense)

针对所有依赖 JAVDB 账号云端会话的操作，构建从“DOM 事件层”到“网络请求层”的四层防御纵深：

1. **第一层：UI 入口点击拦截 (Interactive UI Guard)**
   - 卡片悬停操作组：`.btn-set-wanted`（想看）、`.btn-set-watched`/`.btn-modify`（已看/评分）、`.btn-delete`（删除）在 `click` 回调起始处必须先执行：
     ```javascript
     if (!isJavdbUserLoggedIn()) {
       showToast('⚠️ 未登录 JAVDB 账号，无法标记，请先登录！');
       return;
     }
     ```
   - 详情页女优悬浮气泡中的收藏按钮（`.ap-fav-btn`）与女优个人主页的收藏按钮（`#button-collect-actor`）在触发前必须拦截：
     ```javascript
     if (!isJavdbUserLoggedIn()) {
       showToast('⚠️ 未登录 JAVDB 账号，无法收藏演员，请先登录！');
       return;
     }
     ```

2. **第二层：模态视窗呈现拦截 (Modal Presentation Guard)**
   - 封面评分浮层控制器 `showCoverRatingModal`：若检测到 `!isJavdbUserLoggedIn()`，严禁生成或弹出 `.cover-modal-base` DOM 结构，直接弹窗提醒并中断流程，杜绝因外部调用导致的穿透。

3. **第三层：数据底层与穿透请求拦截 (Underlying Action Guard)**
   - `updateCoverReviewStatus`、`FavoriteActressManager.remoteToggleFavorite`、`FavoriteActressManager.sync`：在组织请求体之前，强制校验 `isJavdbUserLoggedIn()`，未登录时直接抛出异常或提示退出，严禁进入网络交互。

4. **第四层：302 隐式重定向防护网 (Redirected Response Guard)**
   - 在所有 fetch 响应接收处，显式检验重定向目标：
     ```javascript
     if (res.redirected || (res.url && /sign_in|login/i.test(res.url))) {
       throw new Error('未检测到登录状态或登录已过期，请重新登录 JAVDB');
     }
     ```
   - 彻底封死因浏览器自动跟随 HTTP 302 到 `/login` 且返回 200 OK 导致的“伪成功入库”隐患。

### 2. 数据库孤儿清洗对称性修正 (Symmetrical Orphan Purging Invariant)

明确定义作品实体的资产权属与生命周期：
1. **纯本地资产（永久保全）**：
   - `m.customMeta`：用户在 Emby 皮肤内手动进行的片名、演员、片商纠错，属于绝对不可丢失的人工本地资产；
   - `m.notes`：用户针对作品撰写的本地个人备注，属于独立于 JAVDB 云端的私有数据。
2. **云端伴生属性（非独立资产）**：
   - `m.userScore` 仅为 JAVDB 官方“看過”标记的伴生评级参数（1~5星），其生命周期应当与 `reviewStatus === 'watched'` 严格绑定，绝不能赋予超越标记本身的免清洗特权。
3. **清洗逻辑重构**：
   - 从 `dbPurgeOrphanMovies` 的免死白名单中移除 `(typeof m.userScore === 'number' && m.userScore > 0)`：
     ```javascript
     // 修复后：严格仅保全真正的本地自定义纠错与备注
     if (m.customMeta || (m.notes && typeof m.notes === 'string' && m.notes.trim())) return false;
     ```
   - **登录状态下的常态静默清洗（`cleanUnlinkedStatus === false`）**：
     由于 `hasStatus = !!m.reviewStatus` 为 true，无论“想看”还是“看過”（带评分），均正常受 `(!loggedIn || cleanUnlinkedStatus)` 保护，不会被静默清理；
   - **未登录状态或深度清理（`!loggedIn || cleanUnlinkedStatus === true`）**：
     无清单归属的“想看”与“看過”（无论带不带打分），均作为无归属残留孤儿被平权且对称地彻底清理。

---

## 架构影响与收益 (Consequences & Benefits)

1. **数据一致性与数据库纯净度**：
   彻底杜绝游客在浏览 JAVDB 时因无意触发标记或女优收藏而向本地写入无法同步至云端的虚假残留，避免本地收藏夹与 JAVDB 账号状态撕裂。
2. **清洗逻辑严密对称**：
   纠正了历史版本中“想看可被清理，看过永不消除”的逻辑缺陷，使背景自愈与手动深度清理行为完全符合用户直觉与产品设计规范。
3. **网络与性能保护**：
   所有同步任务（清单同步、标记同步、女优同步）在未登录状态下秒级拒绝，避免了无意义的网络轮询、异常日志刷屏以及不必要的 Cloudflare 风控消耗。
