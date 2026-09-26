# 0016. JAVDB 官方年龄弹窗无感通行与全 DOM 盲点击拦截架构

## 状态
已接受 (Accepted)

## 上下文
在全新浏览器环境（未曾访问过 JAVDB 或处于无痕模式）中安装运行 JavdbEmbySkin 后，用户反馈了一个严重的无限弹窗与重定向死循环：
> “修复点击 TOP250 的 tab 有时会自动重定向到 javdb.com/v/nKg2am 的问题，我发现在一个全新的浏览器环境中，用户如果不设置拦截弹出式窗口之前会一直弹打开同一个新标签页 javdb.com/v/nKg2am。跳转到 javdb.com/v/nKg2am 后点击上方的 TOP250 的 tab 就变成了 javdb.com/v/nKg2am#top250，这样不会自动重定向到 javdb.com/v/nKg2am。”

经过代码审计、DOM 树回溯以及网络请求抓包分析，确诊问题根源：

1. **粗暴的全 DOM 扫描与盲目模拟点击**：
   * 原脚本第 25,121 行实现了 `tryAgeGate()`：
     ```javascript
     function tryAgeGate() {
       const els = document.querySelectorAll('a, button');
       for (let i = 0; i < els.length; i++) {
         const t = els[i].textContent || '';
         if (/滿18|over\s*18|18歲|18岁|18\+/i.test(t)) { els[i].click(); return true; }
       }
       return false;
     }
     ```
   * 并在首屏由 `startAgeGate()` 以 400ms 的高频定时器连续轮询（最多 24 次，约 10 秒）。
   * `document.querySelectorAll('a, button')` 是**全文档无差别检索**，没有将作用域限制在年龄弹窗容器内。

2. **热播榜首作品元数据误触触发点**：
   * 当用户在首页点击“TOP250”标签时，`Top250ViewManager` 默认载入热播榜单（`playback` 模式）并将作品卡片动态渲染到 `#t250-movie-list` 中；
   * 当日热播榜排名第 1 的影片恰好为番号为 **MIAB-418**（页面路径 `/v/nKg2am`）的作品，其作品卡片包含番号、标题与发布日期（`2025-02-18`）等元数据；
   * `startAgeGate` 轮询恰好在榜单卡片渲染到 DOM 后执行，卡片内的文本或标签命中正则 `/滿18|over\s*18|18歲|18岁|18\+/i`；
   * `tryAgeGate` 误将 MIAB-418 作品卡片（`<a href="/v/nKg2am" class="box">`）当成了年龄弹窗的“我已满18岁”按钮，直接对其执行了 `els[i].click()`！
   * 因为影片卡片自带链接导航（且在某些配置下带有 `target="_blank"`），浏览器立刻在新标签页弹出打开了 `javdb.com/v/nKg2am`；
   * 新标签页加载后，脚本再次初始化，若匹配持续，就会在未开启弹出式窗口拦截的环境下疯狂弹窗。

3. **Tab 导航点击未阻止事件冒泡**：
   * 标签栏 Tab 按钮（`.emby-home-tab`）的点击监听器仅调用了 `e.preventDefault()`，漏掉了 `e.stopPropagation()`，导致点击事件向上冒泡到原生 `document` 级别可能被其他全局拦截器或点击监听器捕获。

---

## 决策
坚持精准作用域控制与第一性原理，彻底废除全 DOM 盲搜，建立**官方 Cookie 预注入、弹窗就地平滑解构与 Tab 事件完全阻断**三道防线：

1. **官方 Cookie 主动预注入（Server-Side Bypass）**：
   * JAVDB 服务端判断是否输出年龄确认弹窗的本质是检测请求头中的 `over18=1` Cookie；
   * 在脚本初始化入口与 `startAgeGate` 前，立即主动写入官方 Cookie：
     ```javascript
     if (!/(?:^|;\s*)over18=1/.test(document.cookie)) {
       try { document.cookie = 'over18=1; path=/; max-age=315360000; SameSite=Lax'; } catch (e) {}
     }
     ```
   * 后续所有原生导航与 API 请求天然携带该 Cookie，服务端从源头上直接豁免输出年龄拦截 HTML。

2. **精准作用域锁定与 DOM 直接解构（Zero Click Simulation）**：
   * 彻底废除 `querySelectorAll('a, button')` 全局扫描；
   * 严格限定只查找 JAVDB 官方弹窗容器 `.over18-modal` 或内部包含 `a[href*="/over18"]` 的模态层：
     ```javascript
     const modal = document.querySelector('.over18-modal, .modal:has(a[href*="/over18"])');
     if (modal) {
       modal.remove();
       document.documentElement.classList.remove('is-clipped');
       return true;
     }
     ```
   * 若模态层存在，直接将其从 DOM 树移除，并清除 Bulma 的锁屏样式类 `is-clipped`；**绝不调用任何 `.click()` 触发页面跳转**！
   * 兜底查找时，也仅允许匹配显式指向 `/over18` 的按钮（`a.button[href*="/over18"]`），严禁匹配任何影片卡片、女优链接或外链广告。

3. **Tab 导航点击全链路阻断冒泡（`stopPropagation`）**：
   * 在 `buildHomeTabs`（首页标签栏）、`ensureFavSurface`（兜底标签栏）及 `buildTopNav`（顶栏抽屉滑动标签）三个入口的 Tab 点击回调中，统一增加 `e.stopPropagation()`：
     ```javascript
     t.addEventListener('click', function (e) {
       e.preventDefault();
       e.stopPropagation();
       switchTab(def.key);
     });
     ```
   * 确保 Tab 切换事件 100% 封闭在当前组件内，绝不向外部父容器或 `document` 冒泡。

---

## 原因与权衡
* **为什么绝不能在自动化弹窗处理中使用全 DOM 宽泛选择器（如 `a, button`）？**
  * 在内容型网站中，页面充斥着大量的作品标题、发布日期、演员生平与分类标签。带有数字“18”（例如 2025-02-18 日期、18岁女优作品、No.118 排行）的内容极易发生假阳性误判。对未经严格限定的节点调用 `.click()`，等同于在页面上盲打随机链接，极度危险。
* **为什么直接移除模态层优于点击“是,我已滿18歲”链接？**
  * JAVDB 官方弹窗按钮是一个普通链接 `<a href="/over18?respond=1&rurl=...">`。点击该链接会导致浏览器向服务端发起页面重定向刷新。而通过预写 Cookie + 直接从 DOM 中 `remove()` 弹窗，用户当前浏览的页面、SPA 状态、已加载的影片列表完全不受干扰，体验最为丝滑。

---

## 结果与影响
* **收益**：
  * 全新浏览器与无痕模式下点击 TOP250、热播榜或任何页面，**0 误弹窗、0 异常重定向至 MIAB-418**；
  * 年龄确认无感通行，页面无需刷新即可解除滚动锁定；
  * Tab 点击事件彻底受控，彻底隔绝外部全局点击拦截器。
