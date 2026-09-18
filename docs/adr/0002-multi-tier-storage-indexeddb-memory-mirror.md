# 0002. IndexedDB + 内存镜像多级存储架构

## 状态
已接受 (Accepted)

## 上下文
随着用户在 JAVDB 上建立庞大的收藏库（通常包含数千部甚至上万部作品数据），早期版本使用 `localStorage` 存储面临严重的容量与性能瓶颈；而在迁移至 `IndexedDB` 后，又遇到了异步读取延迟导致的首屏渲染白屏，以及万条数据单条写入导致的浏览器卡死。

## 决策
我们决定采用**三级分层存储架构**：
1. **Tier 1 (同步内存镜像)**：启动时一次性将核心实体与索引加载至内存中的 `Map` / `Object`，供所有 UI 渲染函数同步读取；
2. **Tier 2 (持久层 IndexedDB)**：全量业务数据（movies, lists, gallery, meta, actor_genders, notes）持久化于 IndexedDB `javdb-emby-fav-db`；
3. **Tier 3 (标量配置持久化)**：`localStorage` / `GM_setValue` 仅用于存储总容量 < 500KB 的轻量布尔开关、选中的皮肤模式与鉴权 Token；
4. **批量写入协议**：所有多条数据写回操作强制使用分片事务（以 50 条为一批切片）。

## 原因与权衡
* **为什么弃用 localStorage 存影片**：浏览器对 `localStorage` 有 5MB~10MB 的硬性配额限制，存入上千条带封面与演员元数据的 JSON 会触发不可逆的 `QUOTA_EXCEEDED_ERR` 导致数据丢失；且 `localStorage` 的所有 I/O 均阻塞浏览器主线程。
* **为什么必须加内存镜像**：IndexedDB 全部为异步 Promise API。如果卡片渲染必须逐条 `await dbGetMovie`，列表页与瀑布流在滚动时会出现明显的文字和头像闪烁跳动；内存镜像使渲染保持纯同步，首屏毫秒级就绪。
* **为什么必须 50 条分批**：实测表明，万条数据导入时，如果逐条开启 `await dbPutMovie`，会产生上万次跨进程 IPC 事务切换，耗时超过数分钟；而若放入一个超级大事务，低配设备或 Safari 隐私模式极易超时崩溃。按 50 条切片是性能吞吐与防卡死的黄金平衡点。

## 结果与影响
* **收益**：轻松支撑 20,000+ 影片本地秒开；彻底杜绝配额溢出；UI 渲染完全无异步跳动。
* **代价**：当数据发生新增或更新时，必须严格保持内存镜像（如 `favMoviesCache`、`scoreMemCache`）与 IndexedDB 物理数据的双向同步。
