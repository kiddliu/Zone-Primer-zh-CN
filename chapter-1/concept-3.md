# 概念 3：为异步方法打补丁

&emsp;&emsp;接下来我们会了解zone在异步回调之间传递的底层实现。

```javascript
// 保存原始的setTimeout
let originalSetTimeout = window.setTimeout;
// 用一个在zone内包装回调的函数覆盖这个API
window.setTimeout = function(callback, delay) {
  // 调用原始API但是在zone内包装回调
  return originalSetTimeout(
    // 包装回调方法
    Zone.current.wrap(callback),
    delay
  );
}

// 返回回调的包装版本，用来恢复zone
Zone.prototype.wrap = function(callback) {
  // 抓取当前zone
  let capturedZone = this;
  // 返回在zone内执行原始闭包的闭包
  return function() {
    // 在抓取的zone内执行原始回调
    return capturedZone.runGuarded(callback, this, arguments);
  };
};
```

&emsp;&emsp;**核心点：**

* Zone只会打一次本地补丁。
* 进入、离开zone只有修改`Zone.current`值的代价
* `Zone.prototype.wrap`提供了包装回调的便利性。（包装了的回调是通过`Zone.prototype.runGuarded()`执行的）（注意，这里的`Zone.prototype.wrap`只是为了简化实现，目前zone会利用任务调度为所有的异步操作打补丁）
* `Zone.prototype.runGuarded()`与`Zone.prototype.run()`类似，只是多了为了处理异常的try-catch代码块，稍后我们会提到。

```javascript
// 保存原始的Promise.prototype.then.
let originalPromiseThen = Promise.prototype.then;
// 用包装了参数的函数替换这个API
// 注意: 这里简化了，实际API有更多的参数。
Promise.prototype.then = function(callback) {
  // 抓取当前zone
  let capturedZone = Zone.current;
  // 返回在zone内执行原始闭包的闭包
  function wrappedCallback() {
    return capturedZone.run(callback, this, arguments);
  };
  // 在抓取的zone内执行原始回调
  return originalPromiseThen.call(this, [wrappedCallback]);
};
```

&emsp;&emsp;**核心点：**

* Promise处理自己的异常，于是无法使用`Zone.prototype.wrap()`。（我们可以有不同的API，但是promise是这条规则的例外，所以我不觉得创建它自己的API有合理性。）
* Promise API很大，所以在这个例子中没有全部展示出来。
* Zone为微任务的补丁实现了ZoneAwarePromise，于是包装只是为了简化这个概念。
