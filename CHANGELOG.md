# 更新日志 (CHANGELOG)

<!-- 
[AI ASSISTANT INSTRUCTION / 维护者更新契约]
任何后续接手本项目的 AI 智能体或开发者在更新本文件时，必须严格遵守以下准则：
1. 规范遵循：严格遵循 Keep a Changelog (1.0.0) 与 Semantic Versioning (2.0.0) 规范；
2. 结构固定：新版本必须插入在最上方，格式统一为 ## [vX.Y.Z] - YYYY-MM-DD；
3. 五大分类：条目必须严格按维度分类，仅允许以下标准分类标题：
   - ### 新增 (Added)
   - ### 修复 (Fixed)
   - ### 优化 (Optimized)
   - ### 安全与防御 (Security)
   - ### 核心架构 (Core)
4. 深度关联：若改动涉及底层重要决策或历史踩坑，必须提供指向 docs/adr/ 对应 ADR 文档的相对链接；
5. 禁止行为：严禁生成无分类流水账，严禁省略版本号或发布日期，严禁覆盖或删除既有版本的历史记录。
-->

本项目的版本演进严格遵循 [Semantic Versioning (语义化版本 2.0.0)](https://semver.org/lang/zh-CN/) 规范，所有重大架构决策与历史踩坑复盘均已归档于 [`docs/adr/`](./docs/adr/)。

\---

## [v7.335] - 2026-09-30

### 新增 (Added)

* **PreviewSys 7天 IndexedDB 本地持久化缓存**：在 PreviewSys 剧照嗅探层引入 L1 内存 + L2 IndexedDB（`meta` 实体表，`jhs_prev_` 前缀）双层缓存与 7 天 TTL（`7 * 24 * 60 * 60 * 1000`），实现二次访问 0 毫秒秒开与 0 字节网络请求，有效避免频繁跨域抓取触发第三方 CDN 429/403 风控；严格执行仅成功态持久化法则，防止负向缓存污染；详见 [ADR-0022](./docs/adr/0022-preview-sys-multi-site-sniffing-and-sanitization-pipeline.md)。

### 修复 (Fixed)

* **主页视图导航栏 DOM 级幂等与多重去重**：修复因多脚本实例并发运行或 JAVDB 官方 Turbolinks 页面快照缓存（Snapshot Cache）脱水重入时，`buildHomeTabs()` 仅依赖 JS 闭包变量导致页面顶部出现上下重叠两排导航标签栏的视觉异常；在入口处引入 `document.querySelectorAll('.emby-home-tabs')` DOM 扫描与僵尸节点清理机制，保证页面始终唯一呈现；`removeEmbyDOM()` 同步升级为全量清理。

### 核心架构 (Core)

* **零号工程公理与交付前自检清单建立**：在 [`CONTEXT.md`](./CONTEXT.md) 中正式确立「Axiom 0: 第一性原理深度穿透本质与极简手术级微调原则」，坚决抵御用大重构掩盖排查不足；设立 4 步交付前自检清单，将版本演进协议升级为第 31 条核心铁律。
* **完备更新日志建立**：基于 Keep a Changelog 格式建立 [`CHANGELOG.md`](./CHANGELOG.md)，记录从 v5.00+ 到 v7.335 的完整演化历史。

---

## [v7.334] - 2026-09-29

### 新增 (Added)

* **PreviewSys 多站点嗅探扩充**：新增对 `ProjectJav`（`https://projectjav.com`）的剧照预览图嗅探支持，配备专属 96×96 高清网站图标；
* **BSD-3-Clause 开源协议**：项目开源许可证正式切换为 `BSD-3-Clause License`，强化作者商誉与名誉保护，增加明确的禁止背书/借名宣传条款，同步更新 [`LICENSE`](./LICENSE) 与脚本头部 `@license` 元数据；
* **架构决策档案群（ADR-0018 \~ ADR-0027）**：

  * [ADR-0018: 详情页多语言环境元数据健壮性解析与容错架构](./docs/adr/0018-multilingual-detail-metadata-resilient-parsing.md)
  * [ADR-0019: 嵌套 DOM 跨层级安全插入与网格重构沙盒保护架构](./docs/adr/0019-nested-dom-safe-insertion-and-grid-restructure-sandbox.md)
  * [ADR-0020: 未登录状态全链路行为拦截与数据库孤儿清洗对称性架构](./docs/adr/0020-unauthenticated-session-interception-and-symmetrical-orphan-purging.md)
  * [ADR-0021: STEAM 风格卡牌 3D 物理倾斜与全息流光渲染流水线](./docs/adr/0021-steam-3d-card-hover-tilt-and-specular-pipeline.md)
  * [ADR-0022: PREVIEWSYS 多站点预览图嗅探、高清图床推导与双域名清洗流水线](./docs/adr/0022-preview-sys-multi-site-sniffing-and-sanitization-pipeline.md)
  * [ADR-0023: 动态图片 HOST 嗅探与零迁移相对路径持久化架构](./docs/adr/0023-dynamic-image-host-detection-and-relative-path-storage.md)
  * [ADR-0024: GFRIENDS 高清头像映射、别名桥接与四级 CDN 容灾架构](./docs/adr/0024-gfriends-avatar-mapping-alias-bridge-and-cdn-disaster-recovery.md)
  * [ADR-0025: 云端多端同步增量字段胜者合并算法与冲突裁决矩阵](./docs/adr/0025-cloud-sync-field-level-completeness-merge-and-conflict-resolution.md)
  * [ADR-0026: 媒体库多维流式筛选器即时草稿状态机与共演交集算子](./docs/adr/0026-multi-dimensional-filter-draft-snapshot-and-coact-intersection-engine.md)
  * [ADR-0027: DYNAMIC FOCUS 动态聚焦布局、视口防碰撞大图浮层与环境流光跟随架构](./docs/adr/0027-dynamic-focus-layout-collision-avoidance-and-ambient-preview.md)

### 修复 (Fixed)

* **ProjectJav 双域名拼接 Bug 净化**：解决目标站点在返回 HTML 时将外链图片写为畸形双域名（`https://domain.comhttps://img...`）的问题，构建通用域名剥离与净化流水线；
* **Pixhost 高清图床反查推导**：自动将 Pixhost 缩略图路径 `//tXXX.pixhost.to/thumbs/` 向上提升还原为 `//imgXXX.pixhost.to/images/`，抓取的预览剧照清晰度提升 400%；
* **输入框防嗅探防护越界修复**：修复因全局防密码管理器嗅探规则覆盖过宽，导致收藏夹清单范围下拉菜单中的多选框（`type="checkbox"`）以及多维筛选器中的评分滑动条（`type="range"`）误被强制篡改为文本输入框的严重交互 Bug；
* **空预览图回退防崩**：若第三方嗅探源返回空数组或请求超时，自动优雅回退官方剧照或默认大图，杜绝控制台抛出未捕获异常。

### 优化 (Optimized)

* **视觉舒适度提升**：将状态快捷标记「已看」按钮星级评分弹窗界面的背景毛玻璃模糊浓度降低 50%，极大减轻了视觉沉闷感，提升在各类高动态封面下的对比可读性；
* **架构核心铁律扩充**：[`CONTEXT.md`](./CONTEXT.md) 核心铁律扩充至 30 条，完整记录动态聚焦、预览图图床探测、筛选器草稿状态机与云端增量合并裁决准则。
* **重写液态玻璃视觉文档** [ADR-0003: LiquidGlass 液态玻璃 2.0 物理透镜折射流水线与 Chromium 合成层防御架构](./docs/adr/0003-liquidglass-svg-displacement-pipeline.md)

\---

## \[v7.333] - 2026-09-26

### 修复 (Fixed)

* **多语言元数据提取容错**：修复在简体中文、日文镜像及第三方代理环境下，详情页 Hero 区演员、导演、时长、评分及类别标签大面积出现空白或“未知”的缺陷；
* **多重特征双向嗅探**：构建大小写无关的全语言匹配正则（简/繁/英/日变体完整覆盖），演员列表区域引入 `/actors/` 路由特征选择器进行第二道防护，杜绝硬编码繁体字导致的数据解析腰斩。

\---

## \[v7.332] - 2026-09-26

### 修复 (Fixed)

* **嵌套 DOM 跨层级直接子节点安全插入契约**：针对 JAVDB 官方前端近期重写时将 `.tabs.main-tabs` 深度嵌套进 `<div class="main-tabs-wrap">` 容器的变更，实现 `resolveDirectChildAnchor` 向上追溯，严格确保传给 `parent.insertBefore` 的必然是父容器的直接子节点；
* **多级安全挂载降级**：实现 `safeInsertBefore` 处理挂载回退，避免触发 W3C 标准的致命异常 `DOMException: The child can not be found in the parent`；
* **核心网格沙盒隔离保护**：对主页常驻导航条 `buildHomeTabs` 施加独立 `try-catch` 沙盒保护，阻断顶部导航栏的任何偶发异常，彻底根治全站卡片悬停封面大图瘫痪、左上角导航切换失效等雪崩问题。

\---

## \[v7.331] - 2026-09-26

### 安全与防御 (Security)

* **未登录游客状态全链路拦截**：在卡片快捷标记（想看/已看/删除）、评分模态弹窗、底层提交、女优悬浮卡片收藏及所有云端同步任务中，前置校验 `isJavdbUserLoggedIn()` 并在收到响应时探测 `res.redirected`，严防游客态下 JAVDB 服务端 302 伪成功导致向本地写入无云端归属的幽灵记录；
* **数据库孤儿清洗对称性准则重构**：重构 `dbPurgeOrphanMovies` 孤儿清洗逻辑，除用户本地核心资产（`customMeta` 自定义纠错、`notes` 本地笔记）享有永久保护外，未登录状态下标记的“想看”与带 `userScore` 的“看過”记录一视同仁进行彻底清理，彻底消灭单向残留漏洞。

\---

## \[v7.330] - 2026-09-26

### 新增 (Added)

* **首次安装零侵入开关**：初次安装或关闭态时默认保持原生状态（`localStorage.getItem(...) === '1'`），绝对不强行接管界面，仅在右下角挂载优雅的切换微标与引导气泡；
* **JAVDB 官方年龄弹窗（Age Gate）无感通行**：启动入口主动向根域写入官方 `over18=1` Cookie 实现服务端免检，从 DOM 树中安全移除 `.over18-modal`；**严禁全 DOM 盲点击 `a, button`**，杜绝误命中番号含 18 的普通影片卡片（如 MIAB-418）引发的重定向死循环。

### 安全与防御 (Security)

* **搜索框凭据嗅探深度免疫体系**：由 `type="search"`、`role="searchbox"`、11 项反嗅探忽略属性（`data-lpignore`、`data-1p-ignore` 等）、`sanitizeSearchInputs()` 全局生命周期动态清洗与聚焦主动冲刷（Focus Flush）构筑四层立体纵深防御，彻底根除 Chromium `PasswordAutofillAgent` 误填已保存账号密码及点击时弹出系统密码管理下拉框的顽疾。

### 优化 (Optimized)

* **封面融合模式（Cover Fusion）防毒守卫**：在 `cover-fusion` 滚动监听中加入动态阈值防毒与绝对顶部守卫，彻底消灭快速滑屏时的两栏裁切海报跳闪问题。

\---

## \[v7.322] - 2026-09-19

### 修复 (Fixed)

* **Blocker 级别引用崩溃自愈**：补全因重构遗漏的 `getCachedMovies` 导出定义，彻底恢复脚本正常启动生命周期；
* **Rails UJS DELETE 会话生命周期与登出管道**：修复 Emby 抽屉登出链接在原生页面中触发 GET 导致 404 的问题，模拟 Rails 原生 `method="delete"` 表单提交，确保安全登出并不残留脏会话；
* **元素级防盗链穿透**：针对 JAVDB 封面与第三方图床的 403 防盗链阻断，全面注入 `referrerPolicy="no-referrer"` 策略，同时排除人机验证码表单保持同源凭证完整。

\---

## \[v7.321] - 2026-09-18

### 修复 (Fixed)

* **皮肤关闭态原生网格（CSS Grid）恢复与布局引擎启闭熔断**：修复关闭皮肤切换回原生网页时，若干闭包引用与 `MutationObserver` 观察器未复位导致的全局样式残留与页面布局断裂；
* **数据流安全熔断机制**：修复被动采集在判断取消标记时误把“网络未爬完”当成“用户未标记”而误删本地记录的重大安全隐患，引入数量断崖熔断机制。

\---

## \[v7.320] - 2026-09-17

### 新增 (Added)

* **媒体库数据统计中心 (StatisticsCenter)**：基于 IndexedDB 全表异步并发聚合的核心资产仪表盘，提供片量、女优、笔记、清单、订正数 KPI，四档评分阶梯进度条及全维度排行榜；
* **多源在线播放站点矩阵 (PlaySitesService)**：整合 123AV, njavTV, MissAV, Jable, NetFlav 等 10+ 外部在线播放站，支持 `{code}`、`{code\_lower}`、`{code\_num}` 动态变量替换与连通性嗅探；
* **突破 JAVDB 服务端 VIP 302 拦截**：针对 `/plans/ypay` 强制跳转拦截进行前置路由规避与重定向穿透，免登录畅行高级搜索。

\---

## \[v7.311] - 2026-09-15

### 优化 (Optimized)

* **Material Symbols 图标统一映射**：部分淘汰旧版 Material Icons 与字符字体混用方案，规范化命名映射，消除图标对齐基线抖动。

\---

## \[v7.308] - 2026-09-14

### 新增 (Added)

* **状态快捷标记与预览图系统 (Cover Status \& Preview Modal)**：海报卡片右上角悬停快捷状态标记浮层，支持一键切换想看、已看（联动评分模态框）及删除；
* **元数据自定义纠错与自愈引擎 (Metadata Editor \& Self-Healing UI)**：详情页番号旁提供「編輯」与「備註」入口，允许用户手动修正番号、标题、女优名单及自定义封面大图，数据以不可变时间戳加锁持久化至 `customMeta`，享有绝对免死金牌，终身免疫外部被动采集覆盖。

\---

## \[v7.261 \~ v7.257] - 2026-09-08 \~ 2026-09-10

### 新增 (Added)

* **JAVDB 荣誉榜单金标 (Awards) 体系**：精细化重构金标展示，引入全实心金色皇冠徽章，收藏夹数据同步与 HUD 联动呈现；
* **封面融合模式重构 (Cover Fusion)**：详情页四态排版引擎核心分支——原版高清横版封套展示，页面下滚标题触顶时平滑收缩过渡为左海报右详情的两栏紧凑布局。

\---

## \[v7.238] - 2026-09-01

### 新增 (Added)

* **请求代数令牌 (Sequence Token / top250LoadSeq)**：在异步榜单视图与无刷新分页中引入递增代数令牌机制，彻底解决快速点击切换 Tab 时因网络延迟不同导致的异步乱序回包覆盖 Bug。

\---

## \[v7.146 \~ v7.144]

### 修复 (Fixed)

* **动态内联高度残留治理**：彻底治理详情页与海报栏在重构注销与窗口缩放时的 `inline height` 属性残留，消除页面排版折叠错位。

\---

## \[v7.132 \~ v7.128]

### 新增 (Added)

* **全量图标系统迁移**：全站样式架构从旧版字体图标全面迁移至 Google Material Symbols；
* **专属页面快速同步通道**：开辟针对「想看 / 看過」专属路由的高速无痛同步批处理流程；
* **荣誉金标存量作品检测**：引入 `awardsChecked` 标志位，全量对存量影视库启动一次性金标元数据自动回填补齐任务。

\---

## \[v7.115 \~ v7.111]

### 新增 (Added)

* **STEAM 风格卡牌 3D 物理倾斜与全息高光 (attachCard3D)**：基于相对归一化坐标空间结算微透视倾角，采用 `transform-origin: top left` 黄金锚点，配合 `--mx`/`--my` 实时全息径向反光层与 RAF 单帧锁；
* **动态聚焦 (Dynamic Focus) 视觉升级**：动态聚焦网格与 STEAM 3D 流光合体，聚焦单元放大 10%（scale 1.28），相邻单元微缩（scale 0.88）营造流体呼吸感；
* **解耦式未裁切大封面气泡预览 (dfPopup)**：从动态聚焦中解耦，成为网格/瀑布流/聚焦通用全局单例浮层，全盘复用内存已加载图片纹理，实现零网络开销与视口双向智能防碰撞避让；
* **覆盖导入真·完整还原**：重构 `FAV.importJSON`，覆盖模式严格清空全部 6 张底层数据表，并在写回后自动重建内存集。

\---

## \[v7.108 \~ v7.107]

### 新增 (Added)

* **LiquidGlass 液态玻璃 2.0 物理透镜折射流水线**：彻底舍弃引发 CORS 污染与内存泄漏的 HTML5 Canvas 像素运算，全面转入 2D 物理有向距离场 (RoundedBoxSDF) + 离线轻量位移贴图 + SVG `<feDisplacementMap>` 硬件片元着色器执行流水线；引入 Micro-Saturate Jitter（微饱和抖动）破除 Chromium 合成层 GPU 缓存冻结；
* **动态聚焦网格委托解耦与包围盒守护**：修复 `attachDynamicFocus` 中事件对象未传导致的卡片放大永久卡死 Bug，实现卡片脱水期惰性挂载，建立全局 `mousemove` 物理包围盒安全巡检守卫。

\---

## \[v7.88 \~ v7.86]

### 新增 (Added)

* **全封面模式 (Full Cover)**：纯正横版原始封套展示；
* **悬浮固定紧凑条 (Compact Bar)**：页面滚动时优雅浮现，集成番号、标题、评分与核心操作。

\---

## \[v5.00+] - 核心存储引擎奠基（HY3、deepseek-v4-flash）

### 核心架构 (Core)

* **多层混合持久化存储 (FAV)**：构建 IndexedDB `javdb-emby-fav-db` (v5) 实体数据库与同步内存双向镜像双层存储架构；
* **事务分块切片机制**：确立批量写库严格按 50 条一批开启写事务的铁律，兼顾吞吐量与主线程抗卡死；
* **单文件 Monolithic UserScript 架构**：建立完整的隔离沙盒与无外部打包器依赖的自给自足架构基石；
* \*\*整体功能框架结构、EMBY与毛玻璃效果皮肤、数据采集、预览画廊图库、三种封面展现模式、互动动效等等内容。

