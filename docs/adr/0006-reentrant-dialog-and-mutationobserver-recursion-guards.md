# 0006. DOM MutationObserver 递归守卫与 window.confirm 可重入锁

## 状态
已接受 (Accepted)

## 上下文
在复杂油猴脚本的生命周期中，脚本一方面需要作为“观察者”监听 JAVDB 原生 DOM 树或自定义界面的变化（例如监听列表加载、评分组件状态同步）；另一方面脚本又是“操作者”，会频繁修改 DOM 结构与文本。在早先版本中，曾偶发因自身修改触发自身监听而导致的“微任务死循环卡死（Infinite Mutation Loop）”，以及在自动化测试或批量覆盖导入时 `window.confirm` 弹窗被外部或自身重复调用导致的主线程死锁。

## 决策
我们决定**确立严格的 DOM 观察器递归防护范式与全局弹窗可重入保护锁**：
1. **MutationObserver 前置比对守卫**：
   * 严禁在观察器回调内进行无条件的 DOM 赋值；必须遵循“比对后才写入”：
     ```javascript
     if (span && span.textContent !== nextIcon) {
       span.textContent = nextIcon;
     }
     ```
   * 对复合观察器引入 `hasExternalChange` 标志位，由用户真实交互或外部事件触发，避免内部同步逻辑自我触发。
2. **`window.confirm` 全局可重入锁与 finally 强保证**：
   * 当批量操作（如全量数据导入覆盖）需要临时静默或重写 `window.confirm` 行为时，必须使用防重入锁并强制在 `finally` 块中复原：
     ```javascript
     if (!window.__origConfirm) window.__origConfirm = window.confirm;
     try {
       window.confirm = function () { return true; };
       await executeBatchTask();
     } finally {
       window.confirm = window.__origConfirm;
     }
     ```

## 原因与权衡
* **微任务栈溢出的致命性**：MutationObserver 的回调是在当前宏任务结束前清空微任务队列时同步执行的。如果回调中的 `element.textContent = 'A'` 触发了对该节点的 `characterData` 变动监听，观察器会无限循环加入微任务，导致浏览器标签页直接冻结并报“Maximum call stack size exceeded”。
* **油猴沙箱与全局原型的微妙性**：如果异步操作报错而未能执行到恢复 `confirm` 的语句，不仅脚本功能损坏，甚至用户在 JAVDB 上的所有原生确认弹窗都会被劫持破坏。

## 结果与影响
* **收益**：彻底消除了长时间挂机浏览时的 CPU 异常占用、避免了批量数据还原时的意外锁死。
* **代价**：所有在观察器内部操作 DOM 的代码行均需多写一层条件判断，增加了少量字符，但换来了 100% 的运行时鲁棒性。
