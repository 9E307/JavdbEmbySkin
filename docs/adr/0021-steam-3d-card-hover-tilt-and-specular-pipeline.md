# 0021. STEAM 风格卡牌 3D 物理倾斜与全息高光渲染流水线

## 状态
已接受 (Accepted)

## 上下文与交互哲学 (Context & Design Philosophy)

在传统的 Web 多媒体与影视信息展示界面中，电影海报卡片的悬停（Hover）交互通常非常简陋单调：
* **常见 2D 平移方案的平庸性**：
  绝大多数网站仅采用简单的 `transform: translateY(-4px); box-shadow: 0 8px 16px rgba(0,0,0,.3);`。这种二维纸片式位移缺乏物理实体感、机械分量感与互动回馈，难以激发用户探索海量影片的兴趣；
* **传统 3D 方案的破坏性陷阱**：
  若简单引入大型 3D 库（如 Three.js）或生硬地套用 CSS 3D，极易产生：
  1. **布局穿模与视口裁切**：卡片放大后撞入顶栏或左侧屏幕边缘被切掉半截；
  2. **主线程高频重排（Layout Thrashing）**：在 `mousemove` 监听中未加帧率锁直接修改 DOM style，引发剧烈卡顿掉帧；
  3. **多维布局兼容断裂**：标准 CSS Grid、瀑布流（Masonry column-count）与动态聚焦（Dynamic Focus）对 `transform` 的包含块（Containing Block）判定截然不同，容易导致定位错乱。

为此，JavdbEmbySkin 独立构建了**轻量级、零依赖、高性能的 STEAM 风格卡牌 3D 物理倾斜与全息流光渲染流水线 (`attachCard3D`)**。

---

## 核心实现机制 (Architectural Pillars)

### 1. 相对归一化坐标空间与透视微倾斜运算
当光标划过影片卡片时，算法在 $[0, 1] \times [0, 1]$ 归一化局部几何空间内结算三维旋转角：
* **空间归一化**：
  $$p_x = \frac{e.clientX - rect.left}{rect.width} \in [0, 1]$$
  $$p_y = \frac{e.clientY - rect.top}{rect.height} \in [0, 1]$$
* **四元平衡偏航与俯仰计算**：
  设最大倾角 $TILT\_MAX$（通常为 $8^\circ \sim 12^\circ$）：
  $$r_y = (p_x - 0.5) \times (2 \times TILT\_MAX)$$
  $$r_x = (0.5 - p_y) \times (2 \times TILT\_MAX)$$
* **硬件级 3D 矩阵合成**：
  应用 `perspective(700px)` 构建符合人眼近大远小生理规律的三维景深空间，结合 `scale(1.05)` 形成向外跃出的悬浮视觉冲击：
  ```javascript
  box.style.transform = `perspective(700px) rotateX(${rx.toFixed(2)}deg) rotateY(${ry.toFixed(2)}deg) scale(1.05)`;
  ```

### 2. `--mx` / `--my` 全息动态流光层 (`.emby-card-shine`)
为了模拟物理实体收藏卡片（如宝可梦闪卡或 Steam 交易集换卡）的镭射与全息反光质感，系统为每个卡片 DOM 节点注入专用的流光层：
* **CSS 变量动态注入**：
  光标移动时，将归一化百分比坐标写入卡片的 CSS Custom Properties：
  ```javascript
  box.style.setProperty('--mx', (px * 100).toFixed(1) + '%');
  box.style.setProperty('--my', (py * 100).toFixed(1) + '%');
  ```
* **全息径向渐变着色**：
  `.emby-card-shine` 依托 CSS 变量实时改变高光聚焦中心点，光芒随光标在海报表面灵动流转，赋予静止的海报鲜活的物理生命力。

### 3. RAF 帧率锁与零垃圾回收流水线
`mousemove` 事件在现代高刷电竞鼠标上可达到 500Hz ~ 1000Hz 的触发频率。若每次触发均操作 DOM，会导致严重的掉帧。
* **单帧事务节流（requestAnimationFrame Lock）**：
  在触发回调时，先执行 `if (raf) cancelAnimationFrame(raf);`，将计算结果推入下一个屏幕刷新垂直同步信号（VSync，120Hz/144Hz）周期统一绘制；
* **即时释放与物理弹簧回正**：
  光标移出（`mouseleave`）时，立即取消未执行的 RAF，将 `box.style.transform` 恢复为空字符串，利用 CSS `transition: transform .25s ease-out` 实现丝滑的物理弹簧归位阻尼感。

---

## 历史惨痛教训与四大避坑铁律 (Critical Invariants & Gotchas)

### 铁律 1：`transform-origin: top left` 黄金锚点决策
* **致命陷阱**：
  最初采用浏览器默认的 `transform-origin: center center`（中心锚点）。当页面第一列的卡片放大 1.05 倍并向左倾斜时，卡片左边缘会**直接超出屏幕左侧视口边界**被永久截断；当顶部第一行的卡片向上倾斜时，会直接撞入上方的固定导航栏，造成难看的视觉穿模。
* **架构解法**：
  全局强制设定卡片锚点为左上角：
  ```css
  .movie-list .item .box {
    transform-origin: top left;
  }
  ```
  卡片受到悬停激活时，物理缩放与倾斜**严格向右侧与下方延展生长**，彻底消除视口左缘截断与顶栏遮挡风险！

### 铁律 2：`dataset.tilt` 幂等防重守卫
* **致命陷阱**：
  在无限滚动分页、分类标签切换、收藏夹重新挂载或 DOM Mutation 发生时，卡片网格会频繁调用 `restructureGrid` 与 `attachCard3D`。若无防护，同一张卡片会被重复绑定几十次事件监听器，导致内存泄漏与鼠标移动时动画抖动撕裂。
* **架构解法**：
  在 `attachCard3D` 头部注入原子化幂等标记：
  ```javascript
  if (!box || box.dataset.tilt) return;
  box.dataset.tilt = '1';
  ```
  严格保证任意 DOM 节点在其生命周期内仅挂载一次 3D 倾斜监听。

### 铁律 3：`noPreview` 场景解耦机制
* **致命陷阱**：
  卡片 3D 倾斜机制同时被复用于：
  1. 首页与收藏夹普通海报卡片（需触发悬停大图封面预览气泡 `dfHoverStart`）；
  2. 详情页右侧的金标荣誉卡片（`.emby-trophy-card`，只需 3D 流光，绝不能弹封面预览）；
  3. 详情页底部“相关推荐影片”横排滑动条（纯展示推荐，弹出大图会阻挡阅读）。
* **架构解法**：
  在 `attachCard3D(box, opts)` 中提供显式解耦参数 `{ noPreview: true }`，在保留极致 3D 物理倾斜手感的同时，精确熔断不合时宜的封面弹窗逻辑。

### 铁律 4：三大核心布局（Grid / Masonry / Dynamic Focus）统一兼容
* **架构解法**：
  * **标准 CSS Grid 布局**：依靠 `preserve-3d` 与 `minmax` 维持网格单元物理间距；
  * **瀑布流（Masonry）**：在多列断点下通过 `break-inside: avoid` 防止倾斜时列间穿模；
  * **动态聚焦（Dynamic Focus）**：自 v7.115 起打破历史隔离，移出 `.emby-card-shine` 隐藏名单，使聚焦模式同样畅享 3D 镭射流光。

---

## 结果与长远影响 (Consequences)
* **游戏级的桌面交互质感**：让网页端 JAVDB 拥有了媲美 Steam、PS5 游戏库卡牌的高级物理重量感；
* **极度稳定的帧率表现**：在高分辨率视网膜屏与 144Hz 高刷屏下均能保持稳帧运行；
* **通用复用性**：统一赋能了普通卡片、紧凑条小封面与金标荣誉卡，构筑了高一致性的视觉语言。
