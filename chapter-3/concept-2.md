# 概念 2：拦截

&emsp;&emsp;跟之前一样，我们来看同一个的例子，但是这一次使用了合理的`Zone.prototype.fork()` API

```javascript
let timingZone = Zone.current.fork({
  name: 'timingZone', 
  onInvoke: function(parentZoneDelegate, currentZone, targetZone,
               callback, applyThis, applyArgs, source) {
    var start = performance.now();
    parentZoneDelegate.invoke(
       targetZone, callback, applyThis, applyArgs, source);
    var end = performance.now();
    console.log(
      'Zone:', targetZone.name,
      'Intercepting zone:', currentZone.name,
      'Duration:', end - start);  
  }
};
let logZone = timingZone.fork({
  name: 'logZone',
  onInvoke: function(parentZoneDelegate, currentZone, targetZone,
               callback, applyThis, applyArgs, source) {
    console.log(
      'Zone:', targetZone.name,
      'Intercepting zone:', currentZone.name,
      'enter');
    parentZoneDelegate.invoke(
       targetZone, callback, applyThis, applyArgs, source);
    console.log(
      'Zone:', targetZone.name,
      'Intercepting zone:', currentZone.name,
      'leave');
  }
});
let appZone = logZone.fork({name: 'appZone'});

appZone.run(function myApp() {
  console.log('Zone:', Zone.current.name, 'Hello World!');
});
```

&emsp;&emsp;输出的结果是：

```log
Zone: appZone Intercepting zone: logZone enter
Zone: appZone Hello World! 
Zone: appZone Intercepting zone: timingZone duration 0.123
Zone: appZone Intercepting zone: logZone leave
```

&emsp;&emsp;**核心点：**

* 因为设计时父zone是未知的，所以继承是不管用的。
* `ZoneDelegate`允许在保持正确的zone上下文的情况下进行拦截。

&emsp;&emsp;使用promise的例子：

```javascript
let logZone = Zone.current.fork({
  name: 'logZone',
  onInvoke: function(parentZoneDelegate, currentZone, targetZone,
               callback, delegate, applyThis, applyArgs, source) {
    console.log(targetZone.name, 'enter');
    parentZoneDelegate.invoke(
       targetZone, callback, applyThis, applyArgs, source)
    console.log(targetZone.name, 'leave');
  }
});

logZone.run(function myApp() {
  console.log(Zone.current.name, 'queue promise');
  Promise.resolve('OK').then((v) =>
    console.log(Zone.current.name, 'Promise', v));
});
```

&emsp;&emsp;输出的结果是：

```log
logZone enter
logZone queuePromise
logZone leave
logZone enter
logZone Promise OK
logZone leave
```

&emsp;&emsp;**核心点：**

* Zone进入了两次。第一次是为了解析promise，第二次是为了调用then回调。
