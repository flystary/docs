## 数据库

### Redis

#### 概念

##### 出现原因

###### 关系型数据库面临的问题

- 拓展困难
- 读写慢
- 成本高
- 有限的支撑容量

###### 数据库访问的新需求

- 低延迟的读写速度
- 支撑海量的数据和流量
- 大规模集群的管理
- 庞大运营成本的考量

CAP定理（CAP theorem）

- `一致性`（Consistency)—所有节点在同一时间具有相同的数据`
- `可用性`（Availability）—保证每个请求不管成功或者失败都有响应
- `分隔容忍`（Partition tolerance）—系统中任意信息的丢失或失败不会影响系统的继续运作

##### 优缺点

###### 优点

- 简单的扩展
- 快速的读写
- 低廉的成本

###### 缺点

- 不提供对SQL的支持
- 支持的特性不够丰富
- 现有产品的不够成熟

##### Nosql分类和部分代表

![](./img/377.png)

##### 企业常见Nosql应用

- 纯NoSQL架构（Nosql为主）
- 以NoSQL为数据源的架构（Nosql为主）
- NoSQL作为镜像（nosql为辅）
- NoSQL为镜像（同步模式，nosql为辅）
- MySQL和NoSQL组合（nosql为辅）
- 其他应用（L缓存数据可以分为`内存`模式和磁盘`持久化`模式）

#### Redis

​		redis是一个`key-value`存储系统。和Memcached类似，它支持存储的value类型相对更多，包括`string(字符串)`、`list(链表)`、`set(集合)`、`zset(sortedset --有序集合)`和`hash（哈希类型）`。数据都是缓存在内存中。redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步

##### 特性

1. 完全居于内存，数据实时的读写内存，定时闪回到文件中。采用单线程,避免了不必要的上下文切换和竞争条件；
2. 支持高并发量，官方宣传支持10万级别的并发读写；
3. 支持持久存储，机器重启后的，重新加载模式，不会掉数据；
4. 海量数据存储，分布式系统支持，数据一致性保证，方便的集群节点添加/删除；
5. Redis不仅仅支持简单的k/v类型的数据，同时还提供list，set，zset，hash等数据结构的存储；
6. 灾难恢复–memcache挂掉后，数据不可恢复; redis数据丢失后可以通过aof恢复；
7. 虚拟内存–Redis当物理内存用完时，可以将一些很久没用到的value 交换到磁盘；
8. Redis支持数据的备份，即master-slave模式的数据备份。

##### 架构

![](./img/757.png)

各功能模块说明如下：
`File Event`: 处理文件事件，接受它们发来的命令请求（读事件），并将命令的执行结果返回给客户端（写事件）)
`Time Event`: 时间事件(更新统计信息，清理过期数据，附属节点同步，定期持久化等)
`AOF`: 命令日志的数据持久化
`RDB`：实际的数据持久化
`Lua Environment` : Lua 脚本的运行环境. 为了让 Lua 环境符合 Redis 脚本功能的需求，Redis 对 Lua 环境进行了一系列的修改，包括添加函数库、更换随机函数、保护全局变量，等等
`Command table(命令表)`：在执行命令时，根据字符来查找相应命令的实现函数。
`Share Objects（对象共享）`：
主要存储常见的值：
　　a.各种命令常见的返回值，例如返回值OK、ERROR、WRONGTYPE等字符；
　　b. 小于 redis.h/REDIS_SHARED_INTEGERS (默认1000)的所有整数。通过预分配的一些常见的值对象，并在多个数据结构之间共享对象，程序避免了重复分配的麻烦。也就是说，这些常见的值在内存中只有一份。
`Databases`：Redis数据库是真正存储数据的地方。当然，数据库本身也是存储在内存中的。

##### 启动流程

![](./img/931.png)

#####  安装

###### 安装方式

- yum安装
- 源码包安装

###### 程序路径

配置文件：`/etc/redis.conf`
主程序：`/usr/bin/redis-server`
客户端：`/usr/bin/redis-cli`
Unit `File:/usr/lib/systemd/system/redis.service`
数据目录：`/var/lib/redis`
监听：`6379/tcp`

##### 使用

###### 配置文件

 `网络`配置项(NETWORK)

```shell
### NETWORK ###
bind IP   #监听地址
port PORT    #监听端口
protected-mode yes    #是否开启保护模式，默认开启。要是配置里没有指定bind和密码。开启该参数后，redis只会本地进行访问，拒绝外部访问。
tcp-backlog 511   #定义了每一个端口最大的监听队列的长度
unixsocket /tmp/redis.sock    #也可以打开套接字监听
timeout 0    #连接的空闲超时时长；
```

`通用`配置项

```
### GENERAL ###
daemonize no    #是否以守护进程启动
supervised no    #可以通过upstart和systemd管理Redis守护进程，这个参数是和具体的操作系统相关的
pidfile "/var/run/redis/redis.pid"    #pid文件
loglevel notice   #日志等级
logfile "/var/log/redis/redis.log"    #日志存放文件
databases 16     #设定数据库数量，默认为16个，每个数据库的名字均为整数，从0开始编号，默认操作的数据库为0；
    切换数据库的方法：SELECT <dbid>
```

`快照`配置(SNAPSHOTTING)

```shell
### SNAPSHOTTING  ###
save 900 1         #900秒有一个key变化，就做一个保存
save 300 10       #300秒有10个key变化，就做一个保存，这里需要和开发沟通
save 60 10000   #60秒有10000个key变化就做一个保存
stop-writes-on-bgsave-error yes   #在出现错误的时候，是不是要停止保存
rdbcompression yes   #使用压缩rdb文件，rdb文件压缩使用LZF压缩算法，yes：压缩，但是需要一些cpu的消耗；no：不压缩，需要更多的磁盘空间
rdbchecksum yes    #是否校验rdb文件。从rdb格式的第五个版本开始，在rdb文件的末尾会带上CRC64的校验和。这跟有利于文件的容错性，但是在保存rdb文件的时候，会有大概10%的性能损耗，所以如果你追求高性能，可以关闭该配置。
dbfilename "along.rdb"     #rdb文件的名称
```

` 限制`相关配置(LIMITS)

```shell
### LIMITS ###
maxclients 10000    #设置能连上redis的最大客户端连接数量
maxmemory <bytes>    #redis配置的最大内存容量。当内存满了，需要配合maxmemory-policy策略进行处理。
maxmemory-policy noeviction    #淘汰策略：volatile-lru, allkeys-lru, volatile-random, allkeys-random, volatile-ttl, noeviction
内存容量超过maxmemory后的处理策略：
① # volatile-lru：利用LRU算法移除设置过过期时间的key。
② # volatile-random：随机移除设置过过期时间的key。
③ # volatile-ttl：移除即将过期的key，根据最近过期时间来删除（辅以TTL）
④ # allkeys-lru：利用LRU算法移除任何key。
⑤ # allkeys-random：随机移除任何key。
⑥ # noeviction：不移除任何key，只是返回一个写错误。
# 上面的这些驱逐策略，如果redis没有合适的key驱逐，对于写命令，还是会返回错误。redis将不再接收写请求，只接收get请求。写命令包括：set setnx
maxmemory-samples 5   #淘汰算法运行时的采样样本数；
```

`持久化`配置(APPEND ONLY MODE)

```shell
### APPEND ONLY MODE ###
# 默认redis使用的是rdb方式持久化，这种方式在许多应用中已经足够用了。但是redis如果中途宕机，会导致可能有几分钟的数据丢失，根据save来策略进行持久化，Append Only File是另一种持久化方式，可以提供更好的持久化特性。Redis会把每次写入的数据在接收后都写入 appendonly.aof 文件，每次启动时Redis都会先把这个文件的数据读入内存里，先忽略RDB文件。
appendonly no    #不启动aof模式
appendfilename "appendonly.aof"    #据读入内存里，先忽略RDB文件，aof文件名(default: "appendonly.aof")
appendfsync
Redis supports three different modes:
no：redis不执行主动同步操作，而是OS进行；
everysec：每秒一次；
always：每语句一次；
no-appendfsync-on-rewrite no
#在aof重写或者写入rdb文件的时候，会执行大量IO，此时对于everysec和always的aof模式来说，执行fsync会造成阻塞过长时间，no-appendfsync-on-rewrite字段设置为默认设置为no。如果对延迟要求很高的应用，这个字段可以设置为yes，否则还是设置为no，这样对持久化特性来说这是更安全的选择。设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,等rewrite完成后再写入，默认为no，建议yes。Linux的默认fsync策略是30秒。可能丢失30秒数据。

auto-aof-rewrite-percentage 100 aof自动重写配置。当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，即当aof文件增长到一定大小的时候Redis能够调用bgrewrite aof对日志文件进行重写。当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。

auto-aof-rewrite-min-size 64mb #设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写。上述两个条件同时满足时，方会触发重写AOF；与上次aof文件大小相比，其增长量超过100%，且大小不少于64MB;

aof-load-truncated yes #指redis在恢复时,会忽略最后一条可能存在问题的指令。aof文件可能在尾部是不完整的，出现这种现象，可以选择让redis退出，或者导入尽可能多的数据。如果选择的是yes，当截断的aof文件被导入的时候，会自动发布一个log给客户端然后load。
如果是no，用户必须手动redis-check-aof修复AOF文件才可以。
注意：持久机制本身不能取代备份；应该制订备份策略，对redis库定期备份；Redis服务器启动时用持久化的数据文件恢复数据，会优先使用AOF；
```

redis之持久化存储读取

![](./img/66.png)

`RDB`：snapshotting, 二进制格式；按事先定制的策略，周期性地将数据从内存同步至磁盘；数据文件默认为dump.rdb；
　　客户端显式使用SAVE或BGSAVE命令来手动启动快照保存机制；
　　SAVE：同步，即在主线程中保存快照，此时会阻塞所有客户端请求；
　　BGSAVE：异步；backgroud
`AOF`：Append Only File, fsync
　　记录每次写操作至指定的文件尾部实现的持久化；当redis重启时，可通过重新执行文件中的命令在内存中重建出数据库；
　　BGREWRITEAOF：AOF文件重写；
　　不会读取正在使用AOF文件，而是通过将内存中的数据以命令的方式保存至临时文件中，完成之后替换原来的AOF文件；



慢查询日志相关配置(SLOW LOG)

```shell
### SLOW LOG ###
slowlog-log-slower-than 10000    #当命令的执行超过了指定时间，单位是微秒；1s=10^6微秒
slowlog-max-len 128   #慢查询日志长度。当一个新的命令被写进日志的时候，最老的那个记录会被删掉。
ADVANCED配置：
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
设置ziplist的键数量最大值，每个值的最大空间；
```

###### redis命令

└── bin
├── redis-benchmark #redis性能测试工具，可以测试在本系统本配置下的读写性能
├── redis-check-aof #对更新日志appendonly.aof检查，是否可用
├── redis-check-dump #用于检查本地数据库的rdb文件
├── redis-cli #redis命令行操作工具，也可以用telnet根据其纯文本协议来操作
├── redis-sentinel Redis-sentinel 是Redis实例的监控管理、通知和实例失效备援服务，是Redis集群的管理工具
└── redis-server #redis服务器的daemon启动程序

`redis-cli`

```shell
redis-cli -p 6379　　#默认选择 db库是 0
redis 127.0.0.1:6379> keys *　　#查看当前所在“db库”所有的缓存key
redis 127.0.0.1:6379> select 8　　#选择 db库
redis 127.0.0.1:6379> FLUSHALL　　#清除所有的缓存key
redis 127.0.0.1:63798> FLUSHDB　　#清除当前“db库”所有的缓存key
redis 127.0.0.1:6379> set keyname keyvalue　　#设置缓存值
redis 127.0.0.1:6379> get keyname #获取缓存值
redis 127.0.0.1:6379> del keyname #删除缓存值：返回删除数量（0代表没删除）
```

`服务端`的相关命令

```shell
time：返回当前服务器时间
client list: 返回所有连接到服务器的客户端信息和统计数据 参见http://redisdoc.com/server/client_list.html
client kill ip:port：关闭地址为 ip:port 的客户端
save：将数据同步保存到磁盘
bgsave：将数据异步保存到磁盘
lastsave：返回上次成功将数据保存到磁盘的Unix时戳
shundown：将数据同步保存到磁盘，然后关闭服务
info：提供服务器的信息和统计
config resetstat：重置info命令中的某些统计数据
config get：获取配置文件信息
config set：动态地调整 Redis 服务器的配置(configuration)而无须重启，可以修改的配置参数可以使用命令 CONFIG GET * 来列出
config rewrite：Redis 服务器时所指定的 redis.conf 文件进行改写
monitor：实时转储收到的请求
slaveof：改变复制策略设置
debug：sleep segfault
slowlog get：获取慢查询日志
slowlog len：获取慢查询日志条数
slowlog reset：清空慢查询
```

常用`数据类型`

![](./img/5140.png)

Redis内部使用一个`redisObject`对象来表示所有的key和value,redisObject最主要的信息如上图所示：
　　`type`代表一个value对象具体是何种数据类型
　　`encoding`是不同数据类型在redis内部的存储方式
　　比如：`type=strin`g代表value存储的是一个普通字符串，那么对应的encoding可以是raw或者是int,如果是int则代表实际redis内部是按数值型类存储和表示这个字符串的，当然前提是这个字符串本身可以用数值表示，比如:“123” "456"这样的字符串。
　　Redis的键值可以使用物种数据类型：`字符串`，`散列表`，`列表`，`集合`，`有序集合`。



对`KEY`操作的命令

```shell
exists(key)：确认一个key是否存在
del(key)：删除一个key
type(key)：返回值的类型
keys(pattern)：返回满足给定pattern的所有key
randomkey：随机返回key空间的一个
keyrename(oldname, newname)：重命名key
dbsize：返回当前数据库中key的数目
expire：设定一个key的活动时间（s）
ttl：获得一个key的活动时间
move(key, dbindex)：移动当前数据库中的key到dbindex数据库
flushdb：删除当前选择数据库中的所有key
flushall：删除所有数据库中的所有key
```

 对`String`操作的命令

```shell
set(key, value)：给数据库中名称为key的string赋予值value
get(key)：返回数据库中名称为key的string的value
getset(key, value)：给名称为key的string赋予上一次的value
mget(key1, key2,…, key N)：返回库中多个string的value
setnx(key, value)：添加string，名称为key，值为value
setex(key, time, value)：向库中添加string，设定过期时间time
mset(key N, value N)：批量设置多个string的值
msetnx(key N, value N)：如果所有名称为key i的string都不存在
incr(key)：名称为key的string增1操作
incrby(key, integer)：名称为key的string增加integer
decr(key)：名称为key的string减1操作
decrby(key, integer)：名称为key的string减少integer
append(key, value)：名称为key的string的值附加value
substr(key, start, end)：返回名称为key的string的value的子串
```

对`Hash`操作的命令

```shell
hset(key, field, value)：向名称为key的hash中添加元素field
hget(key, field)：返回名称为key的hash中field对应的value
hmget(key, (fields))：返回名称为key的hash中field i对应的value
hmset(key, (fields))：向名称为key的hash中添加元素field
hincrby(key, field, integer)：将名称为key的hash中field的value增加integer
hexists(key, field)：名称为key的hash中是否存在键为field的域
hdel(key, field)：删除名称为key的hash中键为field的域
hlen(key)：返回名称为key的hash中元素个数
hkeys(key)：返回名称为key的hash中所有键
hvals(key)：返回名称为key的hash中所有键对应的value
hgetall(key)：返回名称为key的hash中所有的键（field）及其对应的value
```

 对`List`操作的命令

```shell
rpush(key, value)：在名称为key的list尾添加一个值为value的元素
lpush(key, value)：在名称为key的list头添加一个值为value的 元素
llen(key)：返回名称为key的list的长度
lrange(key, start, end)：返回名称为key的list中start至end之间的元素
ltrim(key, start, end)：截取名称为key的list
lindex(key, index)：返回名称为key的list中index位置的元素
lset(key, index, value)：给名称为key的list中index位置的元素赋值
lrem(key, count, value)：删除count个key的list中值为value的元素
lpop(key)：返回并删除名称为key的list中的首元素
rpop(key)：返回并删除名称为key的list中的尾元素
blpop(key1, key2,… key N, timeout)：lpop命令的block版本。
brpop(key1, key2,… key N, timeout)：rpop的block版本。
rpoplpush(srckey, dstkey)：返回并删除名称为srckey的list的尾元素，并将该元素添加到名称为dstkey的list的头部
```

 对`Set`操作的命令

求交集、并集、差集

```shell
sadd(key, member)：向名称为key的set中添加元素member
srem(key, member)：删除名称为key的set中的元素member
spop(key)：随机返回并删除名称为key的set中一个元素
smove(srckey, dstkey, member)：移到集合元素
scard(key)：返回名称为key的set的基数
sismember(key, member)：member是否是名称为key的set的元素
sinter(key1, key2,…key N)：求交集
sinterstore(dstkey, (keys))：求交集并将交集保存到dstkey的集合
sunion(key1, (keys))：求并集
sunionstore(dstkey, (keys))：求并集并将并集保存到dstkey的集合
sdiff(key1, (keys))：求差集
sdiffstore(dstkey, (keys))：求差集并将差集保存到dstkey的集合
smembers(key)：返回名称为key的set的所有元素
srandmember(key)：随机返回名称为key的set的一个元素
```

##### 应用-主从复制

###### 复制方式

Redis的复制方式有两种；

一种是主（master）-从（slave）模式，

一种是从（slave）-从（slave）模式，

因此Redis的复制拓扑图会丰富一些，可以像星型拓扑，也可以像个有向无环。一个Master可以有多个slave主机，支持链式复制；Master以非阻塞方式同步数据至slave主机；

![](./img/619.png)

###### 复制优点

-  **高可用性**
-  **高性能**
-  **水平扩展性**

###### 需要解决的问题

-  数据一致性问题： 如何保证master服务器写入的数据能够及时同步到slave机器上。
-  **读写分离：** 如何在客户端提供读写分离的实现方案，通过客户端实现将读写请求分别路由到master和slave实例上。
-  Redis服务实现一直在演变，致力于解决的一个问题：`复制实时性和数据一致性矛盾`,Redis的复制实时性和数据一致性是存在矛盾的

#### 常见问题

##### 缓存穿透和缓存雪崩

###### 缓存穿透

1、概念
缓存穿透的概念很简单，用户想要查询一个数据，发现redis内存数据库没有，也就是缓存没有命中，于是向持久层数据库查询。发现也没有，于是本次查询失败。当用户很多的时候，缓存都没有命中，于是都去请求了持久层数据库。这会给持久层数据库造成很大的压力，这时候就相当于出现了缓存穿透。

即黑客故意去请求缓存中不存在的数据，导致所有的请求都怼到数据库上，从而数据库连接异常。
注意：和缓存击穿的区别，缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。
解决方案:
(一)利用互斥锁，缓存失效的时候，先去获得锁，得到锁了，再去请求数据库。没得到锁，则休眠一段时间重试
(二)采用异步更新策略，无论key是否取到值，都直接返回。value值中维护一个缓存失效时间，缓存如果过期，异步起一个线程去读数据库，更新缓存。需要做缓存预热(项目启动前，先加载缓存)操作。
(三)提供一个能迅速判断请求是否有效的拦截机制，比如，利用布隆过滤器，内部维护一系列合法有效的key。迅速判断出，请求所携带的Key是否合法有效。如果不合法，则直接返回

###### 缓存雪崩

1、概念
缓存雪崩是指，缓存层出现了错误，不能正常工作了。于是所有的请求都会达到存储层，存储层的调用量会暴增，造成存储层也会挂掉的情况
缓存雪崩，即缓存同一时间大面积的失效，这个时候又来了一波请求，结果请求都怼到数据库上，从而导致数据库连接异常。

解决方案:
（1）redis高可用
这个思想的含义是，既然redis有可能挂掉，那我多增设几台redis，这样一台挂掉之后其他的还可以继续工作，其实就是搭建的集群。
（2）限流降级
这个解决方案的思想是，在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待。
（3）数据预热
数据加热的含义就是在正式部署之前，我先把可能的数据先预先访问一遍，这样部分可能大量访问的数据就会加载到缓存中。在即将发生大并发访问前手动触发加载缓存不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀
(4) 给缓存的失效时间，加上一个随机值，避免集体失效。

### MongoDB

### MySQL

##### 优化思路和方向

MySQL主从读写分离

分库分表

### PostgreSQL

### InfluxDB


