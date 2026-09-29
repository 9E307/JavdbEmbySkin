# 0022. 预览图嗅探矩阵与跨域原图清洗流水线

## 状态
已接受 (Accepted)

## 上下文与痛点 (Context & Problem Statement)

JAVDB 官方在部分影片详情页中并不提供分段预览剧照（Sample Images），或者剧照清晰度受限于早期低分辨率切图；同时在列表页、收藏夹与动态聚焦浏览时，用户需要快速查阅多张剧照图集以评估影片内容。为此，脚本内置了 `PreviewSys` 预览图嗅探与多源图集提取服务。

然而，抓取第三方番号预览图面临严峻的工程挑战：
1. **第三方站点反爬与 CORS 限制**：浏览器常规 `fetch` / `XMLHttpRequest` 无法直接跨域访问第三方站点；
2. **番号模糊检索导致“张冠李戴”**：许多第三方站点的站内搜索返回模糊匹配（如搜索 `SSIS-001` 返回几十条 `SSIS` 结果），若直接抓取搜索列表第 1 条，极高概率抓取到无关影片的剧照；
3. **站点前端模板拼接缺陷**：部分站点（如 ProjectJav）存在恶性的 URL 拼接 Bug（如把自身图床域名和外部 CDN 链接生硬拼接为 `https://images.projectjav.comhttps://cdn.hpify.com/...`），直接访问必定 404；
4. **第三方图床缩略图锁死与反向还原**：许多博客站点（如 BlogJav）在页面中仅插入 Pixhost 缩略图（`thumbs`），或附带 `?width=xxx` / `?height=xxx` 参数，若不进行反向推导，用户只能看到马赛克级别的微缩剧照；
5. **并发洪水与网络阻塞**：列表页包含数十张卡片，若在滑动时盲目发起无节制请求，极易触发第三方 CDN 的频率风控（HTTP 429/403）。

---

## 核心实现机制 (Architectural Pillars)

### 1. 多引擎注册矩阵与 GM 跨域透明代理
`PreviewSys` 将抓取通道抽象为多引擎矩阵，支持预设站点与用户自定义正则模板：
* **核心内置源**：
  * `javfree`（JavFree API / 页面解析）
  * `javstore`（JavStore 全文提取）
  * `blogjav`（BlogJav WordPress 内容图集）
  * `projectjav`（ProjectJav 实时透镜提取）
  * `local`（JAVDB 官方剧照 DOM 秒提与本地降级）
* **跨域通道**：底层强制通过油猴特权函数 `GM_xmlhttpRequest`（`gmHttp.request`）发起网络请求，彻底绕过浏览器的同源策略（Same-Origin Policy）限制。

### 2. 番号严格归一化比对（Strict Alphanumeric Verification）
为彻底消灭“张冠李戴”现象，所有引擎在解析搜索结果 HTML 时，必须执行严格的归一化比对准则：
```javascript
const normInput = code.replace(/[\s\-_]/g, '').toUpperCase();
```
解析器遍历搜索候选列表，必须在候选卡片的 `href`、图片 `alt`、`title` 或标题文本中找到包含 `normInput` 的精确条目才准予跟进；**若未命中精确条目，算法宁可返回 `null` 也绝不盲目回退到第 1 条**。

### 3. 多源图床反向工程与缺陷修复管线
根据各站点的图床特征，提取层内置了专用的清洗管道：
* **ProjectJav 双重域名拼接 Bug 修复**：
  ```javascript
  s = s.replace(/^https?:\/\/images\.projectjav\.comhttps?:\/\//i, 'https://');
  ```
* **Pixhost 缩略图原画还原**：
  将缩略图服务器及路径映射为大图服务器：
  ```javascript
  s = s.replace(/t(\d+)\.pixhost\.to\/thumbs/, 'img$1.pixhost.to/images');
  ```
* **动态参数与几何尺寸剥离**：
  利用 `URL.searchParams` 剥离 `width` 与 `height` 限制参数，同时剔除末尾残留的 `?` 与 `&`，直接索取 1080P/4K 原画文件。
* **噪音过滤器**：
  严格过滤女优肖像、Logo、横幅广告与站标：
  ```javascript
  !/avatar|logo|icon|banner|favicon|actress/i.test(s)
  ```

### 4. 降级保底机制（Graceful Cover Fallback）
若某部稀有影片在第三方站点存在条目但尚未切出分段剧照，解析器会自动捕获其原画级别电影大封面（如 `/data/covers/...`），保证嗅探成功率最大化，杜绝空白卡死。

### 5. 双层 LRU 缓存与状态隔离
* 针对抓取结果建立内存级 `this.cache = new Map()`，以 `normCode` 为主键；
* 成功结果缓存 `{ status: 'success', img, gallery, source }`；
* 失败或无数据结果写入 `{ status: 'no-data' }`，有效拦截同一会话期内对无剧照影片的重复网络重试。

---

## 历史惨痛教训与避坑铁律 (Critical Gotchas)

1. **铁律 1：绝不可用搜索首项作为全局回退**：
   在第三方搜索中，首项往往是权重最高的热门影片或赞助广告。直接取首项曾导致几百部冷门影片被错误关联上热门作品的剧照，造成灾难性的元数据污染；
2. **铁律 2：清理 URL 时严禁破坏合法查询参数**：
   部分 CDN（如 `cdn.hpify.com`）依赖版本签名参数（如 `?v=1790328549`）放行访问。剥离 `width`/`height` 缩略图参数时，必须使用标准的 `URL.searchParams.delete()`，绝不可用粗暴的正规表达式将整个 Query String 抹掉，否则会导致 CDN 鉴权失败返回 403 Forbidden；
3. **铁律 3：`localStorage` 站点配置无缝升级守卫**：
   老用户的浏览器 `localStorage` 中保存了自定义站点顺序。新增内置站点时（如 `projectjav`），必须在 `getPreviewSiteConfigs()` 中采用“差集检测 + 锚点插入（插在 `local` 官方剧照之前）”，严禁直接清空用户配置或覆盖重置。

---

## 结果与长远影响 (Consequences)
* 实现了高准确率、无图床死链、高保真画质的跨域剧照嗅探体验；
* 建立起自愈、防穿透、具备保底降级的健壮爬虫体系。
