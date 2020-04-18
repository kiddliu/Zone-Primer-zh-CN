# 用例 2：长堆栈记录

&emsp;&emsp;很多时候当应用程序抛出错误，错误会指向一处没有太多帮助的代码，因为无法得知任务是如何调度的额外上下文。试想这样一个错误：

```log
Error: ReferenceError: name is not defined
  paintResponse()
```

&emsp;&emsp;我们知道发生了错误，但是我们不知道任务是被如何调度的。长堆栈记录提供了这样一个上下文，它可以打印出跨任务执行的堆栈信息。

```log
Error: ReferenceError: name is not defined
  at paintResponse()
---- async gap ----
  at requestAnimationFrame()
  at XMLHttpRequest.resolve()
---- async gap ----
  at XMLHttpRequest.send()
  at fetchData()
  at clickHandler()
---- async gap ----
  at addEventListener()
  at main()

```

&emsp;&emsp;有了长堆栈记录，于是把一个复杂的场景，例如创建事件监听方法，然后点击，接着通过XHR获取数据，最终尝试用`requestAnimationFrame`渲染数据失败的整个过程都连接到了一起。

```javascript
class LongStackTraceZoneSpec {
  constructor() {
    this.name = 'LongStackTrace';
  }

  onScheduleTask(parentZoneDelegate, currentZone, targetZone, task) {
    var task =  parentZoneDelegate.scheduleTask(targetZone, task);
    // 每当一个新任务创建时，我们抓取
    // 创建位置的堆栈信息
    task.data.trace = new Error('LongStackTrace');
    // 连接新任务与创建它的父任务
    task.data.parentTask = Zone.currentTask;
    return task;
  }

  onHandleError: function(parentZD, current, target, error) {
    error.stack += this.getLongStackTrace();
    return parentZD.handleError(target, error);
  }

  // 组建一条长堆栈记录
  getLongStackTrace() {
    var trace = [''];
    var task = Zone.currentTask;
    while(task) {
      trace.push(task.data.trace);
      task = task.data.parentTask;
    }
    return trace.join('\n--- async gap --\n');
  }
}
```

&emsp;&emsp;**核心点：**

* 生产环境中的app发送堆栈信息回服务器，于是开发者可以更好的了解问题所在。
