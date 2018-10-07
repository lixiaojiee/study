通道在NIO中，就是用来进行数据传输的通路,通道是用于I/O操作的链接，更具体地讲，通道代表数据到硬件设备、文件、网络套接字的链接。通道可处于打开或关闭这两种状态，当创建通道时，通道就处于打开状态，一旦将其关闭，则保持关闭状态。一旦关闭了某个通道，则试图对其调用I/O操作时就会导致ClosedChannelException异常被抛出，但可以通过调用通道的isOpen()方法测试通道是否处于打开状态以避免出现ClosedChannelException异常。**一般情况下，通道对于多线程的访问时安全的**

Channel主要有以下几个实现：

| 实现名称                | 特点                                                         |
| ----------------------- | ------------------------------------------------------------ |
| AsynchronousChannel     | 使通道支持异步I/O操作，线程安全                              |
| AsynchronousByteChannel | 使通道支持异步I/O操作，操作单位为字节，线程不安全            |
| ReadableByteChannel     | 使通道允许对字节进行**读操作**<br />1）将通道当前位置中的字节序列读入一个ByteBuffer中<br />2）`read()`方法是同步的 |
| ScatteringByteChannel   | 使通道中读取字节到多个缓冲区中                               |
| WritableByteChannel     | 使通道允许对字节进行写操作<br />1）将一个字节缓冲区中的字节序列写入通道中的当前位置<br />2）`write()`方法是同步的 |
| GatheringByteChannel    | 使其可以将多个缓冲区总的数据写入到通道中                     |
| ByteChannel             | 其是ReadableByteChannel和WritableByteChannel的规范，支持双向操作，既支持读，也支持写 |
| SeekableByteChannel     | 其主要作用是在字节通道中维护position，以及允许position发生变化 |
| NetworkChannel          | 使通道与Socket进行关联，使通道中的数据能在Socket技术上进行传输 |
| MulticastChannel        | 使通道支持Internet Protocol（IP）多播 ^注1^                  |
| InterruptibleChannel    | 使通道能以异步的方式进行关闭与中断                           |

**注1：** IP多播就是将多个主机地址进行打包，形成一个组，然后将IP报文向这个组进行发送，也就相当于同时向多个主机传输数据