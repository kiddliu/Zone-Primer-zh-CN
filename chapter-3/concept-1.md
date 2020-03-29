# 概念 1：ZoneDelegate场景

&emsp;&emsp;拦截zone时间有些复杂，毕竟zone是在运行时构成的，并且类似创建子类以及本地打补丁等标准方式只有在设计时父zone已知才有效。为了更好地说明这个问题，假设我们想要在`run()`方法执行前与执行后拦截，从而可以度量执行时间并打印zone。

&emsp;&emsp;为了说明问题，下面是一个失败的例子。

```javascript
// 在设计时，是没有办法知道哪一个zone是父zone
// 于是，父zone只能通过构造函数传入
class TimingZone extends Zone {
  constructor(parent) { super(parent, 'timingZone');}

  // 我们想要拦截run，于是我们重写它
  run() {
    // 抓取起始时间
    var start = performance.now();
    // 看上去在这里调用super.run()是对的
    // 但是super.run()内部必然调用parent.run。
    // 所以super.run是不能用的
    // 我们更应该了解parent.run()的影响。参考下一个例子。
    super.run(...arguments);
    // 抓取结束时间
    var end = performance.now();
    // 打印持续时长，以及当前zone
    console.log(this.name, 'Duration:', end - start);
  }
}

// 在设计时，是没有办法知道哪一个zone是父zone
// 于是，父zone只能通过构造函数传入
class LogZone extends Zone {
  constructor(parent) { super(parent, 'logZone');}

  run() {
    // 记录zone名称与'enter'
    console.log(this.name, 'enter');
    // 调用parent.run的问题, 在于它会导致当前zone变为父zone
    // 我们需要的是调用parent run的钩子方法，而不是修改当前zone
    this.parent.run.apply(this, arguments);
    // 记录zone名称与'leave'
    console.log(this.name, 'leave');
  }
}
```

&emsp;&emsp;让我们用上边的zone创建一个简单的例子。

```javascript
// 构建几个zone
let rootZone = Zone.current;
let timingZone = new TimingZone(rootZone);
let logZone = new LogZone(timingZone);

logZone.run(() => {
  console.log(Zone.current.name, 'Hello World!');
});
```

&emsp;&emsp;以下是我们期望从上边例子得到的结果。尤其是期望`run`代码块中当前的zone是那个`logZone`。

```log
logZone enter
logZone Hello World;
logZone Duration: 0.123
logZone leave
```

&emsp;&emsp;以下是我们从上边例子得到的真实结果。

```log
logZone enter
rootZone Hello World;
timingZone Duration: 0.123
logZone leave
```

&emsp;&emsp;注意每一个zone拦截都打印了它们自己的zone名称（而不是原始zone），并且最终run代码块运行在`rootZone`而不是`logZone`。

## 怎么标准工具不管用了

&emsp;&emsp;拦截调用（super或是parent）的标准方式不起作用的原因在于父zone在设计时是未知的。如果设计时父zone是可知的，那么我们可以这样写：

```javascript
class TimingZone extends RootZone {
  run() {
    …
    super.run(...arguments);
    …
  }
}

class LogZone extends TimingZone {
  run() {
    …
    super.run(...arguments);
    …
  }
}
```

&emsp;&emsp;如果我们在设计时可以确定zone的继承关系，那么`super.run()`的调用会如我们期望的方式工作。然而，因为直到运行时才知道父zone，我们需要把改变zone的操作与调用父zone钩子的操作区分开。为了解决这个问题，钩子被描述为`fork()`的一部分，同时钩子接受一个父ZoneDelegate（它只处理钩子）而不是父zone（它会导致zone的变换）。

| Zone | ZoneDelegate | Description |
| - | - | - |
| `run` | `invoke` | 当zone的`run()`代码块被执行时，`ZoneDelegate`的`invoke()`钩子被调用。这使得钩子被代理，而没有该表当前zone。 |
| `wrap` | `intercept` | 当zone的`wrap()`代码块被执行时，`ZoneDelegate`的`intercept()`钩子被调用。这使得钩子被代理，而没有重新包装回调。 |
