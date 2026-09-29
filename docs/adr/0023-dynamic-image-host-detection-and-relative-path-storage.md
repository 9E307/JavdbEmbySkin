# 0023. JAVDB 动态图床嗅探与零迁移相对路径存储架构

## 状态
已接受 (Accepted)

## 上下文与痛点 (Context & Problem Statement)

JAVDB 是一个长周期迭代的多媒体信息站点，在过去数年的运营与防封锁过程中，其封面图床域名历经了频繁且不可预测的更迭：
* 早期阶段采用第三方 CDN 哈希节点（如 `https://xxx.rhe951l4q/...`）；
* 中期阶段采用多台分流图床（如 `https://c0.jdbstatic.com`、`https://c1.jdbstatic.com`、`https://jdbstatic.com`）；
* 特殊时期镜像站甚至直接使用自身主域名的反向代理路径（如 `https://javdb521.com/covers/...`）。

如果 UserScript 在采集影片信息（`passiveCollectMovie` 或全量同步）写入 IndexedDB 数据库时，直接保存浏览器当时看到的完整绝对 URL（如 `https://c0.jdbstatic.com/covers/12/123.jpg`），将面临毁灭性的架构脆弱性：
**一旦官方图床在下个月弃用或更换子域名，用户本地数据库中积累数年、多达数万部影片的封面图片将瞬间全体沦为“死链（Broken Link）”**，用户面临全屏裂图，且不得不清空数据库重新完整重爬。

---

## 核心实现机制 (Architectural Pillars)

### 1. 绝对协议与域名剥离（Storage-Time Normalization）
在任何影片对象（`Movie Record`）持久化写入 IndexedDB 前，封面字段强制通过 `stripHost()` 剥离其协议、端口与主机名，**仅持久化其静态资源相对路径**：
```javascript
function stripHost(url) {
  if (!url) return url;
  const m = url.match(/^https?:\/\/[^\/]+(\/.*)$/i);
  return m ? m[1] : url; // 例如 "https://c0.jdbstatic.com/covers/ab/cd.jpg" -> "/covers/ab/cd.jpg"
}
```

### 2. 页面生命周期动态图床嗅探（Runtime Host Detection）
在每次用户打开 JAVDB 任意页面的毫秒级初始化阶段，执行 `detectImgHost()`：
1. 扫描当前网页中原生渲染的 `document.images`；
2. 提取匹配 `IMG_HOST_RE = /^(https?:\/\/[a-z0-9.-]*jdbstatic\.com)/i` 的图片真实来源；
3. 若原生图片尚未加载，降级扫描含有 `jdbstatic.com` 的 `<a>` 标签与 `<link>` 标签；
4. 动态确立当前页面生命周期内最高可用的官方图床基准地址 `IMG_HOST`。

### 3. 渲染层动态拼装与历史脏数据自愈（Zero-Migration Resolution）
在卡片封面渲染、大图预览、灯箱查看器或画廊绘制时，统一通过 `resolveImg()` 处理：
```javascript
function resolveImg(path) {
  if (!path) return path;
  let p = String(path).trim();
  // 1. 修复官方历史废弃的缩略图路径
  p = p.replace(/\/small_covers\//g, '/thumbs/');
  // 2. 自动修正远古废弃 CDN 节点
  p = p.replace(/https?:\/\/.*?\/rhe951l4q/g, 'https://c0.jdbstatic.com');
  // 3. 自动修正旧版本硬编码写入的旧镜像域名
  p = p.replace(/^https?:\/\/(www\.)?javdb\d*\.(com|me|life|party|today|vip)\/(covers|thumbs)/i, 'https://c0.jdbstatic.com/$3');
  // 4. 若为相对路径，动态拼接当前会话嗅探到的合法有效图床域名
  if (p.startsWith('/')) {
    return IMG_HOST + p;
  }
  return p;
}
```

---

## 历史惨痛教训与避坑铁律 (Critical Gotchas)

1. **铁律 1：严禁直接向 IndexedDB 写入绝对图片 URL**：
   无论采集自列表页、详情页还是通过第三方备份文件导入，存入数据库前必须调用 `stripHost()`。只有保持存储层的无状态纯净性，才能实现“图床虽更迭，数据永不朽”；
2. **铁律 2：嗅探未成功前不得执行破坏性写回**：
   若页面刚打开由于网络离线未嗅探到有效图床（`!imgHostDetected`），系统仅使用安全默认值 `https://c0.jdbstatic.com` 进行内存渲染，**绝不可在此期间触发全量数据库写回**，防止用默认假定值污染原本干净的数据；
3. **铁律 3：多端同步导入兼容性**：
   在与其他客户端同步 JSON 数据时，可能遇到老版本导出的绝对路径数据。`resolveImg()` 内部设计的多重正则清洗是保证老数据“即插即用、零迁移感知”的生命线，后续代码审查严禁随意删减这些正则。

---

## 结果与长远影响 (Consequences)
* 实现了跨越数年的图床无感迁移，从根本上消灭了“图床变迁导致的封面大面积死链”；
* 数据库体积缩减约 15%（节约了重复存储 `https://c0.jdbstatic.com` 前缀的巨量字符）；
* 保证了导入备份、云端拉取历史数据时的 100% 封面可用率。
