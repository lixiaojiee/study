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

# 三、客户端连接

实现一个Sentinel客户端的基本步骤：

1、遍历Sentinel节点集合获取一个可用的Sentinel节点

2、通过`sentinel get-master-addr-by-name maserName`这个API获取对应主节点的相关信息

3、验证当前获取的主节点是真正的主节点（使用`role`或者`info replication`命令），这样做的目的是为了防止故障转移期间主节点的变化

4、保持和Sentinel节点集合的联系，时刻获取关于主节点的相关信息

# 四、Sentinel的实现原理

## 1、三个定时监控任务

1）每隔10s，每个Sentinel节点会向主节点和从节点发送info命令获取最新的拓扑结构

这个定时任务的命令：

- 通过向主节点执行info命令，获取从节点的信息，这也是为什么Sentinel节点不需要显式配置基恩孔从节点的原因
- 当有新的从节点加入时都可以立刻感知出来
- 节点不可达或者故障转移后，可以通过info命令实时更新节点的拓扑信息

2）每隔2秒，每隔Sentinel节点会向redis数据节点`__sentin__:hello`频道上发送Sentinel节点对于主节点的判断以及当前Sentinel节点的信息，同时每隔Sentinel节点也会订阅该频道，来了解其它Sentinel节点以及它们对主节点的判断，所以这个订单任务可以完成以下两个操作：

- 发现新的Sentinel节点
- Sentinel节点之间交换主节点的状态，作为后面客观下线以及领导者选举的依据

3）每隔1秒，每个Sentinel节点后向主节点、从节点、其余Sentinel节点发送一条ping命令做一次心跳检测，来确认这些节点当前是否可达。

## 2、主观下线和客观下线

### 1）主观下线

每个Sentinel节点会每隔1秒对主节点、从节点以及其它Sentinel节点发送ping命令做心跳检测，当这些节点超过down-after-milliseconds没有进行有效回复，Sentinel节点就会对该节点做失败判定，这个行为叫做主观下线，主观下线只是当前Sentinel节点的一家之言，存在误判的可能

### 2）客观下线

 当Sentinel主观下线的节点是主节点时，该Sentinel节点会通过`sentinel is_master-down-by-addr`命令向其它Sentinel节点询问对主节点的判断，当超过<quorum>个数，Sentinel节点认为主节点确实有问题，这时候该Sentinel节点会做出客观下线判断

从节点、Sentinel节点在主观下线后没有后续的故障转移操作

关于sentinel is_master-down-by-addr命令的介绍：

```
sentinel is_master-down-by-addr <ip> <port> <current_epoch> <runid>
```

- id：主节点IP

- port：主节点端口

- current_epoch：当前配置纪元

- runid：该参数有两种类型，不同类型决定了此API的作用不同

  a、当runid 等于“*”时，作用是Sentinel节点直接交换对主节点下线的判定

  b、当runid等于当前Sentinel节点的runid时，作用是当前Sentinel节点希望目标Sentinel节点统一自己成为领导者的请求

## 3、领导者Sentinel节点选举

Redis使用了Raft算法实现领导者选举

redis Sentinel进行领导者选举的大体思路：

1）每个在线的Sentinel节点都有资格成为领导者，当它确认主节点主观下线时，会向其它Sentinel节点发送`sentinel is_master-down-by-addr`命令，要求将自己设置为领导者

2）收到命令的Sentinel节点，如果没有统一过其它Sentinel节点的`sentinel is_master-down-by-addr`命令，将同意该请求，否则拒绝

3）收到命令的Sentinel节点发现自己的票数已经大于等于`max(quorum,num(sentinel)/2+1)`，那么它将成为领导者

4）如果此过程没有选举出领导者，将进入下一次选举

**选举的过程非常快，基本上谁先完成客观下线，谁就是领导者**

## 4、故障转移                          

领导者选举出的Sentinel节点负责故障转移，步骤如下：

1）在从节点列表中选出一个节点作为新的主节点，选择方法如下：

​	a、过滤：不健康（主观下线、断线）、5s内没有回复过Sentinel节点ping相应、与主节点失联超过down-milliseconds*10秒

​	b、选择slave-priority（从节点优先级）最高的从节点列表，如果存在则返回，不存在则继续

​	c、选择复制偏移量最大的从节点（复制的最完整），如果存在则返回，不存在则继续

​	d、选择runid最小的从节点

2）Sentinel领导者节点会对第一步选出来的从节点执行slave no one命令让其成为主节点

3）Sentinel领导者节点会向剩余的从节点发送命令，让它们成为新主节点的从节点，复制规则和parallel-syncs参数有关

4）Sentinel节点集合会将原来的主节点更新为从节点，并保持着对其关注，当其恢复ii后命令它去复制新的主节点