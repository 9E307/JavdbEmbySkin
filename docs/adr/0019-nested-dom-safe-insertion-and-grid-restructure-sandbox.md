# 0019. 嵌套 DOM 跨层级安全插入与网格重构沙盒保护架构

## 状态
已接受 (Accepted)

## 上下文
在 JavdbEmbySkin 运行生命周期中，主页（`/`）排版重构是全功能初始化的核心枢纽。主页网格重构函数 `restructureGrid()` 负责：
1. 注入顶部 4 视图导航条（`buildHomeTabs()`：主页、收藏夹、画廊、TOP250）；
2. 遍历全页作品卡片 `.movie-list > .item`，注入 STEAM 风格卡牌高光层（`.emby-card-shine`）并挂载 3D 倾斜倾角（`attachCard3D()`）与悬停大图浮层（`attachDynamicFocus()` / `dfHoverStart()`）。

### 崩溃机理排查分析
在 JAVDB 官方近期进行的前端重构中，官方对其顶部分类导航条引入了 Stimulus 控制器容器，将原有的 Bulma 分类条 `<div class="tabs main-tabs is-boxed">` 包裹进了一层新容器 `<div class="main-tabs-wrap" data-controller="main-tabs">`。

此时，旧脚本中解析导航栏插入位置的代码产生严重的层级不匹配：
```javascript
const parent = movieListEl.parentNode; // parent 为 .container
const topAnchor = parent ? (parent.querySelector('.tabs, .main-tabs, .toolbar') || parent.firstElementChild || movieListEl) : movieListEl;
// topAnchor 匹配到了嵌套在 .main-tabs-wrap 内部的 .tabs
parent.insertBefore(tabsBar, topAnchor);
```

依据 W3C WHATWG DOM 标准规范，执行 `parent.insertBefore(newNode, refNode)` 时，`refNode` **必须是 `parent` 的直接子节点**（Direct Child）。由于 `.tabs` 的直接父级是 `.main-tabs-wrap` 而非 `parent`（`.container`），浏览器运行时立即抛出未捕获的严重异常：
```text
DOMException: Failed to execute 'insertBefore' on 'Node':
The node before which the new node is to be inserted is not a child of this node.
(The child can not be found in the parent.)
```

### 级联灾难后果
由于 `restructureGrid()` 中此前未对 `buildHomeTabs()` 实施异常隔离保护，这一异常导致了整条功能链路的中断与多重并发缺陷：
1. **主页常驻导航栏丢失**：`buildHomeTabs()` 在 `insertBefore` 行抛出异常并瞬间中断，`.emby-home-tabs` 根本未能挂载到 DOM 中，收藏夹、画廊与 TOP250 全屏面板也未创建；
2. **主页全卡片悬停封面大图完全失效**：`restructureGrid()` 中途崩溃，后续负责卡片初始化的 `items.forEach(...)` 循环被完全跳过，导致主页 40+ 张卡片**完全没有被赋予 `attachCard3D(box)` 互动层与悬停监听**，悬停封面大图处于死亡态；
3. **左上角 LOGO 滑出导航点击无法切换**：用户在主页滑出 LOGO 导航点击切换时，调用的 `ensureFavSurface()` 内部同样执行了 `ml.parentNode.insertBefore(tabsBar, topAnchor)`，同样抛出该 DOM 异常而崩溃，导致主页无法切换面板；
4. **详情页与子面板表面正常**：在作品详情页中由于不存在 `.movie-list`，`ensureFavSurface()` 降级为直接向 `document.body` 插入，未触发此 BUG，因此在详情页可以跳转；而在收藏夹/画廊/TOP250 面板内部，卡片由 `FAVUI` 或 `Top250ViewManager` 自行绑定 `attachCard3D`，因而表现正常。

---

## 决策
秉持 DOM 合规性、容错冗余与错误隔离原则，实施**三层防御性安全重构**：

1. **直接子节点递归溯源（保证 100% 符合 DOM 规范）**：
   引入 `resolveDirectChildAnchor(parent, target)` 递归溯源解析函数：无论传入的目标节点在 `parent` 内部嵌套有多深（如嵌套在 `.main-tabs-wrap` 内部），都会自动顺着 `parentNode` 向上溯源，直到找到它在 `parent` 下的**真正直接子节点容器**：
   ```javascript
   function resolveDirectChildAnchor(parent, target) {
     if (!parent || !target) return null;
     let cur = target;
     while (cur && cur.parentNode && cur.parentNode !== parent) {
       cur = cur.parentNode;
     }
     return (cur && cur.parentNode === parent) ? cur : null;
   }

   function findTopAnchor(parent, movieListEl) {
     if (!parent) return movieListEl;
     const candidate = parent.querySelector('.main-tabs-wrap, .tabs, .main-tabs, .toolbar');
     const directCandidate = resolveDirectChildAnchor(parent, candidate);
     return directCandidate || parent.firstElementChild || movieListEl;
   }
   ```

2. **多级平滑回退安全插入引擎（`safeInsertBefore`）**：
   将所有涉及外部容器插入的操作（`buildHomeTabs` 与 `ensureFavSurface`）统一委托给 `safeInsertBefore`：
   ```javascript
   function safeInsertBefore(parent, newChild, refChild, fallbackRef) {
     if (!parent || !newChild) return;
     const directRef = resolveDirectChildAnchor(parent, refChild);
     if (directRef && directRef !== newChild) {
       try { parent.insertBefore(newChild, directRef); return; } catch (e) {}
     }
     const directFallback = resolveDirectChildAnchor(parent, fallbackRef);
     if (directFallback && directFallback !== newChild) {
       try { parent.insertBefore(newChild, directFallback); return; } catch (e) {}
     }
     try { parent.appendChild(newChild); } catch (e) {}
   }
   ```
   * 优先在解析后的直接子节点前插入；
   * 若失败，自动回退到原生 `.movie-list` 节点前插入（`.movie-list` 必然是其父节点的直接子节点）；
   * 自带安全捕获，**绝对不会向外部抛出任何未捕获异常**。

3. **核心排版流程沙盒化隔离（Failure Containment）**：
   在 `restructureGrid()` 中将 `buildHomeTabs()` 包裹在独立的 `try ... catch` 沙盒中：
   ```javascript
   if (location.pathname === '/' || /^\/(\?|$)/.test(location.pathname + location.search)) {
     try {
       buildHomeTabs(movieLists[0]);
     } catch (e) {
       log('buildHomeTabs 异常: ' + e.message);
     }
   }
   ```
   **架构保证**：即使未来 JAVDB 官方标签栏发生任何极其诡异的结构突变，标签栏自身的任何异常也绝不会中断后续卡片的重构流程，**100% 保证主页所有卡片的 STEAM 3D 悬停、高光层以及悬停超清大图（`dfPreview`）永远稳固生效**。

---

## 后果

### 正向收益
* **根治 DOMException 崩溃**：彻底杜绝了因原生 DOM 嵌套层次变更引发的 `The child can not be found in the parent` 报错；
* **主页全局功能 100% 复苏**：顶部常驻 4 导航按钮恢复显示，左上角 LOGO 滑出导航随时随地顺畅跳转；
* **封面悬停大图 100% 稳固**：核心卡片网格重构与顶部 Tab 彻底解耦，无论外部环境如何变动，卡片悬停大图浮层永远按预期挂载；
* **健壮性与未来兼容性**：引入直接子节点溯源算法后，未来 JAVDB 无论在顶栏嵌套多少层容器，算法均能自动提升至当前父容器的直接子级。

### 潜在代价与防御
* 节点向上溯源至多遍历 3~5 层父节点（`.container` 之内层级较浅），时间复杂度为 $O(k)$（$k \le 5$），计算耗时不到 0.01ms，完全无性能损耗。

---

## 验证
* 使用真实的 JAVDB 在线抓取首页 HTML 快照（包含 `<div class="main-tabs-wrap">`）构建完整的 JSDOM 自动化回归套件 `test_suite_v7334.js`；
* 验证 5 项核心指标全部一次性通过：
  1. `.emby-home-tabs` 成功注入，包含全部 4 个 Tab 按钮；
  2. 40 张主页作品卡片 100% 获得 `data-grid-done="1"` 与 `.emby-card-shine`；
  3. LOGO 滑出导航点击 TOP250 顺利挂载并切换为 `emby-fav-view` 视图；
  4. 点击“主页”Tab 顺利退出 `emby-fav-view` 并还原主页网格；
  5. 鼠标悬停卡片顺利触发 `dfHoverStart` 并弹出 `#emby-df-preview` 大图浮层；
  6. 全流程控制台未记录任何未捕获的 DOM 异常。
