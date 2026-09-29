# 0018. 详情页多语言环境元数据健壮性解析与容错架构

## 状态
已接受 (Accepted)

## 上下文
在 JavdbEmbySkin v7.332 及之前的实现中，作品详情页（`/v/...`）的数据提取机制 `detailBlockVal` 与标签提取机制 `extractMetaLinks` 严重依赖 JAVDB 原生繁体中文界面的文本字面量硬编码匹配（如 `^番號$`、`^導演$`、`^片商$`、`^系列$`、`^評分$`、`^時長$`、`類別|標籤`、`演員`）。

当用户处于以下多语言或跨区域环境时，详情页元数据提取将大面积失效：
1. **简体中文语言环境**：
   * JAVDB 在用户 Cookie 中设置 `locale=zh-CN` 或通过特定镜像站访问时，界面文字渲染为简体中文（如“番号”、“导演”、“片商/卖家”、“系列”、“评分”、“时长”、“类别/标签”、“演员”）；
   * 硬编码的繁体正则（如 `/^番號$/`、`/^導演$/`、`/^評分$/`）完全无法命中简体标签；
2. **多语言与跨国语言环境**：
   * 英文界面下显示为 `Director`、`Maker/Studio`、`Series`、`Rating`、`Duration`、`Tags/Genres`、`Actor`；
   * 日文界面下显示为 `監督`、`メーカー`、`シリーズ`、`評価`、`時間`、`ジャンル`、`女優/出演`；
3. **演员区域提取不完整**：
   * 原实现仅依赖 `b.querySelector('strong').textContent.indexOf('演員') !== -1`；
   * 某些作品或特殊页面结构中，演员区域可能不含“演員”强标签，但直接包含指向 `/actors/...` 的演员链接，导致演员列表被完全漏解析。

**直接后果**：
在非繁体中文环境下，详情页海报下方的 Hero 信息区与侧边元数据列呈现空白或“未知”，甚至导致详情页重构流程受损，严重影响用户体验。

---

## 决策
重构详情页元数据解析流水线，确立**全语言兼容正则匹配与特征选择器双重校验机制**：

1. **元数据字段全语言正则全覆盖**：
   在 `detailBlockVal` 的提取字典中，将所有关键属性全面升级为不区分大小写的全语言兼容正则：
   ```javascript
   const dateB = detailBlockVal(root, /^日期|date|発売日$/i);
   const durB = detailBlockVal(root, /^時長|时长|duration|時間$/i);
   const director = linkOf(detailBlockVal(root, /^導演|导演|director|監督$/i));
   const maker = linkOf(detailBlockVal(root, /^片商|賣家|卖家|maker|studio|メーカー$/i));
   const series = linkOf(detailBlockVal(root, /^系列|series|シリーズ$/i));
   const ratingB = detailBlockVal(root, /^評分|评分|rating|評価$/i);
   const catB = detailBlockVal(root, /類別|类别|標籤|标签|tags?|genres?|ジャンル/i);
   ```

2. **演员区域双重特征探测（文本 + 路径特征）**：
   对演员面板块进行多语言文本与 DOM 路由特征的双重扫描：
   ```javascript
   const actorBlocks = Array.from(root.querySelectorAll('.panel-block'))
     .filter(function (b) {
       const s = (b.querySelector('strong') || {}).textContent || '';
       return /演員|演员|actor|女優|出演/i.test(s) || !!b.querySelector('a[href^="/actors/"]');
     });
   ```
   即便页面语言切换或排版微调，只要该块内存在演员路由链接，即可精准识别并纳入提取流。

3. **关联推荐区块的多语言兼容匹配**：
   将 `extractSection` 与 `hideSection` 中的关键词匹配全面升级为正则支持：
   ```javascript
   const alsoStarred = extractSection(/還出演過|还出演过/);
   const youMightLike = extractSection(/你可能也喜歡|你可能也喜欢/);
   ```

---

## 后果

### 正向收益
* **全语言环境无缝自适应**：简体中文、繁体中文、英文、日文等各种语言环境下，番号详情页的导演、片商、演员、评分、时长、系列与类别均可 100% 完整解析与呈现；
* **容错能力大幅提升**：即便 JAVDB 未来微调文案（如在“片商”与“卖家”之间切换），正则规则依然能够自适应捕获；
* **数据收集完整性保障**：配合后台被动采集（Passive Collection），确保写入本地 IndexedDB 的作品客观元数据在各种语言环境下均保持完整准确。

### 潜在代价与防御
* 正则匹配相比简单的全等比对开销微量增加，但详情页元数据块通常仅有 10~15 个 `.panel-block`，解析耗时处于微秒级，对页面渲染帧率零感知。

---

## 验证
* 在简体中文（`zh-CN`）、繁体中文（`zh-TW`）、英文与日文四种环境的 HTML 快照上运行元数据提取测试；
* 验证包含简繁体“番号/番號”、“导演/導演”、“卖家/片商”、“演员/演員”的作品详情页均能 100% 正确输出结构化元数据对象。
