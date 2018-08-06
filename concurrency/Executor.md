# 一、Executor框架的结构

Executor框架柱与奥由3大部分组成：

1、任务。包括被执行任务需要实现的接口：Runnable和Callable

2、任务的执行。

3、异步计算结果。

![](/Users/lixiaojie/学习/脑图/图片/Executor.png)

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