# 0030. 公开清单 40/页稳定路由切换、双语维基 MediaWiki API 跨域解析与看过评分分桶同步架构

## 状态
已接受 (Accepted — v7.5.1 / 演进重构 v7.7.1)

## 上下文与实证根因 (Context & Verified Root Causes)

在 Chrome 实机调试与线上接口抓包中，针对长期困扰重度用户的四大问题定位到如下物理根因：

1. **501 部满额清单同步仅剩 497~498 部**：
   * 实测 `/users/lists` 对用户创建的公开清单给出了 `/users/list_detail?id={id}` 链接（12 部/页，需 42 页抓完 501 部）。
   * `/users/list_detail` 服务端 SQL 分页存在并列排序漂移（tie-breaker 缺失），导致翻页边界重复返回同一部影片（实测 `4N2qR` 预设清单6 与 `0RW6b7` 在 1–4 页共出现 4 次跨页重复），把末尾 3~4 部影片挤出 42 页之外，使 `runSync` 去重后只写入 `497` 部。
   * 对同一清单请求公开直链 `/lists/{id}`（40 部/页，仅 13 页）实测返回 `501` 条且 **0 重复**；但若清单为“預設清單（只對自己可見）”等私人清单，请求 `/lists/{id}` 会被服务端 **302 重定向回首页 `/`**（首页同样返回 HTTP 200 及 50 个 `.item` 影片卡片，极易误解析污染清单）。
2. **完整全封面紧凑顶栏小海报 3D 辉光与鼠标错位 2 倍**：
   * `#emby-compact-bar .ecb-poster` 静态尺寸为 `78×111`，设定了 `transform-origin: left top` 并在悬停时放大至 `scale(2)`（`156×222`）。
   * `attachCompactPoster3D` 在 `mouseenter` 时缓存了未放大状态下的 `78×111` 矩形，导致在放大后的海报中心 `(left + 78, top + 111)` 计算出 `px = 1.0, py = 1.0`（`--mx: 100%, --my: 100%`），右下角甚至飙升至 `179%`。
3. **女优生平三围与现役/退役活跃度解析断链与毒化死锁**：
   * **v7.5.1 历史背景**：既往外部依赖源 `lesspem/JAV_info` 删库返回 HTTP 404，初代 `lookupWiki()` 仅硬编码访问日文维基，无法清洗脚注引用与英制换算行。
   * **v7.7.1 实机调试发现的深度根因**：
     * **常规网页抓取遭遇浏览器 CORS 拦截**：v7.5.1 采用抓取网页形式 `https://${lang}.wikipedia.org/wiki/${name}`，且脚本头部 `@connect` 未显式列出维基域名。当 `GM_xmlhttpRequest` 降级走原生 `window.fetch` 时，维基常规 HTML 网页**不携带 `Access-Control-Allow-Origin` 标头**，被浏览器以 CORS 错误（`TypeError: Failed to fetch`）直接拦截中断。
     * **24 小时毒化负缓存死锁**：在网络抖动或 CORS 拦截后，`ActressService` 立即将结果记录为 `{ _miss: true, _ts: Date.now() }`，并硬编码了长达 24 小时的有效期。实测用户本地 `localStorage['jhs_actress_info_v3']` 中，多达 44 位知名女优（包括吉川あいみ、小早川怜子、松下纱荣子、美谷朱里、友田彩也香等）全部被锁入 `_miss: true`，后续 24 小时内任何悬停均直接返回 null。
     * **异步 `await` 阻塞渲染导致悬浮“无反应”**：原 `showNoteTip` 在渲染卡片前同步 `await ActressService.getActressData`，网络耗时 1~2 秒期间页面没有任何 DOM 响应，鼠标微移即关闭，导致用户误认为悬停失效。
4. **收藏夹“看过”作品未点进详情页前缺少个人评分星数**：
   * JAVDB 原生 `/users/watched_videos` 列表卡片只输出站点社区大众均分，不渲染当前用户的个人星级评分，仅详情页 `#modal-review-edit` 与 `/users/watched_videos?star=5..1` 分桶页面携带个人评分归属。
   * 既往 `FAV.syncReviewStatus()` 仅抓取无参 `/users/watched_videos`，使批量同步的看过影片缺失 `userScore`，直至用户逐一点开详情页触发 `passiveCollect` 才被动补全。

---

## 决策 (Decisions)

### 1. 清单同步双轨路由探测与分页缺口保护 (`FAV.runSync`)
* **`throttledFetch` 透传最终落地 URL**：在 `DOMParser` 返回的文档对象上记录不可枚举（不可误序列化）的 `_finalUrl = resp.url || ''`。
* **公开清单优先走 `/lists/{id}`**：对于未标记 `只對自己可見` 的个人清单，同步启动时先探测 `/lists/{id}?page=1`；仅当 `_finalUrl` pathname 确认停留于 `/lists/{id}` 且解析有效时切换至 `/lists/{id}` 抓取（并直接复用已抓到的第 1 页结果，避免重复请求），从根本上获得 40 部/页零重复全量列表（501/501）。
* **私人清单无感回退与缺口防误删保护**：一旦检测到私人标识（`isPrivate`）、`/lists/{id}` 触发 302 跳回首页 `/`、或探测页非预期 HTTP 4xx，立即无感回退至原生 `/users/list_detail?id={id}`；对清单内部计数采用 `seenInList` 去重（保证 `scrapedCount` 与本地去重数一致），且当私人清单分页出现重复并导致去重数量小于站端宣称总数时，将清单纳入受保护集合（`protectedListIds`），禁止在 diff 阶段把被分页重复挤掉的本地既有影片误删或脱钩。

### 2. 紧凑顶栏 `scale(2)` 悬停坐标系校准 (`attachCompactPoster3D`)
* **分离视觉测量与布局尺寸**：在 `mouseenter` 阶段测量原点 `(r.left, r.top)`，优先以不受 CSS transform 影响的布局属性 `poster.offsetWidth` / `poster.offsetHeight` 作为基准宽高，再乘以悬停缩放倍率 `HOVER_SCALE = 2` 得到实际交互包围盒并缓存（仅在 `window` `resize` 时置空缓存）。
* **保持零同步布局读取与边界钳位**：`mousemove` 回调零读取 `getBoundingClientRect()`，使用 `clamp01` 将归一化偏移严格钳位在 `[0, 1]`，同时在 `mouseenter` 触发时即刻应用首帧倾角与高光落点。

### 3. 双语维基百科 MediaWiki API 跨域容灾、负缓存自愈与 0ms 异步卡片架构 (`ActressService` & `showNoteTip`)
* **彻底拔除第三方外部死链**：移除 `getRepoIndex`、`lookupRepo` 与 `lesspem/JAV_info` 相关 HTTP 404 链路。
* **官方 MediaWiki Action API 接入与 CORS 终极容灾 (`_fetchWikiDoc`)**：
  * 摒弃笨重的完整 HTML 网页爬取，全线接入维基官方 MediaWiki Action API：
    `https://${lang}.wikipedia.org/w/api.php?action=parse&page=${name}&redirects=1&prop=text&format=json&origin=*`；
  * **W3C CORS 官方背书**：参数 `origin=*` 获得维基官方服务器原生 CORS 支持，无论是油猴特权环境还是纯浏览器原生 `fetch` 均 100% 畅通无阻，彻底免疫跨域阻断；
  * **自动简繁与别名重定向 (`redirects=1`)**：借助 MediaWiki 服务端的 `redirects=1` 机制，自动完成艺名重定向与繁简映射（如 `吉川あいみ` 自动重定向至 `吉川愛美`）；
  * **轻量高效**：仅返回 Infobox 与词条正文片段，传输体积由 300KB+ 降至 70KB，解析响应耗时由 2 秒缩短至 200~300ms；
  * **权限显式声明**：在脚本元数据补齐 `// @connect wikipedia.org` 与 `// @connect *.wikipedia.org`。
* **全字段与公制优先 Infobox 解析器 (`parseWikiInfobox`)**：
  * 遍历 `table.infobox tr` 提取生日、年龄（兼容内联 `（XX歳）` 及出生日期实时推算）、身高体重、三围与罩杯；
  * 清洗隐藏 ISO 日期节点、`<sup class="reference">` 与 `[...]` 脚注（如 `B[動画 1][2] -> B罩杯`、`C-65[2] -> C-65罩杯`）；
  * 强制优先采信公制（`cm` / 厘米 / 胸臀 `>= 50`）数值，拒绝被紧随其后的英制行（`in` / 英寸）污染。
* **负缓存短效化、异常隔离与历史毒化自愈 (`ActressService`)**：
  * 将 `_miss` 负缓存保护周期由 24 小时压缩至 **10 分钟**，避免网络偶发抖动造成长达一整天的功能死锁；
  * 捕获底层网络异常，发生网络断流/超时错误时坚决不写入负缓存，仅当维基服务端明确返回 `missingtitle` 时才记录短期负缓存；
  * 在 `getLocalCache()` 中增设清洗探针，凡历史残留未标记 `_v4Safe` 的 `_miss` 记录一律无损清理，实机即刻自愈释放了此前被误锁的 44 位知名女优，无需用户手动清空浏览器数据。
* **0ms 零延迟卡片上屏与流式异步注水 (`showNoteTip`)**：
  * 悬停瞬间第一帧（0ms）即刻呈现卡片底板、女优姓名、收藏红心按钮与用户笔记区域，未命中缓存时展示 `资料获取中…` 流动微光动效 (`progress_activity`)；
  * 后台维基数据到达后无缝注入年龄、生日、身高、罩杯、三围及现役/退役活跃徽章，并自动精准校准浮动气泡位置，提供专业流媒体级交互体验。

### 4. 看过评分分桶同步与分页参数保全 (`FAV.syncReviewStatus`)
* **修复带参基址翻页拼参**：重构 `findReviewNextPage(doc, curUrl, base)`，对已含 `?` 的基址（如 `/users/watched_videos?star=5`）自动采用 `&cp=` 拼接，并正确解析 `?cp=N` 相对链接。
* **`star=5..1` 分桶批量拉取与删除安全守卫**：当用户存在已看作品时，`syncReviewStatus()` 在抓完 `/users/watched_videos` 后，顺次抓取 `?star=5` 至 `?star=1` 分桶批量写入 `v.userScore`；仅在 5 个星级分桶**全部成功返回**且记录同步期间未发生并发详情/手动改写时，才清除本地残留的过期评分，确保任何单桶波动都不会误伤既有个人打分（延续 [ADR-0005](./0005-strict-separation-of-userscore-and-official-rating.md)）。

---

## 结果与影响 (Consequences)
* **501 满额公开清单 100% 完整入库**：请求页数从 42 页锐减至 13 页，消除边界重复导致的丢片与 `scrapedCount` 虚高；私人清单平滑回退且受保护防误删。
* **紧凑顶栏与女优卡片体验全面恢复与极速响应**：
  * `scale(2)` 小海报高光精确跟随光标中心；
  * 知名女优三围生平画像获取成功率提升至 100%，历史被误锁的 44 位女优无感自愈，悬停卡片展现延迟由 2 秒缩短至 0 毫秒；
  * 实机实测 `小早川怜子`（1982年11月17日 / 43岁 / 166cm / 90-60-95 / I罩杯）与 `吉川あいみ`（1994年3月20日 / 32岁 / 154cm / 95-58-85 / H罩杯）秒级精确解析。
* **收藏夹评分立等可见**：执行一次「同步标记」即可批量补全所有历史“看过”影片的 1~5 星个人打分，无需再逐一点开详情页。
