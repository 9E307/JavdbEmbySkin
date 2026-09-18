# 0011. 异步榜单视图基于请求代数令牌（Sequence Token）的防乱序架构

## 状态
已接受 (Accepted)

## 上下文
在榜单与热播排行榜管理器（`Top250ViewManager`，第 9052-10713 行）中，提供了总榜（All-Time）、日榜（Daily）、周榜（Weekly）和月榜（Monthly）的多维切换。因为每个榜单的数据都需要通过 `gmHttp` 异步发起跨域抓取或解析本地缓存，当用户快速在不同榜单 Tab 之间频繁点击切换时，产生了严重的“网络竞态乱序覆盖（Asynchronous Race Condition）”缺陷：先点击的慢速网络请求后返回，导致用户选中的是“月榜”，界面却最终被迟到的“日榜”内容强行覆盖。

## 决策
我们决定**在所有多源异步抓取与多 Tab 切换模块中，强制引入“请求代数令牌（Request Generation Token / Sequence Token）”模式**：
```javascript
// 每次用户触发新维度查询时，单调自增全局代数令牌
top250LoadSeq++;
const mySeq = top250LoadSeq;

const html = await gmHttp.get(targetUrl);

// 关键屏障：若当前执行链路的代数令牌与全局最新代数不一致，说明用户已发起更新的操作，直接作废并丢弃回包
if (mySeq !== top250LoadSeq) {
  log('丢弃过期的滞后网络回包: seq=' + mySeq + ', currentSeq=' + top250LoadSeq);
  return;
}

renderRankingsUI(html);
```

## 原因与权衡
* **解决异步竞态最轻量、确定性最高的方式**：传统方法可能尝试调用 `AbortController.abort()` 中止请求，但 `GM_xmlhttpRequest` 的中止机制在不同油猴脚本管理器（Tampermonkey vs Violentmonkey）中实现不一致甚至不支持中断底层已发送的网络连接。通过整数序列令牌在 JavaScript 层面做回包有效性校验，逻辑确定性为 100%。

## 结果与影响
* **收益**：无论用户多么疯狂快速地连续切换榜单周期或分页，最终渲染的页面内容必然且严格与当前高亮的 Tab 保持一致，彻底杜绝“张冠李戴”的乱序 BUG。
* **代价**：所有编写异步 UI 渲染的开发者必须养成声明 `mySeq` 并做前置校验的工程习惯。
