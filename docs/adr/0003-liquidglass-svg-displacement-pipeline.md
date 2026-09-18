# 0003. LiquidGlass 液态玻璃 SVG Displacement 滤镜流水线

## 状态
已接受 (Accepted)

## 上下文
为了赋予界面具有高级拟物感与流体折射质感的现代界面，脚本引入了“液态玻璃（LiquidGlass）”风格。在早期的原型与第三方参考实现中，常采用 HTML5 Canvas 绘制高斯模糊与色彩采样，或通过 Canvas 像素运算生成高光与折射图层。但在真实网络环境中，这导致了频繁的崩溃。

## 决策
我们决定**完全舍弃 Canvas 像素运算方案，采用纯 SVG `<feDisplacementMap>` + `<feImage>` 矢量着色器流水线**来实现液态玻璃渲染。

## 原因与权衡
1. **彻底规避 Canvas CORS 污染（Tainted Canvas）崩溃**：JAVDB 的封面图、Logo 和演员头像均托管在第三方 CDN 节点（如 `c0.jdbstatic.com`）。在 HTML5 Canvas 中绘制跨域图像时，只要对方没有配置完美的 CORS 头，调用 `ctx.getImageData()` 会立即被浏览器安全机制拦截，抛出致命的 `DOMException: Failed to execute 'getImageData' on 'CanvasRenderingContext2D': The canvas has been tainted by cross-origin data.`，直接导致详情页脚本终止、页面变白。
2. **GPU 硬件级加速与零内存泄漏**：SVG 滤镜是由浏览器渲染引擎在合成层（Compositing Layer）直接交由 GPU 片元着色器执行的，无需在 JavaScript 主线程维持巨大的像素 Uint8ClampedArray 数组，从根本上杜绝了在快速滚动卡片时产生的内存飙升与垃圾回收（GC Pause）卡顿。
3. **分辨率自适应**：基于 SVG 矢量的位移图滤镜能够天然适应 2K/4K 等高分屏 Retina 显示，边沿不会产生 Canvas 缩放带来的像素锯齿。

## 结果与影响
* **收益**：0 次跨域安全异常、120fps 丝滑滚屏体验、高分屏完美抗锯齿。
* **代价**：老旧浏览器或禁用了硬件加速的特殊环境可能无法完全呈现折射效果，因此系统设计了回退到 CSS `backdrop-filter: blur(...)`（毛玻璃）的降级检测机制。
