# 基础知识

&emsp;&emsp;JavaScript虚拟机嵌入在例如浏览器或是Node一样的宿主环境中，藉由任务（tasks）负责调度JavaScript的执行。Zone是可以在异步任务间持久化的执行上下文，它允许创建者观察、控制在zone里的代码的执行情况。

&emsp;&emsp;Zone负责：

* 在异步任务执行过程中持久化zone
* 暴露宿主环境中任务调度与处理过程的可见性（范围仅限当前zone）
