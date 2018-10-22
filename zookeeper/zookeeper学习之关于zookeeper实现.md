# 一、系统模型

## 1、数据模型

**事物ID：** 在zookeeper中，事物是指能够改变zookeeper服务器状态的操作，我们也称之为事物操作或更新操作，一般包括数据节点创建与删除、数据节点内容更新和客户端会话创建与失效等操作

## 2、节点特性

### 1）节点类型

在zookeeper中，节点类型可以分为持久节点(PERSISTENT)、临时节点(EPHEMERAL)和顺序节点(SEQUENTIAL)三大类

在节点具体的创建过程中，通过组合使用，可以生成以下四种组合型节点类型：

- **持久节点：**

  所谓持久节点，是指该数据节点被创建后，就会一直存在于zookeeper服务器上，直到有删除操作来主动清除这个节点

- **持久顺序节点**

  持久顺序节点的基本特性和持久节点是一致的，额外的特性表现在顺序性上。在zookeeper中，每个父节点都会为**它的第一级子节点**维护一份顺序，用于记录下每个子节点创建的先后顺序。zookeeper会自动为给定节点名加上一个数字后缀，作为一个新的、完整的节点名。另外，**这个数字后缀的上限是整形的最大值**

- **临时节点**

  临时节点的生命周期和客户端的会话绑定在一起，也就是说，如果客户端会话失效，那么这个节点就会被自动清理掉。这里提到的客户端会话失效，不是TCP连接断开

  zookeeper规定了不能基于临时节点来创建子节点，即临时节点只能作为叶子节点

- **临时顺序节点**

  临时顺序节点的特性和临时节点也是一致的，同样是在临时节点的基础上，添加了顺序的特性

### 2）状态信息

zookeeper中的数据节点，，除了数据节点的数据内容外，还包含节点的状态信息

| 状态属性       | 说明                                                         |
| -------------- | :----------------------------------------------------------- |
| czxid          | 数据节点被创建时的事物ID                                     |
| mzxid          | 该节点最后一次被更新时的事物ID                               |
| ctime          | 标识节点被创建的时间                                         |
| mtime          | 该节点最后一次被更新的时间                                   |
| version        | 数据节点的版本号                                             |
| cversion       | 子节点的版本号                                               |
| aversion       | 节点的ACL版本号                                              |
| ephemeralOwner | 创建该临时节点的会话ID，如果该节点是持久节点，则这个属性的值为0 |
| dataLength     | 数据内容的长度                                               |
| numChildren    | 当前节点的子节点个数                                         |
| pzxid          | 该节点的子节点列表最后一次被修改时的事物ID，注：子节点内容变更不影响该值 |

## 3、版本-保证分布式数据原子性操作

zookeeper中为数据节点引入了版本的概念，每个数据节点都具有三种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化

| 版本类型 | 说明                             |
| -------- | -------------------------------- |
| version  | 当前数据节点**数据内容的版本号** |
| cversion | 当前数据节点子节点的版本号       |
| aversion | 单签数据节点ACL变更版本号        |

zookeeper中的版本概念和传统意义上的软件版本有很大的区别，它表示的是对数据节点的数据内容、子节点列表、或是节点ACL信息的修改次数

注：version版本号表示的是对数据节点数据内容变更的次数，强调的是变更次数，因此即使前后两次变更并没有使得数据内容的值发生变化，version的值依然会变更

## 4、Watcher—数据变更通知

**watcher的工作机制：**

客户端在向zookeeper服务器注册watcher的同时，会将watcher对象存储在客户端的WatcherManager中。当zookeeper服务器端触发watcher事件后，会向客户端发送通知，客户端线程从WatcherManager中取出对应的watcher对象来执行回调逻辑

**watcher的几个事件类型说明：**

- NodeDataChanged（数据节点的数据内容发生变更）：此处说的变更包括节点的数据内容和数据版本号
- NodeChildrenChanged（数据节点的子节点列表发生变更）：这里说的子节点列表的变化特指子节点个数和组成情况的变更，即新增子节点或删除子节点，而**子节点内容的变化是不会触发这个事件的**

**关于回调：**

- WatchedEvent：是一个逻辑事件，用于服务端和客户端程序执行过程中所需的逻辑对象
- WatcherEvent：其和WatchedEvent表示的是同一个事物，都是对一个服务端事件的封装，但其因为实现了序列化接口，因此可以用于网络传输

WatcherEvent存在的原因：服务端在生成WatchedEvent事件之后，会调用getWrapper方法将自己包装成一个可序列化的WatcherEvent事件，以便通过网络传输到客户端。客户端在接收到服务端这个事件对象后，首先会将WatcherEvent事件还原成一个WatchedEvent事件，并传递给process方法处理，回调方法process根据入参就能够解析出完整的服务端事件了

客户端无法直接从该事件中获取到对应数据节点的原始数据内容以及变更后的新数据内容，而是需要客户端再次主动去重新获取数据—这也是zookeeper watcher机制的一个非常重要的特性

**watcher特性总结：**

- 一次性，无论是客户端还是服务端，一旦一个Watcher被触发，zookeeper都会将其从相应的存储中移除
- 客户端串行执行，客户端回调的过程是一个串行同步的过程，这为我们保证了顺序
- 轻量，WatchedEvent是zookeeper整个watcher通知机制的最小通知单元，这个数据结构中，只包含了三部分内容：通知状态、事件类型和节点路径

另外，客户端向服务端注册watcher的时候，并不会把客户端真实的watcher对象传递到服务端，仅仅在客户端请求中使用boolean类型属性进行标记，同时服务端也仅仅只是保存了当前连接的ServerCnxn对象

## 5、ACL—保障数据的安全

ACL，即访问控制列表，包括三部分：权限模式（Scheme）、授权对象（ID）和权限（Permission），通常使用“scheme : id :permission”来标识一个有效的ACL信息

#### 1）ACL的组成：

##### (1)权限模式：Scheme

权限模式用来确定权限验证过程中使用的检验策略，在zookeeper中，使用最多的就是以下四种权限模式：

###### a、IP

IP模式通过IP地址粒度来进行权限控制，如：“ip:127.0.0.1”，即表示权限控制都是针对这个ip地址的。同时，IP模式也支持按照网段的方式进行配置，例如：“ip:192.168.1/24”，表示针对192.168.1.*这个IP段进行权限控制

###### b、Digest

Digest是最常用的权限控制模式，其以类似于"username:password"的形式的权限标识来进行权限配置。当我们通过"username:password"形式配置了权限标识后，zookeeper会对其先后进行两次编码处理，分别是SHA-1算法加密和BASE64编码

###### c、World

World是一种最开发的权限控制模式，这种权限控制模式没有任何作用，数据节点的访问权限对所有的用户开放

###### d、Super

Super模式，顾名思义就是超级用户的意思，是一种特殊的Digest模式，，在Super模式下，超级用户可以对任意zookeeper上的数据节点进行任何操作

##### (2）授权对象：ID

授权对象指的是权限赋予的用户或一个指定的实体。在不同的权限模式下，授权对象是不同的

下表是权限模式和授权对象的对应关系：

| 权限模式 | 授权对象                                         |
| -------- | ------------------------------------------------ |
| IP       | 通产是一个IP地址或IP段                           |
| Digest   | 自定义，通常是"username:BASE61(SHA-1(password))" |
| World    | 只有一个ID：“anyone”                             |
| Super    | 与Digest模式一致                                 |

##### (3）权限：Permission

权限就是指那些通过权限检查后可以被允许执行的操作。在zookeeper中，所有对数据的操作权限分为以下五大类

- CREATE（C）：数据节点的创建权限，允许授权对象在该数据节点下创建子节点
- DELETE（D）：**子节点**的删除权限，允许授权对象删除该数据节点的子节点
- READ（R）：数据节点的读取权限，允许授权对象**访问该数据节点**并**读取其数据内容**或**子节点列表**等
- WRITE（W）：数据节点的更新权限，允许授权对象对该数据节点进行更新操作
- ADMIN（A）：数据节点的管理权限，允许授权对象对该数据节点进行ACL相关设置操作

#### **2)权限扩展体系：**

zookeeper允许开发人员通过制定的方式对zookeeper的权限进行扩展：

**实现自定义权限控制器：**

可以通过实现zookeeper提供的AuthenticationProvider接口来实现自定义权限控制器

**注册自定义的权限控制器：**

zookeeper支持通过系统属性和配置文件两种方式来注册自定义的权限控制器。

系统属性-Dzookeeper.authProvier.X(其中，X代表序号)，如下：

```
-Dzookeeper.authProvieer.1=com.zk.CustomerAuthenticationProvicer
```

配置文件的方式：在zookeeper的配置文件zoo.cfg中配置类似如下配置项：

```
authProvider.1=com.zk.CustomerAuthenticationProvicer
```

权限控制器的注册，zookeeper采用延迟加载的策略，即只有在第一次处理包含权限控制的客户端请求时，才会进行权限控制器的初始化。

#### 3)ACL的管理

设置ACL，可以通过两种方式进行ACL设置，一种是在数据节点创建的同时进行ACL权限的设置，命令格式如下：

```
create [-s] [-e] path data acl
```

另一种方式则是使用setAcl命令单独对已经存在的数据节点进行ACL设置：

```
setAcl path acl
```

#### 4)Super模式的用法

如果一个持久节点包含了ACL权限控制，而其创建者客户端已经退出或已不再使用，那这些数据节点该如何清理呢？这个时候，就需要在ACL的Super模式下，使用超级管理员权限来进行处理了。要使用超级管理员权限，首先需要在zookeeper服务器上开启Super模式，方法是在zookeeper服务器启动的时候，添加如下系统属性：

```
-Dzookeeper.DigestAuthenticationProvicer.superDigest=username:password
```

# 二、客户端

zookeeper的客户端主要由以下几个核心组件组成

- **ZooKeeper实例：** 客户端的入口
- **ClientWatchManager：** 客户端Watcher管理器
- **HostProvider：** 客户端地址列表管理器
- **ClientCnxn：** 客户端核心线程，其内部又包含两个线程，即SendThread和EventThread，前者是一个I/O线程，主要负责ZooKeeper客户端和服务端之间的网络I/O通信；后者是一个事件线程，主要负责对服务端事件进行处理

**客户端的整个初始化和启动过程大体可以分为以下三个步骤：**

- 设置默认的Watcher
- 设置ZooKeeper服务器地址列表
- 创建ClientCnxn

如果再ZooKeeper的构造方法中传入了一个Watcher对象的话，那么ZooKeeper就会将这个Watcher对象保存在ZKWatchManager的defaultWatcher中，作为整个客户端回话期间的默认Watcher

## 1、一次会话的创建过程

**初始化阶段：**

**1)初始化ZooKeeper对象**

在zookeeper的初始化过程中，会创建一个客户端的Watcher管理器：ClientWatchManager

**2)设置回话默认Watcher**

如果在构造方法中传入了一个Watcher对象，那么客户端会将这个对象作为默认Watcher保存在ClientWatchManager中

**3)构造ZooKeeper服务器地址列表管理器：HostProvider**

对于构造方法中传入的服务器地址，客户端会将其存放在服务器地址列表管理器HostProvider中

**4)创建并初始化客户端网络连接器：ClientCnxn**

ZooKeeper客户端首先会创建一个网络连接器ClientCnxn，用来管理客户端与服务器的网络交互。另外，客户端在创建ClientCnxn的同时，还会初始化客户读研的两个核心队列**outgoingQueue**和**pendingQueue**，分别作为客户端的请求发送队列和服务端响应的等待队列

**5)初始化SendThread和EventThread**

客户端会创建两个核心网络线程SendThread和EventThread，前者用于管理客户端和服务端的所有网络I/O，后者则用于进行客户端的事件处理。同时，客户端还会降ClientCnxnSocket分配给SendThread作为底层网络I/O处理器，并初始化EventThread的待处理事件队列waitingEvents，用于存放素有等待被客户端处理的事件

**会话的创建阶段：**

**6)启动SendThread和EventThread**

SendThread首先会判断当前客户端的状态，进行一系列清理性工作，为客户端发送“会话创建”请求做准备

**7)获取一个服务器地址**

在开始创建TCP连接之前，SendThread首先需要获取一个ZooKeeper服务器的目标地址，这通常是从HostProvider中随机获取一个地址，然后委托给ClientCnxnSocket去创建与ZooKeeper服务器之间的TCP连接

**8)创建TCP连接**

获取一个服务器地址后，ClientCnxnSocket负责和服务器创建一个TCP长连接

**9)构造ConnectRequest请求**

在TCP连接创建完毕之后，SendThread会负责根据当前客户端的实际设置，构造出一个ConnectRequest请求，该请求代表了客户端试图与服务器创建一个会话。同事，ZooKeeper客户端还会进一步将该请求包装成网络I/O层的Packet对象，放入请求发送队列outgoingQueue中去

**10)发送请求**

当客户端请求准备完毕后，就可以开始向服务端发送请求了。ClientCnxnSocket负责从outgoingQueue中取出一个待发送的Packet对象，将其序列化成ByteBuffer后，向服务端进行发送

**响应处理阶段：**

**11)接受服务端响应**

ClientCnxnSocket接收到服务端的响应后，会首先判断当前的客户端状态是否是”已初始化“，如果尚未完成初始化，那么就认为该响应一定是会话创建请求的响应，直接交由readConnectionResult方法来处理

**12)处理Response**

ClientCnxnSocket会对接收到的服务端响应进行反序列化，得到ConnectionResponse对象，并从中或得到ZooKeeper服务端分配的会话sessionId

**13)连接成功**

连接成功后，一方面需要通知SendThread线程，进一步对客户端进行会话参数的设置，包括readTimeout和connectionTimeout等，并更新客户端状态；另一方面，需要通知地址管理器HostProvider当前成功连接的服务器地址

**14)生成事件：SyncConnected-None**

为了能让上层应用感知到会话的成功创建，SendThread会生成一个时间SyncConnected-None，代表客户端与服务器会话创建成功，并将该时间传递给EventThread线程

**15)查询Watcher**

EventThread线程收到事件后，会从ClientWatchManager管理器中查询出对应的Watcher，针对SyncConnected-None事件，那么就直接找出步骤2)中存储的默认Watcher，然后将其放到EventThread的waitingEvents队列中去

**16)处理事件**

EventThread不断地从waitingEvents队列中取出待处理的Watcher对象，然后直接调用该对象的process接口方法，以达到触发Watcher的目的

## 2、服务器地址列表

这里需要了解的是，ZooKeeper如何从这个服务器列表中选择服务器机器的呢？是按顺序访问，还是随机访问呢？

### 1)Chroot：客户端隔离命名空间

Chroot允许每个客户端为自己设置一个命名空间。如果一个ZooKeeper客户端设置了Chroot，那么该客户端对服务器的任何操作，都将会被现在在其自己的命名空间下

客户端可以通过在connectionString中添加后缀的方式来设置Chroot：

```
192.168.0.1:2181,192.168.0.1:2182/apps/X
```

### 2)HostProvider：地址列表管理器

ZooKeeper的服务器地址保存在一个List集合中，并且这些地址顺序被随机打散，然后将这些地址拼装成一个环形的循环队列，这个环形队列包含两个游标：currentIndex和lastIndex。currentIndex标识循环队列中当前遍历到的那个元素的位置，lastIndex则标识当前正在使用的服务器地址位置。初始化的时候，currentIndex和lastIndex的值都为-1。在每次尝试获取一个服务器地址的时候，都会首先将currentIndex游标向前移动1位，如果发现游标移动超过了整个地址列表的长度，那么就重置未为0，回到开始的位置重新开始，这样一来，就实现了循环队列

## 3、zookeeper中的各服务器角色

zookeeper集群中，分别有Leader、Follower和Observer三种类型的服务器角色

### 1）Leader

Leader服务器是整个zookeeper的集群工作机制的核心，其主要工作有以下两个：

- 事物请求的唯一调度和处理者，保证集群事物处理的顺序性
- 集群内部各服务器的调度者

**什么是事物请求？**

在zookeeper中，我们将那些会改变服务器状体啊的请求称为“事物请求”——通常指的就是那些创建节点、更新数据、删除节点以及创建会话等请求。

### 2）Follower

从角色名字上可以看出，Follower服务器是zookeeper**集群状态的跟随者**，其主要工作有以下 三个：

- 处理客户端的非事物请求，转发事物请求给Leader服务器
- 参与事物请求Proposal的投票
- 参与Leader选举投票

### 3）Observer

Observer时zookeeper自3.0.0版本开始运入的一个全新的服务器角色。从字面意思看，该服务器充当了一个观察者的角色——其观察zookeeper集群的最新状态变化并将这些状体的变更同步过来。Observer服务器在工作原理上和Follower基本时一致的，对于非事物请求，都可以进行独立处理，而对于事物请求，则会转发给Leader服务器进行处理。**其和Follower服务器的唯一区别在于，Observer不参与任何形式的投票，包括事物请求Proposal的投票和Leader选举投票。简单地讲，Observer服务器只提供非事物服务，通常用于在不影响集群事物处理能力的前提下提升集群的非事物处理能力**

## 4、zookeeper集群间通信

学了这么久zookeeper，了解了很多内部处理机制以及工作原理，但是集群间是根据什么规则通信的还是比较好奇，现在开始学习这块。

zookeeper的消息类型大体上可以分为四类：**数据同步型、服务器初始化型、请求处理型和会话管理型**

### 1）数据同步型

数据同步型消息是指在Learner和Leader扶额 u 为 i进行数据同步的时候，网络通信所用到的消息，通常有DIFF、TRUNC、SNAP和UPTODATE四种

### 2）服务器初始化型

服务器初始化型消息是指在整个集群或是某些新机器初始化的时候，Leader和Learner之间相互通信所使用的消息类型，常见的有OBSERVERINFO、FOLLOWERINFO、LEADERINFO、ACKEPOCH和NEWLEADER五种

### 3）请求处理型

请求处理型消息是指在进行请求处理的过程中，Leader和Learner服务器之间互相通信所使用的消息常见的有REQUEST、PROPOSAL、ACK、COMMIT、INFORM和SYNC六种

### 4）会话管理型

会话管理型消息是指zookeeper在进行会话管理的过程中，和Leader扶额 u 为 i之间相互通信所使用的消息，常见的有PING和REVALIDATE两种

**注：什么是提案？**

所谓提案，是zookeeper中针对事物请求所展开的一个投票流程中对事物操作的包装

**zookeeper中的Proposal流程：**

zookeeper的实现中，每一个事物请求都需要及群众过半及其投票认可才能被真正应用到zookeeper的内存数据库中去，**这个投票与统计过程被称为“Proposal流程”**

## 5、SetData请求

服务端对于SetData请求的处理，大体可以分为4大步骤，分别是请求的预处理、事物处理、事物应用和请求的响应
