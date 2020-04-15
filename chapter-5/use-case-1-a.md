# 用例 1.a：测试——测试中禁止异步代码

&emsp;&emsp;有快速流畅的测试是很值得期待的。一种达成的方式是完全使用同步测试，因为这样的测试又快，行为又可以预测。并不是所有的测试都可以完全同步执行的，即使是在构建海量模拟模型的情况下。更重要的是可以有一种方式，可以标记测试是倾向于同步的，并且可以强制这种意图，从而不至于意外的具备了异步的行为。

```javascript
var syncZoneSpec = {
  name: 'SyncZone',
  onScheduleTask: function() {
    throw new Error('No Async work is allowed in test.');
  }
}

function sync(fn) {
  return function(...args) {
    Zone.current.fork(new EventReleaseZoneSpec).run(fn, args, this);
  }
}

it('should fail when doing async', sync(() => {
  Promise.resolve('value');
}));
```

&emsp;&emsp;**核心点：**

* 在`SyncZone`内执行测试，如果创建了异步任务就会自动失败。
