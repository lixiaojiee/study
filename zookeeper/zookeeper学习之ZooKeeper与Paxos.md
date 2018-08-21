# 一、zookeeper的基本概念

## 1、数据节点（Znode）

ZNode是zookeeper数据模型中的数据单元，zookeeper讲所有的数据存储在内存中，数据模型是一棵树（ZNode Tree），由斜杠（/）进行分割的路径，就是一个ZNode。在Zookeeper中，ZNode可以分为持久节点和临时节点两类：

持久节点：是指一旦这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在zookeeper上 

临时节点：临时节点的生命周期和客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的所有临时节点都会被移除

zookeeper允许用户为每个节点添加一个特殊的属性：SEQUENTIAL。一旦 节点被标记上这个属性，那么在这个节点被创建的时候，zookeeper会自动在其节点名后面追加一个整型数字，这个整型数字是一个由父节点维护的自增数字

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