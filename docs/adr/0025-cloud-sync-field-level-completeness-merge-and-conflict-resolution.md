# 0025. 云端多端同步增量字段胜者合并算法与冲突裁决矩阵

## 状态
已接受 (Accepted)

## 上下文与多端冲突痛点 (Context & Problem Space)

在跨设备、跨平台的实际使用场景中（如台式机、笔记本、平板以及手机端 Kiwi / Firefox 浏览器），用户对影视收藏夹的使用具有高度并发性与分布式特征：
1. **多端状态割裂**：
   用户可能在办公室电脑上将某部影片加入清单「待看」，在下班通勤路上用手机标记为「看過」并给出 4.5 星评分，随后在家中电脑上为该影片手动修正了「演员别名」或补充了「个人观后备注」；
2. **整条记录覆盖的毁灭性灾难（Naive Overwrite Disaster）**：
   如果采用传统粗暴的「全量覆盖导入」或以最后修改时间戳（LWW: Last-Write-Wins）盲目覆盖整条记录，那么后同步的设备必然会抹除另一台设备上新写入的笔记、评分或清单归属；
3. **海量数据导入时的 IndexedDB 事务饥饿风暴**：
   早期方案采用逐条 `await dbGetMovie(code)` 后 `await dbPutMovie(merged)` 的模式。在影片库达到 5,000 ~ 20,000 部时，这会导致上万次 IndexedDB 事务排队往返，主线程被彻底锁死，浏览器频频弹出「页面无响应」；
4. **隐私与敏感凭证泄露风险**：
   备份文件中不仅包含影视元数据，还涉及用户的 JAVDB Session Cookie、WebDAV 账号密码及 GitHub/Gitee Personal Access Token。若未做边界隔离，极易在导出分享时造成严重的信息泄露。

为此，JavdbEmbySkin 设计并实现了**纯内存索引裁决、字段完整度评分胜者算法与增量批量合并流水线 (`FAV.importJSON`)**。

---

## 核心实现机制 (Architectural Pillars)

### 1. 事务降维：单次读全集 + 纯内存裁决 + 分块批量写回
为了从根本上消除 IndexedDB 事务往返开销，数据合并流水线重构为三段式批处理：
* **单次全量内存索引（1 次只读事务）**：
  在合并前执行单次 `dbAllMovies()`，在内存中构建以番号 `code` 为键的哈希索引表 `oldById`：
  ```javascript
  const oldById = Object.create(null);
  (await dbAllMovies()).forEach(m => { if (m && m.code) oldById[m.code] = m; });
  ```
* **纯内存冲突裁决与去重**：
  所有记录的字段比对、完整度算分、清单并集均在纯 JavaScript 内存中微秒级完成，并利用 `decided[code] = 1` 防御备份文件内重复番号的冗余计算；
* **分块批量写回（Chunked Batch Writes, O(N/50)）**：
  定稿后的影片数组交由 `dbPutMovies` 按 50 条一批开启写入事务，使得万条数据的导入事务总数从 20,000+ 骤降至 200 次以内，耗时由数分钟缩短至 1 秒内。

### 2. 字段丰富度评分模型 (`fieldRichness`)
当面对远端（云端备份/新抓取数据）与本地现有数据的元数据差异时，系统不以单纯的时间先后为标准，而是引入**字段丰富度加权评分函数**：
$$\text{Richness}(v) = \sum_{k \in K_{\text{base}}} \mathbb{I}(v[k]) + 2 \cdot \mathbb{I}(|\text{actors}| > 0) + 2 \cdot \mathbb{I}(|\text{categories}| > 0) + 5 \cdot \mathbb{I}(\text{detailComplete})$$
* **基础元数据** $K_{\text{base}}$：`title`, `date`, `coverThumb`, `coverLarge`, `duration`, `maker`, `series`, `director`, `rating`（各占 1 分）；
* **深度元数据**：演员列表占 2 分，类别标签占 2 分；
* **详情完整度权威标**：`detailComplete`（已爬取详情页完整元数据）赋予 5 分绝对权重。

两端比较中，得分更高的一方成为**基底容器宿主（Base Metadata Winner）**，确保海报封面清晰度、导演、系列等信息始终保留最充实的一版。

### 3. 用户主观数据冲突裁决矩阵 (Conflict Arbitration Matrix)
在基底元数据裁决胜出后，算法会严格执行**属性级保护规则**，将用户的主观修正与动态状态精准嫁接回胜出实体：

| 字段类别 | 字段名 | 冲突裁决策略 | 业务设计哲学 |
| :--- | :--- | :--- | :--- |
| **所属清单** | `listIds` | **数学并集 (Union)**：`Set(old.listIds ∪ nv.listIds)` | 多端归档互不冲突，清单只增不减 |
| **用户评分** | `userScore` | **既有评分绝对优先**：`old.userScore > 0 ? old : nv` | 尊重用户已付出心智打出的星级，不被新导入空置 |
| **自定义修正** | `customMeta` | **逻辑时间戳最新胜者**：比较 `updatedAt` 毫秒数 | 尊重用户最近一次对番号、标题或封面的纠错 |
| **观看状态** | `reviewStatus` | **显式状态覆盖 (State Override)**：`nv.status || old.status` | 新产生的观看或想看标记优先同步 |
| **浏览次数** | `views` | **单调递增最大值 (Monotonic Max)**：$\max(nv.views, old.views)$ | 统计值跨设备物理累加或取峰值，绝不倒退 |
| **个人备注** | `notes` | **非空优先与保全**：`nv.notes || old.notes` | 用户文字批注绝不允许静默清空 |
| **自定义封面** | `customCover` | **非空优先与保全**：`nv.customCover || old.customCover` | 本地挑选的高清大图绝对保全 |

### 4. 敏感凭据防御性脱敏隔离
在执行云端推送（WebDAV PUT / Git Commit API）及本地 JSON 导出时：
* 默认调用 `getBackupDataObject(false)`，自动过滤剥离用户的 JAVDB Authorization Cookie、WebDAV 登录密码及 Git 访问令牌；
* 仅当用户在设置面板中主动开启「备份包含敏感凭据」复选框时，方透传凭据。从底层阻断了多设备同步文件在私有云或公共代码仓托管时的泄露隐患。

---

## 历史惨痛教训与避坑铁律 (Critical Invariants & Gotchas)

### 铁律 1：严禁以覆盖模式抹除画廊真理源
在早期版本中，用户执行「全量覆盖导入」时仅清空了影片表，但未重置 IndexedDB 的画廊表及内存集（`collectedSet`）。导致导入完成后，详情页中的画廊按钮高亮状态与实际本地库不一致。**覆盖导入必须按序完整清空全部 6 张表，并在写回后立即调用 `dbAllGallery()` 强制重建内存集合**。

### 铁律 2：严禁非安全类型参与评分比对
在解析外部 JSON 时，`userScore` 可能被错误序列化为字符串 `"0"` 或 `null`。在裁决逻辑中，必须严格执行 `typeof val === 'number' && val > 0` 守卫，杜绝无效类型导致已有评分被抹除。

---

## 效果与收益 (Outcomes & Value)
1. **千秒降至一秒**：万条记录多端合并耗时由 120+ 秒下降至 0.8 秒以内，零掉帧、零卡死；
2. **多端无缝漫游**：无论在何种设备上打标签、写备注、标星级，同步后均能无损融合，彻底告别数据覆盖丢失的恐惧。
