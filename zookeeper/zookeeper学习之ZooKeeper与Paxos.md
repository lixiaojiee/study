# 一、zookeeper的基本概念

## 1、数据节点（Znode）

ZNode是zookeeper数据模型中的数据单元，zookeeper将所有的数据存储在内存中，数据模型是一棵树（ZNode Tree），由斜杠（/）进行分割的路径，就是一个ZNode。在Zookeeper中，ZNode可以分为**持久节点**和**临时节点**两类：

**持久节点：**是指一旦这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在zookeeper上 

**临时节点：**临时节点的生命周期和客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的所有临时节点都会被移除

zookeeper允许用户为每个节点添加一个特殊的属性：`SEQUENTIAL`。一旦 节点被标记上这个属性，那么在这个节点被创建的时候，zookeeper会自动在其节点名后面追加一个整型数字，这个整型数字是一个由父节点维护的自增数字

## 2、版本

Zookeeper的每个ZNode上都会存储数据，对应于每个ZNode，ZooKeeper都会为其维护一个叫做Stat的数据结构，Stat中记录了这个ZNode的三个数据版本，分别为：

- **version：**当前ZNode的版本
- **cversion：**当前ZNode子节点的版本
- **aversion：**当前ZNode的ACL版本

## 3、Watcher

ZooKeeper允许用户在指定节点上注册一些Watcher，并且在一些特定事件触发的时候，ZooKeeper服务端会将事件通知到感兴趣的客户端上去，该机制是ZooKeeper实现分布式协调服务的重要特性

## 4、ACL

ZooKeeper采用ACL（Access Control Lists）策略来进行权限控制，主要定义了如下5种权限：

- **CREATE：**创建子节点的权限
- **READ:**获取节点数据和子节点列表的权限
- **WRITE：**更新节点数据的权限
- **DELETE：**删除子节点的权限
- **ADMIN：**设置节点ACL的权限

其中，DELETE和CREATE这两种权限都是针对子节点的权限控制

# 二、ZooKeeper的ZAB协议

## 1、ZAB协议：

**所有的事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为Leader服务器**，而**余下的其它的服务器则称为Follower服务器**。Leader服务器负责将一个客户端事务请求转换成一个事物Proposal（提议），并将该Proposal分发给集群中所有Follower服务器。之后Leader服务器需要等待所有Follower服务器的反馈，一旦超过半数的Follower服务器进行的正确的反馈后，那么Leader就会再次向所有的Follower服务器分发Commit消息，要求其将前一个Proposal进行提交

## 2、协议介绍

ZAB协议包括两种高级本的模式，分别是**崩溃恢复**和**消息广播**

**崩溃恢复：**当整个服务框架在启动过程中，或是当Leader服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB协议就会进入恢复模式并选举产生新的Leader服务器。当选举产生了新的Leader服务器，同时集群中已经有过半的机器与该Leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式。

所谓的状态同步是指数据同步，用来保证集群中存在过半的机器能够和Leader服务器的数据状态保持一致

**消息广播：**当集群中已经有过半的Follower服务器完成了Leader服务器的状态同步，那么整个服务框架就可以进入消息广播模式了

当一台同样遵守ZAB协议的服务器启动后加入到集群中时，如果此时集群中已经存在一个Leader服务器在负责进行消息广播，那么新加入的服务器就会自觉地进入数据恢复模式：找到Leader所在的服务器，并与其进行数据同步然后一起参与到消息广播流程中去。

## 3、基本特性

**1）ZAB协议需要确保那些已经在Leader服务器上提交的事物最终被所有的服务器都提交**

崩溃恢复过程需要确保已经被Leader提交的Proposal也能被所有的Follower提交，例如Leader提交一个Proposal后崩溃了，那么在崩溃恢复过程中需要确保所有的Follower提交了该Proposal

**2）ZAB协议需要确保丢弃那些只在Leader服务器上被提出的事物**

崩溃恢复过程需要跳过那些已经被丢弃的事物Proposal，例如Leader所在的机器server提出一个Proposal后崩溃，那么当server恢复过来再次加入集群的时候，ZAB协议需要确保丢弃崩溃之前提出的未提交的事物

**综上:ZAB是这样一个选举算法：能够确保提交已经被Leader提交的事物Proposal，同时丢弃已经被跳过的事物proposal**

## 4、数据同步

**关于ZXID：**ZXID是一个64位的数字，其中低32位可以看做是一个简单的单调递增的计数器，针对客户端的每一个事物请求，Leader服务器在产生一个新的事物Proposal的时候，都会对该计数器进行加1操作；而高32位则代表了Leader周期epoch的编号，每当选举产生一个新的Leader服务器，都会从这个Leader服务器上取出其本地日志中最大事物Proposal的ZXID，并从该ZXID中解析出对应的epoch值，然后在对其进行加1操作，之后就会以此编号作为新epoch，并将低32位置0来开始生成新的ZXID。

## 5、ZAB与Paxos算法的联系与区别

### 1）两者联系

a、两者都存在一个类似于Leader进程的角色，由其负责协调多个Follower进程的运行

b、Leader进程都会等待超过半数的Follower作出正确的反馈后，才将一个提案进行提交

c、在ZAB协议中，每个Proposal中都包含了一个epoch值，用来代表当前的Leader周期，在Poxos算法中，同样存在这样的标识，只是名字变成了Ballot

### 2）区别

两者的设计目标不太一样，ZAB协议主要用于构建一个高可用的分布式数据主备系统，而Paxos算法则是用于构建一个分布式的一致性状态机系统

