# 一、慢查询分析

1、redis客户端执行一条命令分为如下4个部分：

1）发送命令

2）命令排队

3）命令执行

4）返回结果

慢查询只统计3）的时间，1）+4）称为Round Trip Time（RTT，往返时间）所以没有慢查询并不代表客户端没有超时问题

慢查询的两个配置参数：

`showlog-log-slower-than`:预设的阈值，单位为微秒，默认为10 000，超过预设阈值的查询操作将会被记录进慢查询日志中

`showlog-max-len`:慢查询日志列表的最大长度，当慢查询日志列表已处于其最大长度时，最早插入的一个命令将从列表中移出

redis中有两种修改配置的方法，一种是修改配置文件，另一种是使用config set命令动态修改。如下所示：

```
config set showlog-log-slower-than 20000
config set showlog-max-len 1000
config rewrite #将修改的配置持久化到本地配置文件中
```

2、慢查询日志的访问和管理

1）获取慢查询日志

```
showlog get [n]
```

慢查询日志的组成：

a、标示id

b、发生的时间戳

c、命令耗时

d、执行的命令和参数

2）获取慢查询日志列表的当前长度

```
showlog len
```

3)慢查询日志重置

```
showlog reset
```

3、关于慢查询的相关建议

a、showlog-max-len配置建议：线上建议调大慢查询列表，记录慢查询时redis会对长命令做截断操作，并不会占用大量内存。增大慢查询列表可以减缓慢查询被剔除的可能

b、showlog-log-slower-than配置建议：默认值超过10毫秒判定为慢查询，需要根据redis并发量调整该值。

c、慢查询只记录命令执行时间，并不包括命令排队和网络传输时间，因此客户端执行命令的时间会大于命令实际执行的时间。因为命令执行的排队机制，慢查询会导致其他命令级联阻塞，因此当客户端出现请求超时，需要检查该时间点是否有对应的慢查询，从而分析是否为慢查询导致命令级联阻塞

d、由于慢查询日志是一个先进先出的队列，也就是说在慢查询比较多的情况下，可能会丢失部分慢查询命令，为了防止这种情况的发生，可以定期执行showlog get命令将慢查询日志持久化到其他存储中，然后可以制作可视化界面进行查询。 

# 二、Pipeline

pipelie可以将一组redis命令进行组装，通过一次RTT传输给redis，再将这组redis命令的执行结果按顺序返回给客户端

原生批量命令与pipeline对比：

- 原生批量命令是原子的，pipeline是非原子的
- 原生批量命令是一个命令对应多个key，pipeline支持多个命令
- 原生批量命令是redis服务端支持实现的，而pipeline需要服务端和客户端的共同实现

pipeline组装的命令个数不能没有节制，否则一次组装pipeline数据过大，一方面会增加客户端的等待时间，另一方面会造成一定的网络阻塞

pipeline只能操作一个redis实例

# 三、事物与LUA

## 1、事物

redis提供了简单事物功能，将一组需呀一起执行的命令放到`multi`和`exec`两个命令之间。`multi`命令代表事物开始，`exec`命令代表事物结束，它们之间的命令是原子顺序执行的，每执行完一条指令，返回结果为QUEUED，代表命令并没有真正执行，而是暂时保存在redis。只有执行`exec`命令后，两个操作才真正执行，事物执行结束后，会返回各个操作的执行结果，如果结果为nil，则说明事物没有执行

如果要停止事物的执行，可以使用`discard`命令代替`exec`命令即可。

如果事物中的命令出现错误，redis也有不同的处理机制：

1）命令错误，比如命令拼写错误，会造成整个事物无法执行，事物不回滚

2）运行时错误，比如zdd写成了sadd，也会造成事物无法运行，但是事物不会回滚

有时在事物之前，确保事物中的key没有被其他客户端修改过，才执行事物，否则不执行。redis提供了`watch`命令来解决这个问题

之所以说redis只是支持简单事物，就是因为其不支持事物的回滚特性。

## 2、LUA用法

### 1）数据类型及其逻辑处理

a、字符串（strings）

```lua
local strings val = "world"
```

local代表val是一个局部变量，如果没有local，则代表是全局变量。

2）数组（tables）

```lua
local tables myArray == {"redis", "jedis", true, 88.0}
```

lua的数组下标从1开始计算

如果想要遍历这个数组，可以使用for和while。

a、for

```lua
--计算2）中数组的长度
for i = 1, #myArray
do
    print(myArray[i])
end
```

要获取数组的长度，只需在数组的变量前加#

除此之外，lua还提供了内置函数ipairs，使用for index,value ipairs(tables)可以遍历出所有的索引下标和值，如下：

```lua
for index,value in ipairs(myArray)
do
    print(index)
    print(value)
end
```

b、while

计算1到100的和

```lua
--求1到100的和
local int num = 0
local int i = 0
while i <= 100
do
    sum = sum + i
    i = i + 1
end
--输出求和结果
print(num)
```

c、if-else

```lua
local tables myArray = {"redis", "jedis", true, 88.0}
for i = 1, #myArray
do
    if myArray[i] == "jedis"
    then
    	print("true")
       	break
    else
        --do nothing
    end
end
```

3)哈希

数组中的每个元素都是键值对组合

```lua
local tables user = {age = 28, name= "lixiaojie"}
```

遍历哈希

```lua
for key,value in pairs(user)
do 
    print(key .. value)
end
```

### 2)函数定义

在lua中，函数以function开头，以end结尾，functionName为函数名，中间部分是函数体

```lua
function functionName()
    ...
end
```

##  3)在redis中使用lua

在redis中使用lua脚本有两种方法：`eval`和`evalsha`

a、eval

```shell
eval 脚本内容 key个数 key列表 参数列表
```

示例：

```shell
eval 'return "hello " .. KEYS[1] .. ARGV[1]' 1 redis word
```

结果为:hello redisword

如果lua脚本过长，还可以使用redis-cli --eval直接执行文件

eval命令和--eval参数本质是一样的，客户端如果想执行lua脚本，首先在客户端编写好lua脚本代码，然后把脚本作为字符串发送给服务端，服务端会执行结果返回给客户端

b、evalsha

首先要将lua脚本加载到redis服务端，得到该脚本的SHA1校验和，evalsha命令使用SHA1作为参数可以直接执行对应的lua脚本，避免每次发送lua脚本的开销。这样客户端就不需要每次执行脚本内容，而脚本也会常驻服务端，脚本功能得到复用

**加载脚本：**script load命令可以将脚本内容加载到redis内存中

**执行脚本：**evalsha的使用方法如下，参数使用SHA1值，执行逻辑和eval一致

```shell
evalsha 脚本 SHA1值 key个数 key列表 参数列表
```

lua脚本功能有三个好处：

- lua脚本在redis中是原子执行的，执行过程中间不会插入其它命令
- 可以定制自己的命令，并可以将这些命令常驻在red is 内存中
- lua脚本可以将多条命令一次性打包，有效减少网络开销

redis如何管理lua脚本：

a、script load

```
script load script
```

此命令用于将lua脚本加载到redis内存中

b、script exists

```
script exists sha1 [sha1 ...]
```

此命令用来判断指定sha1是否已经加载到redis里，返回的结果代表sha1被加载到内存中的个数

c、script flush

```
script flush
```

此命令用于清除redis内存已经加载的所有lua脚本

d、script kill

```
script kill
```

此命令用于杀掉正在执行的lua脚本，如果lua脚本比较耗时，甚至lua脚本存在问题，那么此时lua脚本的执行会阻塞redis，直到脚本执行完毕或着外部进行干预将其结束

如果当前脚本正在执行写操作，那么`script kill`命令将不会生效。 

## 4）Bitmaps

Bitmaps是一种redis实现的位操作的“数据结构”

- Bitmaps本身不是一种数据结构，实际上它就是字符串，但是它可以对字符串的位进行操作
- Bitmaps单独提供了一套命令，所以在Redis中使用Bitmaps和使用字符串的方法不太相同。可以把Bitmaps想象成一个以位为单位的数组，数组的每个单元只能存储0和1，数组的下标在Bitmaps中叫偏移量

a、命令

i、设置值

```
setbit key offset value
```

设置键的第offset个位的值（从0开始算）

很多用户id以一个指定的数字开头，直接将用户id和bitmaps的偏移量对应势必会造成一定的浪费，通常的做法是每次做setbit操作时将用户id减去这个指定的数字。在第一次初始化Bitmaps时，加入偏移量非常大，那么整个初始化过程执行会比较慢，可能会造成Redis阻塞

ii、获取值

```
getbit key offset
```

 获取键的第offset位的值（从0开始算）

iii、获取Bitmaps指定范围值为1的个数

```
bitcount [start] [end]
```

[start]和[end]代表起始和结束子节数

iv、Bitmaps间的运算

```
bitop op destkey key [key1 ..]
```

bitop是一个符合操作，它可以做多个Bitmaps的and（交集）、or（并集）、not（非）、xor（异或）操作并将结果保存在destkey中

v、计算Bitmaps中第一个值为targetBit的偏移量

```
bitops key targetBit [start] [end]
```

  Bitmaps相比于set的优缺点：

1）Bitmaps存储活跃用户统计是用位统计，相比于set要节省很多空间

2）如果有很多非活跃用户，Bitmaps的优势就不那么明显了

## 5）HyperLogLog

HyperLogLog并不是一种新的数据结构（实际类型为字符串类型），而是一种基数算法，通过HyperLogLog可以利用极小的空间完成独立总数的统计，数据集可以是IP、Email、ID等

命令：

i、添加

```
pfadd key element [element1 ...]
```

pfadd用于向HyperLogLog添加元素，如果添加成功则返回1

ii、计算独立用户数

```
pfcount key [key1 ...]
```

pfcount用于计算一个或多个HyperLogLog的独立总数

iii、合并

```
pfmerge destkey sourcekey [sourcekey1 ...]
```

pfmerge可以求出多个HyperLogLog的并集并赋值给destkey

HyperLogLog内存占用量非常小，但是存在错误率，开发者在进行数据结构选型时只需要确认如下两条即可：

- 只为了计算独立总数，不需要获取单条数据
- 剋容忍一定的误差率，毕竟HyperLogLog在内存的占用量上有很大的优势

## 6）发布订阅

Redis提供了基于“发布/订阅”模式的消息机制，此种模式下消息发布者和订阅者不进行直接通信，发布者客户端向指定频道（channel）发布消息，订阅该频道的每个客户端都可以收到该消息

常用命令：

i、发布消息

```
publish channel message
```

返回结果为订阅者个数

ii、订阅消息

```
subscribe channel [channel1 ...]
```

订阅消息有以下几点需要注意：

- 客户端在执行订阅命令后进入了订阅状态，只能接收subscribe、psubscribe、unsubscribe、punsubscribe的四个命令
- 新开启的订阅客户端，无法收到该频道之前的消息，因为Redis不会对发布的消息进行持久化

iii、取消订阅

```
unsubscribe [channel [channel1 ...]]
```

iv、按照模式进行订阅和取消订阅

```
psubscribe pattern [pattern1 ...]
punsubscribe pattern [pattern1 ...]
```

v、查询订阅

查看活跃的频道

```
pubsub channels [pattern]
```

所谓活跃的频道是指当前频道至少有一个订阅者，其中pattern是指具体的模式

查看频道订阅数

```
pubsub numsub [channel ...]
```

查看模式订阅数

```
pubsub numpat
```

也就是查看通过模式订阅的客户端数

使用场景：

聊天室、公告牌、服务之间利用消息进行解藕

## 7、GEO

GEO支持存储地理位置信息用来实现诸如附近位置、摇一摇这类依赖于地理位置信息的功能

常用命令：

i、添加地理位置信息

```
geoadd key longitude latitude memer [longitude1 latitude1 member1 ...]
```

Longitude、latitude和member分别是该地理位置的经度、纬度、成员，返回的结果为成功添加的个数，如果集合中没有，添加时返回为1，如果已经有了，再添加，则返回0

如果需哟啊更新地理位置信息，仍然可以使用geoadd命令，虽然返回的结果是0。

ii、获取地理位置信息

```
geopos key member [member1 ...]
```

iii、获取两个地理位置的距离

```
geodist key member1 member2 [unit]
```

其中unit代表返回结果的单位，包含以下四种：

- m（meters）代表米
- km（kilometers）代表千米
- mi（miles）代表英里
- ft（feet）代表尺

iv、获取指定位置范围内的地理信息位置集合

```
georadius key longitude latitude radiusm|km|ft|mi [withcoord] [withdist] [withhash] [COUNT count] [asc|desc] [store key] [storedist key]
getradiusbymember key member longitude latitude radiusm|km|ft|mi [withcoord] [withdist] [withhash] [COUNT count] [asc|desc] [store key] [storedist key]
```

georadius和georadiusbymember两个命令的作用是一样的，都是以一个地理位置为中心算出指定半径内的其它地理信息位置，不同的是georadius命令的中心位置给出了具体的经纬度，georadiusbymember只需要给出成员即可，其中radiusm|km|ft|mi是必须参数，指定了半径，这两个命令有很多参数可选，如下：

- withcoord：返回结果中包含经纬度
- withdist：返回结果中包含离中心节点位置的距离
- withhash：返回结果中包含geohash
- COUNT count：指定返回结果的数量
- asc|desc： 返回结果按照离中心节点的距离做升序或者降序
- store key： 将返回的结果的地理位置信息保存到指定的键
- storedist key：将返回结果离中心的距离保存到指定键

v、获取geohash

```
geohash key member [member1 ...]
```

将二维经纬度转换成一维字符串

geohash有如下特点：

- GEO的数据类型为zset，redis将所有的地理位置信息的geohash存放在zset中
- 字符串越长，表示的位置更精确
- 两个字符串越相似，它们之间的距离越近，redis利用字符串的前缀匹配算法实现相关的命令
- geohash编码和经纬度是可以相互转换的

vi、删除地理位置信息

```
zrem key member
```

借用zset的zrem命令，因为其底层实现是zset
