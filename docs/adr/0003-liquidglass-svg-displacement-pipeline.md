# 0003. LiquidGlass 液态玻璃 2.0 物理透镜折射流水线与 Chromium 合成层防御架构

## 状态
已接受 (Accepted)

## 上下文 (Context)

为了赋予界面具有高级拟物感与流体折射质感的现代界面，脚本引入了“液态玻璃（LiquidGlass）”风格。在早期的原型与第三方参考实现中，常采用 HTML5 Canvas 绘制高斯模糊与色彩采样，或通过 Canvas 像素运算生成高光与折射图层。但在真实网络环境中，这导致了频繁的崩溃。

在现代 Web UI 与多媒体界面设计中，“毛玻璃（Frosted Glass）”与“液态玻璃（Liquid Glass）”有着截然不同的物理光学原理与交互体验：
* **普通毛玻璃（Frosted Glass）的局限**：
  仅采用单调的高斯模糊（`backdrop-filter: blur(20px)`），本质上是将底层光线做各向同性无序漫反射（Lambertian Diffusion），把画面抹成一块雾蒙蒙的平面毛毡。这会导致底层精美的影片封面、动态聚焦光晕被彻底吞噬，界面沉闷厚重，文字浮在雾气上方易产生眩光。
* **液态玻璃 2.0（Liquid Glass 2.0）的极致追求**：
  深度借鉴 **Apple visionOS 材质规范** 与 **childrentime/liquid-glass 物理透镜算法**，将其构建为具有光学物理厚度的真实水晶凸透镜：
  1. **中心零失真、100% 锐利保真**：透镜中央主视野区域的光学偏折为 0，底层画面细节清晰呈现，绝非糊成一片；
  2. **边缘向心折射（Edge Refraction Bevel）**：光线在穿过晶体倒角边缘时发生物理偏折，呈现逼真的三棱镜微位移；
  3. **镜面高光（Specular Highlights）与环境光泽**：多层边缘内阴影与顶部极细渐变流光，塑造晶莹透亮的水晶质感；
  4. **高对比度无障碍文字系统**：彻底解决晶莹通透透镜在强光浅色背景下的眩光看不清问题。

该效果是经由大量极端场景调试与用户高度共创打磨出的核心视觉资产，后续维护与接手者必须严格遵循本架构准则，**严禁将其回退为简陋的单层 CSS 模糊**。

---

## 决策 (Decision)

我们决定**完全舍弃 Canvas 像素运算方案，采用纯 SVG `<feDisplacementMap>` + `<feImage>` 矢量着色器流水线**来实现液态玻璃渲染。

---

## 核心原因与权衡 (Core Rationales & Trade-offs)

1. **彻底规避 Canvas CORS 污染（Tainted Canvas）崩溃**：
   JAVDB 的封面图、Logo 和演员头像均托管在第三方 CDN 节点（如 `c0.jdbstatic.com`）。在 HTML5 Canvas 中绘制跨域图像时，只要对方没有配置完美的 CORS 头，调用 `ctx.getImageData()` 会立即被浏览器安全机制拦截，抛出致命的 `DOMException: Failed to execute 'getImageData' on 'CanvasRenderingContext2D': The canvas has been tainted by cross-origin data.`，直接导致详情页脚本终止、页面变白。
2. **GPU 硬件级加速与零内存泄漏**：
   SVG 滤镜是由浏览器渲染引擎在合成层（Compositing Layer）直接交由 GPU 片元着色器执行的，无需在 JavaScript 主线程维持巨大的像素 `Uint8ClampedArray` 数组，从根本上杜绝了在快速滚动卡片时产生的内存飙升与垃圾回收（GC Pause）卡顿。
3. **分辨率自适应与无锯齿**：
   基于 SVG 矢量的位移图滤镜能够天然适应 2K/4K 等高分辨 Retina 显示，边沿不会产生 Canvas 缩放带来的像素锯齿。

---

## 核心技术实现要素 (Six Core Architectural Pillars)

### 1. 2D 物理有向距离场 (RoundedBoxSDF) 与表面法线差分求导
真实晶体的倒角并非生硬的二维描边，而是连续曲率的曲面。
* **RoundedBoxSDF 算法**：
  给定容器宽高与实际边框圆角 $r$，计算任意点 $(x, y)$ 到圆角矩形物理轮廓的精确欧几里得距离 $d$：
  $$q_x = |x| - \frac{W}{2} + r, \quad q_y = |y| - \frac{H}{2} + r$$
  $$SDF(x, y) = \min(\max(q_x, q_y), 0) + \sqrt{\max(q_x, 0)^2 + \max(q_y, 0)^2} - r$$
* **表面法向量求导（Surface Normal）**：
  在极小邻域 $\epsilon = 0.5\text{px}$ 内采用中心差分算子对 SDF 进行梯度差分：
  $$n_x = SDF(x + \epsilon, y) - SDF(x - \epsilon, y)$$
  $$n_y = SDF(x, y + \epsilon) - SDF(x, y - \epsilon)$$
  归一化后获得三维凸透镜曲面在此处精确的表面法向量 $\mathbf{n} = (n_x, n_y)$。

### 2. $smoothStep$ 三次非线性缓动与“中心零失真、边缘向心偏折”
* **倒角带宽度（Bevel Width）**：
  - 药丸按钮与小控件：自适应取 $6\text{px} \sim 14\text{px}$；
  - 大面板与全屏抽屉：自适应取 $16\text{px} \sim 28\text{px}$。
* **非线性折射缓动**：
  仅在物理倒角带 $d \in [-bevel, 0]$ 内激活偏折计算；在内部核心区（$d < -bevel$），位移恒为零！
  令相对归一化深度 $t = \frac{d + bevel}{bevel} \in [0, 1]$，应用 Hermite 三次插值：
  $$sc = t^2(3 - 2t)$$
  光线穿透凸透镜边缘时向内汇聚折射，位移方向严格为表面法线负方向 $-\mathbf{n}$。

### 3. Canvas 离线编码 + SVG `<feDisplacementMap>` 硬件片元着色器执行
* **避免主线程重度计算**：
  HTML5 Canvas **仅在初始化与容器尺寸改变（resize）时**用于离线烘焙一张轻量级（$240 \times 240$ 级别降采样，单次执行耗时 $< 1\text{ms}$）的位移图（Displacement Map），并以 Data URI 注入 SVG `<feImage>`。
* **中位灰度偏移基准**：
  - 默认无位移基底：$R = 128, G = 128, B = 0, A = 255$；
  - 边缘向心位移编码：
    $$R = \text{clamp}\left(0, 255, \operatorname{round}\left((-n_x \cdot sc \cdot 0.5 + 0.5) \times 255\right)\right)$$
    $$G = \text{clamp}\left(0, 255, \operatorname{round}\left((-n_y \cdot sc \cdot 0.5 + 0.5) \times 255\right)\right)$$
* **GPU Compositing 合成流水线**：
  ```html
  <filter id="lg-xxx" filterUnits="userSpaceOnUse" color-interpolation-filters="sRGB">
    <feImage id="lg-xxx_map" href="data:image/png;base64,..." preserveAspectRatio="none" />
    <feDisplacementMap in="SourceGraphic" in2="lg-xxx_map" xChannelSelector="R" yChannelSelector="G" scale="..." />
  </filter>
  ```
  实际的光学逐像素搬家直接由浏览器的 GPU 合成层（Compositing Layer）并行片元着色器完成，JavaScript 主线程零负担。

### 4. 分层模糊与视差阶梯（Blur Hierarchy）
液态玻璃绝不可“一刀切”施加相同的模糊度，必须根据构件的物理功能建立严格的模糊阶梯：
1. **清透水滴层（顶部导航栏、紧凑条、圆形操作按钮）**：
   `blur(0.25px) brightness(1.04) saturate(1.08)`
   仅采用 0.25px 亚像素抗锯齿微平滑，消灭折射边缘的走样与摩尔纹，底层大封面与流动画廊 100% 锐利透出；
2. **半透晶体层（侧边导航抽屉、快捷悬浮窗）**：
   `blur(1.6px) brightness(1.04) saturate(1.15)`
   微度柔化背景干扰，保证常驻交互图标清晰识别；
3. **沉浸重晶体层（数据设置大窗、筛选弹窗、多源播放矩阵）**：
   `blur(8px)` ~ `blur(14px)`
   配合暗度底色，沉淀复杂表单文字。

### 5. 双层高对比度防眩文字阴影（Apple visionOS 规范）
透明与折射晶体最致命的问题是文字遇浅色底图时“眩光失焦看不清”。
本方案严格落实 visionOS 规范，对透镜上的一切文字应用**局部双层高对比度阴影**：
```css
text-shadow: 0 1px 2px rgba(0,0,0,.95), 0 0 8px rgba(0,0,0,.7);
```
外层 8px 大模糊黑色气晕压暗眩光背景，内层 1px 锐利投影勾勒字形，在任何高动态对比画面上均能实现 AAA 级可读性。

### 6. 多层镜面高光 (Specular) 与环境光线反射
* **内外双向倒角反光（Inset Specular Box-Shadow）**：
  ```css
  box-shadow:
    inset 0 1.5px 0 rgba(255, 255, 255, 0.65), /* 顶部天光反射高光 */
    inset 0 -1px 0 rgba(255, 255, 255, 0.12),  /* 底部地面暗反光 */
    0 18px 48px rgba(0, 0, 0, 0.45);           /* 悬浮柔和投影 */
  ```
* **顶部主光源亮条**：
  在 `#emby-topbar::after`、`.jhs-qs-popover::before` 等元素上方注入 2px 细流光线：
  ```css
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.9), transparent);
  ```

---

## 历史惨痛教训与五大避坑铁律 (Critical Invariants & Gotchas)

### 铁律 1：Chromium 的 Backdrop-Filter GPU 缓存冻结与微抖动破除机制（Micro-Saturate Jitter）
* **陷阱现象**：
  在页面向下滚动浏览时，背景卡片在滑动，但顶栏透镜里的画面**完全静止、冻结在首屏**，滑过的海报在透镜里残留难看的重影与残相。
* **底层根因**：
  Chromium 内核对 `backdrop-filter: url(#id)` 施加了极端激进的位图缓存策略。只要该 CSS 属性字符串未改变，合成器线程就认为宿主元素下方的视口不需要重新执行 SVG 滤镜片元着色器。
* **架构解法（Micro-Saturate Jitter）**：
  在 `window.onScrollInvalidate` 流水线中，以 50ms 节流频率向滤镜的 `saturate()` 参数注入**万分之五**的微小浮点抖动：
  ```javascript
  const sat = (1.08 + tick).toFixed(4); // 例如 1.0800 与 1.0805 之间交替
  const bp = 'url(#' + entry.filterId + ') blur(0.25px) brightness(1.04) saturate(' + sat + ')';
  entry.el.style.setProperty('backdrop-filter', bp, 'important');
  ```
  这一微小变动对人眼完全不可见，但足以迫使 Chromium 渲染管线判定样式失效并击穿 GPU 静态缓存，实时以 120fps 重绘底层滚动的影片画面！滚动停止 120ms 后由 `scrollEndTimer` 优雅复位至基准值。

### 铁律 2：严禁给液态玻璃容器添加 `contain: strict` 或 `contain: paint`
* **陷阱现象**：
  容器瞬间失去通透折射效果，背景变成纯黑色或纯灰色实色板。
* **底层根因**：
  CSS Containment 规范要求隔离渲染树。`contain: strict` 或 `contain: paint` 会强行截断浏览器合成层对祖先视口的采样通道，导致 `backdrop-filter` 无法获取其后方的 DOM 像素。
* **架构解法**：
  顶栏等容器必须严格保持：
  ```css
  contain: none !important;
  will-change: backdrop-filter;
  transform: translateZ(0);
  ```

### 铁律 3：绝不可在 JS 主线程用 Canvas 逐像素运算渲染动态背景
* **陷阱现象 1（致命白屏）**：
  JAVDB 图片托管在第三方 CDN，调用 `ctx.getImageData()` 触发 CORS 污染（Tainted Canvas），抛出不可捕获的 `DOMException`，整页白屏崩溃。
* **陷阱现象 2（滚动雪崩）**：
  主线程逐像素读写 `Uint8ClampedArray` 带来巨大的内存垃圾，频繁触发垃圾回收（GC Pause），滚屏帧率掉至个位数。
* **架构解法**：
  Canvas 仅用于离线静态生成轻量级位移图，全部动态折射完全交付 SVG `<feDisplacementMap>` 由 GPU 执行。

### 铁律 4：物理振幅安全限幅（Physical Clamping）防撕裂穿模
* **陷阱现象**：
  用户调节折射滑块到较大值时，边缘图像发生剧烈拉伸甚至“穿模”露底。
* **架构解法**：
  振幅系数 `liquidAmp` 严格限制在 `[0, 0.44]` 安全物理区间。
  并在计算 scale 时按构件几何尺度分别限幅：
  - 大面板：`finalScale = Math.min(12 + liquidAmp * 65, Math.max(12, minDim * 0.65))`
  - 小按钮：`finalScale = Math.min(6 + liquidAmp * 35, Math.max(6, minDim * 0.95))`

### 铁律 5：风格切换（Theme Switching）资源熔断与过渡暗幕
* **架构解法**：
  当用户切换到毛玻璃（Glass）或经典 Emby 时，必须调用 `LiquidGlass.clear()`：
  1. 彻底从 DOM 中 `removeChild` 移除注入的所有隐藏 SVG、`<filter>`、`<feImage>` 节点；
  2. 释放 Canvas 引用；
  3. 彻底断开 `MutationObserver` 监听；
  4. 注入 1 秒的 `#efav-style-morph` 柔和渐变过渡暗幕，平滑掩盖 GPU 合成层管线重建期间的闪烁。

---

## 结果与长远影响 (Consequences)

* **收益**：0 次跨域安全异常、120fps 丝滑滚屏体验、高分屏完美抗锯齿。在 JAVDB 全网实现了媲美原生桌面应用的流体折射晶莹质感；
* **代价与降级保护**：老旧浏览器或禁用了硬件加速的特殊环境可能无法完全呈现折射效果，因此系统设计了回退到 CSS `backdrop-filter: blur(...)`（毛玻璃）的降级检测机制；
* **不可退化性**：为后续开发者与 AI 树立了坚实的架构壁垒，防止液态玻璃被误改为平庸的高斯模糊。
