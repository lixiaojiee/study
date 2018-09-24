#  一、redis启动命令

启动redis的三种方法：

1）默认配置：redis-server（生产环境不建议使用）

2）运行启动：redis-server加上要修改的配置名和值（可以是多对），没有设置的配置将使用默认配置。虽然运行配置可以自定义配置，但是如果需要修改的配置较多或者希望将配置保存到文件里，不建议使用这种配置

3）配置文件启动：将要修改的配置写到文件里，这样我们就可以按照下边方式启动

```
# redis-server /opt/redis/redis.conf
```

redis-cli可以使用两种方式连接redis服务器：

1）交互方式：

```
# redis-cli -h {host} -p {port} {command}
```

2)命令方式：

```
# redis-cli -h ip {host} -p {port} {command}
```

第二种需要注意的是，如果没有-h参数，那么默认连接127.0.0.1；如果没有-p，那么默认端口就是6379.

# 二、redis api的使用

## 1、redis的一些全局命令

1）查看所有的键

```
keys *
```

keys命令会遍历所有的键，所以它的时间复杂度为*O*(n)，因此，<u>当redis保存大量键时，线上禁止使用</u>

2）键总数

```
dbsize
```

键总数会返回当前数据库中键的总数

dbsize命令在计算键总数时不会遍历所有的键，而是直接获取redis内置的键总数变量，所以dbsize命令的时间复杂度为*O*(1)

3）检查键是否存在

```
exists key
```

存在返回1，不存在返回0

4）删除键

```
del key [key1 ...]
```

成功了就返回删除成功的个数，不存在返回0

5）键过期

```
expire key seconds
```

超时时间单位为秒

可以用ttl命令查看键的剩余过期时间

```
ttl key
```

它有三种返回值：

a、大于等于零的整数，键剩余的过期时间

b、-1：键没设置过期时间

c、-2:键不存在

6）键的数据结构类型

```
type key
```

如果键不存在，则返回none

## 2、redis数据结构的内部编码

redis的数据结构包括string、hash、list、set、zset等，这些都是redis对外的数据结构，实际上每种数据结构都有其底层的内部编码实现。这种内部编码实现有两个好处：

1）可以改进内部编码，而对外的数据结构和命令没有影响

2）多种内部编码实现可以在不同场景下发挥各自的优势

## 3、redis的单线程处理机制

一条命令从客户端到服务端，不会被立即执行，而是都要进一个队列里，然后才会逐个执行

redis即使在单线程下，其执行速度仍然快，归结于以下几点：

1）纯内存访问，redis将所有的数据放在内存内，而内存的响应时长大约为100ns，这是redis达到每秒万级访问的重要基础

2）非阻塞io，redis使用epoll作为I/O多路复用技术的实现，再加上redis自身的事件处理模型将epoll中的连接、读写、关闭都转换为事件，不在网络I/O上浪费过多的时间

3）单线程避免了线程切换和静态产生的消耗

单线程也是有缺点的，如果某个命令的执行时间过长，会造成其他命令的阻塞，这对于redis这种高性能的服务来说是致命的，所以<u>redis是面向快速执行场景的数据库</u>

## 4、字符串

字符串可以存储字符串、数字、甚至是二进制，但最大不得超过512MB

常用命令：

1）设置值

```
set key value [ex seconds] [px milliseconds] [nx|xx]
```

nx：键必须不存在，才可以设置成功，用于添加

xx：键必须存在，才可以设置成功，用于更新

也有专门支持nx的命令setnx，其中setnx可以用来实现分布式锁

2）获取值

```
get key
```

如果键不存在，则返回nil

3）批量设置值

```
mset key value [key1 value1 ...]
```

4)批量获取值

```
mget key [key1 ...]
```

如果分别通过get获取n个键的值，则其消费的时间就是`n次get时间=n次网络时间+n次命令时间`

如果通过mget获取n个键的值，则其消费的时间就是`n次get时间=1次网络时间+n次命令时间`

5）计数

```
incr key
```

incr用于对值进行自增操作，返回结果有三种情况

a、值不是整数，返回错误

b、值是整数，返回自增后的结果

c、键不存在，返回值为0自增，返回的结果为1

redis的内部技术实现时使用了CAS，CAS对cpu的开销在redis不存在，因为redis是单线程的

6）追加值

```
append key value
```

向字符串的末尾追加值

7）字符串长度

```
strlen key
```

字符串的内部编码：

a、int：8个字节的长整型

b、embstr：小于等于39个字节的字符串

c、raw，大于39个字节的字符串

字符串的典型使用场景：

a）缓存功能

b）计数

c）共享Session（需要保证redis是高可用和可扩展的）

d）限速，如用户输入手机验证码，一分钟内输入多少次就不让输入，网站对ip的访问限制也是该道理

## 5、哈希

哈希类型指键值本身又是一个键值对结构

常用命令：
1）设置值

```
hset key field value
```

设置成功返回1，反之返回0，redis提供了hsetnx命令，作用同setnx，只是其作用域从key变为field

2）获取值

```
hget key field
```

3）删除field

```
hdel key field [field1 ...]
```

删除成功后会返回删除的个数

4）计算field个数

```
hlen key
```

5）批量设置或获取field-value

```
hmget key field [field1 ...]
hmset key field value [field1 value1 ...]
```

6)判断field是否存在

```
hexists key field
```

存在返回1，否则返回0

7）获取所有的field

```
hkeys key
```

8）获取所有的value

```
hvals key
```

9）获取所有的field-value

```
hgetall key
```

内部编码：

a、ziplist（压缩列表）：

​	sum(field)<hash-max-ziplist-entries(默认512个)&&value<hash-max-ziplist-value(默认64字节)

zip使用更加紧凑的结构实现多个元素的连续存储，所以在节省内存方面比hashtable优秀很多

b、hashtable（哈希表）：

当不满足a条件时，会采用hashtable编码，因为此时ziplist的读写效率会下降，而hashtable的读写时间复杂度为*O*(1)

使用场景：

可以存储一个用户的个人信息

## 6、列表

列表类型是用来存储多个有序字符串，列表中的每个字符串称为元素。一个列表中最多可以存储2^32^-1个元素

列表的特点：

a、列表中的元素都是有序的

b、列表中的元素是**可以重复**的

常用命令：

1）添加操作

a、从右边插入元素

```
rpush key value [value1 ...]
```

b、从左边插入元素

```
lpush key value [value1 ...]
```

c、向某个元素前或后插入元素

```
linsert key before|after pivot value
```

linsert命令会从列表中找到等于pivot的元素，在其前或者后插入一个新的元素value

2）查找

a、获取指定范围内的元素列表

```
lrange key start end
```

list的索引下标有两个特点：

i、索引下标从左到右分别是0到N-1，从右到左分别是-1到-N

ii、**lrange中end选项包含了自身**

b、获取列表指定索引下标的元素

```
lindex key index
```

c、获取列表长度

```
llen key
```

3）删除

a、从列表左侧弹出元素

```
lpop key
```

b、从列表右侧弹出

```
rpop key
```

c、删除指定元素

```
lrem key count value
```

lrem命令会从列表中找到等于value的元素进行删除，根据count的不同分为三种情况：

i、count>0:从左到右，删除至多count个元素

ii、count<0：从右到左，删除最多count绝对值个元素

iii、count=0:删除所有

d、按照指定范围修剪列表

```
ltrim key start end
```

4）修改

a、修改指定索引下标的元素

```
lset key index nowValue
```

5）阻塞操作

阻塞式弹出如下：

```
blpop key [key1 ...] timeout
brpop key [key1 ...] timeout
```

该操作可能有以下几种情况：

i、列表为空：如果timeout = 3，那么客户端要等到3s后返回，如果timeout=0，那么客户端会一直阻塞等下去，如果在阻塞期间添加了key，客户端立即返回

ii、列表不为空，客户端立即返回

在使用brpop时，有两点需要注意：

i、如果是多个键，那么brpop会从左至右遍历键，一旦有一个键能弹出元素，客户端将立即返回

ii、如果多个客户端对同一个键执行brpop，那么最先执行brpop命令的客户端客户获取到弹出的值

内部编码：

a、ziplist（压缩列表）

​		sum(field)<list-max-ziplist-entries(默认512个)&&value<list-max-ziplist-value(默认64字节)

b、linkedlist（链表）

使用场景：

a、消息队列

可以使用lpush+brpop实现阻塞队列

b、文章列表

##  7、集合

集合类型也是用来保存多个字符串元素，但和列表类型不一样的是，集合中不允许有重复元素，并且集合中的元素是无序的，不能通过索引下标获取元素。一个集合最多可以存储2^31^-1个元素

常用命令：

1）集合内操作

a、添加元素

```
sadd key element [element1 ...]
```

返回的结果是添加成功的元素的个数

b、删除元素

```
srem key element [element1 ...]
```

返回结果为删除成功元素的个数

c、计算元素的个数

```
scard key
```

该操作的时间复杂度为*O*(1)，它不会遍历集合所有元素，而是直接用Redis的内部变量

d、判断元素是否在集合中

```
sismember key element
```

在集合内返回1，否则返回0

e、随机从集合返回指定个数的元素

```
srandmember key [count]
```

不指定count，默认返回1个元素

f、从集合随机弹出元素

```
spop key 
```

与srandmember的区别：

spop命令执行后，元素会将元素从集合中删除，二srandmember不会

g、获取所有元素

```
smembers key
```

smembers和lrange、hgetall一样，都是比较重的操作，建议用sscan来替代

2）集合间操作

a、求多个集合的交集

```
sinter key [key1 ...]
```

b、求多个集合的并集

```
sunion key [key1 ...]
```

取并集会去重

c、求多个集合的差集

```
sdiff key [key1 ...]
```

d、将交集、并集、差集的结果保存

```
sinterstore destination key [key1 ...]
sunionstore destination key [key1 ...]
sdiffstore destination key [key1 ...]
```

集合间的运算在元素较多的情况下会比较耗时，所以redis提供了上面三个命令将集合的交集、并集、差集保存在destination key中，其结果也是一个集合类型

内部编码：

1）intset（整数集合）：当集合中的元素都是整数且元素的个数小于512个时，采用此编码

2）hashtable（哈希表）：当集合类型不满足1）时，采用此编码 

使用场景：

a、标签（sadd）：给用户添加标签或者给标签添加用户，这两个操作要在一个事物里运行，防止部分命令失败造成的数据不一致。同时，删除用户的标签以及删除标签下的用户也需要在同一个事物里面运行

b、生成随机数，抽奖（spop/srandmember）

c、社交需求（sadd+sinter）

## 8、有序集合

有序集合有如下特点：

a、和集合一样不能有重复元素

b、可以排序，但是它和列表使用索引下标作为排序依据不同的是，它给每个元素设置一个分数（score）作为排序的依据

c、虽然有序集合中不能有重复元素，但是score可以重复

下表是列表、集合、有序集合的异同点

| 数据结构 | 允许重复 | 是否有序 | 有序实现方式 | 应用场景     |
| -------- | -------- | -------- | ------------ | ------------ |
| 列表     | 允许     | 是       | 索引下标     | 时间轴、队列 |
| 集合     | 否       | 否       | 无           | 标签、社交   |
| 有序集合 | 否       | 是       | 分值         | 排行榜、社交 |

常用命令：

1）集合内操作：

a、添加成员

```
zadd key score member [score1 member1 ...]
```

返回结果为成功添加成员的个数

该命令有两点需要注意：

i、redis 3.2为zadd命令添加了nx、xx、ch、incr四个选项：

- nx：member必须不存在，才可以设置成功，用于添加
- xx：member必须存在，才可以设置成功，用于更新
- ch：返回此操作后，有序集合元素和分数发生变化的个数
- incr：对score做增加，相当于后面介绍的zincrby

ii、有序集合相比集合提供了排序字段，但是也产生了代价，zadd的时间复杂度为*O*(log(*n*)),add的时间复杂度为*O*(1)

b、计算成员的个数

```
zcard key
```

和集合一样，其时间复杂度为*O*(1)

c、计算某个成员的分数

```
zscore key member
```

d、计算成员的排名

```
zrank key member
zrevrank key member
```

zrank是按照分数从低到高返回排名，zrevrank是按照分数从高到低返回排名（排名从0开始计算）

e、删除成员

```
zrem key member [member1 ...]
```

返回结果为删除成功的个数

f、增加成员分数

```
zincrby key increment member
```

返回结果为成员的最新分数

g、返回指定排名范围内的成员

```
zrange    key start end [withscores]
zrevrange key start end [withscores]
```

zrange是从低到高返回，zrevrange是从高到低返回

h、返回指定分数范围的成员

```
zrangeByscore    key mix max [withscores] [limit offset count]
zrevrangeByscore key mix max [withscores] [limit offset count]
```

withscores 选项会同时返回每个成员的分数。[limit offset count]选项可以限制输出的起始位置和个数

-inf和+inf分别代表无限小和无限大

指定范围可以支持开区间()和闭区间[]

i、返回指定分数范围成员个数

```
zcount key min max
```

j、删除指定范围内的升序元素

```
zremrangebyrank key start end
```

k、删除指定范围分数的成员

```
zremrangebyscore key min max
```

2）集合间操作：

a、交集

```
zinterstore destination numkeys key [key1 ...] [weights weight [weight1 ...]] [aggregate sum|min|max]
```

该命令的参数说明如下：

i、destination：交集计算结果保存到这个键

ii、numkeys：需要做交集计算键的个数

iii、key[key1 ...]：需要做交集的计算的键

iv、weights weight[weight1 ...]：每个键的权重，默认为1

v、aggregate sum|min|max：计算成员交集后，分值可以按照sum、min、max做汇总，默认值为sum

b、并集

```
zunionstore destination numkeys key [key1 ...] [weights weight [weight1 ...]] [aggregate sum|min|max]
```

参数说明同a

内部编码：

1）ziplist（压缩列表）：元素个数小于128个且每个元素的值的大小小于64字节时，会使用该编码，可以节省内存空间

2）skiplist（跳跃表）：当1）不满足时，会选择该种编码格式

使用场景：排行榜

1）添加用户赞数，可以使用zadd+zincrby

2）取消用户赞数，zrem

3）展示获取赞数最多的十个用户zrevrange

4）展示用户信息以及用户分数，zscore+zrank

## 9、键管理

### 1）单个键管理

#### a、键重命名

```
rename key newkey
```

如果在rename之前，键newkey已经存在，那么它的值将被覆盖

为了防止被强行rename，redis提供了renamenx命令，确保只有newkey不存在时候才被覆盖，若返回结果为0，则表示没有完成重命名

注意：

i、由于重命名期间会执行del命令删除旧的键，如果键对应的值比较大，会村子阻塞redis的可能，这点不要忽视

ii、如果rename和renamenx中的key和newkey是相同的，在redis3.2之前是可以rename的，之后会报错

#### b、随机返回一个键

```
randomkey
```

#### c、键过期

```
expire key seconds #键在seconds秒后过期
expireat key timestamp #键在秒级时间戳timestamp后过期
```

ttl命令和pttl命令都可以查询键的声誉过期时间，但是pttl精确度更高，可以达到毫秒级别，其有三种返回值：

i、大于等于0的整数：键剩余的过期时间（ttl是秒，pttl是毫秒）

ii、-1:键没有设置过期时间

iii、-2:键不存在

```
pexpire key milliseconds #键在milliseconds毫秒后过期
pexpireat key milliseconds-timestamp #键在毫秒级时间戳后过期
```

使用redis相关过期命令需要注意以下几点：

i、如果expire key的键不存在，返回结果为0

ii、如果过期时间为负值，键会立即被删除，此时就是del

iii、persist命令可以将键的过期时间清除

```
persist key
```

==iv、对于字符串类型键，执行set命令会去掉过期时间==

v、redis不支持二级数据结构（例如哈希、列表）内部元素的过期功能

vi、setex命令作为set+expire的组合，不但是原子执行，同时减少了一次网络通讯时间

#### d、键迁移

i、move

```
move key db
```

move命令用于在redis内部进行数据迁移

ii、dump+restore

```
dump key
restore key ttl value
```

dump+restore可以实现在不同的redis实例之间进行数据迁移的功能，整个恰伊工作分两步：

i）在源redis，dump命令会将键值序列化，格式采用的是RDB格式

ii）在目标redis上，restore命令将上面序列化的值进行复原，其中ttl参数代表过期时间，如果ttl=0，代表没有过期时间

注意：

i）整个迁移过程并非原子性的，而是通过客户端分步完成的

ii）迁移过程是开启了两个客户端连接，所以dump的结果不是在源redis和目标redis之间进行传输

iii、migrate

```
migrate host port key|"" destination-db timeout [copy] [replace] [keys key [key1 ...]]
```

migreate命令也是用于在redis实例间进行数据迁移的，实际上migrate命令就是将dump+restore+del三个命令进行组合，从而简化了操作流程。migrate命令具有原子性。

该命令和dump+restore的区别：

i）整个过程是原子执行的，不需要在多个redis实例上开启客户端的，只需要在源redis上执行migrate命令即可

ii）migtate命令的数据传输直接在源redis和目标redis上完成的

iii）目标redis完成restore后会发送OK给源redis，源redis接收后会根据migrate对应的选项来决定是否在源redis上删除对应的键

参数说明：

i）host：目标redis的ip地址

ii）port：目标redis的端口

iii）key|“”：如果要迁移一个键，key就是要迁移的键，如果要迁移多个键，则此处是“”

iv）destination-db：目标redis的数据库索引，如果要迁移到0号数据库，则这里就填0

v）timeout）迁移的超时时间，单位为毫秒

vi）[copy]：如果添加此项，迁移后并不删除源键

vii）[replace]：如果添加此项，不管目标redis是否存在该键，都会正常迁移进行数据覆盖

viii）[keys key [key1 ...]]：迁移多个键

下表是三种迁移方式的比较

| 命令         | 作用域        | 原子性 | 支持多个键 |
| ------------ | ------------- | ------ | ---------- |
| move         | redis实例内部 | 是     | 否         |
| dump+restore | redis实例之间 | 否     | 否         |
| migrate      | redis实例之间 | 是     | 是         |

这三种迁移模式中，建议使用第三种进行迁移

### 2）遍历键

#### a、全量遍历键

```
keys pattern
```

pattern可以使用glob风格的通配符：

- *代表匹配任意字符
- ？代表匹配一个字符
- []代表匹配部分字符，如[1，3]代表匹配1，3，[1-10]代表匹配1到10的任意数字

遍历键的使用场景：

- 在一个不对外提供服务的redis从节点上执行，这样不会阻塞到客户端的请求，但是会影响到主从复制
- 如果确认键值总数却是比较少，可以执行该命令
- 使用scan命令渐进式的遍历所有键，可以有效防止阻塞

#### b、渐进式遍历

```
scan cursor [match pattern] [count number]
```

每次执行scan，可以想象成只扫描一个字典中的一部分键，直到将字典中的所有键遍历完毕，其返回结果是下次遍历scan需要的cursor

参数说明：

i）cursor：必须参数，实际上cursor是一个游标，第一次遍历从0开始，每次scan遍历完都会返回当前游标的值，直到游标值为0，表示遍历结束

ii）match pattren：模式匹配，和keys的模式匹配类似

iii）count number：表明每次要遍历的键的个数，默认是10个，该参数可以适当增大

除了scan外，redis提供了面向哈希类型、集合类型、有序集合的扫描遍历命令，解决诸如hgetall、smembers、zrange可能产生的阻塞问题，对应命令分别是hscan、sscan、zscan，它们的用法和scan类似

渐进式遍历的缺点：如果在scan的过程中有键发生变化（如增加、删除、修改），那么遍历效果可能会碰到如下问题：新增的键可能没有遍历到，遍历出了重复的键等情况。

### 3）数据库管理

a、切换数据库

```
select dbindex
```

dbindex为数据库编号，redis中，一个实例默认为16个数据库，编号从0开始

b、flushdb/flushall

flushdb/flushall用于清除数据库，两者之间的区别是flushdb只清除当前数据库，flushall会清除所有数据库

这两个命令带来的两个问题：

a、flushdb/flushall命令会将所有的数据清除，一旦误操作后果不堪设想，可以rename-command配置规避这个问题

b、如果当前数据库键值对比较多，flushdb/flushall存在阻塞redis的可能性

