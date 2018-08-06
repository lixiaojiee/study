# 一、Executor框架的结构

Executor框架主要由3大部分组成：

1、任务。包括被执行任务需要实现的接口：Runnable和Callable

2、任务的执行。

3、异步计算结果。

![](/pictures/Executor.png)

# 二、ScheduledThreadPoolExecutor

## 1、ScheduledThreadPoolExecutor的执行步骤

1）线程首先从DelayQueue重获取已到期的ScheduledFutureTask

2）线程执行这个ScheduledFutureTask

3）线程修改ScheduledFutureTask的time变量为下次将要被执行的时间

4）线程把这个修改time后的ScheduledFutureTask放回到DelayQueue中

## 2、DelayQueue的take方法执行步骤

1）获取Lock

2）获取周期任务

​	a、如果PriorityQueue为空，当前线程到Condition中等待

​	b、如果PriorityQueue的头元素的time时间比当前时间大，到Condition中等待到time时间

​	c、获取PriorityQueue的头元素，如果PriorityQueue不为空，则唤醒在Condition中等待的所有线程

3）释放Lock

## 3、FutureTask

FutureTask可以处于下面三种状态：

1）未启动。FutureTask.run()方法还没有被执行之前，FutureTask处于未启动状态

2）已启动。FutureTask.run()方法被执行的过程中，FutureTask处于已启动状态

3）已完成。FutureTask.run()方法执行完成后正常结束，或被取消，或执行FutureTask.run()方法时抛出异常而异常结束，FutureTask处于已完成状态

当FutureTask处于未启动或启动状态时，执行FutureTask.get()方法将导致调用线程阻塞；

当FutureTask处于已完成状态时，执行FutureTask.get()方法将导致调用线程立即返回结果或抛出异常

当FutureTask处于未启动状态时，执行FutureTask.cancel()方法将导致此任务永远不会被执行

当FutureTask处于已启动状态时，执行FutureTask.cancel(true)方法将以中断执行此任务线程的方式来试图停止任务

当FutureTask处于已启动状态时，执行FutureTask.cancel(false)方法将不会对正在执行此任务的线程产生影响（郑子啊执行的任务将会运行完成）

当FutureTask处于已完成状态时，执行FutureTask.cancel()方法将返回false