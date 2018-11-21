# 一、InnoDB体系架构

InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：

- 维护所有进程/线程需要访问的多个内部数据结构
- 缓存磁盘上的数据，方便快速地读取，同时对磁盘文件的数据修改在这里缓存
- 崇左日志缓冲

后台线程主要作用是负责刷新内存池中的数据，宝恒缓冲池中的内存缓存的事最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常的运行状态

## 1、后台线程

InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理不同的任务

### 1）Master Thread

Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、undo页的回收等

### 2）IO Thread

在InnoDB存储引擎中大量使用了AIO来处理IO请求，而IO Thread的工作主要是负责这些IO请求的回调处理,可以通过以下命令查看InnoDB版本号：

```mysql
mysql> show variables like 'innodb_version'\G;
*************************** 1. row ***************************
Variable_name: innodb_version
        Value: 8.0.12
1 row in set (0.02 sec)
```

查看InnoDB引擎中的IO线程数：

```mysql
mysql> show variables like 'innodb_%io_threads'\G;
*************************** 1. row ***************************
Variable_name: innodb_read_io_threads
        Value: 4
*************************** 2. row ***************************
Variable_name: innodb_write_io_threads
        Value: 4
2 rows in set (0.01 sec)
```

可以通过以下命令观察InnoDB中的IO Thread

```mysql
mysql> show engine innodb status\G;
*************************** 1. row ***************************
  Type: InnoDB
  Name:
Status:
=====================================
2018-11-20 23:03:07 0x700004857000 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 21 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 1 srv_active, 0 srv_shutdown, 4474 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 1
OS WAIT ARRAY INFO: signal count 1
RW-shared spins 0, rounds 0, OS waits 0
RW-excl spins 0, rounds 0, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 0.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 19972
Purge done for trx's n:o < 19970 undo n:o < 0 state: running but idle
History list length 2
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 281479732221760, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for i/o request (insert buffer thread)
I/O thread 1 state: waiting for i/o request (log thread)
I/O thread 2 state: waiting for i/o request (read thread)
I/O thread 3 state: waiting for i/o request (read thread)
I/O thread 4 state: waiting for i/o request (read thread)
I/O thread 5 state: waiting for i/o request (read thread)
I/O thread 6 state: waiting for i/o request (write thread)
I/O thread 7 state: waiting for i/o request (write thread)
I/O thread 8 state: waiting for i/o request (write thread)
I/O thread 9 state: waiting for i/o request (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
855 OS file reads, 164 OS file writes, 12 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 3 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number          20534551
Log buffer assigned up to    20534551
Log buffer completed up to   20534551
Log written up to            20534551
Log flushed up to            20534551
Added dirty pages up to      20534551
Pages flushed up to          20534551
Last checkpoint at           20534551
11 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 370735
Buffer pool size   8191
Free buffers       7237
Database pages     950
Old database pages 370
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 818, created 132, written 140
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 950, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=517, Main thread ID=0x700004286000 , state=sleeping
Number of rows inserted 0, updated 311, deleted 0, read 4246
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.00 sec)
```

### 3）Purge Thread

事物被提交后，其所使用的undolog可能不再需要，因此需要Purge Thread来回收已经使用并分配的undo页。用户可以在mysql数据库的配置文件中添加如下命令来启用独立的Purge Thread：

```idl
[mysqld]
innodb_purg_threads=1
```

可以通过如下命令查看当前设置的Purge Thread数：

```mysql
mysql> show variables like 'innodb_purge_threads'\G;
*************************** 1. row ***************************
Variable_name: innodb_purge_threads
        Value: 4
1 row in set (0.00 sec)
```

### 4）Page Cleaner Thread

Page Cleaner Thread的作用是将之前版本中脏页的刷新操作都放入到单独的线程中来完成。而其目的是为了减轻原Master Thread的工作及对与用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能

## 2、内存

### 1）缓冲池

   InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。因此可视其为基于磁盘的数据库系统。基于磁盘的数据库系统通常都是使用缓冲池技术来提高数据库的整体性能

缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响

对于读取操作，首先将从磁盘读取到的页放在缓冲池中，这个过程称为将页“FIX”在缓冲池中，下一次读取相同的页时，首先判断该页是否在缓冲池中，若在缓冲池中，称该页在缓冲池中被命中，直接读取该页，否则读取磁盘上的页

对于修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。这里需要注意的是，页从缓冲池刷新会磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为Checkpoint的机制刷新回磁盘，同样，这也是为了提高数据库的整体性能

综上所述，缓冲池的大小直接影响着数据库的整体性能，对于InnoDB存储引擎来说，其缓冲池的配置通过参数innodb_buffer_pool_siez来设置，可以通过如下命令查看当前的缓冲池大小：

```mysql
mysql> show variables like 'innodb_buffer_pool_size'\G;
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728
1 row in set (0.00 sec)
```

具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo页、插入缓冲、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等

### 2）LRU List、Free List、Flush List

mysql中每个页的大小为16k

在LRU列表中的页被修改后，称该页为脏页，即缓冲池中的页和磁盘上的页的数据产生了不一致。这时数据库会通过CHECKPOINT机制将脏页刷新回磁盘，而Flush列表中的页即为脏页列表。需要注意的是，脏页即存在于LRU列表中，也存在于Flush列表中。LRU列表用来管理缓冲池中的页的可用性，Flush列表用来管理将页刷新回磁盘，二者互不影响

### 3）重做日志缓冲

InnoDB存储引擎首先将崇左日志信息放入到这个缓冲区，然后按一定频率将其刷新到重做日志文件。重做日志在下列三种情况下回将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中：

- Master Thread每一秒将重做日志缓冲刷新到重做日志文件
- 每个事物提交时会将重做日志缓冲刷新到重做日志文件
- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件

### 4）额外的缓冲池

## 3、CheckPoint技术

