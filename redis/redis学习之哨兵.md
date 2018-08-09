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

当Sentinel节点集合对主节点故障判定达成一致时，Sentinel领导者节点会做故障转移操作，选出新的主节点，原来的从节点会向新的主节点发起复制操作，`parallel-syncs`就是用来限制在一次故障转移之后，每次向新的主节点同时发起复制操作，parallel-syncs=3表示三个从节点同时向主节点发起复制操作，parallel-syncs=1表示一个从节点同时向主节点发起复制操作，这个时候会发起轮询复制

4、sentinel failover-timeout

```
sentinel failover-timeout <master-name> <times>
```

failover-timeout通常被解释成故障转移的超时时间，但实际上它作用于故障转移的各个阶段：

a、选出合适的从节点

b、晋升选出的从节点为主节点

c、命令其余从节点复制新的主节点

d、等待原主节点恢复后命令它去复制新的主节点

failover-timeout的作用具体体现在四个方面：

1）如果Redis Sentinel对一个主节点故障转移失败，那么下次再对该主节点做故障转移的起始时间时failover-timeout的2倍

2）在b阶段时，如果Sentinel节点向a阶段选出来的节点执行`slaveof no one`操作一只失败（例如该从节点此时出现故障），当过程超过failover-timeout时，则故障转移失败

3）在b阶段如果执行成功，Sentinel节点还会执行info命令确认a阶段选出来的节点确实晋升为主节点，如果此过程执行时间超过failover-timeout时，则故障转移失败

4）如果c阶段执行时间超过了failover-timeout（不包含复制时间），则故障转移失败，注意即使超过了这个时间，Sentinel节点也会最终配置从节点去同步最新的主节点

5、sentinel auth-pass

```
sentinel auth-pass <master-name> <password>
```

如果Sentinel监控的主节点配置了密码，sentinel auth-pass配置通过添加主节点的密码，防止Sentinel节点对主节点无法监控

6、sentinel notification-script

```
sentinel notification-script <master-name> <script-path>
```

Sentinel notification-script的作用是在故障转移期间，当一些警告级别的Sentinel时间发生（指重要事件，例如-sdown：客观下线，-odown：主观下线）时，会触发对应路径的脚本，并向脚本发送相应的事件参数作为邮件或者短信报警依据

7、sentinel client-reconfig-script

```
sentinel client-reconfig-script <master-name> <script-path>
```

Sentinel client-reconfig-script的作用时在故障转移结束后，会触发对应路径的脚本，并向脚本发送故障转移结果的相关参数，该脚本会接收每个Sentinel节点传过来的故障转移结果参数，并触发类似短信或邮件报警

监控多个主节点：

Sentinel监控多个主节点的配置就是只需要指定多个masterName来区分不同的节点即可

