# ADR-0031: 主页「每週推薦」精选横栏隔离改造与 `/lucky`「手氣不錯」全景发现架构 (v7.6.0)

* **状态**: 已采纳 (Accepted)
* **日期**: 2026-11-12
* **关联版本**: `v7.6.0`
* **关联模块**: `applyHomeRecMode`, `extractRecommendMeta`, `enhanceCardItem`, `InfiniteScrollManager`, `LuckyViewManager`, `findTopAnchor`, `applyLayout`, `ensureLayoutClasses`, `removeEmbyDOM`, `ensureFavSurface`, `SET_KEYS`, `HOME_TAB_DEFS`, `SKIN_CHROME_SEL`

---

## 1. 背景 (Context)

JAVDB 官方针对首页 `/` 与探索能力进行了两项结构性改版：

1. **首页 `/` 新增「每週推薦(週一/四更新)」区块**：
   * 在第一页 `.main-tabs-wrap` 与主新片网格之前插入 `<h1 class="title is-size-5" data-controller="recommend" data-cookie-key="expand_key_XXX" data-container-id="recommend-container">` 及 `<div class="movie-list cols-5 mb-4" id="recommend-container">`。
   * 该容器同时带有 `.movie-list` 类，但内部卡片结构与主信息流存在三处关键差异：
     1. 内部 `<div class="item"><a>...</a></div>` 原生**缺少 `.box` 类**，导致皮肤面向 `.movie-list > .item > .box` 编写的封面圆角、内边距与 3D 悬停高光样式失效；
     2. 原生点击「收起到下次更新」仅对该期 `data-cookie-key` 写入 `style="display: none;"`，而皮肤全局 `html.emby-skin .movie-list { display:grid !important; }` 会直接击穿原生内联 `display:none`，强制展开 10 部推荐占满首屏并遮挡新片信息流；
     3. 同时存在多个 `.movie-list` 后，原先仅取 `document.querySelector('.movie-list')` 或遍历所有 `.movie-list` 的逻辑（瀑布流、动态聚焦、无限滚动 `InfiniteScrollManager` 抓下一页、首页导航栏锚点查找 `findTopAnchor`、无损退出 `removeEmbyDOM`）会出现错位甚至抓错容器问题。
2. **新增 `/lucky`「手氣不錯」路由**：
   * 结构为单组 `.movie-list.h.cols-4.vcols-8`（含 20 部随机影片，无分页 `.pagination`），底部配有 `<div class="has-text-centered my-5"><a class="button is-info" data-controller="lucky-shuffle" href="/lucky">換一批</a></div>`。
   * 由于无 `.pagination`，原 `InfiniteScrollManager` 会误将该页判为末页并在底部注入无效的「已经到底了」提示条；同时老用户难以从首页标签栏和顶栏一跳触发原生 `/lucky` 的随机探索流。

---

## 2. 决策与架构设计 (Decision — 方案 D)

经方案比对后采纳**方案 D（主页推荐横栏降噪 + `/lucky` 全屏发现双轨融合理想方案）**：

### 2.1 主页「每週推薦」与主信息流的严格解耦与隔离
1. **统一谓词识别**：引入 `isRecommendList(el)` 与 `isPanelInternalList(el)`，与既有 `applyLayout`、`ensureLayoutClasses`、瀑布流 `WaterfallEngine`、动态聚焦 `ensureDynamicFocusGrid` 联动，明确**排除** `#recommend-container` 参与任何瀑布流或动态聚焦全宽排布。
2. **可逆 `.box` 适配 (`enhanceCardItem`)**：仅对 `#recommend-container > .item > a:not(.box)` 动态补全 `.box` 并标记 `data-emby-added-box="1"`，绑定 `.emby-card-shine` 与 3D 倾角交互；关闭皮肤时 `removeEmbyDOM()` 精确识别该标记并还原原生 DOM。
3. **高特异性 CSS 与折叠持久化**：
   * 在 CSS 层面使用 `html.emby-skin #recommend-container.emby-rec-rail:not(.emby-rec-collapsed):not(.emby-rec-hidden)`（单行横向 Flex 滚动轨）与 `html.emby-skin #recommend-container.emby-rec-collapsed` / `.emby-rec-hidden` (`display:none !important`) 解决与 `.movie-list { display:grid !important }` 的优先级冲突。
   * 折叠态独立持久化至 `SET_KEYS.homeRecCollapsed` (`javdbEmbyHomeRecCollapsed`)，并配合 `REC_SEEN_COOKIE_KEY` (`javdbEmbyRecSeenCookie`) 记录已读 `data-cookie-key`。即便官方周更期数变化导致 `cookie-key` 轮换，仍保持用户的收起偏好并在胶囊头 Bar (`#emby-rec-bar`) 亮起 `NEW · 本期上新` 未读角标提醒，点击展开后自动消标并记录已读。
   * 提供三种展示策略 `SET_KEYS.homeRecMode`（`rail` 单行精选横栏 / `pill` 默认收起胶囊 / `hide` 彻底隐藏），在快捷设置 (`#qs-home-rec-mode`) 与完整设置 (`#full-home-rec-mode`) 面板实时双向同步，并接入导出/云端配置恢复管线。
4. **顶部锚点与原生浮层协调**：
   * `findTopAnchor(parent, movieListEl)` 优先锚定 `h1[data-controller="recommend"], #emby-rec-bar, #recommend-container`，确保 `.emby-home-tabs` 始终稳居页面最顶部；
   * `html.emby-skin.emby-fav-view` 下统一隐藏 `.site-announce`、`.main-tabs-wrap`、`h1[data-controller="recommend"]`、`#emby-rec-bar`、`#recommend-container` 与 `.has-text-centered.my-5`，避免毛玻璃/液态玻璃全屏子面板透出原生高亮公告条或标签条（同时显式豁免 `#emby-lucky-panel`、`#emby-favorites-panel` 与 `#emby-top250-panel` 内部网格与控制条）。

### 2.2 无限滚动精准选取 (`InfiniteScrollManager`)
* 通过 `findMainMovieList()` 过滤掉 `isPanelInternalList(l)` 与 `isRecommendList(l)`；在抓取下一页 HTML（若第一页或服务端子模板回传推荐容器）时，同样排除 `isRecommendList(l)` 再提取 `.item`。
* 当 `isLuckyPage()` 成立或文档中存在 `a[data-controller="lucky-shuffle"]` 时，跳过创建并清理 `#jhs-infinite-loader`，避免「已经到底了」误报。

### 2.3 `/lucky`「手氣不錯」全屏发现视图与原生入口增强 (`LuckyViewManager`)
1. **统一导航体系联动**：
   * 将 `lucky` 纳入 `HOME_TAB_DEFS`（位列 `home` 与 `favorites` 之间），通过 `isLuckyTabEnabled()` 与 `populateTabContainer()` / `refreshHomeTabsAndTopNav()` 保证顶部首页标签栏（`.emby-home-tabs`）与顶栏悬停导航（`.emby-topnav`）同步增减，并开放 `SET_KEYS.luckyTab`（`#qs-lucky-tab` / `#full-lucky-tab`）实时显隐开关（关闭当前激活面板时自动回落至 `home`）。
   * 同步在侧边抽屉站点导航 `buildSiteNavLinksHtml()` 新增 `<a class="nav-item emby-drawer-lucky-link" href="/lucky">` 并在 `/lucky` 路由自动高亮 `.sel`。
2. **原位 AJAX 抽卡面板 (`#emby-lucky-panel`)**：
   * 首次切入 `lucky` 标签或点击「換一批」按钮 (`#lucky-shuffle-btn`) 时，由 `LuckyViewManager.loadData(forceShuffle)` 通过 `gmHttp.get(location.origin + '/lucky?_t=' + Date.now())` 拉取最新随机列表，复用 `enhanceCardItem`、图床地址重写、`referrerPolicy="no-referrer"`、`.efav-bg` 随机虚化背景及三种布局模式，并监听 `efav-randombg-change` 实时刷新背景。
   * 同步美化原生 `/lucky` 页面底部 `a.button[data-controller="lucky-shuffle"]`，并在 Emby / Glass / Liquid 样式与 `LiquidGlass.TARGETS` 引擎下保持统一质感。

---

## 3. 结果与权衡 (Consequences)

* **正向收益**：
  * 首页新片网格 100% 夺回首屏主导权，周更推荐栏降噪为优雅的单行精选轨，新期上新提醒与折叠习惯互不干扰；
  * 无需离开首页即可原位高频「換一批」发现随机作品，与收藏夹、预览画廊、TOP250 共享统一的沉浸式面板与动态背景体系；
  * 对原生 DOM 保持可逆改造，一键关闭皮肤或切换页面时无任何残留污染。
* **维护约束**：
  * 任何在全局扫描 `.movie-list` 的新模块必须优先调用 `isRecommendList(el)` 与 `isPanelInternalList(el)` 进行容器隔离（见 `CONTEXT.md` Golden Rule 34）。
