# 一、关于Buffer

`Buffer`类是一个抽象类，它具有七个直接子类，分别是`ByteBuffer`、`CharBuffer`、`DoubleBuffer`、`FloatBuffer`、`IntBuffer`、`LongBuffer`和`ShortBuffer`，也就是缓冲区中存储的类型并不像普通I/O流只能存储byte或者char类型数据，Buffer类能存储的类型是多样的。

在java的nio包中并不包含StringBuffer缓冲区，**在nio中存储自负的缓冲区可以使用CharBuffer类**

**注：** `Buffer`不存储`Boolean`类型的数据

`Buffer`类的七个直接子类并不能使用`new`关键字直接实例化，而是将七种数据类型的数组包装进缓冲区中，此时就需要借助静态方法`wrap()`进行实现。`wrap()`方法的作用是将数组放入缓冲区中，来构建存储不同数据类型的缓冲区

**注：** 缓冲区是非线程安全的

# 二、包装数据与获得容量

在nio技术的缓冲区中，存在4个核心的技术点，分别是：

- capcacity（容量）
- limit（限制）
- position（位置）
- mark（标记）

这四个技术点之间的值的大小关系如下：

​							$0<=mark<=position<=limit<=capacity$	

## 1、capacity

capacity代表缓冲区包含元素的数量。缓冲区的capacity不能为负值，并且capacity也不能更改

## 2、限制获取与设置

什么是限制？

缓冲区中的限制代表第一个不应该读取或写入的元素的index（索引）。**缓冲区的限制不能为负，并且limit不能大雨其capacity**。**如果position大于新的limit，则将position设置为新的limit**。**如果mark已定义切大于新的limit，则丢弃该mark**

limit的使用场景就是当反复地向缓冲区中存取数据时使用，比如第一次向缓冲区中存入9个数据A、B、C、D、E、F、G、H、I。然后读取全部9个数据，完成后再进行第2次向缓冲区中存储数据，第2次只存储4个数据，分别是1、2、3、4，当读取时却出了问题，如果读取全部数据1、2、3、4、E、F、G、H、I时就是错误的，所以要结合limit来限制读取的范围，在E处设置limit，从而实现了只能读取1、2、3、4这四个正确的数据

## 3、位置的获取与设置

什么是位置？

它代表“下一个”要读取或者写入元素的index（索引），**缓冲区的position（位置）不能为负**，并且**position不能大于其limit**。**如果mark已定义且大于新的position，则丢弃该mark**。

**Buffer中默认的position为0**

## 4、甚于空间大小的获取

Buffer中剩余空间的获取，可以通过remaining()方法获取，其返回当前位置（position）与limit之间的元素数

## 5、使用Buffer mark()方法处理标记

方法`mark()`的作用：再次缓冲区的位置设置标记

标记有什么作用呢？

缓冲区的标记是一个索引，在调用`reset()`方法时，会将缓冲区的position位置重置为该索引。标记并不是必须的。定义mark时，并不能将其定义为负数，并且不能让它大于position。如果定义了mark，则在将position或limit调整为小于该mark的值时，该mark被丢弃，丢弃后mark的值是-1.如果未定义mark，那么调用`reset()`方法将导致抛出InvalidMarkException一异常

## 6、直接缓冲区

通过ByteBuffer向硬盘存取数据时时需要将数据暂存在JVM的中间缓冲区的，如果有频繁操作数据的情况发生，则在每次操作时都会将数据暂存在JVM的中间缓冲区，再交给ByteBuffer处理，这样就大大降低了软件的吞吐量，提高内存占有率，导致软件运行效率降低，这就是非直接缓冲区保存数据的过程，使用直接缓冲区，可以解决非直接缓冲区的这个问题

如果使用直接缓冲区实现两端的数据交互，则直接在内核空间中就进行了处理，无需JVM创建新的缓冲区，这样就减少了在JVM中创建中间缓冲区的步骤，增加了程序的运行效率

Buffer中可以通过`allocateDirect(capacity)`方法来创建直接缓冲区

## 7、还原缓冲区的状态

还原缓冲区到初始状态，包括将位置设置为0，将限制设置为容量，并丢弃标记，即“一切为默认”

Buffer中可以使用`clear()`方法实现

