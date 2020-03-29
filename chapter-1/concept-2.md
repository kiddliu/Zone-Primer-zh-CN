# 概念 2: 跟踪异步操作

&emsp;&emsp;我们了解了zone是如何进入退出的，现在我们来了解一下如何在异步操作之间保持当前zone。

```javascript
let rootZone = Zone.current;
let zoneA = rootZone.fork({name: 'A'});

expect(Zone.current).toBe(rootZone);
// 当前执行setTimeout调用的zone是rootZone
expect(Error.captureStackTrace()).toEqual(rootLocation)
setTimeout(function timeoutCb1() {
  // 回调是执行在rootZone内的
  expect(Zone.current).toEqual(rootZone);
  expect(Error.captureStackTrace()).toEqual(rootLocationRestored)
}, 0);


zoneA.run(function run1() {
  expect(Zone.current).toEqual(zoneA);
  // 当前执行setTimeout调用的zone是zoneA
  expect(Error.captureStackTrace()).toEqual(zoneALocation)
  setTimeout(function timeoutCb2() {
    //回调是执行在zoneA内的
    expect(Zone.current).toEqual(zoneA);
    expect(Error.captureStackTrace()).toEqual(zoneALocationRestored)
  }, 0);
});
```

```log
rootLocation:
  at <anonymous>()[rootZone]

rootLocationRestored:
  at timeoutCb1()[rootZone]
  at Zone.prototype.run()[<root> -> <root>]

zoneALocation:
  at run1()[zoneA]
  at Zone.prototype.run()[<root> -> zoneA]
  at <anonymous>()[rootZone]

zoneALocationRestored:
  at timeoutCb2()[zoneA]
  at Zone.prototype.run()[<root> -> zoneA]
```

&emsp;&emsp;**核心点：**

* 当异步任务被调度时，回调函数会执行在调用这个异步API时就存在的同一个zone内。这使得zone可以在多个异步调用之间被跟踪。

&emsp;&emsp;如果使用promise的话。（由于在回调中处理异常，Promise于是有些不同。）

```javascript
let rootZone = Zone.current;
// libZone代表了某些不受开发者控制的第三方库
// 这里的zone只是起到说明作用
// 在实际中大多数第三方库不可能有这样精细的zone控制
let libZone = rootZone.fork({name: 'libZone'});
// 代表了开发者控制的app的zone。
let appZone = rootZone.fork({name: 'appZone'});
let appZone1 = rootZone.fork({name: 'appZone1'});

// 在这个例子中我们想要展示完成promise与监听promise的区别
// Promise的完成可以发生在一个第三方库libZone中
let promise = libZone.run(() => {
  return new Promise((resolve, reject) => {
    expect(Zone.current).toBe(libZone);
    // 在这个例子中，这个promise可以立即返回，也可以稍后再返回
    setTimeout(() => {
      expect(Zone.current).toBe(libZone);
      // Promise是在libZone中完成的，但这对promise的监听者没有影响
      resolve('OK');
    }, 500);
  });
});

appZone.run(() => {
  promise.then(() => {
    // 由于开发者控制着.then()在哪个zone执行
    // 他们会期望回调可以执行在同一个zone，也就是这里的appZone
    expect(Zone.current).toBe(appZone);
  });
});

appZone1.run(() => {
  promise.then(() => {
    // 并且不同的thenCallback可以在不同的zone内调用
    expect(Zone.current).toBe(appZone1);
  });
});
```

&emsp;&emsp;**核心点：**

* 对于promise来说，thenCallback是调用在`.then()`调用时起效的zone内的。
  * 或者，对于thenCallback我们可以使用另一个zone，比如创建promise的zone或是完成promise的zone。这二者都不是一个好选择，毕竟promise可以在第三方库创建、完成，它们也都可以有自己的zone。最终产生的promise将被传入app，app也可以有自己的zone。如果app注册`.then()`在自己的zone，那么它会期望传递的是自己的zone。
  * 举例来说：调用`fetch()`返回promise。在内部`fetch()`由于自己的原因使用了自己的zone。调用`.then()`的应用则会期望应用的zone。（因为我们不想让`fetch()`的zone暴露在我们的应用中）
