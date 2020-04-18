# 用例 1.a：测试——自动等待测试完成

&emsp;&emsp;进行异步测试时，是很难知道所有任务是什么时候完成的。通常，测试会等待某个特定任务完成，然后对结果进行断言。在一个有许多异步任务的测试中，是很难知道所有任务是什么时候完成的。未完成的任务很可能对后续的测试产生消极的影响，特别是在它们修改了全局状态的情况下，比如DOM。我们需要一种方式，标记测试是异步的，然后让测试在所有未完成任务执行完毕时自动完成。

```javascript
class TrackTaskZoneSpec {
  constructor(done) {
    this.name = 'TaskTrackingZone';
    this.done = done;
  }

  onHasTask(delegate, current, target, hasTaskState) {
    if (!hasTaskState.microTask && !hasTaskState.macroTask) {
      this.done();
    }
  }
}

function async(fn) {
  return function(done) {
    Zone.current.fork(new TrackTaskZoneSpec(done)).run(fn);
  }
}

it('should auto-wait for async test', async(() => {
  setTimeout(() => {
    Promise.resolve(0).then(() => {
      // 只有在这行代码执行之后，测试才会完成
      console.log('wait for me');
    }, 0);
  });
}));
```

&emsp;&emsp;**核心点：**

* 这样，测试会自动等待所有异步任务结束之后再进行下一项测试。
* 当zone在测试完成时进行自动检测的情况下，测试提前结束（调用`done`）就不再是问题了。这样做降低了后续测试受到当前测试消极影响的可能性。
