# JavdbEmbySkin 完整架构上下文与领域模型 (CONTEXT.md)

JavdbEmbySkin 是一个在浏览器油猴环境（Tampermonkey / Violentmonkey）中运行的大型单文件 UserScript（25,367 行，v7.331）。它将 JAVDB 原生站点全面重构为现代化的 Emby 视觉风格，并内置了从数据采集、多级持久化、多维流式画廊、元数据纠错自愈、媒体库统计分析、多源播放矩阵，到云端多端同步与高清头像映射在内的完整多媒体数据管理系统。

---

## 领域词汇表 (Domain Language & Glossary)

### 1. 核心实体与元数据体系

**Movie Record (m)**:
收藏夹与列表页的核心影片实体对象，主键为唯一番号（`code`），包含标题 (`title`)、封面 (`cover`)、时长、日期、演员列表、自定义纠错及主观标记。
_Avoid_: Video, Film, TorrentItem, MediaItem

**userScore**:
用户对影片的**个人主观星级评分**（数字 1~5，0 或缺失代表未评分）。具有最高主观优先级，不得被任何外部抓取数据覆盖。
_Avoid_: rating, starCount, score, myScore

**rating.score**:
JAVDB 官方社区大众评分（浮点数 0.0~5.0），属于影片只读客观元数据，随官方页面更新而浮动。
_Avoid_: userScore, personalScore, myRating

**reviewStatus**:
用户对影片的看片标记状态，严格限制为 `'want'`（想看）、`'watching'`（在看）、`'watched'`（已看）或 `null`（未标记）。
_Avoid_: status, watchState, markType

**customMeta**:
用户在元数据纠错管理器中手动修正并加锁的自定义元数据字典（覆盖片名、演员名单、片商、标签、封面）。
_Avoid_: userMeta, editData, patchInfo, overrideMeta

**officialMeta**:
从 JAVDB 原始详情页或搜索接口解析出的未经人工修改的官方元数据快照。
_Avoid_: rawData, siteMeta, defaultInfo

**Entity Token & Noise Filter**:
从页面 `<h2>` 标题中提取实体候选名时，针对演员按标点分割，但针对片商（如 `S1 NO.1 STYLE`）保留空格且保留日文间隔号（`・`），并剔除 `ENTITY_NAME_NOISE`（如“部影片”、“作品”、“出演”等噪声词）。
_Avoid_: rawTitle, cleanString, entitySlug

---

### 2. 详情页排版与封面四态引擎

**Cover Crop (裁切海报模式)**:
海报钉住原宽幅封面右侧主体内容，裁掉左侧留白与多余区域，使 2:3 竖版框内满铺，封面顶部与标题永远保持水平平行锚定。
_Avoid_: PosterMode, LeftCrop, RightFit

**Cover Full (全封面模式)**:
原汁原味呈现完整 16:9 宽幅大封面；当页面向下滚动大图离开视口时，顶栏下方自动浮现固定紧凑条。
_Avoid_: BigCover, WideMode, RawImage

**Cover Fusion (融合模式 - v7.257 重构)**:
顶态时呈现纯正大封面，下方舒展排布标题；当滚动导致标题触顶时，通过滚动监听与 CSS 矩阵变换平滑收缩转为两栏裁切海报布局。
_Avoid_: StickyCover, DynamicLayout, MorphPoster

**Cover Hybrid (混合模式 - 参考 Letterboxd)**:
三栏排布体系：左栏大封面固定，中栏展示标题、详细元数据与剧情简介，右栏展示操作按钮行与多源播放列表。
_Avoid_: ThreeColumn, LetterboxdStyle, FlexHero

---

### 3. 视觉与渲染引擎

**LiquidGlass (液态玻璃)**:
基于纯 SVG 矢量位移滤镜 (`<feDisplacementMap>`) 和 CSS 动态混合的拟物流体玻璃态渲染风格，彻底避开 Canvas 跨域安全污染。
_Avoid_: Glassmorphism, BlurEffect, CanvasFilter

**Glass (毛玻璃)**:
基于 CSS `backdrop-filter: blur(...)` 的经典现代化磨砂半透明视觉风格。
_Avoid_: FrostedGlass, BlurSkin, LightTheme

**Emby Native (经典皮肤)**:
高度还原 Emby Web 客户端深色卡片布局与排版的经典 UI 模式。
_Avoid_: DarkMode, DefaultTheme, EmbyStyle

**WaterfallEngine (瀑布流引擎)**:
根据视口宽度动态计算多列高度、调度卡片绝对定位、并在演员折叠徽章展开与图片懒加载时执行防抖重排的布局控制器。
_Avoid_: MasonryGrid, ColumnLayout, FlexGrid

**Native Grid Coexistence Guard (原生网格共存守卫)**:
当用户关闭 Emby 皮肤时，通过同步拔除容器 `.emby-masonry` 类名并切断所有后台布局引擎异步调度的防御机制，确保 JAVDB 原生基于 CSS Grid 的 4/5 列排版 100% 无缝复原。
_Avoid_: ClearLayout, SkinDisabler, ResetGrid

**Element-level Image Referrer Policy (元素级防盗链免溯源策略)**:
严禁在 `<head>` 注入全局 `<meta name="referrer" content="no-referrer">`（避免破坏 Rails CSRF 同源校验与表单登录 POST 标头），仅在常规图片属性级施加 `referrerPolicy = 'no-referrer'` 并主动排除图形验证码（`rucaptcha-image`）。配合鉴权路由严格避让守卫，从根本上免疫官方图床 403 Forbidden 拦截同时确保整站会话与登录完整。
_Avoid_: GlobalNoReferrer, Anti403, ImageFix, RefererHack

**Steam 3D Tilt (卡牌高光倾角)**:
鼠标在卡片悬停移动时，实时计算光标偏移比例，赋予卡牌透视 3D 倾斜（`transform: perspective(1000px) rotateX(...) rotateY(...)`）并渲染跟随高光。
_Avoid_: HoverShine, TiltEffect, CardHover

---

### 4. 头像与媒体资产

**Gfriends Avatar (高清头像)**:
由 Gfriends 开源仓库索引收录的 400x400 分辨率定妆照，覆盖男女演员，具有最高头像展示优先级。
_Avoid_: GirlAvatar, ActressPic, HDAvatar

**Official Avatar (官方头像)**:
JAVDB 官方托管在 `c0.jdbstatic.com` 的演员缩略图，作为 Gfriends 未收录或加载失败时的平滑降级底图。
_Avoid_: SiteAvatar, JdbPic, DefaultAvatar

**Avatar Fallback Badge (首字渐变徽章)**:
当官方头像亦不存在时，通过演员姓名哈希算法生成的彩色渐变背景配单字纯文本微标。
_Avoid_: DefaultIcon, EmptyAvatar, Placeholder

---

### 5. 存储、持久化与同步底层

**FAV (收藏夹存储系统)**:
封装 IndexedDB `javdb-emby-fav-db` 数据库访问、事务队列、内存双向映射与实体持久化的底层单例模块。
_Avoid_: DBManager, StorageService, IndexedDBHelper

**MetaStore (元数据倒排表)**:
IndexedDB 中独立的 `meta` 表（对象仓库），用于存储非影片实体数据（如 7天 TTL 的 Gfriends 头像倒排索引树、系统配置标记）。
_Avoid_: ConfigTable, SystemKV, CacheStore

**Passive Collection (被动静默采集)**:
用户在日常浏览 JAVDB 影片列表或详情页时，后台自动增量补全本地数据库缺失字段的数据采集机制。
_Avoid_: AutoScrape, BackgroundSync, AutoSave

**Desensitized Backup (脱敏备份)**:
排除所有第三方授权凭据（WebDAV 密码、Git Token、App 鉴权凭证）的纯净配置与收藏夹数据导出结构。
_Avoid_: PublicBackup, SafeJson, CleanExport

---

### 6. 数据统计、榜单与播放矩阵

**StatisticsCenter (媒体库数据统计中心)**:
基于 IndexedDB 全表异步并发聚合的数据统计面板，提供核心资产 KPI 汇总（片量、女优、笔记、清单、订正数）、评分阶梯文字与百分比进度条（神作/优秀/良好/普通）、年代标签以及女优/片商/导演/系列排行榜卡片。
_Avoid_: ChartDashboard, DataVisualization, ChartCenter

**Top250ViewManager (热播与 TOP250 榜单)**:
管理总榜、日榜、周榜、月榜的异步抓取与渲染，利用**请求代数令牌（Sequence Token / `top250LoadSeq`）**严格防御异步乱序回包覆盖。
_Avoid_: RankingManager, BillboardService, TopList

**PlaySitesService (多源播放站点矩阵)**:
整合 123AV, njavTV, MissAV, Jable, NetFlav 等 10+ 外部在线播放站，支持 `{code}`, `{code_lower}`, `{code_num}` 动态模板变量替换与有效性探测。
_Avoid_: VideoLinks, PlayerHub, OnlineSource

---

## 架构子系统全景地图 (Architecture Map Across 25,179 Lines)

```
                       ┌──────────────────────────────────────────────┐
                       │           JAVDB 原生页面 (DOM & Fetch)         │
                       └──────────────────────┬───────────────────────┘
                                              │ 拦截与注入
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ JavdbEmbySkin 核心运行时 (Monolithic Sandbox, Host Guard: javdb*.com / jdforrepam.com)       │
│                                                                                             │
│ 1. 路由拦截与限制突破层 (Lines 167 - 340 & Lines 8278 - 9006)                               │
│    • 规避 JAVDB 服务端 302 拦截 (/plans/ypay 强转高级搜索，免登录穿透)                     │
│    • ReviewListInfiniteManager: 突破限制，无刷新异步抓取并拼接长短评与相关清单              │
│    • InfiniteScrollManager: 瀑布流滚动探底无缝加载下一页                                    │
│                                                                                             │
│ 2. 详情页重构与多模式排版引擎 (Lines 2010 - 2362 & Lines 22282 - 22400)                     │
│    • 四大封面形态引擎：Crop(裁切) | Full(全宽) | Fusion(触顶收缩融合) | Hybrid(Letterboxd) │
│    • 实体备注小气泡 (.emby-note-tip): 女优/片商/系列悬浮卡片 (生平+现役退役+穿透筛选)       │
│    • 详情页操作行 (.emby-action-row) 与 播放按钮联动                                        │
│                                                                                             │
│ 3. 视觉风格渲染栈与特效 (Lines 679 - 1637 & Lines 7310 - 8196)                               │
│    • 风格切换：Emby Native / Glass 毛玻璃 / LiquidGlass 液态玻璃真折射层 (SVG Filter)        │
│    • Steam 3D Tilt: 卡牌跟随光标 3D 倾斜与反光层                                            │
│    • WaterfallEngine: 绝对定位横向流，布局抖动节流与共演折叠展开重排                        │
│                                                                                             │
│ 4. FAV 核心存储系统 (Lines 15412 - 18429)                                                   │
│    • Tier 1: 同步内存镜像 (favMoviesCache, scoreMemCache, noteMap, actorGenderMap)          │
│    • Tier 2: IndexedDB (javdb-emby-fav-db, v5) 实体表: movies, lists, gallery, meta, notes │
│    • 事务控制: dbPutMovies 严格按 50 条分批切片事务，兼顾吞吐与防卡死                      │
│    • 原型链防护: 导入反序列化全量采用 Object.create(null) 字典                             │
│                                                                                             │
│ 5. 交互界面与业务中心 (Lines 9052 - 15411 & Lines 18430 - 22107)                           │
│    • FAVUI: 收藏夹复合面板 (清单管理、演员/片商/标签/年份/评分多维筛选、批量打标)           │
│    • StatisticsCenter: 媒体库数据统计中心 (核心KPI卡片、评分阶梯占比、女优/片商Top榜)       │
│    • Top250ViewManager: 日榜/周榜/月榜/总榜 (基于 Sequence Token 请求代数令牌防串流)        │
│    • Quick Status & Preview Modal: 封面快捷四态评分标记与多源剧照预览                       │
│                                                                                             │
│ 6. 外部服务与云端网关 (Lines 11120 - 12800)                                                 │
│    • GfriendsAvatarService: 全量索引树 (TTL 7天) + 883条 aliases 桥接 + 原生 onerror 降级   │
│    • ActressService: JAV_info 现役/退役检测与维基百科异步回退                               │
│    • PlaySitesService: 123AV / njav / MissAV 等 10+ 在线播放源与模板替换                    │
│    • CloudSync: WebDAV / GitHub / Gitee 双向同步，强制凭据脱敏白名单过滤                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心“弯弯绕绕”与历史避坑铁律 (15 Critical Invariants & Gotchas)

这 15 条铁律是整个项目 25,179 行代码历经数十次对抗式审计与实战迭代沉淀出的硬核准则，**后续维护者与 AI 绝不可触犯**：

### 1. 绝不可混淆 `userScore` 与 `rating.score`
* **铁律**：`m.userScore` 是 1~5 整数（用户主观打星），`m.rating.score` 是 0.0~5.0 浮点数（网站大众分）。
* **陷阱**：在数据合并、备份导入或详情页抓取时，绝对不能用 JAVDB 官方的 `rating.score` 覆盖用户的 `userScore`；当用户未打分时，`m.userScore` 必须保持缺失或 `undefined`，**绝不可默认赋值为 0 或把官方评分充填进去**（0 会被星级组件渲染为 1 星差评并破坏评分筛选统计）。
* **参见决策**：[ADR-0005: 评分隔离机制](./adr/0005-strict-separation-of-userscore-and-official-rating.md)

### 2. 头像替换绝不可加“区分男女优”的前置过滤
* **铁律**：开启头像替换后，**只要 Gfriends 仓库能匹配到头像（无论是男优还是女优），统一替换为高清头像；未匹配或加载失败时，平滑降级官方头像**。
* **陷阱**：Gfriends 社区早已收录清水健、森林原人等知名男优真实肖像（保存在 `9-Javrave` 目录）。所谓“清水健套上女装”是历史误判。切勿加诸如 `if (isMale) return` 或跳过 `9-Javrave` 的条件，这不仅多余，而且会破坏 403 位演员的高清肖像并引入脆弱的性别判断代码。
* **参见决策**：[ADR-0004: 无性别偏见的头像匹配](./adr/0004-gender-agnostic-avatar-matching-and-fallback.md)

### 3. 被动采集 (`passiveCollectMovie`) 绝不可覆盖 `customMeta`
* **铁律**：用户在“元数据纠错”界面手动纠正的片名、演员、片商和封面具有最高裁量权。
* **陷阱**：当用户在 JAVDB 列表页滑动浏览时，被动采集器会快速爬取页面数据写入 DB。写入前必须提取 `chosenCustomMeta` 并予以保全，严禁被网页原生旧数据洗掉。
* **参见决策**：[ADR-0007: 被动采集熔断与元数据自愈](./adr/0007-passive-collection-and-custom-metadata-healing.md)

### 4. IndexedDB 批量写回必须以 50 条为一批进行切片事务
* **铁律**：全量导入或同步数千部影片时，必须调用 `dbPutMovies(list)`（按 50 条一批开启读写事务），严禁在循环内逐条 `await dbPutMovie(m)`。
* **陷阱**：浏览器的 IndexedDB 进程与 JS 主线程跨进程通信开销极高。上万次连续小事务会导致页面 UI 冻结数分钟甚至触发浏览器崩溃；而单一大事务若包含数万条，在移动端或低内存设备极易触发 `QuotaExceededError` 或超时回滚。50 条是兼顾响应度与吞吐的最佳平衡点。
* **参见决策**：[ADR-0002: 多级存储架构](./adr/0002-multi-tier-storage-indexeddb-memory-mirror.md)

### 5. DOM MutationObserver 内部严禁无条件写入 DOM
* **铁律**：任何由 MutationObserver 监听的节点，在对其子节点进行状态更新（如 `textContent`、`className`、`innerHTML`）前，**必须先判断当前值是否已是目标值**，或标记 `hasExternalChange` 守卫。
* **陷阱**：如行 8120 的 `reviewRefreshObserver`，在修改文本或图标前必须加比对守卫：`if (span && span.textContent !== nextIcon) span.textContent = nextIcon;`。否则观察器因自身修改再次触发，微任务队列死循环，瞬间吃满单核 CPU。
* **参见决策**：[ADR-0006: 递归守卫与可重入锁](./adr/0006-reentrant-dialog-and-mutationobserver-recursion-guards.md)

### 6. 原生 `confirm` 猴子补丁必须具备全局重入锁与 finally 恢复
* **铁律**：在需要连续向用户确认并自动执行批量操作（如全量覆盖导入）时，若临时覆写 `window.confirm`，必须先将原函数保存在 `window.__origConfirm`，且必须在 `try ... finally` 块中无条件还原。
* **陷阱**：油猴脚本运行在特定上下文中，如果异步任务抛出异常导致 `window.confirm` 未能还原，整个 JAVDB 页面的所有原生交互弹窗将永久失效。
* **参见决策**：[ADR-0006: 递归守卫与可重入锁](./adr/0006-reentrant-dialog-and-mutationobserver-recursion-guards.md)

### 7. LiquidGlass 绝不可使用 HTML5 Canvas 做动态色彩提取
* **铁律**：液态玻璃的核心流体效果必须使用 SVG 矢量位移（`<feDisplacementMap>` + `feImage`）；代表色提取若遇跨域图片，必须有防御性 try-catch 并回退 CSS 默认主题色变量。
* **陷阱**：第三方图片在未经 CORS 授权的情况下绘制到 Canvas 会立即“污染（Taint）”画布。一旦调用 `getImageData` 会直接抛出致命的 `SecurityError` 导致详情页崩溃白屏。
* **参见决策**：[ADR-0003: LiquidGlass 滤镜流水线](./adr/0003-liquidglass-svg-displacement-pipeline.md)

### 8. 演员共演堆叠徽章展开/收起后必须主动通知 WaterfallEngine
* **铁律**：卡片上的多女优折叠徽标在被点击展开（显示剩余女优头像）或点击收起时，必须调用 `WaterfallEngine.refresh()`。
* **陷阱**：由于 Emby 模式卡片是绝对定位计算高度的，徽章展开会增加卡片实际高度；如果不主动触发重排，该卡片会直接与其正下方卡片发生难看的物理重叠（Layout Overlap）。
* **参见决策**：[ADR-0008: SPA 路由与渲染生命周期](./adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md)

### 9. 备份导入反序列化必须使用 `Object.create(null)` 抵御原型链污染
* **铁律**：解析用户上传的 JSON 文件并建立临时索引字典时，必须使用 `const dict = Object.create(null)`，严禁使用普通字面量 `{}`。
* **陷阱**：恶意构造的备份文件可能包含形如 `"__proto__": { "polluted": true }` 的属性，导致全局 Object 原型污染，进而引发任意条件绕过或 XSS 注入风险。
* **参见决策**：[ADR-0009: 敏感凭据脱敏与防污染](./adr/0009-two-tier-sync-and-credential-desensitization.md)

### 10. 实体候选提取必须区分演员与片商（空格与中点保护）
* **铁律**：从详情页标题提取实体名时，演员名可以用逗号/空格拆分为多候选人；但片商名必须使用 `extractEntityNamesWhole` 保持整行完整性。
* **陷阱**：片商名（如 `S1 NO.1 STYLE`）内部含有合法空格，日文片商（如 `ケイ・エム・プロデュース`）内部含有中点间隔号（`・`）。若盲目按空格或符号切碎，片商名会被切成碎片导致收藏夹筛选完全匹配落空。

### 11. 异步榜单切换必须使用请求代数令牌（Sequence Token）
* **铁律**：在 TOP250 管理器（行 9052）中，所有异步跨域拉取榜单前必须单调递增 `top250LoadSeq`，并在回包到达后校验 `if (mySeq !== top250LoadSeq) return;`。
* **陷阱**：用户在总榜、日榜、周榜、月榜之间快速切换时，慢速的日榜回包可能比快速的月榜回包更晚到达。如果不校验代数令牌，界面会被迟到的旧回包覆盖，产生严重竞态乱序 BUG。
* **参见决策**：[ADR-0011: 异步榜单代数令牌防乱序](./adr/0011-request-sequence-tokens-for-rankings.md)

### 12. 详情页封面融合模式（Fusion）阈值防毒与绝对顶部守卫
* **铁律**：融合模式下收缩折叠计算由 `requestAnimationFrame` 驱动；在 `fusionOnScroll` 入口必须设立 `y <= 30` 绝对顶部守卫（无条件强制移除折叠类名与 `--fusion-shift`），从数学物理上 100% 杜绝因滞后死区或测量污染导致的“顶部 700px 空白卡死、大图丢失”；在 `measureFusionMetrics()` 中严禁在折叠态（`.cover-fusion-collapsed`）下测量几何尺寸（折叠态海报与信息并排会导致 `infoTop === heroTop` 严重毒化缓存）；展开态测算必须限制物理安全保底 `titleTop >= 450px` 确保滞后死区 `titleTop - 80 >= 370px` 恒正；图片 `load` 与窗口 `resize` 仅在非折叠态下才允许更新缓存，彻底杜绝液态玻璃（Liquid Glass）强制回流与 MutationObserver 引发的尺寸震荡（Chattering）。
* **陷阱**：单纯依赖 `y < titleTop - 80` 恢复大图存在致命设计隐患：一旦因折叠态测量导致 `titleTop <= 80px`，滞后死区变成负数，用户滑到最顶部 `y = 0` 时条件永远不成立，英雄区带着数百像素的 `--fusion-shift` 上边距卡死在页面中下部，顶部留出巨大黑色空白；此外在液态玻璃下频繁测量会触发 Layout Thrashing 导致海报残缺。
* **参见决策**：[ADR-0010: 详情页封面融合模式（Cover Fusion）与四态排版引擎架构](./adr/0010-cover-fusion-and-multi-layout-engine.md)

### 13. 规避 VIP 302 拦截墙时避免循环重定向
* **铁律**：针对非会员或未登录用户触发 302 重定向到 `/plans/ypay` 时，脚本拦截后使用 `location.replace('/advanced_search?...')` 穿透替换。必须严格判断目标页面的有效性与来源。
* **陷阱**：严禁出现自身重定向至自身引发浏览器死循环报错（`ERR_TOO_MANY_REDIRECTS`）；替换前必须前置判断当前是否已经在高级搜索页面或已开启绕过限制。
* **参见决策**：[ADR-0012: JAVDB 服务端 VIP 302 拦截墙规避与多源在线播放矩阵](./adr/0012-vip-wall-bypass-and-multi-source-playback.md)

### 14. 关注女优同步必须带断崖式下跌熔断（30% 保护）
* **铁律**：在 `FavoriteActressManager.sync` 中，若远端抓取到的关注列表少于本地的 30%（`cur.size >= 10 && onlineSet.size < 0.3 * cur.size`），必须强制触发熔断并降级为合并模式。
* **陷阱**：当 JAVDB 临时改版或用户网络丢包导致仅返回部分数据时，无脑覆写会将用户积累数年的本地收藏演员全部清空。
* **参见决策**：[ADR-0007: 被动采集熔断与元数据自愈](./adr/0007-passive-collection-and-custom-metadata-healing.md)

### 15. 皮肤开闭与 SPA 路由跳转必须严格清理常驻资源
* **铁律**：当用户在设置中关闭 Emby 皮肤时，不仅要移除 `#emby-style` 样式表，还必须同步解绑路由拦截器、清理定时守护器（`clearInterval`）、隐藏设置浮层遮罩（`#jhs-settings-mask`），恢复原生 Navbar。
* **陷阱**：残留的定时器或未清理的全局样式会在用户切回原生 JAVDB 界面后继续运行，导致内存泄漏及原生排版严重错乱。
* **参见决策**：[ADR-0008: SPA 路由与渲染生命周期](./adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md)

### 16. 关闭皮肤时必须清除容器 Masonry 类名并对所有布局引擎实施启闭熔断
* **铁律**：`removeEmbyDOM()` 必须彻底移除 `.movie-list` 上的 `emby-masonry` 与 `emby-dynamic-focus` 类名；所有异步排版引擎（`WaterfallEngine.layout`、`WaterfallEngine.schedule`、`ensureLayoutClasses`、`restructureGrid`）入口必须设置 `if (!enabled) return;` 熔断守卫。
* **陷阱**：JAVDB 原生主页采用现代 CSS Grid 布局。若遗留 Masonry 类名或因图片 `load` 事件唤醒瀑布流重新写入 `translate3d`，绝对定位与 CSS Grid 坐标系会相互叠加放大数十像素，导致原生排版严重错位出界。
* **工程警示**：坚决抵御“用大重构掩盖排查不足”的过度设计冲动（如推翻重写已有稳定成熟的 `removeEmbyDOM`）；必须坚持第一性原理直击 CSS Grid 叠加本质，优先实施 6 行代码以内的最小侵入手术级微调。
* **参见决策**：[ADR-0008: SPA 路由与渲染生命周期](./adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md) 及 [ADR-0013: 皮肤关闭态原生网格（CSS Grid）恢复与布局引擎启闭熔断架构](./adr/0013-native-grid-restoration-and-layout-engine-circuit-breaker.md)

### 17. 抽屉账户登出必须遵循 Rails UJS RESTful DELETE 规范与严格 Cookie 判定
* **铁律**：Emby 抽屉提取原生 navbar 菜单项时必须完整保全 `data-method`、`data-confirm` 与 `rel` 属性；登出操作必须通过专属安全管道（优先触发隐藏的原生 Rails UJS 登出节点，兜底构造 `_method=delete` + CSRF Token 的 POST 表单），并清空本地临时凭据缓存；`isJavdbUserLoggedIn()` 的 Cookie 匹配必须使用严格正则排除 `_rucaptcha_session_id`。
* **陷阱**：JAVDB 服务端严格拒绝 `GET /logout`。若以 GET 发送请求，服务端不仅不会销毁 Session，反而会设置 `redirect_to=%2Flogout` 并 302 重定向至登录页，使用户误以为成功退出；后续刷新页面或开关皮肤时服务端的有效 Session 仍然存在，导致账号“幽灵复活”；此外，粗放正则匹配 `/_session/i` 会误命中图形验证码 Cookie，导致游客状态被误判为登录态。
* **参见决策**：[ADR-0014: Rails UJS DELETE 会话生命周期与 Emby 抽屉安全登出管道架构](./adr/0014-rails-ujs-delete-session-lifecycle-and-safe-logout.md)

### 18. 统计中心观影标记上下列排列与全生命周期数据一致性
* **铁律**：数据统计中心（StatisticsCenter）核心 KPI 矩阵中展示的观影标记（已看 / 想看作品数）必须严格呈上下列垂直排列（上列宝石绿已看、下列琥珀金想看）；必须保持为纯粹稳健的统计卡片，不绑定多余的跨模块跳转交互；其上游统计数据源以本地 IndexedDB 全量电影表（`allDbMovies` 优先）的 `reviewStatus` 为真相源，在备份导出时随 `movies` 完整导出，备份导入还原时自动由 `collectStats()` 重新汇算，确保全生命周期数据绝对闭环。
* **陷阱**：在纯统计 KPI 指标卡片中随意绑定未经充分设计的路由筛选跳转不仅违反极简与单一职责原则，还可能因收藏夹当前分页或筛选状态冲突引发意外行为；若在上游数据采集时漏算未加入自定义清单但已被标记的作品，会导致统计数量与用户实际标记严重不符。

### 19. 鉴权路由避让与元素级 Referrer 隔离机制
* **铁律**：在 `/login`、`/user_sessions`、`/users/sign_in`、`/users/new`、`/users/password` 等关键鉴权路由上，脚本必须在入口处立即 return 避让，坚决不注入任何皮肤 DOM、样式或 MutationObserver，100% 保全原生 Rails Form POST 导航生命周期；严禁向 `<head>` 注入全局 `<meta name="referrer" content="no-referrer">`，避免破坏 Rails 同源 CSRF 校验（`verify_same_origin_request`）导致服务端 Session 重置与登录重定向死循环；防盗链需求一律由元素级 `referrerPolicy = 'no-referrer'` 承载，且显式豁免图形验证码（`.rucaptcha-image`）。
* **陷阱**：忽视 Rails 体系对同源 `Referer` / `Origin` 标头的严格校验，采用全局 Meta 粗暴阻断 Referer，导致合法的整页表单 POST 请求被服务端判定为 CSRF 攻击并被动清空 Session 验证码；无差别重写页面全部 `<img>` 标签导致图形验证码丢失与 Cookie 解绑。
* **参见决策**：[ADR-0015: Rails CSRF 同源 Referer 完整性保障与元素级防盗链穿透架构](./adr/0015-rails-csrf-referer-integrity-and-image-level-anti-hotlink.md)

---

## 架构决策档案索引 (Architectural Decision Records)

完整技术演化背景、备选方案与决策权衡已记录于 `docs/adr/`：

1. [ADR-0001: 单文件 Monolithic UserScript 架构](./adr/0001-monolithic-userscript-architecture.md)
2. [ADR-0002: IndexedDB + 内存镜像双层存储架构](./adr/0002-multi-tier-storage-indexeddb-memory-mirror.md)
3. [ADR-0003: LiquidGlass 液态玻璃 SVG Displacement 滤镜流水线](./adr/0003-liquidglass-svg-displacement-pipeline.md)
4. [ADR-0004: 无性别偏见的演员高清头像解析与平滑降级](./adr/0004-gender-agnostic-avatar-matching-and-fallback.md)
5. [ADR-0005: 用户主观评分 (`userScore`) 与官方评分 (`rating.score`) 的物理隔离体系](./adr/0005-strict-separation-of-userscore-and-official-rating.md)
6. [ADR-0006: DOM MutationObserver 递归守卫与 window.confirm 猴子补丁可重入锁](./adr/0006-reentrant-dialog-and-mutationobserver-recursion-guards.md)
7. [ADR-0007: 被动采集熔断器与自定义纠错元数据 (`customMeta`) 永久保全](./adr/0007-passive-collection-and-custom-metadata-healing.md)
8. [ADR-0008: JAVDB SPA 局部路由生命周期接管与原生浮层/设置遮罩同步](./adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md)
9. [ADR-0009: 敏感凭据双重脱敏导出与 `Object.create(null)` 原型链污染防御](./adr/0009-two-tier-sync-and-credential-desensitization.md)
10. [ADR-0010: 详情页封面融合模式（Cover Fusion）与四态排版引擎架构](./adr/0010-cover-fusion-and-multi-layout-engine.md)
11. [ADR-0011: 异步榜单视图基于请求代数令牌（Sequence Token）的防乱序架构](./adr/0011-request-sequence-tokens-for-rankings.md)
12. [ADR-0012: JAVDB 服务端 VIP 302 拦截墙规避与多源在线播放矩阵](./adr/0012-vip-wall-bypass-and-multi-source-playback.md)
13. [ADR-0013: 皮肤关闭态原生网格（CSS Grid）恢复与布局引擎启闭熔断架构](./adr/0013-native-grid-restoration-and-layout-engine-circuit-breaker.md)
14. [ADR-0014: Rails UJS DELETE 会话生命周期与 Emby 抽屉安全登出管道架构](./adr/0014-rails-ujs-delete-session-lifecycle-and-safe-logout.md)
15. [ADR-0015: Rails CSRF 同源 Referer 完整性保障与元素级防盗链穿透架构](./adr/0015-rails-csrf-referer-integrity-and-image-level-anti-hotlink.md)

