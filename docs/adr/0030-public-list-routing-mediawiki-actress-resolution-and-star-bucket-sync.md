# 0030. 公开清单 40/页稳定路由切换、无依赖双语维基女优解析与看过评分分桶同步架构

## 状态
已接受 (Accepted — v7.5.1)

## 上下文与实证根因 (Context & Verified Root Causes)

在 Chrome 实机调试与线上接口抓包中，针对长期困扰重度用户的四大问题定位到如下物理根因：

1. **501 部满额清单同步仅剩 497~498 部**：
   * 实测 `/users/lists` 对用户创建的公开清单给出了 `/users/list_detail?id={id}` 链接（12 部/页，需 42 页抓完 501 部）。
   * `/users/list_detail` 服务端 SQL 分页存在并列排序漂移（tie-breaker 缺失），导致翻页边界重复返回同一部影片（实测 `4N2qR` 预设清单6 与 `0RW6b7` 在 1–4 页共出现 4 次跨页重复），把末尾 3~4 部影片挤出 42 页之外，使 `runSync` 去重后只写入 `497` 部。
   * 对同一清单请求公开直链 `/lists/{id}`（40 部/页，仅 13 页）实测返回 `501` 条且 **0 重复**；但若清单为“預設清單（只對自己可見）”等私人清单，请求 `/lists/{id}` 会被服务端 **302 重定向回首页 `/`**（首页同样返回 HTTP 200 及 50 个 `.item` 影片卡片，极易误解析污染清单）。
2. **完整全封面紧凑顶栏小海报 3D 辉光与鼠标错位 2 倍**：
   * `#emby-compact-bar .ecb-poster` 静态尺寸为 `78×111`，设定了 `transform-origin: left top` 并在悬停时放大至 `scale(2)`（`156×222`）。
   * `attachCompactPoster3D` 在 `mouseenter` 时缓存了未放大状态下的 `78×111` 矩形，导致在放大后的海报中心 `(left + 78, top + 111)` 计算出 `px = 1.0, py = 1.0`（`--mx: 100%, --my: 100%`），右下角甚至飙升至 `179%`。
3. **第三方依赖仓库删库导致女优悬停生平卡片空白与三围/罩杯缺失**：
   * 既往主数据源 `cdn.jsdelivr.net/gh/lesspem/JAV_info@main/dist/` 与 GitHub 源站已全线删库返回 HTTP 404，所有冷启动全部跌落至初代 `lookupWiki()`。
   * 初代 `lookupWiki()` 仅硬编码访问 `https://ja.wikipedia.org/wiki/<原名>`（无中文译名桥接、无 MediaWiki 重定向/跨语言查询），且遗漏 `スリーサイズ`、`ブラサイズ`、`活動内容` 与中文维基 Infobox 字段，无法清洗脚注引用（如 `B[動画 1][2]`）与英制换算行（`5 ft 3 in` / `36 - 22 - 34 in`），且 `jhs_actress_info_v2` 的 7 天缓存固化了缺漏条目。
4. **收藏夹“看过”作品未点进详情页前缺少个人评分星数**：
   * JAVDB 原生 `/users/watched_videos` 列表卡片只输出站点社区大众均分，不渲染当前用户的个人星级评分，仅详情页 `#modal-review-edit` 与 `/users/watched_videos?star=5..1` 分桶页面携带个人评分归属。
   * 既往 `FAV.syncReviewStatus()` 仅抓取无参 `/users/watched_videos`，使批量同步的看过影片缺失 `userScore`，直至用户逐一点开详情页触发 `passiveCollect` 才被动补全。

---

## 决策 (Decisions)

### 1. 清单同步双轨路由探测与分页缺口保护 (`FAV.runSync`)
* **`throttledFetch` 透传最终落地 URL**：在 `DOMParser` 返回的文档对象上记录不可枚举（不可误序列化）的 `_finalUrl = resp.url || ''`。
* **公开清单优先走 `/lists/{id}`**：对于未标记 `只對自己可見` 的个人清单，同步启动时先探测 `/lists/{id}?page=1`；仅当 `_finalUrl` pathname 确认停留于 `/lists/{id}` 且解析有效时切换至 `/lists/{id}` 抓取（并直接复用已抓到的第 1 页结果，避免重复请求），从根本上获得 40 部/页零重复全量列表（501/501）。
* **私人清单无感回退与缺口防误删保护**：一旦检测到私人标识（`isPrivate`）、`/lists/{id}` 触发 302 跳回首页 `/`、或探测页非预期 HTTP 4xx，立即无感回退至原生 `/users/list_detail?id={id}`；对清单内部计数采用 `seenInList` 去重（保证 `scrapedCount` 与本地去重数一致），且当私人清单分页出现重复并导致去重数量小于站端宣称总数时，将清单纳入受保护集合（`protectedListIds`），禁止在 diff 阶段把被分页重复挤掉的本地既有影片误删或脱钩。

### 2. 紧凑顶栏 `scale(2)` 悬停坐标系校准 (`attachCompactPoster3D`)
* **分离视觉测量与布局尺寸**：在 `mouseenter` 阶段测量原点 `(r.left, r.top)`，优先以不受 CSS transform 影响的布局属性 `poster.offsetWidth` / `poster.offsetHeight` 作为基准宽高，再乘以悬停缩放倍率 `HOVER_SCALE = 2` 得到实际交互包围盒并缓存（仅在 `window` `resize` 时置空缓存）。
* **保持零同步布局读取与边界钳位**：`mousemove` 回调零读取 `getBoundingClientRect()`，使用 `clamp01` 将归一化偏移严格钳位在 `[0, 1]`，同时在 `mouseenter` 触发时即刻应用首帧倾角与高光落点。

### 3. 无外部删库依赖的 `ja/zh` 双语维基直连与别名自学习 (`ActressService`)
* **彻底拔除 `JAV_info` HTTP 404 请求链**：移除 `getRepoIndex`、`lookupRepo` 与 `lesspem/JAV_info` 相关死链。
* **多阶段维基百科解析管线**：
  1. 候选名拆分与按假名优先排序，首选直连 `ja.wikipedia.org/wiki/<cand>`（对绝大多数现役女优 1 次请求直接命中）；
  2. 未命中时调用 `zh.wikipedia.org/w/api.php`（`action=query&redirects=1&converttitles=1&prop=langlinks&lllang=ja&origin=*`）完成简繁转换、页面重定向与日文标准词条桥接，再抓对应日文页；
  3. 再回退中文维基条目本身 Infobox（同时兼容简体与繁体字段标签）与 `opensearch` 关键词校验匹配。
* **全字段与公制优先 Infobox 解析器 (`parseWikiInfobox`)**：遍历 `table.infobox tr` 提取生日、年龄（兼容内联 `（XX歳）` 及出生日期实时推算）、身高体重、三围与罩杯；清洗隐藏 ISO 日期节点、`<sup class="reference">` 与 `[...]` 脚注（如 `B[動画 1][2] -> B罩杯`、`C-65[2] -> C-65罩杯`），强制优先采信公制（`cm` / 厘米 / 胸臀 `>= 50`）数值，拒绝被紧随其后的英制行（`in` / 英寸）污染。
* **缓存升版与全站别名联动**：将缓存键升级为 `jhs_actress_info_v3`（自动忽略并清理残留残缺 `v2` 缓存及废库索引），对无词条女优写入 24 小时 `_miss` 负缓存，并将运行期通过 MediaWiki `redirects` 与 `langlinks` 发现的跨语言/艺名映射组持久化至 `jhs_actress_aliases_v1`，由 `ActressService.getAliases(name)` 统一赋能 `FavoriteActressManager` 与 `GfriendsAvatarService`。

### 4. 看过评分分桶同步与分页参数保全 (`FAV.syncReviewStatus`)
* **修复带参基址翻页拼参**：重构 `findReviewNextPage(doc, curUrl, base)`，对已含 `?` 的基址（如 `/users/watched_videos?star=5`）自动采用 `&cp=` 拼接，并正确解析 `?cp=N` 相对链接。
* **`star=5..1` 分桶批量拉取与删除安全守卫**：当用户存在已看作品时，`syncReviewStatus()` 在抓完 `/users/watched_videos` 后，顺次抓取 `?star=5` 至 `?star=1` 分桶批量写入 `v.userScore`；仅在 5 个星级分桶**全部成功返回**且记录同步期间未发生并发详情/手动改写时，才清除本地残留的过期评分，确保任何单桶波动都不会误伤既有个人打分（延续 [ADR-0005](./0005-strict-separation-of-userscore-and-official-rating.md)）。

---

## 结果与影响 (Consequences)
* **501 满额公开清单 100% 完整入库**：请求页数从 42 页锐减至 13 页，消除边界重复导致的丢片与 `scrapedCount` 虚高；私人清单平滑回退且受保护防误删。
* **紧凑顶栏与女优卡片体验全面恢复**：`scale(2)` 小海报高光精确跟随光标中心；女优悬停恢复完整生日、年龄、身高、三围、罩杯与现役/退役状态，并彻底清除后台控制台 `404` 无效外链请求。
* **收藏夹评分立等可见**：执行一次「同步标记」即可批量补全所有历史“看过”影片的 1~5 星个人打分，无需再逐一点开详情页。
