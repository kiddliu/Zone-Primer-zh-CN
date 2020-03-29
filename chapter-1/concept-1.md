# 概念 1: 进入zone, 创建zone与堆栈帧

&emsp;&emsp;首先需要了解的是zone是如何创建（fork）的，在系统内是如何传递的，以及如何在心智模型中想象zone与堆栈帧。

```javascript
// RootZone在这里指代不清，与没有zone的情况没有区别。
let rootZone = Zone.current;
// 通过分叉，我们从一个已有的zone创建一个新的zone。
let zoneA = rootZone.fork({name: 'zoneA'});

// 为方便调试，每个zone都有名字
expect(rootZone.name).toEqual('<root>');
expect(zoneA.name).toEqual('zoneA');
// 子zone知道父zone。 （单向指向）
expect(zoneA.parent).toBe(rootZone);


function main() {
  // zone只可以使用`run`、`runGuarded`或`runTask`方法进入或是退出。
  zoneA.run(function fnOuter() {
    // 在`run`方法内部，Zone.current被更新了。
    expect(Zone.current).toBe(zoneA);
    // 心智模型: 每个堆栈帧都与zone连结了起来。
    expect(Error.captureStackTrace()).toEqual(outerLocation)

    // zone嵌套的方式与堆栈帧的嵌套方式一摸一样。
    rootZone.run(function fnInner() {
      // 内层堆栈帧必须是父堆栈帧zone的子堆栈帧，没有什么为什么。
      // 下边是"逃离"zone的方法。
      expect(Zone.current).toBe(rootZone);
      expect(Error.captureStackTrace()).toEqual(innerLocation)
    });
  });
}

main();
```

&emsp;&emsp;核心点：

* 对`Zone.current`赋值是一个运行时错误。改变`Zone.current`的唯一方式是通过`Zone.prototype.run()`、`Zone.prototype.runGuarded()`或者`Zone.prototype.runTask()`。在之后的内容中，方便起见我们只会使用`Zone.prototype.run`。
* 一个给定的堆栈帧有且只有一个对应的zone。之上或之下的堆栈帧一定有一样的zone，除非这一帧是`Zone.prototype.run()`。
* 子zone指向父zone（但是父zone不指向子zone）。只有与父zone的关系使得zone在垃圾回收时只需要释放zone的引用。

## 堆栈帧

&emsp;&emsp;理解一个给定的堆栈帧有且只有一个对应的zone很重要（比如，函数的上半部分与下半部分运行在不同的zone中是不可能的。而同一个函数由于调用方式的不同有不同的zone是完全可能的）。只能通过进入或是退出`Zone.prototype.run()`进入或是离开zone。为了更好的可见性，zone会更新堆栈帧，显示相关的zone。下边是来自上边示例的两个堆栈快照，其中的每个堆栈帧都显示了对应的zone。

```log
outerLocation:
  at fnOuter()[zoneA];
  at Zone.prototype.run()[<root> -> zoneA]
  at main()[<root>]
  at <anonymous>()[<root>]


innerLocation:
  at fnInner()[<root>];
  at Zone.prototype.run()[zoneA -> <root>]
  at fnOuter()[zoneA];
  at Zone.prototype.run()[<root> -> zoneA]
  at main()[<root>]
  at <anonymous>()[<root>]
```
