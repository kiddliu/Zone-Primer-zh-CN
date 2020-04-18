# 用例 1.a：测试——可以自动清理的事件

&emsp;&emsp;修改了全局状态，测试就可能把信息暴露给下一个测试。发生此类问题的一种常见方式是向DOM节点附加上了事件处理函数，而这些节点在测试过程中是一直存在的，比如Body元素。Zone可以跟踪这样的时间注册，当测试完成时自动清理掉它们。

```javascript
class EventReleaseZoneSpec {
  constructor() {
    this.name = 'CleanupZone';
    // 在这里持续记录未处理的EventTask
    this.eventTasks = [];
  }

  onScheduleTask(parentZoneDelegate, currentZone, targetZone, task) {
    // 每当调度了新的EventTask，就把它添加到未处理任务的列表去
    if (task.type == 'eventTask') {
       this.eventTasks.push(task);
    }
    return parentZoneDelegate.scheduleTask(targetZone, task);
  }

  cleanup() {
    // 取消所有未处理的（任务）
    while(this.eventTasks.length) {
      Zone.current.cancelTask(this.eventTasks.pop());
    }
  }
}

// 包装测试函数
// 然后当测试结束自动释放EventTask监听函数
function cleanup(fn) {
  return function(done) {
    let zoneSpec = new EventReleaseZoneSpec();
    let args = [() => {
      zoneSpec.cleanup();
      done();
    }];
    Zone.current.fork(zoneSpec).run(fn, this, args);
  }
}

it('should auto-cleanup events', cleanup((done) => {
  someElement.addEventListener('click', () => console.log('click'));
  someElement.click();
  done();
}));
```

&emsp;&emsp;**核心点：**

* 一项测试可以在完成之后自动清理所有的事件注册动作。
