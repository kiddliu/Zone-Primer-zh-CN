# Node领域

## [postmortem](https://gist.github.com/trevnorris/d19addf4aa09a0a31a4a14e90cb5c302)

&emsp;&emsp;笔记：

* 摘自Domain[文档](https://nodejs.org/api/domain.html)
  * 如果使用了domain，那么所有新的`EventEmitter`对象（包括`Stream`对象，`request`，`response`等等）都会隐式地绑定到构造时活动的domain上
    * 这我不认为是对的，绑定发生在连接事件监听函数时比`EventEmitter`创建时好。
* Domain的叠加与构建是同步的，可一旦执行了异步函数，只能恢复最顶层的domain。
  * 这与domain是一种zone是一致的，但是domain无法组成。
  * 在讨论zone的时候，区分zone的堆叠与zone的组成是很重要的。两个概念完全无关。请看下边这个例子

```javascript
let logs = [];
let zoneA = Zone.current.fork({
  name: 'zoneA',
  onInvoke: function(delegate, currentZone, targetZone, callback, applyThis, applyArgs) {
    logs.push('zoneA onInvoke');
    return delegate.invoke(targetZone, callback, applyThis, applyArgs);
  }
});
let zoneB = Zone.current.fork({
  name: 'zoneB',
  onInvoke: function(delegate, currentZone, targetZone, callback, applyThis, applyArgs) {
    logs.push('zoneB onInvoke');
    return delegate.invoke(targetZone, callback, applyThis, applyArgs);
  }
});
let zoneAChild = zoneA.fork({
  name: 'zoneAChild',
  onInvoke: function(delegate, currentZone, targetZone, callback, applyThis, applyArgs) {
    logs.push('zoneAChild onInvoke');
    return delegate.invoke(targetZone, callback, applyThis, applyArgs);
  }
});

zoneA.run(() => {
  zoneB.run(() => {
    zoneAChild.run(function test() {
      logs.push('begin run' + Zone.current.name);
      console.log('logs', logs);

      const error = new Error();
      console.log('trace', error.stack);
    });
  });
});
```

&emsp;&emsp;在这个例子中，

1. Zone堆栈会是root -> zoneA -> zoneB -> zoneAChild，于是你可以从`error.stack`中找到zone的切换信息。
2. Zone的组成则是zoneAChild -> zoneA -> rootZone，所以在测试方法中，zoneB的回调是不会触发的。

    * 新的zone是从当前zone分叉出来的，而新的domain的创建是完全隔绝的
