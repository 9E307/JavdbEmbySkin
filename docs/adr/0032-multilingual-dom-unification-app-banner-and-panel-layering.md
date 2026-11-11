# ADR-0032: 全链路多语言 DOM 统一解析、新版官方 App 推广条结构性拦截与全屏面板图层隔离架构 (v7.6.1)

* **状态**: 已采纳 (Accepted)
* **日期**: 2026-11-12
* **关联版本**: `v7.6.1`
* **关联模块**: `hideJavdbAds`, `observeMutations`, `DETAIL_FIELD_RES`, `DETAIL_ALSO_STARRED_RE`, `DETAIL_YOU_MIGHT_LIKE_RE`, `parseJavdbRatingText`, `detectReviewTitleStatus`, `classifyReviewActionEl`, `buildAccountMenuHtml`, `Top250ViewManager`, `FavoriteActressManager`, `FAV.parseListLink`, `FAV.parseListPage`, `FAV.parseDetail`, `FAV.detectActorGenders`, `extractReviewButtons`, `extractDetailData`, `restructureDetail`, `extractMetaLinks`, `ENTITY_NAME_NOISE`, `ENTITY_DIM_LABELS`, `extractMakerNames`, `extractRecommendMeta`, `LuckyViewManager`, `HOME_TAB_DEFS`

---

## 1. 背景 (Context)

在 `v7.6.0` 发布与官方新一轮前端改版后，实测与用户反馈集中暴露出四类影响稳定性与一致性的跨模块问题：

1. **新版固定底部官方 App 推广条绕过旧拦截规则**：
   * 官方下发的新版底部悬浮推广条（含 `JAV.APP`、`官方 App`、`標籤追蹤、內建磁鏈搜尋、線上播放更順暢`、`下載 App` 等特征）更换了容器结构与类名，甚至在主文档挂载后再由前端脚本动态注入，导致既有 `.app-desktop-banner` / `.app-mobile-banner` 规则失效，横跨视口底部遮挡影片卡片与左下角 `#javdb-emby-toggle` 开关按钮。
2. **首页推荐未收起时切换到「手气不错」发生背景透底**：
   * 桌面端及窄屏媒体查询中 `html.emby-skin #recommend-container.emby-rec-rail:not(.emby-rec-collapsed):not(.emby-rec-hidden)` 的 CSS 选择器权重为 `(0, 4, 1)`，高于 `html.emby-skin.emby-fav-view #recommend-container` 的 `(0, 3, 1)`。当用户在首页未折叠「每周推荐」直接切入 `#emby-lucky-panel`（或其他 `.emby-fav-view` 全屏子面板）时，`display:flex !important` 反胜过全屏隐藏规则，导致半透明顶栏/背景下方透出推荐栏封面。
3. **`v7.6.0` 新增界面文案繁体混用**：
   * 「手氣不錯」「每週推薦」「換一批」「展開推薦」「原頁打開」「已載入」等新增 UI 文案直接沿用官方繁体词条，与插件既有的简体中文主控体系（主页、收藏夹、预览画廊、设置中心）视觉割裂；从 `h1[data-controller="recommend"]` 提取的官方标题与更新频次也未经本地化规整即直接上屏。
4. **繁体 / 简体 / 英文 / 日文多语言环境 DOM 解析碎片化**：
   * 在 [ADR-0018](./0018-multilingual-detail-metadata-resilient-parsing.md) 之后，随着业务扩张，不同模块各自维护了一套正则与关键词列表：`FAV.parseDetail`、`extractDetailData`、`extractMetaLinks`、`restructureDetail` 相关区隐藏（`hideSection`）、评分人数/分值解析、`review-buttons` / `.review-title` 观影状态识别、`buildAccountMenuHtml` 账户下拉提取、清单卡片影片数解析、演员性别探测与 `FavoriteActressManager` 噪声词过滤仍存在部分硬编码繁体/简中偏倚（如 `hideSection('還出演過')` 与 `extractSection` 未共用全语言正则、英文 `Director`/`Maker`/`4.62, by 1,280 users` 漏抓，以及混合正则末尾误用 `\b` 导致中文词尾紧接标点失效）。

---

## 2. 决策与架构设计 (Decision)

### 2.1 新版官方 App 推广条多维拦截与安全边界 (`hideJavdbAds` + CSS + `observeMutations`)
1. **扩展 CSS 属性通配覆盖**：新增 `nav/div/section/aside[class*="app-banner"]`、`[id*="app-banner"]`、`[class*="app-promo"]`、`[class*="app-download"]`、`[class*="download-app"]`、`[data-controller*="app-*"]` 与 `.emby-hide-ad` 通用隐藏类。
2. **结构 + 文案/外链启发式扫描**：在 `hideJavdbAds()` 内增加面向 `nav, section, aside, div, footer` 的紧凑容器探测（文本长度限 `<= 260`），联合判定 `官方 App / Official App`、`下載/下载/Download App`、`JAV.APP` 外链或特征文案，并向父级安全上溯至最外层广告包裹容器（体积增幅受限且严禁越过 `main`、`header`、`.section`、`.container`）。
3. **防误杀双端边界守护**：通过 `PROTECT_AD_ANCESTOR_SEL` 与 `PROTECT_AD_DESCENDANT_SEL` 显式保护 `#emby-*` 皮肤控件、`#javdb-emby-toggle`、`#footer`、`main`、`#recommend-container`、`.movie-list`、`.video-detail` 绝不被误隐藏。
4. **懒注入实时捕获**：`hideJavdbAds()` 内置 `250ms` 节流与路由变化重置逻辑，并在全局 `observeMutations()` 接收到非纯皮肤内部 DOM 变更（`!allInsideSkinChrome(muts)`）时触发兜底扫除。

### 2.2 全屏发现/收藏视图下推荐横栏的高特异性隔离
1. 将推荐横栏展开规则收窄为仅在非全屏模式生效：在桌面端及 `@media (max-width:700px)` 两处统一使用 `html.emby-skin:not(.emby-fav-view) #recommend-container.emby-rec-rail:not(.emby-rec-collapsed):not(.emby-rec-hidden)`。
2. 在 `.emby-fav-view` 隐藏区块追加高特异性压制规则 `html.emby-skin.emby-fav-view #recommend-container.emby-rec-rail` 与 `html.emby-skin.emby-fav-view #recommend-container.emby-rec-rail:not(.emby-rec-collapsed):not(.emby-rec-hidden) { display:none !important; }`，从优先级代数上彻底杜绝任何展开态/窄屏断点下的透底穿透。

### 2.3 全链路多语言解析统一助手与 CJK 词边界铁律
在顶层统一声明共享解析原语，消灭各模块重复且口径不一的多语言判断：
* **`DETAIL_FIELD_RES` 与相关区块常量**：集中维护 `code / date / duration / director / maker / series / rating / categories / actors` 以及 `DETAIL_ALSO_STARRED_RE`、`DETAIL_YOU_MIGHT_LIKE_RE`，由 `FAV.parseDetail`、`extractDetailData`、`extractMetaLinks` 与 `restructureDetail`（同时覆盖 `hideSection(...)` 与 `extractSection(...)`）共用。
* **`parseJavdbRatingText(rawText)`**：统一解析繁中（`4.50分, 由 120 人評價`）、简中（`由 890 人评价`）、英文（`4.62, by 1,280 users`、`4.30 / 5, by 77 ratings`）与日文评分及含千位分隔符的人数，同步服务 `FAV.parseListPage`、`FAV.parseDetail`、`extractDetailData` 与 `Top250ViewManager`。
* **`detectReviewTitleStatus(rawText)` & `classifyReviewActionEl(el)`**：统一识别 `.review-title` 及原生操作按钮（文本 + `data-action` / `href` / `id` / `className` 多特征）的 `wishlist` / `watched` / `list` 类型，由 `FAV.detectPageReviewStatus` 与 `extractReviewButtons` 共享。
* **CJK `\b` 词边界禁区**：在英文与中日文混排正则（如 `FAV.parseListLink` 的影片数提取与 `ENTITY_NAME_NOISE`）中，仅对 ASCII 英文单词分支施加 `\b` 词边界（`(?:部影片|部作品|(?:videos?|movies?|works?|films?)\b)`），严禁在 `\W` 中文字符末尾附加 `\b`（否则当中文直接邻接 `,` / `·` 等非单词字符时匹配必然静默失败）。
* **全链路消费者对齐**：同步补齐 `buildAccountMenuHtml`（`userMenuKeywords` 与 `isLogout` 修正 `sign\.out` 字面点号缺陷为 `sign[\s.-]*out`）、`FAV.detectActorGenders`（`\bactor\b` / `\bmale\b` 并排除 `actress`）、`FavoriteActressManager` `IGNORE_STRINGS` 与未登录判定、`FAV.throttledFetch` 英文登录页检测、`ENTITY_NAME_NOISE` / `ENTITY_DIM_LABELS` / `extractMakerNames` 与 `FAVUI.filterByDetail`。

### 2.4 皮肤 UI 文案全量简体中文规范化
* 统一将 `HOME_TAB_DEFS`、`.emby-drawer-lucky-link`、`LuckyViewManager`、快捷设置 (`#qs-home-rec-mode` / `#qs-lucky-tab`)、完整设置 (`#full-home-rec-mode` / `#full-lucky-tab`) 及 `#emby-rec-bar` 的交互文案规整为简体中文；
* `extractRecommendMeta()` 对 `#emby-rec-bar` 中的 `每週推薦` / `週一/四更新` 进行轻量简体规整，同时保持英文站点 `Weekly Picks` 等原生标题原样显示。

---

## 3. 结果与权衡 (Consequences)

* **正向收益**：
  * 多语言 DOM 解析收敛为单一共享助手族，新增语言变体或字段别名仅需修改一处常量/纯函数，降低简繁英日界面同步漂移风险；
  * 无类名/懒注入官方 App 推广条与 `.emby-fav-view` 推荐横栏透底均在结构与样式层级根治；
  * 皮肤交互文案回归 100% 简体中文一致性，且兼顾英文页面元数据完整采集。
* **验证边界与限制**：
  * 由于运行环境中安全策略限制浏览器子代理访问成人站域且 DevTools MCP 连接不可用，本决策的 DOM 与多语言解析验证基于多语言 JSDOM 仿真测试集（覆盖 `zh-TW` / `zh-CN` / `en` 与动态 DOM 注入场景）；若未来官方继续更迭全新 `App` 推广结构或英文页面文言，可在 `hideJavdbAds()` 与顶层共享助手处做最小增量扩展。
