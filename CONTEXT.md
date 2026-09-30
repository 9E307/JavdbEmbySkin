# JavdbEmbySkin 完整架构上下文与领域模型 (CONTEXT.md)

JavdbEmbySkin 是一个在浏览器油猴环境（Tampermonkey / Violentmonkey）中运行的大型单文件 UserScript（26,046 行，v7.336）。它将 JAVDB 原生站点全面重构为现代化的 Emby 视觉风格，并内置了从数据采集、多级持久化、多维流式画廊、元数据纠错自愈、媒体库统计分析、多源播放矩阵，到云端多端同步与高清头像映射在内的完整多媒体数据管理系统。

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

### 7. 路由、安全与稳定性守卫 (Routing, Security & Resiliency Guards)

**Searchbox Autofill Immunity Guard (搜索框凭据嗅探深度免疫守卫)**:
由 `type="search"`、`role="searchbox"`、11 项反嗅探忽略属性、`sanitizeSearchInputs()` 全局生命周期动态清洗与聚焦主动冲刷（Focus Flush）构筑的四层立体纵深防御体系。彻底根治 Chromium `PasswordAutofillAgent` 误填已保存账号密码以及用户点击时弹出“一键填写账号密码/管理密码”系统悬浮下拉框的问题，同时在 CSS 层消除 WebKit 默认清除小图标。
_Avoid_: DisableAutofill, ClearInput, ResetSearch

**Direct Child Anchor & Safe Insertion (直接子节点锚点与安全挂载契约)**:
在向原生父容器调用 `parent.insertBefore(newChild, refChild)` 插入导航条与面板时，必须经由 `resolveDirectChildAnchor` 递归向上回溯至父容器的直接子节点，杜绝因 JAVDB 结构深层嵌套（如 `.main-tabs-wrap` 包裹 `.tabs`）触发 W3C 标准的致命异常 `DOMException: The child can not be found in the parent`。配合 `safeInsertBefore` 多级回退机制保证页面永不白屏崩溃。
_Avoid_: DirectInsert, QuerySelectorInsert, ForceAppend

**Multilingual Metadata Resilient Parser (多语言元数据健壮性解析器)**:
在番号详情页，`detailBlockVal` 与 `extractMetaLinks` 采用大小写无关的全语言正则（覆盖简中、繁中、英文、日文变体），演员列表区域配合 `/actors/` 路由特征选择器进行双重探测提取，彻底杜绝非繁体中文环境下关键元数据（演员、导演、片商、评分、时长、系列、标签）呈现空白或未知的问题。
_Avoid_: TraditionalChineseOnly, HardcodedLabel, StaticSelector

**Age Gate Cookie Pre-Exemption (年龄弹窗 Cookie 预置豁免)**:
在脚本启动入口提前写入官方 `over18=1` Cookie 实现服务端免检，并仅针对 `.over18-modal` 或显式带有 `a[href*="/over18"]` 的模态层直接从 DOM 树移除解绑；绝对禁止跨全 DOM 扫描 `a, button` 模拟盲点击，杜绝误点普通影片卡片（如番号含 18 的热播作品）引发恶性弹窗死循环。
_Avoid_: AutoClick18, ModalClicker, RegexClick

---

## 架构子系统全景地图 (Architecture Map Across 25,539 Lines)

```
                       ┌──────────────────────────────────────────────┐
                       │           JAVDB 原生页面 (DOM & Fetch)         │
                       └──────────────────────┬───────────────────────┘
                                              │ 拦截与注入
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ JavdbEmbySkin 核心运行时 (Monolithic Sandbox, Host Guard: javdb*.com / jdforrepam.com)       │
│                                                                                             │
│ 1. 路由拦截、安全守卫与限制突破层 (Lines 32 - 198 & Lines 8600 - 9196)                      │
│    • 严格域名守卫与鉴权路由避让 (/login, /user_sessions 等 100% 保持原生 Rails POST)        │
│    • 元素级 Referrer 免溯源策略 (针对普通图片 no-referrer，排除验证码防 403 阻断)           │
│    • 官方 age-gate 年龄确认预写 over18=1 Cookie 免检，严格禁止全 DOM 盲点击                 │
│    • 规避 JAVDB 服务端 302 拦截 (/plans/ypay 强转高级搜索，免登录穿透)                     │
│    • ReviewListInfiniteManager: 突破限制，无刷新异步抓取并拼接长短评与相关清单              │
│    • InfiniteScrollManager: 瀑布流滚动探底无缝加载下一页                                    │
│                                                                                             │
│ 2. 详情页重构与多模式排版引擎 (Lines 7322 - 7410, Lines 7821 - 8050 & Lines 22800 - 24750)  │
│    • 四大封面形态引擎：Crop(裁切) | Full(全宽) | Fusion(触顶收缩融合) | Hybrid(Letterboxd) │
│    • 详情页 Hero 区多语言健壮性解析 (简/繁/英/日正则自适应提取 + /actors/ 路由特征探测)     │
│    • 实体备注小气泡 (.emby-note-tip): 女优/片商/系列悬浮卡片 (生平+现役退役+穿透筛选)       │
│    • 详情页操作行 (.emby-action-row) 与 10+ 多源在线播放按钮矩阵联动                        │
│                                                                                             │
│ 3. 视觉风格渲染栈与特效 (Lines 738 - 6421, Lines 7140 - 7790 & Lines 15613 - 15663)        │
│    • 风格切换：Emby Native / Glass 毛玻璃 / LiquidGlass 液态玻璃真折射层 (SVG Filter)        │
│    • Steam 3D Tilt: 卡牌跟随光标 3D 倾斜与反光层 (attachCard3D)                             │
│    • WaterfallEngine: 绝对定位横向流，布局抖动节流与共演折叠展开重排                        │
│                                                                                             │
│ 4. FAV 核心存储系统 (Lines 15664 - 18676)                                                   │
│    • Tier 1: 同步内存镜像 (favMoviesCache, scoreMemCache, noteMap, actorGenderMap)          │
│    • Tier 2: IndexedDB (javdb-emby-fav-db, v5) 实体表: movies, lists, gallery, meta, notes │
│    • 事务控制: dbPutMovies 严格按 50 条分批切片事务，兼顾吞吐与防卡死                      │
│    • 原型链防护: 导入反序列化全量采用 Object.create(null) 字典                             │
│                                                                                             │
│ 5. 交互界面与业务中心 (Lines 9197 - 11349, Lines 13420 - 15200 & Lines 18677 - 22800)      │
│    • FAVUI: 收藏夹/画廊复合面板 (清单管理、多维筛选器 openFilterModal、批量打标)            │
│    • StatisticsCenter: 媒体库数据统计中心 (核心KPI卡片、评分阶梯占比、排行榜)               │
│    • Top250ViewManager: 日榜/周榜/月榜/总榜 (基于 Sequence Token 请求代数令牌防乱序覆盖)    │
│    • sanitizeSearchInputs: 搜索框全域防嗅探与四层纵深防御 (type=search, 11项属性, focus冲刷)│
│    • buildHomeTabs & restructureGrid: 主页常驻4导航 Tab 与跨层级直接子节点安全挂载沙盒      │
│                                                                                             │
│ 6. 外部服务与云端网关 (Lines 11350 - 13419 & Lines 14640 - 15060 & Lines 21390 - 21630)    │
│    • GfriendsAvatarService: 全量索引树 (TTL 7天) + 883条 aliases 桥接 + 官方头像降级        │
│    • ActressService: JAV_info 现役/退役检测与维基百科异步回退                               │
│    • PlaySitesService: 123AV / njav / MissAV 等 10+ 在线播放源与模板替换                    │
│    • PreviewSys: JavFree / JavStore / BlogJav / ProjectJav 嗅探 + L1/L2 7天本地持久化缓存   │
│    • CloudSync: WebDAV / GitHub / Gitee 双向同步，强制凭据脱敏白名单过滤                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛑 项目最高工程哲学与排查零号准则 (Axiom 0: First Principles & Surgical Fix)

任何后续接手本项目的 AI 智能体或开发者，在开始阅读具体业务代码与排查任何 Bug 前，**必须将以下三大第一性原理作为最高行为公理（Axiom 0）深植于推理上下文**：

1. **坚决抵御“用大重构掩盖排查不足”的浮躁冲动**：
   * 严禁在未定位到真实物理根因前，轻易向用户提议“推翻重写已有成熟模块”（如重写 `removeEmbyDOM`、重写网格重构或重写事件委托）；
   * 本项目是一个拥有 25,000+ 行紧密耦合单文件、历经数十轮实战对抗审计的生产级应用，盲目推翻成熟函数往往会在修好 1 个表象问题的同时，瞬间踩中此前填平的 5 个深层隐性地雷。
2. **第一性原理深度穿透本质（First Principles Root-Cause Analysis）**：
   * 排查任何异常必须追本溯源到最底层的物理本质：到底是不是 Chromium 合成层（Compositing Layer）GPU 缓存未刷？是不是 DOM 脱水期未连接 DOM 树时的 `isConnected === false` 时序竞争？是不是 CSS Specificity 权重被原生类名覆盖？是不是 W3C 标准下的直接子节点（Direct Child）断言失败？是不是 IndexedDB 异步事务未就绪？
   * 必须拿出具有底层依据的确凿结论，严禁靠“我猜可能在这里”、“试着包裹一个 setTimeout”来盲目试错。
3. **极简手术级微调原则（Surgical Precision）**：
   * **能用 3~6 行最小侵入代码在源头精准修复的，绝对不允许擅自动动 50 行以上的周边逻辑**；
   * 修复方案必须像显微外科手术一样，以最小的扰动解决根本问题，对既有庞大生态保持最大的敬畏与零侵入性。

---

## 📋 AI 维护者交付前自检清单 (AI Pre-flight Checklist)

后续任何 AI 智能体在向用户汇报或提交代码前，必须在内部反思链（Chain of Thought）中逐项核对并确保符合以下四步交付准则：

1. **[语法与静态安全]**：是否已执行 `node -c .\JavdbEmbySkin.user.js` 并确保 100% 零语法错误与未闭合语法结构？
2. **[31条铁律红线核查]**：本次改动是否侵犯了后文 31 条业务铁律中的任何一条？（尤其关注：`userScore` 是否被客观评分污染？密码管理器是否误触下拉框？DOM 挂载是否保持直接子节点？图片路径是否仍为无域名相对路径？）。
3. **[侵入度与重构判定]**：本次修改是否严格做到了“最小必要侵入”？是否存在用推翻重构掩盖排查不足的嫌疑？
4. **[文档与版本原子化同步]**：若涉及版本递增，是否已将脚本头部 `@version`、脚本内部 `VERSION` 常量、`CONTEXT.md` 顶栏行数与版本号、以及 [`CHANGELOG.md`](./CHANGELOG.md) 严格按 Keep a Changelog 格式同步更新？

---

## 核心“弯弯绕绕”与历史避坑铁律 (31 Critical Invariants & Gotchas)

这 31 条铁律是整个项目 25,699 行代码历经数十次对抗式审计与实战迭代沉淀出的硬核准则，**后续维护者与 AI 绝不可触犯**：

### 1. 绝不可混淆 `userScore` 与 `rating.score`
* **铁律**：`m.userScore` 是 1~5 整数（用户主观打星），`m.rating.score` 是 0.0~5.0 浮点数（网站大众分）。
* **陷阱**：在数据合并、备份导入或详情页抓取时，绝对不能用 JAVDB 官方的 `rating.score` 覆盖用户的 `userScore`；当用户未打分时，`m.userScore` 必须保持缺失或 `undefined`，**绝不可默认赋值为 0 或把官方评分充填进去**（0 会被星级组件渲染为 1 星差评并破坏评分筛选统计）。
* **参见决策**：[ADR-0005: 评分隔离机制](./docs/adr/0005-strict-separation-of-userscore-and-official-rating.md)

### 2. 头像替换绝不可加“区分男女优”的前置过滤
* **铁律**：开启头像替换后，**只要 Gfriends 仓库能匹配到头像（无论是男优还是女优），统一替换为高清头像；未匹配或加载失败时，平滑降级官方头像**。
* **陷阱**：Gfriends 社区早已收录清水健、森林原人等知名男优真实肖像（保存在 `9-Javrave` 目录）。所谓“清水健套上女装”是历史误判。切勿加诸如 `if (isMale) return` 或跳过 `9-Javrave` 的条件，这不仅多余，而且会破坏 403 位演员的高清肖像并引入脆弱的性别判断代码。
* **参见决策**：[ADR-0004: 无性别偏见的头像匹配](./docs/adr/0004-gender-agnostic-avatar-matching-and-fallback.md)

### 3. 被动采集 (`passiveCollectMovie`) 绝不可覆盖 `customMeta`
* **铁律**：用户在“元数据纠错”界面手动纠正的片名、演员、片商和封面具有最高裁量权。
* **陷阱**：当用户在 JAVDB 列表页滑动浏览时，被动采集器会快速爬取页面数据写入 DB。写入前必须提取 `chosenCustomMeta` 并予以保全，严禁被网页原生旧数据洗掉。
* **参见决策**：[ADR-0007: 被动采集熔断与元数据自愈](./docs/adr/0007-passive-collection-and-custom-metadata-healing.md)

### 4. IndexedDB 批量写回必须以 50 条为一批进行切片事务
* **铁律**：全量导入或同步数千部影片时，必须调用 `dbPutMovies(list)`（按 50 条一批开启读写事务），严禁在循环内逐条 `await dbPutMovie(m)`。
* **陷阱**：浏览器的 IndexedDB 进程与 JS 主线程跨进程通信开销极高。上万次连续小事务会导致页面 UI 冻结数分钟甚至触发浏览器崩溃；而单一大事务若包含数万条，在移动端或低内存设备极易触发 `QuotaExceededError` 或超时回滚。50 条是兼顾响应度与吞吐的最佳平衡点。
* **参见决策**：[ADR-0002: 多级存储架构](./docs/adr/0002-multi-tier-storage-indexeddb-memory-mirror.md)

### 5. DOM MutationObserver 内部严禁无条件写入 DOM
* **铁律**：任何由 MutationObserver 监听的节点，在对其子节点进行状态更新（如 `textContent`、`className`、`innerHTML`）前，**必须先判断当前值是否已是目标值**，或标记 `hasExternalChange` 守卫。
* **陷阱**：如行 8120 的 `reviewRefreshObserver`，在修改文本或图标前必须加比对守卫：`if (span && span.textContent !== nextIcon) span.textContent = nextIcon;`。否则观察器因自身修改再次触发，微任务队列死循环，瞬间吃满单核 CPU。
* **参见决策**：[ADR-0006: 递归守卫与可重入锁](./docs/adr/0006-reentrant-dialog-and-mutationobserver-recursion-guards.md)

### 6. 原生 `confirm` 猴子补丁必须具备全局重入锁与 finally 恢复
* **铁律**：在需要连续向用户确认并自动执行批量操作（如全量覆盖导入）时，若临时覆写 `window.confirm`，必须先将原函数保存在 `window.__origConfirm`，且必须在 `try ... finally` 块中无条件还原。
* **陷阱**：油猴脚本运行在特定上下文中，如果异步任务抛出异常导致 `window.confirm` 未能还原，整个 JAVDB 页面的所有原生交互弹窗将永久失效。
* **参见决策**：[ADR-0006: 递归守卫与可重入锁](./docs/adr/0006-reentrant-dialog-and-mutationobserver-recursion-guards.md)

### 7. LiquidGlass 2.0 物理透镜折射流水线与 Chromium 合成层四大禁忌
* **铁律**：
  1. **绝不可在 JS 主线程用 Canvas 逐像素运算渲染动态背景**：核心折射流体必须走纯离线烘焙位移图（Data URI）+ SVG `<feDisplacementMap>` 由 GPU 片元着色器硬件级执行。严禁对跨域海报调用 `ctx.getImageData()`（避免 CORS Tainted Canvas 崩溃抛错致死）；
  2. **绝不可给液态玻璃容器添加 `contain: strict` 或 `contain: paint`**：Containment 会截断宿主对外部视口内容的采样通道，使 `backdrop-filter` 无法读取下层 DOM，透镜直接退化为黑底或纯灰死色。顶栏等必须严格保持 `contain: none !important; will-change: backdrop-filter; transform: translateZ(0);`；
  3. **必须以 Micro-Saturate Jitter 机制破除 Chromium GPU 缓存冻结**：Chromium 对 `backdrop-filter: url(#id)` 存在激进的位图缓存机制，滚动时不触发重绘导致透镜画面冻结。必须在 `window.onScrollInvalidate` 流水线中以 50ms 节流频率向 `saturate()` 注入万分之五的微小浮点抖动，迫使合成器实时刷新采样，并在滚动停止 120ms 后优雅复位；
  4. **必须遵循透镜分层模糊阶梯与 visionOS 双层防眩文字阴影**：顶栏/药丸按钮采用 `blur(0.25px)` 亚像素微平滑抗锯齿 + `brightness(1.04)` + `saturate(1.08)`（绝不可加重模糊，中心内容必须 100% 锐利保真）；抽屉/快捷弹窗采用 `blur(1.6px)`；透镜文字必须配备 `text-shadow: 0 1px 2px rgba(0,0,0,.95), 0 0 8px rgba(0,0,0,.7);` 保证强光背景下的无障碍可读性。
* **参见决策**：[ADR-0003: LiquidGlass 液态玻璃 2.0 物理透镜折射流水线与 Chromium 合成层防御架构](./docs/adr/0003-liquidglass-svg-displacement-pipeline.md)

### 8. 演员共演堆叠徽章展开/收起后必须主动通知 WaterfallEngine
* **铁律**：卡片上的多女优折叠徽标在被点击展开（显示剩余女优头像）或点击收起时，必须调用 `WaterfallEngine.refresh()`。
* **陷阱**：由于 Emby 模式卡片是绝对定位计算高度的，徽章展开会增加卡片实际高度；如果不主动触发重排，该卡片会直接与其正下方卡片发生难看的物理重叠（Layout Overlap）。
* **参见决策**：[ADR-0008: SPA 路由与渲染生命周期](./docs/adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md)

### 9. 备份导入反序列化必须使用 `Object.create(null)` 抵御原型链污染
* **铁律**：解析用户上传的 JSON 文件并建立临时索引字典时，必须使用 `const dict = Object.create(null)`，严禁使用普通字面量 `{}`。
* **陷阱**：恶意构造的备份文件可能包含形如 `"__proto__": { "polluted": true }` 的属性，导致全局 Object 原型污染，进而引发任意条件绕过或 XSS 注入风险。
* **参见决策**：[ADR-0009: 敏感凭据脱敏与防污染](./docs/adr/0009-two-tier-sync-and-credential-desensitization.md)

### 10. 实体候选提取必须区分演员与片商（空格与中点保护）
* **铁律**：从详情页标题提取实体名时，演员名可以用逗号/空格拆分为多候选人；但片商名必须使用 `extractEntityNamesWhole` 保持整行完整性。
* **陷阱**：片商名（如 `S1 NO.1 STYLE`）内部含有合法空格，日文片商（如 `ケイ・エム・プロデュース`）内部含有中点间隔号（`・`）。若盲目按空格或符号切碎，片商名会被切成碎片导致收藏夹筛选完全匹配落空。

### 11. 异步榜单切换必须使用请求代数令牌（Sequence Token）
* **铁律**：在 TOP250 管理器（行 9052）中，所有异步跨域拉取榜单前必须单调递增 `top250LoadSeq`，并在回包到达后校验 `if (mySeq !== top250LoadSeq) return;`。
* **陷阱**：用户在总榜、日榜、周榜、月榜之间快速切换时，慢速的日榜回包可能比快速的月榜回包更晚到达。如果不校验代数令牌，界面会被迟到的旧回包覆盖，产生严重竞态乱序 BUG。
* **参见决策**：[ADR-0011: 异步榜单代数令牌防乱序](./docs/adr/0011-request-sequence-tokens-for-rankings.md)

### 12. 详情页封面融合模式（Fusion）阈值防毒与绝对顶部守卫
* **铁律**：融合模式下收缩折叠计算由 `requestAnimationFrame` 驱动；在 `fusionOnScroll` 入口必须设立 `y <= 30` 绝对顶部守卫（无条件强制移除折叠类名与 `--fusion-shift`），从数学物理上 100% 杜绝因滞后死区或测量污染导致的“顶部 700px 空白卡死、大图丢失”；在 `measureFusionMetrics()` 中严禁在折叠态（`.cover-fusion-collapsed`）下测量几何尺寸（折叠态海报与信息并排会导致 `infoTop === heroTop` 严重毒化缓存）；展开态测算必须限制物理安全保底 `titleTop >= 450px` 确保滞后死区 `titleTop - 80 >= 370px` 恒正；图片 `load` 与窗口 `resize` 仅在非折叠态下才允许更新缓存，彻底杜绝液态玻璃（Liquid Glass）强制回流与 MutationObserver 引发的尺寸震荡（Chattering）。
* **陷阱**：单纯依赖 `y < titleTop - 80` 恢复大图存在致命设计隐患：一旦因折叠态测量导致 `titleTop <= 80px`，滞后死区变成负数，用户滑到最顶部 `y = 0` 时条件永远不成立，英雄区带着数百像素的 `--fusion-shift` 上边距卡死在页面中下部，顶部留出巨大黑色空白；此外在液态玻璃下频繁测量会触发 Layout Thrashing 导致海报残缺。
* **参见决策**：[ADR-0010: 详情页封面融合模式（Cover Fusion）与四态排版引擎架构](./docs/adr/0010-cover-fusion-and-multi-layout-engine.md)

### 13. 规避 VIP 302 拦截墙时避免循环重定向
* **铁律**：针对非会员或未登录用户触发 302 重定向到 `/plans/ypay` 时，脚本拦截后使用 `location.replace('/advanced_search?...')` 穿透替换。必须严格判断目标页面的有效性与来源。
* **陷阱**：严禁出现自身重定向至自身引发浏览器死循环报错（`ERR_TOO_MANY_REDIRECTS`）；替换前必须前置判断当前是否已经在高级搜索页面或已开启绕过限制。
* **参见决策**：[ADR-0012: JAVDB 服务端 VIP 302 拦截墙规避与多源在线播放矩阵](./docs/adr/0012-vip-wall-bypass-and-multi-source-playback.md)

### 14. 关注女优同步必须带断崖式下跌熔断（30% 保护）
* **铁律**：在 `FavoriteActressManager.sync` 中，若远端抓取到的关注列表少于本地的 30%（`cur.size >= 10 && onlineSet.size < 0.3 * cur.size`），必须强制触发熔断并降级为合并模式。
* **陷阱**：当 JAVDB 临时改版或用户网络丢包导致仅返回部分数据时，无脑覆写会将用户积累数年的本地收藏演员全部清空。
* **参见决策**：[ADR-0007: 被动采集熔断与元数据自愈](./docs/adr/0007-passive-collection-and-custom-metadata-healing.md)

### 15. 皮肤开闭与 SPA 路由跳转必须严格清理常驻资源
* **铁律**：当用户在设置中关闭 Emby 皮肤时，不仅要移除 `#emby-style` 样式表，还必须同步解绑路由拦截器、清理定时守护器（`clearInterval`）、隐藏设置浮层遮罩（`#jhs-settings-mask`），恢复原生 Navbar。
* **陷阱**：残留的定时器或未清理的全局样式会在用户切回原生 JAVDB 界面后继续运行，导致内存泄漏及原生排版严重错乱。
* **参见决策**：[ADR-0008: SPA 路由与渲染生命周期](./docs/adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md)

### 16. 关闭皮肤时必须清除容器 Masonry 类名并对所有布局引擎实施启闭熔断
* **铁律**：`removeEmbyDOM()` 必须彻底移除 `.movie-list` 上的 `emby-masonry` 与 `emby-dynamic-focus` 类名；所有异步排版引擎（`WaterfallEngine.layout`、`WaterfallEngine.schedule`、`ensureLayoutClasses`、`restructureGrid`）入口必须设置 `if (!enabled) return;` 熔断守卫。
* **陷阱**：JAVDB 原生主页采用现代 CSS Grid 布局。若遗留 Masonry 类名或因图片 `load` 事件唤醒瀑布流重新写入 `translate3d`，绝对定位与 CSS Grid 坐标系会相互叠加放大数十像素，导致原生排版严重错位出界。
* **工程警示**：坚决抵御“用大重构掩盖排查不足”的过度设计冲动（如推翻重写已有稳定成熟的 `removeEmbyDOM`）；必须坚持第一性原理直击 CSS Grid 叠加本质，优先实施 6 行代码以内的最小侵入手术级微调。
* **参见决策**：[ADR-0008: SPA 路由与渲染生命周期](./docs/adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md) 及 [ADR-0013: 皮肤关闭态原生网格（CSS Grid）恢复与布局引擎启闭熔断架构](./docs/adr/0013-native-grid-restoration-and-layout-engine-circuit-breaker.md)

### 17. 抽屉账户登出必须遵循 Rails UJS RESTful DELETE 规范与严格 Cookie 判定
* **铁律**：Emby 抽屉提取原生 navbar 菜单项时必须完整保全 `data-method`、`data-confirm` 与 `rel` 属性；登出操作必须通过专属安全管道（优先触发隐藏的原生 Rails UJS 登出节点，兜底构造 `_method=delete` + CSRF Token 的 POST 表单），并清空本地临时凭据缓存；`isJavdbUserLoggedIn()` 的 Cookie 匹配必须使用严格正则排除 `_rucaptcha_session_id`。
* **陷阱**：JAVDB 服务端严格拒绝 `GET /logout`。若以 GET 发送请求，服务端不仅不会销毁 Session，反而会设置 `redirect_to=%2Flogout` 并 302 重定向至登录页，使用户误以为成功退出；后续刷新页面或开关皮肤时服务端的有效 Session 仍然存在，导致账号“幽灵复活”；此外，粗放正则匹配 `/_session/i` 会误命中图形验证码 Cookie，导致游客状态被误判为登录态。
* **参见决策**：[ADR-0014: Rails UJS DELETE 会话生命周期与 Emby 抽屉安全登出管道架构](./docs/adr/0014-rails-ujs-delete-session-lifecycle-and-safe-logout.md)

### 18. 统计中心观影标记上下列排列与全生命周期数据一致性
* **铁律**：数据统计中心（StatisticsCenter）核心 KPI 矩阵中展示的观影标记（已看 / 想看作品数）必须严格呈上下列垂直排列（上列宝石绿已看、下列琥珀金想看）；必须保持为纯粹稳健的统计卡片，不绑定多余的跨模块跳转交互；其上游统计数据源以本地 IndexedDB 全量电影表（`allDbMovies` 优先）的 `reviewStatus` 为真相源，在备份导出时随 `movies` 完整导出，备份导入还原时自动由 `collectStats()` 重新汇算，确保全生命周期数据绝对闭环。
* **陷阱**：在纯统计 KPI 指标卡片中随意绑定未经充分设计的路由筛选跳转不仅违反极简与单一职责原则，还可能因收藏夹当前分页或筛选状态冲突引发意外行为；若在上游数据采集时漏算未加入自定义清单但已被标记的作品，会导致统计数量与用户实际标记严重不符。

### 19. 鉴权路由避让与元素级 Referrer 隔离机制
* **铁律**：在 `/login`、`/user_sessions`、`/users/sign_in`、`/users/new`、`/users/password` 等关键鉴权路由上，脚本必须在入口处立即 return 避让，坚决不注入任何皮肤 DOM、样式或 MutationObserver，100% 保全原生 Rails Form POST 导航生命周期；严禁向 `<head>` 注入全局 `<meta name="referrer" content="no-referrer">`，避免破坏 Rails 同源 CSRF 校验（`verify_same_origin_request`）导致服务端 Session 重置与登录重定向死循环；防盗链需求一律由元素级 `referrerPolicy = 'no-referrer'` 承载，且显式豁免图形验证码（`.rucaptcha-image`）。
* **陷阱**：忽视 Rails 体系对同源 `Referer` / `Origin` 标头的严格校验，采用全局 Meta 粗暴阻断 Referer，导致合法的整页表单 POST 请求被服务端判定为 CSRF 攻击并被动清空 Session 验证码；无差别重写页面全部 `<img>` 标签导致图形验证码丢失与 Cookie 解绑。
* **参见决策**：[ADR-0015: Rails CSRF 同源 Referer 完整性保障与元素级防盗链穿透架构](./docs/adr/0015-rails-csrf-referer-integrity-and-image-level-anti-hotlink.md)

### 20. 严禁跨全 DOM 扫描 `a, button` 模拟点击处理弹窗
* **铁律**：处理 JAVDB 年龄弹窗等交互时，**绝对禁止**使用 `document.querySelectorAll('a, button')` 盲目扫描匹配正则并调用 `.click()`；必须通过在脚本启动入口提前主动写入官方 `over18=1` Cookie 实现服务端豁免，并仅针对 `.over18-modal` 或显式带有 `a[href*="/over18"]` 的模态层直接从 DOM 树移除解绑（零页面跳转、零模拟点击）；所有 Tab 导航栏点击必须调用 `e.stopPropagation()` 阻断事件向上冒泡。
* **陷阱**：全 DOM 扫描 `a, button` 会在榜单加载后误命中番号、标题、发布日期或标签中含有 `18` 的普通影片卡片（典型如热播榜首作品 MIAB-418，`/v/nKg2am`），从而触发不可控的 `.click()` 产生恶性弹窗与重定向死循环；Tab 点击未阻断冒泡会导致事件渗透到原生页面触发全局拦截。
* **参见决策**：[ADR-0016: JAVDB 官方年龄弹窗无感通行与全 DOM 盲点击拦截架构](./docs/adr/0016-age-gate-pre-cookie-injection-and-safe-modal-removal.md)

### 21. 搜索输入框必须强制 `type="search"` 与多重安全属性防 Chrome 账号嗅探
* **铁律**：所有搜索输入框（原版搜索框、Emby 顶栏克隆框、搜索弹窗 `#emby-search-modal`、收藏夹/画廊工具栏 `.efav-toolbar input`、多维筛选浮层 `.efav-dim input`、女优搜索框 `#actor-search-inp`、元数据修正搜索框 `#meta-correct-search` 以及通用 `input[placeholder*="搜索"]` 等）必须在源头创建时与 `sanitizeSearchInputs()` 清洗中统一强制设定为 `type="search"`、`role="searchbox"`，并附带 `autocomplete="off"`、`autocorrect="off"`、`autocapitalize="none"`、`spellcheck="false"`、`data-lpignore="true"`、`data-1p-ignore="true"`、`data-bwignore="true"` 和 `data-form-type="other"`；在非 `/search` 结果页或查询参数为空时，若检测到被浏览器自动回填且用户尚未编辑，必须主动清空重置；并在 CSS 层对 `input[type=search]` 的原生清除按钮（`-webkit-search-cancel-button`）进行 appearance 重置以保持界面精致。
* **陷阱**：Chromium 密码管理器与主流密码管理扩展（1Password、LastPass、Bitwarden 等）的启发式引擎，只要在已保存凭据的域名（如 JAVDB）下检测到普通的 `<input type="text">`（甚至是缺少 name 的匿名文本框），即使声明了 `autocomplete="off"`，仍会强行判定为用户名候选字段，在用户点击或聚焦输入框时弹出“一键填写账号密码/管理密码”提示。若遗漏了收藏夹工具栏或动态筛选器弹窗中的搜索框，会导致点击时再次唤醒凭据管理器。只有从“HTML 属性全屏蔽 + 无障碍角色升级为 searchbox + 全局周期性/挂载清洗 + CSS 兼容重置”四层纵深防御，才能彻底根除任何密码管理器下拉弹窗。
* **参见决策**：[ADR-0017: 首次运行零侵入开关与 Chrome 搜索框账号防误填架构](./docs/adr/0017-first-run-zero-intrusion-and-search-autofill-defense.md)

### 22. 首次安装脚本必须默认保持原生关闭状态（零侵入）
* **铁律**：脚本激活状态判定必须严格匹配 `localStorage.getItem(STORAGE_KEY) === '1'`；在初次安装（值为 `null`）或关闭态（值为 `'0'`）时，严禁注入全屏皮肤、修改原生 DOM 结构或覆盖原生 Navbar，仅挂载右下角切换微标并提示引导气泡，将界面的控制权 100% 交还用户。
* **陷阱**：使用 `!== '0'` 会导致首次安装的新用户在不知情的状态下瞬间被强行接管界面，不仅剥夺了用户的知情权与选择权，还容易在首次网络握手尚未就绪时引发渲染冲突。
* **参见决策**：[ADR-0017: 首次运行零侵入开关与 Chrome 搜索框账号防误填架构](./docs/adr/0017-first-run-zero-intrusion-and-search-autofill-defense.md)

### 23. 详情页元数据多语言与多格式健壮性提取
* **铁律**：`detailBlockVal` 与 `extractMetaLinks` 必须使用不区分大小写的全语言兼容正则（`i` 标志），完整覆盖简体中文、繁体中文、英文与日文标签变体（如“番號/番号”、“導演/导演/director/監督”、“片商/賣家/卖家/maker/studio/メーカー”、“系列/series/シリーズ”、“評分/评分/rating/評価”、“時長/时长/duration/時間”、“類別/类别/標籤/标签/tags/genres/ジャンル”）；演员列表区域必须采用“多语言文本包含 + `/actors/` 路由特征选择器”双重探测，严禁仅依靠单一繁体字面量全等判定。
* **陷阱**：在用户使用简体中文、日文镜像或第三方代理时，硬编码繁体字匹配会导致详情页海报 Hero 区关键元数据（演员、评分、导演、时长等）大面积出现空白或“未知”，甚至导致详情重构流程中断。
* **参见决策**：[ADR-0018: 详情页多语言环境元数据健壮性解析与容错架构](./docs/adr/0018-multilingual-detail-metadata-resilient-parsing.md)

### 24. DOM 跨层级插入直接子节点契约与核心网格沙盒隔离
* **铁律**：所有向原生父容器执行 `parent.insertBefore(newChild, refChild)` 的操作（如 `buildHomeTabs` 与 `ensureFavSurface`），必须通过 `resolveDirectChildAnchor(parent, target)` 递归向上追溯，严格确保传递给 `insertBefore` 的参考节点必然是 `parent` 的**直接子节点**（Direct Child）；必须采用多级回退的 `safeInsertBefore` 处理 DOM 挂载；`restructureGrid` 内部对外部组件（`buildHomeTabs`）的调用必须施加独立的 `try ... catch` 沙盒保护，严禁顶部导航栏的任何异常中断后续卡片的 `attachCard3D` 互动层初始化与 `dfPreview` 封面大图绑定。
* **陷阱**：JAVDB 官方前端不定期重构（如近期将 `.tabs.main-tabs` 嵌套进 `<div class="main-tabs-wrap">` 容器），若使用深度查找的 `querySelector` 获取深层子节点作为 `parent.insertBefore` 的参照物，会直接触发 W3C 标准的致命异常 `DOMException: The child can not be found in the parent`。由于缺乏沙盒隔离，该异常会使整个主页网格重构彻底腰斩，导致全站卡片悬停封面大图完全瘫痪、常驻导航条消失、左上角导航切换崩溃。
* **参见决策**：[ADR-0019: 嵌套 DOM 跨层级安全插入与网格重构沙盒保护架构](./docs/adr/0019-nested-dom-safe-insertion-and-grid-restructure-sandbox.md)

### 25. 未登录状态全链路行为拦截与数据库孤儿清洗对称性准则
* **铁律**：想看（wishlist）、看過（watched/评分）、清单（list）、关注女优等状态写操作与全量同步严格依赖 JAVDB 官方会话。未登录游客态下，卡片快捷标记（想看/看过/删除）、评分模态窗（`showCoverRatingModal`）、底层提交（`updateCoverReviewStatus`）、女优悬浮卡片收藏（`favBtn`/`remoteToggleFavorite`）以及所有云端同步任务，必须在交互层和网络层前置校验 `isJavdbUserLoggedIn()` 并在接收响应时检测 `res.redirected`，严防 302 伪成功向 IndexedDB 写入无云端归属的幽灵记录；后台静默孤儿清洗函数 `dbPurgeOrphanMovies` 必须保持绝对对称性：除真正用户本地资产（自定义纠错 `customMeta`、本地备注 `notes`）享有免死金牌外，无论想看还是看过（哪怕带 `userScore`），在未登录状态下或用户深度清洗时，均作为无清单归属孤儿统一清洗，严禁 `userScore` 成为单向阻断孤儿清洗的例外漏洞。
* **陷阱**：JAVDB 会为游客在 `<head>` 输出 `csrf-token`，且服务端对未登录 POST 请求返回 302 跳转至 `/login`。现代 `fetch` 默认跟随跳转并返回 HTTP 200，若无登录守卫与重定向探测，未登录操作会被误判为成功并写入数据库；而在孤儿清洗算法中，若为“已看”附带的 `userScore` 赋予本地资产豁免权，会导致未登录态下标记的“想看”被正确识别清洗，而“看過”却永久滞留本地数据库，产生严重的非对称残留 Bug。
* **参见决策**：[ADR-0020: 未登录状态全链路行为拦截与数据库孤儿清洗对称性架构](./docs/adr/0020-unauthenticated-session-interception-and-symmetrical-orphan-purging.md)

### 26. STEAM 风格卡牌 3D 物理倾斜锚点与 RAF 帧率锁机制
* **铁律**：所有电影卡片、紧凑条小封面与荣誉卡的 3D 物理倾斜（`attachCard3D`）必须将 `transform-origin` 严格锁定为左上角 `top left`，严禁使用浏览器默认的中心原点 `center center`；高频鼠标移动监听（`mousemove`）必须配备 `requestAnimationFrame` 单帧锁与 `cancelAnimationFrame` 即时取消，严禁在事件回调中同步触碰 layout/reflow；DOM 挂载前必须校验 `box.dataset.tilt` 幂等守卫，严禁重复绑定监听器；非详情卡片（如金标荣誉卡、详情页相关推荐行）必须显式传入 `{ noPreview: true }` 解耦封面预览气泡。
* **陷阱**：若采用默认中心原点 `center center`，卡片悬停放大 1.05 倍并倾斜时，第一列卡片左边缘会直接超出屏幕视口边界被截断，第一行卡片会直接撞入顶部导航栏造成穿模；高刷电竞鼠标（500~1000Hz）若无 RAF 锁，会在每秒内触发数百次 DOM 内联样式修改，引发严重的布局抖动（Layout Thrashing）与滚屏掉帧；若无 `dataset.tilt` 守卫，瀑布流或网格重排时会累加多重监听器导致动画撕裂。
* **参见决策**：[ADR-0021: STEAM 风格卡牌 3D 物理倾斜与全息流光渲染流水线](./docs/adr/0021-steam-3d-card-hover-tilt-and-specular-pipeline.md)

### 27. 动态聚焦布局、大图防碰撞避让与 DOM 节点脱水期惰性挂载契约
* **铁律**：动态聚焦网格（`.movie-list.emby-dynamic-focus`）采用无间距竖版流体宫格，卡片容器强制 $2:3$ 比例，海报图片采用 $200\%$ 宽右对齐黄金人脸裁切（`object-position: right center`），并彻底剥离标题/标签等非封面文本；悬停大图浮层（`dfPopup`）必须解耦运行，通过复用内存已有解码图片纹理实现 0 字节网络请求，并通过光标上下剩余空间动态决策弹出方向（`top = below ? y + 18 : y - 18 - h`），结合 64px 顶栏安全红线与 18px 物理空隙彻底杜绝光标遮挡与高频闪烁；全屏暗色环境遮罩（`#emby-df-scrim`）与气泡必须附带 `pointer-events: none` 确保原生点击穿透；在生成卡片（`buildCard`）等 DOM 脱水期，严禁静态捕获未连接父网格，必须推迟至 `mouseenter` 时惰性嗅探 `box.closest(...)` 并挂载网格级委托 `ensureDynamicFocusGrid`；`mouseenter` 回调必须显式接收 `(ev)` 事件对象并对各步操作加设 `try/catch` 隔离保护；全局常驻 `mousemove` 监听器，一旦光标物理坐标脱离卡片包围盒（`getBoundingClientRect`）或页面发生滚动，立即强制撤回放大缩放与浮层。
* **陷阱**：v7.107 历史重大 Bug 复盘——旧代码在 `attachDynamicFocus` 中未声明形参直接访问 `e.clientX`，抛出未捕获的 `ReferenceError` 导致后续网格委托未能挂载，卡片在触发 `scale(1.28)` 后永久冻结放大无法复原，且浮层永久消失；在收藏夹筛选/排序重绘时，若在卡片挂树前静态获取父网格，会因为 `isConnected === false` 得到 `null`，导致动态聚焦完全失效；若缺少 `mousemove` 包围盒全局巡检，在复杂动画或 DOM 突变时丢失 `mouseleave`，会导致卡片放大状态卡死在页面中。
* **参见决策**：[ADR-0027: DYNAMIC FOCUS 动态聚焦布局、视口防碰撞大图浮层与环境流光跟随架构](./docs/adr/0027-dynamic-focus-layout-collision-avoidance-and-ambient-preview.md)

### 28. 预览图嗅探多源清洗、L1/L2 两级持久化缓存与图片 Host 零迁移相对路径存储准则
* **铁律**：多站点预览图嗅探（`PreviewSys`）跨域抓取时，必须强制对输入番号执行去空格、去斜杠规范化；严格防御目标站点（如 ProjectJav）在返回 HTML 时出现的双域名拼接 Bug（`https://domain.comhttps://img...`），必须经由通用域名剥离流水线净化；Pixhost 图床抓取时，必须自动将缩略图 `//tXXX.pixhost.to/thumbs/` 向上提升为高清大图 `//imgXXX.pixhost.to/images/`；**嗅探结果必须采用「L1 同步内存 Map + L2 IndexedDB (`meta` 表, `jhs_prev_` 前缀) 7 天 TTL（`7*24*60*60*1000`）两级缓存体系」，成功态双写缓存以实现二次展现 0 毫秒秒开与 0 网络请求，大幅降低第三方 CDN 触发 429 风控概率；严禁将偶发网络失败或 `no-data` 写入持久化数据库，负向状态仅限单次会话内存缓存**；持久化至 IndexedDB 的影片封面与图床图片路径必须经由 `stripHost` 剔除 CDN 主机名前缀仅保留相对路径（如 `/covers/xx.jpg`），读取渲染时经由 `detectImgHost()` 动态拼接当前可用镜像域名；自愈正则必须支持历史废弃域名的平滑清洗。
* **陷阱**：若将包含具体 CDN 域名（如 `c0.jdbstatic.com`）的完整 URL 固化写入本地数据库，当 JAVDB 因防封或 CDN 供应商迁移更改图片域名时，用户本地收藏夹中的海量封面将瞬间全部 404 挂掉，造成无法挽回的灾难；若未清洗目标站点返回的双域名字符串，会产生畸形 URL 并触发 CORS/DNS 失败；若未提取 Pixhost 大图，抓取到的预览剧照分辨率仅为 150px 邮票大小；**若缺少 L2 7 天持久化缓存，每次刷新或切换列表都会疯狂重复抓取第三方图床，极易被第三方防爬机制拉黑（HTTP 429）；若把网络异常或 `no-data` 负向结果固化持久化至数据库，用户将遭遇长达 7 天的剧照假死盲区**。
* **参见决策**：[ADR-0022: PREVIEWSYS 多站点预览图嗅探、高清图床推导与双域名清洗流水线](./docs/adr/0022-preview-sys-multi-site-sniffing-and-sanitization-pipeline.md)、[ADR-0023: 动态图片 HOST 嗅探与零迁移相对路径持久化架构](./docs/adr/0023-dynamic-image-host-detection-and-relative-path-storage.md)

### 29. 媒体库筛选器即时草稿状态隔离与共演算子实时交集收敛
* **铁律**：筛选模态窗（`openFilterModal`）在打开时必须基于当前配置深拷贝生成独立草稿对象 `tmp`，用户在弹窗内的任何勾选、取选、滑动均严格限制在 `tmp` 沙箱内运行，点击遮罩或取消时直接丢弃 `tmp`，绝对禁止在点击「确定」之前触碰或污染主列表全局过滤状态；切换状态类开关（如「只看有备注」、「看過」）时，必须通过 `modalVids()` 毫秒级重构当前候选子宇宙，并触发 `refreshModalDims()` 动态重新计算全维度标签的出现频次与累积浏览量，自动剔除计数为 0 的虚假候选项；开启「共演筛选（交集）」后，演员多选逻辑必须从并集（OR）无缝切换为严格子集交集（AND），选定演员后其它维度候选必须实时收敛至共同出演过的作品范围内；演员头像渲染源必须严格在 `modalVids` 就绪后执行；维度搜索框必须配置全套防密码管理器误填属性并经由 `sanitizeSearchInputs` 清洗。
* **陷阱**：若采用即时响应全局模式，用户在多达数十个维度的弹窗中随意尝试组合后若想放弃，页面已千疮百孔无法复原；若在筛选弹窗中仍展示全量静态列表，用户点击某位演员后经常出现 0 部匹配结果的挫败死胡同；若 `vids` 初始化顺序颠倒，会导致勾选演员头像时抛出异常导致整个演员候选列表被完全清空；未防御的搜索框会引来 1Password/Bitwarden 强制弹出账号填充下拉框，阻挡用户点击选项。
* **参见决策**：[ADR-0026: 媒体库多维流式筛选器即时草稿状态机与共演交集算子](./docs/adr/0026-multi-dimensional-filter-draft-snapshot-and-coact-intersection-engine.md)

### 30. 云端多端同步增量字段胜者合并与用户主观资产不可侵犯法则
* **铁律**：多端数据同步与备份导入（`FAV.importJSON`）必须严格遵循「字段丰富度评分胜者算法（`fieldRichness`）」进行属性级智能裁决，严禁采用粗暴的全量记录覆盖或盲目的时间戳（LWW）整行替换；无论远端元数据得分多高，用户本地的主观纠错（`customMeta`，按修改时间戳比对）、用户主观打星评分（`userScore`，既有评分绝对优先保全）、个人文字批注（`notes`，非空保全）、所属清单归属（`listIds`，严格执行数学并集 $A \cup B$）、总浏览量（`views`，单调递增取 $\max$）享有最高法律效力，绝对不可被远端空值或客观官方数据篡改抹除；导入过程必须采用「单次读全集建立内存哈希索引 + 纯内存冲突合并去重 + 50条分块批量写回」流水线，万条数据事务总数严控在 200 次以内；备份导出时默认剔除所有敏感访问令牌与凭据，仅在用户显式勾选授权时方可打包导出。
* **陷阱**：在早期的逐条事务实现中，导入万条记录需触发数万次 IndexedDB 事务往返，导致浏览器完全失去响应长达数分钟；若采用整行覆盖，在一台设备上记录的观影笔记与打星会被另一台设备的同步操作彻底摧毁；若导出时未剥离凭据，分享备份文件会导致用户的 JAVDB 登录令牌与 WebDAV 密码泄露。
* **参见决策**：[ADR-0025: 云端多端同步增量字段胜者合并算法与冲突裁决矩阵](./docs/adr/0025-cloud-sync-field-level-completeness-merge-and-conflict-resolution.md)

### 31. 严格遵循 SemVer 2.0 与 Keep a Changelog 版本演进协议
* **铁律**：后续任何 AI 智能体或开发者在提交功能新增、缺陷修复或架构优化时，必须严格遵守 [Semantic Versioning (语义化版本 2.0.0)](https://semver.org/lang/zh-CN/) 升级规范；`CHANGELOG.md` 必须严格在对应版本标题 `## [vX.Y.Z] - YYYY-MM-DD` 下按标准五大维度分类记录（`### 新增 (Added)`、`### 修复 (Fixed)`、`### 优化 (Optimized)`、`### 安全与防御 (Security)`、`### 核心架构 (Core)`），严禁生成无分类流水账，严禁省略版本号或发布日期；代码头部 `@version`、脚本内部 `VERSION` 常量、`CONTEXT.md` 全局版本与 `CHANGELOG.md` 必须保持强一致原子化同步；涉及重要底层架构决策时必须同步更新或新增对应 ADR 文档。
* **陷阱**：若无版本演进与日志分类约束，后续接手的 AI 极易生成格式混乱的碎片化日志、漏更脚本元数据版本号、甚至粗暴覆写或抹除历史版本记录，导致用户在油猴管理器中无法检测到版本更新或无法溯源破坏性变更。
* **参见决策**：[CHANGELOG.md](./CHANGELOG.md)

### 32. 视频预览统一拉取引擎、双源容灾与详情页分辨率自适应播放架构
* **铁律**：封面悬停小视频预览必须与未裁切大图浮层（`#emby-df-preview`）融合为一体；为防止未预期的流量消耗，卡片悬停视频预览功能默认保持关闭（`'0'`），用户可在设置中主动开启；详情页预览栏新增 `video_template` 视频播放按钮（`DetailVideoPreview`），为显式主动交互按需响应；所有调用方必须统一调用全局唯一的单例拉取器 `getPreviewVideoBlobUrl(rawCode)`，严禁分散自行 fetch；入口必须对番号强制进行 `.trim().toLowerCase()` 归一化清洗，保障 123AV MD5 计算与 MissAV CDN 路径严格匹配，并实现卡片与详情页双向内存会话缓存（`dfVideoCache`）秒开共享；详情页播放弹窗尺寸必须依据 `loadedmetadata` 事件中的 `videoWidth` 与 `videoHeight` 严格自适应计算，等比例约束在视口安全范围内，严禁画面拉伸与黑边变形；弹窗元素 `#emby-detail-video-pop` 必须纳入 `removeEmbyDOM` 卸载清理集与 `SKIN_CHROME_SEL` 过滤集，确保皮肤关闭无残留且弹窗状态变动不触发全局 MutationObserver 递归。
* **陷阱**：若无两阶段延迟防抖，光标快速扫视卡片时会瞬间并发几十个视频网络请求，导致浏览器网络栈堵塞并触发第三方 CDN 429 频控；若未对番号强制转换为小写，大写番号会导致 MD5 哈希错误（如 `IPZZ-934` 计算为 `b9` 而非正确的 `94`）进而触发双源同时 404；若未将弹窗纳入 `SKIN_CHROME_SEL`，弹窗载入动画会造成全局路由监听频繁抖动。
* **参见决策**：[ADR-0028: 封面悬停视频预览双源容灾引擎与防抖播放架构](./docs/adr/0028-hover-preview-video-dual-source-fallback-and-debounced-playback.md)

---

## 架构决策档案索引 (Architectural Decision Records)

完整技术演化背景、备选方案与决策权衡已记录于 `docs/adr/`：

1. [ADR-0001: 单文件 Monolithic UserScript 架构](./docs/adr/0001-monolithic-userscript-architecture.md)
2. [ADR-0002: IndexedDB + 内存镜像双层存储架构](./docs/adr/0002-multi-tier-storage-indexeddb-memory-mirror.md)
3. [ADR-0003: LiquidGlass 液态玻璃 2.0 物理透镜折射流水线与 Chromium 合成层防御架构](./docs/adr/0003-liquidglass-svg-displacement-pipeline.md)
4. [ADR-0004: 无性别偏见的演员高清头像解析与平滑降级](./docs/adr/0004-gender-agnostic-avatar-matching-and-fallback.md)
5. [ADR-0005: 用户主观评分 (`userScore`) 与官方评分 (`rating.score`) 的物理隔离体系](./docs/adr/0005-strict-separation-of-userscore-and-official-rating.md)
6. [ADR-0006: DOM MutationObserver 递归守卫与 window.confirm 猴子补丁可重入锁](./docs/adr/0006-reentrant-dialog-and-mutationobserver-recursion-guards.md)
7. [ADR-0007: 被动采集熔断器与自定义纠错元数据 (`customMeta`) 永久保全](./docs/adr/0007-passive-collection-and-custom-metadata-healing.md)
8. [ADR-0008: JAVDB SPA 局部路由生命周期接管与原生浮层/设置遮罩同步](./docs/adr/0008-spa-route-lifecycle-and-native-mask-synchronization.md)
9. [ADR-0009: 敏感凭据双重脱敏导出与 `Object.create(null)` 原型链污染防御](./docs/adr/0009-two-tier-sync-and-credential-desensitization.md)
10. [ADR-0010: 详情页封面融合模式（Cover Fusion）与四态排版引擎架构](./docs/adr/0010-cover-fusion-and-multi-layout-engine.md)
11. [ADR-0011: 异步榜单视图基于请求代数令牌（Sequence Token）的防乱序架构](./docs/adr/0011-request-sequence-tokens-for-rankings.md)
12. [ADR-0012: JAVDB 服务端 VIP 302 拦截墙规避与多源在线播放矩阵](./docs/adr/0012-vip-wall-bypass-and-multi-source-playback.md)
13. [ADR-0013: 皮肤关闭态原生网格（CSS Grid）恢复与布局引擎启闭熔断架构](./docs/adr/0013-native-grid-restoration-and-layout-engine-circuit-breaker.md)
14. [ADR-0014: Rails UJS DELETE 会话生命周期与 Emby 抽屉安全登出管道架构](./docs/adr/0014-rails-ujs-delete-session-lifecycle-and-safe-logout.md)
15. [ADR-0015: Rails CSRF 同源 Referer 完整性保障与元素级防盗链穿透架构](./docs/adr/0015-rails-csrf-referer-integrity-and-image-level-anti-hotlink.md)
16. [ADR-0016: JAVDB 官方年龄弹窗无感通行与全 DOM 盲点击拦截架构](./docs/adr/0016-age-gate-pre-cookie-injection-and-safe-modal-removal.md)
17. [ADR-0017: 首次运行零侵入开关与 Chrome 搜索框账号防误填架构](./docs/adr/0017-first-run-zero-intrusion-and-search-autofill-defense.md)
18. [ADR-0018: 详情页多语言环境元数据健壮性解析与容错架构](./docs/adr/0018-multilingual-detail-metadata-resilient-parsing.md)
19. [ADR-0019: 嵌套 DOM 跨层级安全插入与网格重构沙盒保护架构](./docs/adr/0019-nested-dom-safe-insertion-and-grid-restructure-sandbox.md)
20. [ADR-0020: 未登录状态全链路行为拦截与数据库孤儿清洗对称性架构](./docs/adr/0020-unauthenticated-session-interception-and-symmetrical-orphan-purging.md)
21. [ADR-0021: STEAM 风格卡牌 3D 物理倾斜与全息流光渲染流水线](./docs/adr/0021-steam-3d-card-hover-tilt-and-specular-pipeline.md)
22. [ADR-0022: PREVIEWSYS 多站点预览图嗅探、高清图床推导与双域名清洗流水线](./docs/adr/0022-preview-sys-multi-site-sniffing-and-sanitization-pipeline.md)
23. [ADR-0023: 动态图片 HOST 嗅探与零迁移相对路径持久化架构](./docs/adr/0023-dynamic-image-host-detection-and-relative-path-storage.md)
24. [ADR-0024: GFRIENDS 高清头像映射、别名桥接与四级 CDN 容灾架构](./docs/adr/0024-gfriends-avatar-mapping-alias-bridge-and-cdn-disaster-recovery.md)
25. [ADR-0025: 云端多端同步增量字段胜者合并算法与冲突裁决矩阵](./docs/adr/0025-cloud-sync-field-level-completeness-merge-and-conflict-resolution.md)
26. [ADR-0026: 媒体库多维流式筛选器即时草稿状态机与共演交集算子](./docs/adr/0026-multi-dimensional-filter-draft-snapshot-and-coact-intersection-engine.md)
27. [ADR-0027: DYNAMIC FOCUS 动态聚焦布局、视口防碰撞大图浮层与环境流光跟随架构](./docs/adr/0027-dynamic-focus-layout-collision-avoidance-and-ambient-preview.md)
28. [ADR-0028: 封面悬停视频预览双源容灾引擎与防抖播放架构](./docs/adr/0028-hover-preview-video-dual-source-fallback-and-debounced-playback.md)


