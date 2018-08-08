#  一、 Redis Sentinel的高可用性

当主节点出现故障时，Redis Sentinel能自动完成故障发现和故障转移，并通知应用方，从而实现真正的高可用.

Redis Sentinel是一个分布式架构，其中包含若干个Sentinel节点和Redis数据节点，每个Sentinel节点会对数据节点和其余的Sentinel节点进行监控，当它发现节点不可用时，会对节点做下线标示。如果被标识的是主节点，他还会和其他Sentinel节点进行协商，当大多数Sentinel节点都认为主节点不可达时，它们会选举出一个Sentinel节点来完成自动故障转移工作，同时会将这个变化实时通知给Redis应用方。

Redis Sentinel进行故障转移的步骤如下：

1、主节点出现故障，此时从节点与主节点失去连接，主从复制失败

2、每个Sentinel节点通过定期监控发现主节点出现了故障

3、多个Sentinel节点对主节点的故障达成一致，选举出一个Sentinel节点作为领导者负责故障转移

4、Sentinel领导者节点进行故障转移

​	1）选取一个从节点，对其执行`slave no one`命令使其成为新的主节点

​      2）原来的从节点成为新的主节点后，更新应用方的主节点信息，重新启动应用方

​      3）客户端命令其他从节点去复制新的主节点

​      4）待原来的主节点恢复后，让它去复制新的主节点

Sentinel的作用就是实现了第4步操作的自动化

从上边可以看出，Redis Sentinel具有以下几个功能：

1、**监控**：Sentinel节点会定期监测Redis数据节点、其余Sentinel数据节点是否可达

2、**通知**：Sentinel节点会将故障转移的结果通知给应用方

3、**主节点故障转移**：实现从节点晋升为主节点并维护后续正确的主从关系

4、**配置提供者**：在Redis Sentinel结构中，客户端在初始化的时候连接的是Sentinel节点结合，从中获取主节点信息

Redis Sentinel节点集合是由多个Sentinel节点组成，这样有两个好处：

1、对于节点的故障判断是由多个Sentinel节点共同完成，这样可以有效地防止误判

2、这样即使个别Sentinel节点不可用，整个Sentinel节点集合仍然是健壮的

# 二、redis哨兵配置说明

1、sentinel monitor

```
sentinel monitor <master-name> <ip> <port> <quorum>
```

该配置说明了Sentinel节点要监控的是一个名字叫<master-name>，ip地址和端口为<ip> <port>的主节点。<quorum>代表要判定主节点最终不可达所需要的票数。但实际上Sentinel节点会对所有的节点进行监控，但是在Sentinel节点的配置中没有看到有关从节点和其余Sentinel节点的配置，那是因为Sentinel节点会从主节点中获取有关从节点以及其余Sentinel节点的相关信息

<quorum>参数用于故障发现和判定，如将quorum配置为2，代表至少有2个Sentinel节点认为主节点不可达，那么这个不可达的判定才是客观的。一般建议将其设置为Sentinel节点的一半加1，同时<quorum>海域Sentinel节点的领导者选举有关，至少要有`max(quorum,num(sentinels)/2+1)`个Sentinel节点参与选举，才能选出领导者Sentinel，从而完成故障转移

2、sentinel down-after-milliseconds

```
sentinel down-after-milliseconds <master-name> <times>
```

 每个Sentinel节点都要通过定期发送ping命令来判断redis数据节点和其余Sentinel节点是否可达，如果超过了down-after-milliseconds配置的时间且没有有效的回复，则判定节点不可达，<times>单位为毫秒，就是超时时间。

3、sentinel parallel-syncs

```
sentinel parallel-syncs <master-name> <nums>
```

