# 第二章：上下文传递

&emsp;&emsp;上下文传递允许把数据附加到zone里。之后数据随着zone一起在异步调用之间传递。

```javascript
let rootZone = Zone.current;
// 从rootZone分叉，并附加属性
let zoneA = rootZone.fork({name: 'zoneA', properties: {a: 1, b:1}});
// 从rootZone分叉并附加属性, 其中一些zoneA的属性被覆盖
let zoneB = zoneA.fork({name: 'zoneB', properties: {a: 2}});

// 查询zoneA，返回了期望的值
expect(zoneA.get('a')).toEqual(1);
expect(zoneA.get('b')).toEqual(1);

//查询zoneB，返回了期望的值，特别是zoneB定义了'a'为2.
expect(zoneB.get('a')).toEqual(2);
// 查询zoneB中的'b'值，返回的是zoneA的值。因为在zoneB中没有'b'值。
expect(zoneB.get('b')).toEqual(1);
```

&emsp;&emsp;**核心点：**

* 附加在zone内的数据是浅-不变的（shallow-immutable）。（一旦zone分叉，它的睡醒就不能变了。）
* 子zone继承父zone的属性。（一段代码，运行在子zone的行为应当与运行在父zone时一致，只要子zone没有显式地修改这段代码依赖的值。）
* 因为zone在异步操作之间流动，对应的属性也随着zone流动，形成了一种形式的通信。
