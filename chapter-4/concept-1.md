# 概念 1：任务调度

&emsp;&emsp;为了跟踪任务，我们需要回顾一下`setTimeout()`是如何打了本地补丁，从而改变了跟踪zone的方式。

```javascript
// 保存setTimeout的原始引用
let originalSetTimeout = window.setTimeout;
// 用把回调包装在zone里的函数覆盖API
window.setTimeout = function(callback, delay) {
  // 在当前zone使用scheduleTask API
  Zone.current.scheduleMacroTask(
    // 调试信息
    'setTimeout',
    // 需要在当前zone执行的回调
    callback,
    // 可选数据，比如任务是否重复执行
    null,
    // 默认的调度行为
    (task) => {
      return originalSetTimeout(
        // 使用任务的invoke方法
        // 于是任务可以在正确的zone调用callback
        task.invoke,
        // 原始的延迟信息
        delay
      );
    });
}
```

&emsp;&emsp;使用示例：

```javascript
// 创建可以记录日志的zone
let logZone = Zone.current.fork({
  onScheduleTask: function(parentZoneDelegate, currentZone,
                           targetZone, task) {
    // 异步任务被调度时打印日志
    console.log('Schedule', task.source);
    return parentZoneDelegate.scheduleTask(targetZone, task);
  },

  onInvokeTask: function(parentZoneDelegate, currentZone,
                         targetZone, task, applyThis, applyArgs) {
    // 异步任务被调用时打印日志
    console.log('Invoke', task.source);
    return parentZoneDelegate.invokeTask(
             targetZone, task, applyThis, applyArgs);
  }
});

console.log('start');
logZone.run(() => {
  setTimeout(() => null, 0);
});
console.log('end');
```

&emsp;&emsp;输出结果：

```log
start
Schedule setTimeout
end
Invoke setTimeout
```

&emsp;&emsp;**核心点：**

* 所有调度任务的API使用`Zone.prototype.scheduleTask()`而不是`Zone.prototype.wrap()`。
* 所有任务使用`Zone.prototype.runGuarded()`执行回调，于是它们会处理所有的错误。
* 对任务的拦截使用的是`invokeTask()`，而对zone的拦截使用的是`invoke()`。
