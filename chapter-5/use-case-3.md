# 用例 3：框架自动渲染

&emsp;&emsp;类似Angular这样的框架需要知道什么时候所有的应用程序任务完成了，然后更新DOM，从而浏览器执行像素级别的渲染。实践中，这意味着框架对什么时候主任务以及附属的微任务执行完成，而虚拟机还没有把控制权还给宿主的时间感兴趣。

```javascript
class VMTurnZoneSpec {
  constructor(vmTurnDone) {
    this.name = 'VMTurnZone';
    this.vmTurnDone = vmTurnDone;
    this.hasMicroTask = false
  }

  onHasTask(delegate, current, target, hasTaskState) {
    this.hasMicroTask = hasTaskState.microTask;
    if (!this.hasMicroTask) {
      this.vmTurnDone();
    }
  }

  onInvokeTask(parent, current, target, task, applyThis, applyArgs){
    try {
      return parent.invokeTask(target, task, applyThis, applyArgs);
    } finally {
      if (!this.hasMicroTask) {
        this.vmTurnDone();
      }
    }
  }
}
```
