# 	一、ConcurrentHashMap

## 1、为什么会有HashMap

1）在并发编程中HashMap可能会导致程序死循环

2）线程安全的HashTable效率非常低下

## 2、ConcurrentHashMap的结构

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁，在ConcurrentHashMap里扮演锁的角色；HashEntry则是存储键值对数据。每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须收看获得与它对应的Segment锁。

concurrentHashMap中相关参数：

segmentShift：segmentShift用于定位参与散列运算的位数，segmentShift等于32减sshift，其中sshift时ssize向左移动的次数

segmentMask：segmentMask是散列运算的掩码，等于ssize-1，其中ssize是Segment数组的大小

## 3、ConcurrentHashMap的操作

### 1）get操作

CHM的get操作的高效之处在于整个get过程不需要加锁，除非读到的值是空才会加锁重读

那么CHM的get操作是如何做到不加锁的呢？原因是它的get方法将要使用的共享变量都定义成volatile类型，如用于 统计当前Segment大小的count字段和用于存储值的HashEntry的value，定义成volatile的共享变量，能够在线程之间保持可见性能够被多线程同时读，并且保证不会读到过期的值，但是只能被单线程写，在get操作离之需要读不需要写共享变量count和value，所以可以不用加锁。之所以不会读到过期的值，是因为根据JMM的happens-before原则，对volatile字段的写入操作先于读操作，即使两个线程同时修改和获取volatile变量，get操作也能拿到最新的值，这是用volatile替换锁的典型场景。

### 2）put操作

因为put操作需要对共享变量进行写入操作，所以为了线程安全，在操作共享变量时必须加锁。put方法先定位到Segment，然后在Segment里进行插入操作，插入操作需要进行两步：

a、判断是否需要对Segment里的HashEntry数组进行扩容

CHM是先判断是否需要扩容，其比HashMap更恰当的是，HashMap是在插入元素后再判断，这样就有可能扩容之后没有新元素插入，这时HashMap就进行了一次无效的扩容，CHM的扩容，不会对整个容器进行扩容，只是对某个Segment进行扩容

b、定位添加元素的位置，然后将其放在HashEntry看数组里

### 3）size操作

CHM通过先尝试两次不锁住Segment的方式来统计各个Segment的大小，如果统计过程中，容器的count发生变化，则再采用加锁的方式来统计所有的Segment的大小

CHM是如何判断在统计的时候容器是否发生了变化呢？答案是使用modCount变量，在put、remove和clean方法里操作元素前都会将变量modCount进行加1，那么统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生了变化

# 二、ConcurrentLinkedQueue

ConcurrentLinkedQueue是一个基于链接节点的误解线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，它会添加到队列的尾部；当我们获取一个元素的时候，它会返回队列头部的元素。它采用“wait-free”算法（即CAS算法）来实现

## 1、入队

入队列就是将入队节点添加到队列的尾部。入队主要做两件事：第一是将入队节点设置成当前队列的尾节点的下一个节点；第二是更新tail节点，如果tail节点的next节点不为空，则将入队节点设置成tail节点，如果tail节点的next节点为空，则将入队节点设置成tail的next节点，所以t==ail节点不总是尾节点==

## 2、出队

# 三、阻塞队列

阻塞队列是一个支持两个附加操作的队列。这两个附加操作支持阻塞的插入和移除方法。

- 支持阻塞的插入方法：意思是当一个队列满时，队列会阻塞插入元素的线程，知道队列不满
- 支持阻塞的移除方法：意思是在队列为空时，获取元素的线程会等待队列变为非空

**阻塞队列插入和删除操作的4种处理方式**

| 方法     | 抛出异常  | 返回特殊值 | 一直阻塞 | 超时退出            |
| -------- | --------- | ---------- | -------- | ------------------- |
| 插入方法 | add(e)    | offer(e)   | put(e)   | offer(e, time,unit) |
| 移除方法 | remove()  | poll()     | take()   | poll(time,unit)     |
| 检查方法 | element() | peek()     | 不可用   | 不可用              |

## 1、Java里的阻塞队列

DelayQueue

DelayQueue是一个支持延时获取元素的无节阻塞队列，队列里的元素必须实现Delayed接口。DelayQueue的使用场景：

- 缓存系统的设计：可以使用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue种获取元素时，表示缓存的有效期到了
- 定时任务调度：使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue种获取到任务就开始执行，比如TimerQueue就是使用DelayQueue实现的

# 四、Fork/Join框架

Fork/Join框架是一个用于并行执行任务的框架。是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架

## 1、工作窃取算法

工作窃取算法是指某个线程从其它队列里窃取任务来执行

## 2、工作窃取算法的优缺点

优点：充分利用线程进行并行计算，减少了线程间的竞争

缺点：在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且该算法会消耗更多的系统资源，比如创建多个线程和多个双端队列

## 3、Fork/Join框架的设计

要实现Fork/Join框架，需要做到下面两步

- 分割任务
- 执行任务并合并结果

Fork/Join通过下面两个类来完成以上两件事情：

- **ForkJoinTask**：要实现Fork/Join框架，首先必须创建一个Fork/Join任务。它提供在任务中执行fork()和join()操作的机制。通常情况下，我们不需要直接继承ForkJoinTask类，只需要继承它们的子类，Fork/Join提供了一下两个子类：

  a、RecursiveAction：用于没有返回结果的任务

  b、RecursiveTask：用于有返回结果的任务

- **ForkJoinPool**：ForkJoinTask需要通过ForkJoinPool来执行。

任务分割出的子任务会添加到当前工作线程锁维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其它工作线程的队列的尾部获取一个任务
