# [返回](/)

# Redis概述

## 作用

K-V + Cache + Persistence

### 3V/3高

3V

- Volume：海量
- Variety：多样
- Velocity：实时

3高

- 高可扩
- 高并发
- 高性能

### 当前多源海量数据是如何存储的？

以某电商平台为例

- 商品基本信息（变动较少的、固定的）：MySQL
- 商品描述、详情、评价（多文字类，IO读写慢）：MongoDB
- 商品图片：分布式的文件系统，如淘宝的TFS，Google的GFS，Hadoop的HDFS
- 商品的关键字：ISearch、ES
- 商品波段性的热点高频信息：Redis
- 商品的交易、价格计算、积分累计：第三方接口

面对海量、多源、重构的数据，淘宝的解决方案：

**UDSL（统一数据平台服务层）**

![img](img\redis\1.png)

### 为什么要NoSQL？

不同类型的数据处理时，对于关系性数据库如mysql，要进行表的join等等操作，而高并发的场景是不太建议有关联查询的

> 往往采用冗余数据的方式来避免

分布式事务支持不了太多并发

对于海量数据分布式场景，mysql由于并非分布式数据库，因此不是那么适合。随后便出现了nosql。nosql并不是为了取代mysql，而是为分布式场景而生，弱结构化，但是对事务支持的不好

NoSQL是聚合模型，json字符串的拼接更改和删除比较容易

## NoSQL四大分类

- K-V键值对：Redis
- 文档型数据库：MongoDB
- 列存储数据库：HBase
- 图数据库（社交网络、推荐系统等）：Neo4J、InfoGrid

## CAP+BASE

[https://www.cnblogs.com/summer108/p/9783033.html](https://www.cnblogs.com/summer108/p/9783033.html)

### CAP

[https://www.zhihu.com/question/54105974](https://www.zhihu.com/question/54105974)

#### 传统的ACID

- Atomicity：原子性
- Consistency：一致性
- Isolation：隔离性
- Durability：持久性

#### CAP

- Consistency：一致性，在分布式系统中的所有数据备份，在同一时刻是否同样的值（等同于所有节点访问同一份最新的数据副本）

- Availability：可用性，主要针对给用户提供的服务层面，保证每个请求不管成功或者失败都能在一定时间内作出响应

- Partition tolerance：分区容忍性，主要针对于分布式系统内的数据通信层面。分布式系统容忍出现分区的情况。即两节点之间可能无法通信，即出现分区，但系统仍然能够持续运行，对外提供满足有效性或一致性的服务

  > 当你一个数据项只在一个节点中保存，那么分区出现后，和这个节点不连通的部分就访问不到这个数据了。这时分区就是无法容忍的

对于传统的关系型数据库，同时满足ACID，然而对于NoSQL数据库，只能满足CAP三个其中的两个，即**CAP的3进2**

#### 理论核心

**一个分布式系统不可能同时很好的满足一致性、可用性和分区容错性这三个需求**

#### 分类（CA、CP、AP）

根据CAP原理将NoSQL数据库分成了三类：

- CA：如果要满足数据一致性和可用性，就必须不能出现分区的情况，因为分区会导致节点之间无法通信，满足一致性需要恢复节点之间的通信，牺牲了可用性；而满足可用性的话，就必须要及时响应请求，数据通信恢复和同步的时间就不足，降低了一致性
  - 所以只能满足于非集群的单点服务，可扩展性不大。如传统的Oracle数据库
- CP：如果要满足分区容忍性和一致性，就必须有及时的数据同步时间，请求可能会被阻塞，可用性降低
  - 通常性能不强
- AP：如果要满足可用性、分区容忍性，就必须及时响应用户请求，数据同步的时间被压缩，牺牲分区后节点之间的数据一致性
  - 大多数网站的架构采用AP

![image-20211008202450997](img\redis\2.png)

由于网络硬件肯定会出现延迟丢包等问题，所以分区容错性是我们必须需要实现的。所以我们只能在一致性和可用性之间进行权衡，没有NoSQL系统能同时保证这三点

#### 总结

总的来说就是，数据存在的节点越多，分区容忍性越高，但要复制更新的数据就越多，一致性就越难保证。为了保证一致性，更新所有节点数据所需要的时间就越长，可用性就会降低

### BASE理论

#### 弱化的关系型数据库特性

对于web2.0网站来说，关系数据库的很多主要特性却往往无用武之地。

1. 数据库事务一致性需求：很多web实时系统并不要求严格的数据库事务，对读一致性的要求很低，有些场合对写一致性要求并不高。允许实现最终一致性
2. 数据库的写实时性和读实时性需求：对关系数据库来说，插入一条数据之后立刻查询，是肯定可以读出来这条数据的，但是对于很多web应用来说，并不要求这么高的实时性，比方说发一条消息之 后，过几秒乃至十几秒之后，我的订阅者才看到这条动态是完全可以接受的
3. 对复杂的SQL查询，特别是多表关联查询的需求：任何大数据量的web系统，都非常忌讳多个大表的关联查询，以及复杂的数据分析类型的报表查询，特别是SNS类型的网站，从需求以及产品设计角 度，就避免了这种情况的产生。往往更多的只是单表的主键查询，以及单表的简单条件分页查询，SQL的功能被极大的弱化了

#### BASE

针对关系型数据库强一致性造成的可用性降低的问题，BASE理论用于提供一个解决方案

- BA：Basically Available，基本可用
  - 响应时间增加或服务稍微降级等等
- S：Soft state，软状态
  - 软状态指允许系统中的数据存在中间状态，主要是同步过程之中，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时，这个过程中可以接受一定的数据弱一致性
- E：Eventually consistence，最终一致性
  - 最终一致性强调的是所有的数据副本，在经过一段时间的同步之后，最终都能够达到一个一致的状态。因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不需要实时保证系统数据的强一致性

它的思想是通过**让系统放松对某一时刻数据一致性的要求来换取系统整体伸缩性和性能上改观**。由于大型系统往往由于地域分布和极高性能的要求，不可能采用分布式事务来完成这些指标，要想获得这些指标，我们必须采用另外一种方式来完成，这里BASE就是解决这个问题的办法

#### 分布式和集群

简单来说：

1. 分布式：不同的多台服务器上面部署不同的服务模块(工程)，他们之间通过Rpc/Rmi之间通信和调用，对外提供服务和组内协作
2. 集群：不同的多台服务器上面部署相同的服务模块、通过分布式调度软件进行统一的调度，对外提供服务和访问

# Redis入门

Redis：REmote Dictionary Server（远程字典服务器）

## 特点

- 完全开源免费
- C语言编写，遵守BSD协议
- 高性能的K-V分布式**内存**数据库，且支持持久化（异步，不影响服务）

Redis和其他K-V缓存数据库有以下三个特点：

1. 支持持久化
2. 不仅支持简单的K-V，还支持list、set、zset、hash等
3. 支持主从模式的数据备份

## 使用

### 本地安装

1. `tar -zxvf redis-6.2.6.tar.gz`
2. `cd redis-6.2.6`
3. `make`
4. `make install`

#### 启动Redis

![image-20211012100652163](img\redis\5.png)

创建一个文件夹用来放`redis.conf`的副本（注意：不要修改原文件）

打开副本.conf文件，修改`daemorize no`为`daemorize yes`，:wq保存退出，这样就可以以后台守护进程的方式启动了

![image-20211012100605526](img\redis\4.png)

用副本文件启动redis，`ps -ef|grep redis`打开进程显示，发现已经启动了

![image-20211012100927381](img\redis\6.png)

测试一下

![image-20211012101119638](img\redis\7.png)

> 关闭：shutdown
>
> ![image-20211012112232334](img\redis\10.png)

### docker安装

- `docker pull redis`

- `docker run --name redis -v /root/data/redis_data:/data -itd -p 6379:6379 redis --requirepass "123456" --appendonly yes`

  > 用户名为default，密码123456

  > **注意！**
  >
  > 1. redis默认镜像是没有redis.conf的，需要从网上下载或从其他地方复制；如果不用redis.conf来启动，则参数只能写在docker命令行中一大串，比较繁琐
  > 2. 可以使用自定义的redis.conf来启动

#### 使用自定义redis.conf安装

1. 在本机的某个目录准备好自己下载的`redis.conf`文件，这里我放在了`/root/data/conf/redis.conf`下

2. 挂载到容器内某个目录，这里我选择了`/etc/redis/redis.conf`

   ![image-20211012152532145](img\redis\3.png)

   可以使用`docker inspect [container's name]`来查看

   ![image-20211012174635573](img\redis\15.png)

3. 使用自定义的`redis.conf`来启动

   `docker run --name redis -v /root/data/redis_data:/data -v /root/data/conf/redis.conf:/etc/redis/redis.conf -itd -p 6379:6379 redis redis-server /etc/redis/redis.conf --requirepass "123456" --appendonly yes`

   > **注意！**
   >
   > 1. 若直接使用网上的redis.conf，外网可能访问不了，打开redis.conf看一下可以发现：
   >
   > ![image-20211012154309006](img\redis\11.png)
   >
   > 只绑定了服务器的本地网卡，自然访问不了，有两种解决方式：① 注释掉bind那一行，监听所有网卡；② protected-mode改为no。个人感觉方式一较好
   >
   > ![image-20211012155923417](img\redis\13.png)
   >
   > 2. 注意改成非守护进程运行，否则运行不了
   >
   > ![image-20211012154558695](img\redis\12.png)
   >
   > 即这里改为no
   >
   > 3. requirepass只能指定default用户的密码，每次重启后其他用户会丢失，需要在redis.conf里面显式记录下来
   >
   > ![image-20211012173054223](img\redis\14.png)

### 常用基本命令

- `select [num]`：切换数据库
- `config get [parameter]`：获取某条配置信息
- `config set [parameter] [value]`：更改某条配置信息
- `set [key] [value]`：添加一条数据
- `get [key]`：查询
- `keys *`：查询所有key
- `keys x?`：查询形如xx的key，?可以匹配任意一个字符
- `exists [key]`：查询是否存在某种key
- `move [key] [db]`：移动某条数据到某个库
- `expire [key] [expireTime]`：设置key的过期时间，以秒为单位
- `ttl [key]`：返回key的剩余生存时间
  - -1：无过期时间
  - -2：键不存在
  - 正数：剩余时间
- `persisit [key]`：去除某key的过期时间
- `flushdb`：清当前库
- `flushall`：清所有库
- `dbsize`：查看有多少key
- `type [key]`：查看key的类型

......

### Redis-SCAN

这个指令使用非常简单，提供一个简单的正则字符串即可，但是有很明显的两个**缺点** 

1. 没有 offset、limit 参数，一次性吐出所有满足条件的 key，万一实例中有几百 w 个key 满足条件，当你看到满屏的字符串刷的没有尽头时，你就知道难受了
2. keys 算法是遍历算法，复杂度是 O(n)，如果实例中有千万级以上的 key，这个指令就会导致 Redis 服务卡顿，所有读写 Redis 的其它的指令都会被延后甚至会超时报错，因为Redis 是单线程程序，顺序执行所有指令，其它指令必须等到当前的 keys 指令执行完了才可以继续

---

Redis 为了解决这个问题，它在 2.8 版本中加入了大海捞针的指令——scan。scan 相比keys 具备有以下特点: 

1. 复杂度虽然也是 O(n)，但是它是通过游标分步进行的，不会阻塞线程
2. 提供 limit 参数，可以控制每次返回结果的最大条数，limit 只是一个 hint，返回的结果可多可少
3. 同 keys 一样，它也提供模式匹配功能
4. 服务器不需要为游标保存状态，游标的唯一状态就是 scan 返回给客户端的游标整数
5. 返回的结果可能会有重复，需要客户端去重复，这点非常重要
6. 遍历的过程中如果有数据修改，改动后的数据能不能遍历到是不确定的 
7. 单次返回的结果是空的并不意味着遍历结束，而要看返回的游标值是否为零

**scan基础使用**

scan 参数提供了三个参数，第一个是 cursor 整数值，第二个是 key 的正则模式，第三个是遍历的 limit hint。第一次遍历时，cursor 值为 0，然后将返回结果中第一个整数值作为下一次遍历的 cursor。**一直遍历到返回的 cursor 值为 0 时结束**

`scan [cursor] match [pattern] count [limit]`：cursor从0开始，以0结束；limit约为遍历的字典槽位数量

#### 大Key

**大** **key** **扫描**

有时候会因为业务人员使用不当，在 Redis 实例中会形成很大的对象，比如一个很大的hash，一个很大的 zset 这都是经常出现的。这样的对象对 Redis 的集群数据迁移带来了很大的问题，因为在集群环境下，如果某个 key 太大，会数据导致迁移卡顿。另外在内存分配上，如果一个 key 太大，那么当它需要扩容时，会一次性申请更大的一块内存，这也会导致卡顿。如果这个大 key 被删除，内存会一次性回收，卡顿现象会再一次产生

**在平时的业务开发中，要尽量避免大** **key** **的产生。**

如果你观察到 Redis 的内存大起大落，这极有可能是因为大 key 导致的，这时候你就需要定位出具体是那个 key，进一步定位出具体的业务来源，然后再改进相关业务代码设计

为了避免对线上 Redis 带来卡顿，这就要用到 scan 指令，对于扫描出来的每一个key，使用 type 指令获得 key 的类型，然后使用相应数据结构的 size 或者 len 方法来得到它的大小，对于每一种类型，保留大小的前 N 名作为扫描结果展示出来

上面这样的过程需要编写脚本，比较繁琐，不过 Redis 官方已经在 redis-cli 指令中提供了这样的扫描功能，我们可以直接拿来即用

`redis-cli -h 127.0.0.1 -p 7001 –-bigkeys`

如果你担心这个指令会大幅抬升 Redis 的 ops 导致线上报警，还可以增加一个休眠参数

`redis-cli -h 127.0.0.1 -p 7001 –-bigkeys -i 0.1`

上面这个指令每隔 100 条 scan 指令就会休眠 0.1s，ops 就不会剧烈抬升，但是扫描的时间会变长

### Redis-unlink

删除指令 del 会直接释放对象的内存，大部分情况下，这个指令非常快，没有明显延迟。不过如果删除的 key 是一个非常大的对象，比如一个包含了千万元素的 hash，那么删除操作就会导致单线程卡顿

Redis 为了解决这个卡顿问题，在 4.0 版本引入了 unlink 指令，它能对删除操作进行懒处理，丢给后台线程来异步回收内存

```sh
\> unlink key
OK
```

如果有多线程的开发经验，你肯定会担心这里的线程安全问题，会不会出现多个线程同时并发修改数据结构的情况存在。实际上不会，当 unlink 指令发出时，它就再也无法被主线程中的其它指令访问到了

#### flush

Redis 提供了 flushdb 和 flushall 指令，用来清空数据库，这也是极其缓慢的操作

Redis 4.0 同样给这两个指令也带来了异步化，在指令后面增加 async 参数就可以将整棵大树连根拔起，扔给后台线程慢慢焚烧

```sh
\> flushall async
OK
```

#### 异步队列

主线程将对象的引用从「大树」中摘除后，会将这个 key 的内存回收操作包装成一个任务，塞进异步任务队列，后台线程会从这个异步队列中取任务。任务队列被主线程和异步线程同时操作，所以必须是一个线程安全的队列

![image-20211115150917437](img\redis\102.png)

不是所有的 unlink 操作都会延后处理，如果对应 key 所占用的内存很小，延后处理就没有必要了，这时候 Redis 会将对应的 key 内存立即回收，跟 del 指令一样

#### AOF Sync

Redis 需要每秒一次(可配置)同步 AOF 日志到磁盘，确保消息尽量不丢失，需要调用sync 函数，这个操作会比较耗时，会导致主线程的效率下降，所以 Redis 也将这个操作移到异步线程来完成。执行 AOF Sync 操作的线程是一个独立的异步线程，和前面的懒惰删除线程不是一个线程，同样它也有一个属于自己的任务队列，队列里只用来存放 AOF Sync 任务

#### 更多异步删除点

Redis 回收内存除了 del 指令和 flush 之外，还会存在于在 key 的过期、LRU 淘汰、rename 指令以及从库全量同步时接受完 rdb 文件后会立即进行的 flush 操作

Redis4.0 为这些删除点也带来了异步删除机制，打开这些点需要额外的配置选项

1. `slave-lazy-flush `从库接受完 rdb 文件后的 flush 操作
2. `lazyfree-lazy-eviction` 内存达到 maxmemory 时进行淘汰
3. `lazyfree-lazy-expire key` 过期删除
4. `lazyfree-lazy-server-del rename` 指令删除 destKey

# Redis五大基本数据类型

redis的结构可以看作一个大的全局的hashmap，其中它的key对应的value有五种不同的结构，每个结构底层又采用不同的实现

> 比如，对于set key 111
>
> ```sh
> Aliyun2G:0>object encoding key
> "int"
> ```
>
> 虽然是redis的string类型，但底层是用int存储的。redis存string类型时，会先尝试强转成int，若失败才会变成embstr或raw

---

命令中：

- ex：set with expire
- nx：set if not exists
- xx：set if exists

---

list/set/hash/zset 这四种数据结构是容器型数据结构，它们共享下面两条通用规则：

1. create if not exists

如果容器不存在，那就创建一个，再进行操作。比如 rpush 操作刚开始是没有列表的，Redis 就会自动创建一个，然后再 rpush 进去新元素

2. drop if no elements

如果容器里元素没有了，那么立即删除元素，释放内存。这意味着 lpop 操作到最后一个元素，列表就消失了

## String

redis最基本的数据结构，K-V键值对，String→String

### 常用命令

- `set/get/del/append/strlen`

- `incr [key]/decr [key]/incrby [key] x/decrby [key] x`：+1，-1，+x，-x

- `getrange [key] left right/setrange [key] begin xxx`：get对应value的 [left, right] 部分（截取字符串，包含right），set对应value的begin之后的部分

- `setex [key] [expireTime] [value]/setnx [key] [value]`：set with expire；set if not exists（如果存在，则会失败）

- `mset [key1] [value1] [key2] [value2] .../mget/msetnx`：批量get/set

  > 例：如果要get n次，用get方法的话，需要n次网络通讯，n次内存操作；而mget只需要一次网络通讯

- `append [key] [str]`：追加字符串

- `strlen`：字符串长度

- `getset [key] [value]`：get原值并set新值，有点像原子操作中的getAndAdd

### 使用场景

- 缓存

- 计数器

- 共享的session

  > 用户通过nginx访问服务器集群，有多个tomcat，登录信息不可能只保存在一个tomcat上，因为下次nginx可能就为用户分配了另一个服务器，这时候可以通过共享的session来保存用户信息

- 限流

- 分布式锁

### 键名规范

企业生产使用string结构，为了防止多个不同业务组的键名冲突，一般建议键名设置为：

**`业务名:对象名:id:属性`**

## Hash

类似Java中的Map<String, Map<String, Object>>，K-V模式不变，但V是一个键值对

比如对于User{id=1, name="zhangsan"}，就可以存为key→user（user实际上就是若干个键值对）

即：**key→hash(field→value)**

> 例：存储User{"name":"zhangsan", "age":"22"}
>
> - 用字符串存
>
>   ```shell
>   set groupA:user:1:name zhangsan
>   set groupA:user:1:age 22
>   ```
>
>   ![image-20211028155415047](img\redis\57.png)
>
> - 用hash存
>
>   ```shell
>   hmset groupA:user:1 name zhangsan age 22
>   ```
>
>   ![image-20211028155649156](img\redis\58.png)

### 常用命令

- `hset [key] [field] [value]`：在key中加入一个field→\value键值对

- `hget [key] [field]`：获取field的value

- `hmset [key] [field1] [value1] [field2] [value2] ...`：插入若干个键值对

- `hmget [key] [field1] [field2] ...`：批量拿出values

- `hgetall [key]`：拿出所有键值对

  > 慎用：由于内存操作是单线程的，可能会导致阻塞

- `hdel [key] [field]`：删除key里面的field这一条

- `hlen [key]`：key中键值对的个数

- `hexists [key] [field]`：key中有无field

- `hkeys [key]`：获取key所有field

- `hvals [key]`：获取key所有value

- `hincrby [key] [field] [N]`：给key中field对应的记录加N

- `hincrbyfloat [key] [field] [N]`：给key中field对应的记录加N（float）

- `hsetnx [key] [field] [value]`：set if not exists

### rehash

不同的是，Redis 的字典的值只能是字符串，另外它们 rehash 的方式不一样，因为 Java 的 HashMap 在字典很大时，rehash 是个耗时的操作，需要一次性全部 rehash。Redis为了高性能，不能堵塞服务，所以采用了渐进式 rehash 策略。渐进式 rehash 会在 rehash 的同时，保留新旧两个 hash 结构，查询时会同时查询两个hash 结构，然后在后续的定时任务中以及 hash 的子指令中，循序渐进地将旧 hash 的内容一点点迁移到新的 hash 结构中

![image-20211108165612901](img\redis\93.png)

### 使用场景

- 存储对象

  > 存储对象的三种方法：
  >
  > 1. 原生字符串
  >
  >    ```shell
  >    set user:1:name zhangsan
  >    set user:1:age 22
  >    ```
  >
  >    - 优点：简单直观，每个键对应一个值
  >    - 缺点：① 键值过多，有多少个属性就要多少个键，占用内存多；② 对象信息分散，不利于生产环境
  >
  > 2. 序列化成字符串后存储
  >
  >    ```shell
  >    set user:1:serialize(User{"name":"zhangsan", "age":22})
  >    ```
  >
  >    - 优点：编程简单，序列化好的话内存利用率高
  >    - 缺点：序列化和反序列化需要开销，更改属性麻烦
  >
  > 3. hash存储
  >
  >    ```shell
  >    hset user:1 name zhangsan age 22
  >    ```
  >
  >    - 优点：简单直观，合理使用的话内存低
  >
  >    - 缺点：要控制ziplist和hashtable两种编码的转换，且hashtable消耗内存高
  >
  >      > redis的hash底层是由ziplist和hashtable实现的
  >      
  >    - 对于商品库存这类数据或者是余额等等信息，用hash比较好，因为用json需要先将对象取出来，然后修改余额，再重新序列化；而hash只需要用一个hset指令
  
- 购物车

![image-20211106225900953](img\redis\79.png)

## List

实际上是个链表，头尾都可以添加

### 常用命令

- `lpush [key] [element1] [element2] .../rpush/lrange [key] [left] [right(include)]` ：从左/右添加/从list中取

  > lpush list1 1 2 3 4 5 → [5 4 3 2 1]
  >
  > lrange list1 0 1 → 5 4

- `lpop [key]/rpop`：从左/右栈顶弹出

- `lindex [index]`：查找列表中索引index的元素

- `llen [key]`：计算长度

- `lrem [key] [N] [value]`：删除n个指定value的记录

- `ltrim [key] [left] [right]`：截取key的[left, right]范围的元素，重新赋给key

  > ![image-20211013004334589](img\redis\16.png)

- `rpoplpush [key1] [key2]`：从key1中rpop一个，lpush到key2中

  > ![image-20211013010451729](img\redis\17.png)
  >
  > [3 2 1] [9 8 7] → [3 2] [1 9 8 7]

- `lset [key] [index] [value]`：将index的位置设置为value

- `linsert [key] before/after [value1] [value] `：在值value1前/后插入一个value

- `blpop/brpop [timeout]`：阻塞式从左边/右边弹出，阻塞timeout秒（为0则一直阻塞）

### 使用场景

- 消息队列等
- 消息流

![image-20211106231026670](img\redis\80.png)

![image-20211107014411436](img\redis\81.png)

写扩散机制，相当于这里的“大V”向每个粉丝的消息列表中写数据，顺序写（粉丝数少的时候可以）

## Set

无重复元素的集合，元素为string类型

### 常用命令

- `sadd [key] [element1] [element2] ...`：添加
- `smembers [key]`：查看所有元素
- `sismember [key] [value]`：判断value是否在set中
- `scard [key]`：获取集合个数
- `srem [key] [value]`：删除某元素
- `srandmember [key] [N]`：随机出N个元素
- `spop [key]`：随机出一个元素
- `smove [key1] [key2] [value]`：在key1中移出一个value给key2
- `sdiff [key1] [key2]`：key1和key2的差集
- `sinter [key1] [key2]`：key1和key2的交集
- `sunion [key1] [key2]`：key1和key2的并集
- `sinterstore [key1] [key2] [key3]`：将key2和key3 inter的结果存储到key1

### 使用场景

- 用户标签、社交、查询有共同兴趣爱好的人
- 智能推荐
- 用户抽奖

![image-20211107021633785](img\redis\82.png)

- 朋友圈点赞

![image-20211107022810664](img\redis\83.png)

问题：

1. 这样的话自己自然是可以看见所有点赞用户，但朋友点进你的朋友圈，只能看到共同好友的点赞记录，这个功能如何实现？

   > 集合实现关注模型
   >
   > - 共同关注：sinter 我的关注列表 他的关注列表
   > - 我关注的人也关注了xxx：sismember 我的关注的人的关注列表 xxx 
   > - 我可能认识的人：sdiff 我关注的人的关注列表 我的关注列表
   >
   > ![image-20211107024513561](img\redis\84.png)
   >
   > 关注模型有什么用？举个例子，微博两个用户共同关注越多，表示兴趣爱好越接近，就可以推荐相同的内容

2. 点赞顺序？

   > zset解决

- 商品筛选

![image-20211107025603278](img\redis\85.png)

## Zset

sorted set，有序集合

每个元素都会关联一个double类型的分数，成员是唯一的，分数可以重复，通过分数实现排序

### 常用命令

- `zadd [key] [score1] [element1] [score2] [element2] ...`：给key添加元素时需附上分数

- `zrange [key] [left] [right(include)] ...`：显示[left, right]的元素（按分数升序）

- `zrevrange [key] [left] [right(include)] ...`：逆序显示[left, right]的元素（按分数降序）

- `zrange [key] [left] [right(include)] withscores ...`：显示[left, right]的元素及分数

- `zrangebyscore [key] [score1] [score2]`：显示分数在[score1, score2]的元素（按分数升序）
  - `zrangebyscore [key] [score1] ([score2]`：加个左括号，表示不包含score2
  - `zrangebyscore [key] [score1] [score2] limit [begin] [steps]`  ：显示从begin下标开始，分数在[score1, score2]的steps个元素
  
- `zrevrangebyscore [key] [score1] [score2]`：逆序显示分数在[score1, score2]的元素（按分数降序）

- `zrem [key] [value]`：删除某个元素

- `zcard [key]`：统计个数

- `zcount [key] [score1] [score2]`：统计[score1, score2]范围内元素个数

- `zrank [key] [value]`：获取元素对应位置

- `zscore [key] [value]`：获取元素对应分数

- `zrevrank [key] [value]`：获取元素对应逆序位置

- `+inf/-inf`：无限大/小

- `zinterstore [n] [key2] [key3] ... [keyn] weights [w1] [w2] ... [wn]`：将key2和key3和......和keyn inter的结果存储到key1，并可以计算n个zset中相同key对应记录的score的加权平均

  > 例子：
  >
  > ```shell
  > Aliyun2G:0>zadd test1 90 zhangsan 85 lisi
  > "2"
  > Aliyun2G:0>zadd test2 80 zhangsan 86 lisi
  > "2"
  > Aliyun2G:0>zinterstore test_avg 2 test1 test2 weights 0.65 0.35
  > "2"
  > Aliyun2G:0>zrange test_avg 0 -1 withscores
  > 1) "lisi"
  > 2) "85.349999999999994"
  > 3) "zhangsan"
  > 4) "86.5"
  > Aliyun2G:0>
  > ```

### 使用场景

- 排行榜，按点赞数排序等等

![image-20211107025824646](img\redis\86.png)

# Redis常用配置(v6.2.6)

配置文件一定要拷贝一份再处理，因为不能保证一定一次性修改正确

可以在redis命令行中查看和修改：`config get [name]`、`config set [name] [value]`

## UNITS

redis.conf文件里面只支持bytes单位，所以可以自定义一些单位如kb、mb等，类似于宏

![image-20211013113217075](img\redis\18.png)

大小写不敏感

## INCLUDES

可以包括其他的配置文件

![image-20211013145857135](img\redis\19.png)

## NETWORK

网络相关功能

![image-20211013150024098](img\redis\20.png)

主要包括：

- bind 127.0.0.1 -::1     指定监听的网卡，如果只绑定本地网卡（默认），则外网访问不了
- protected-mode yes     保护模式
- port 6379
- tcp-backlog 511
- timeout 0     连接闲置N秒后自动断开，默认为0，不断开
- tcp-keepalive 300     每隔N秒，进行一次keepalive检测，判断连接是否还在

## TLS/SSL

是否开启TLS/SSL连接

## GENERAL

一些通用配置

主要包括：

- daemonize no     是否以守护线程模式运行（docker启动时，这个要设置为no，否则会失败），默认为no

- pidfile /var/run/redis_6379.pid     守护进程下运行时，会将pid写入pidfile，这里设置pidfile的位置

- loglevel notice     开启的日志级别，有四种

  ![image-20211013151221525](img\redis\21.png)

- logfile ""     指定日志模式，默认为空字符串，即以标准输出来记录。如果在日志模式为标准输出，redis又是以守护进程运行的情况下，日志会被输出到 /dev/null，即回收站

  ![image-20211013151457921](img\redis\22.png)

- syslog-enabled no     是否将日志输出到系统日志

- syslog-ident redis     指定系统日志里面的日志标志，默认为redis

- syslog-facility local0     指定syslog的设备，可以是user或local0~local7，默认local0

- crash-log-enabled no     崩溃日志

- databases 16     数据库个数

## SNAPSHOTTING

RDB快照，见下方rdb章节

[RDB](#RDB)

## REPLICATION

复制

## KEYS TRACKING

## SECURITY

安全相关，默认是全部被注释掉的，除了`acllog-max-len 128`

主要包括：

- 启动用户的添加，默认为空

  ![image-20211013152636779](img\redis\23.png)

- requirepass foobared     启动时是否给default用户添加密码，默认被注释掉，但最好都添加个密码

## CLIENTS

主要用于设置最大客户端连接数

- maxclients 10000     默认10000

## MEMORY MANAGEMENT

内存相关

主要包括：

- maxmemory \<bytes>     设置最大内存，默认是注释掉的

- maxmemory-policy noeviction     内存满时的缓存淘汰策略，有七种

  ![image-20211013154053697](img\redis\24.png)

  - volatile-lru：针对设置了过期时间的key，采用lru（最近最少使用）移除key
  - allkeys-lru：针对所有key，采用lru移除
  - volatile-lfu：针对针对设置了过期时间的key，采用lfu（最低频率使用）移除key
  - allkeys-lfu：针对所有key，采用lfu（最低频率使用）移除key
  - volatile-random：针对设置了过期时间的key，随机移除key
  - allkeys-random：针对所有key，随机移除
  - volatile-ttl：移除即将过期的key
  - noevicetion：永不自动移除，内存满时再进行写操作，则返回错误信息

- maxmemory-samples 5     LRU和TTL算法都是估计值，根据样本来估计整个库的情况（有点像mysql的优化器选择算法），这里设置样本个数，默认为5，在速度和准确率上可以取得综合较好结果

> Redis 使用的是一种近似 LRU 算法，它跟 LRU 算法还不太一样。之所以不使用 LRU 算法，是因为需要消耗大量的额外的内存，需要对现有的数据结构进行较大的改造
>
> 近似 LRU 算法则很简单，在现有数据结构的基础上使用随机采样法来淘汰元素，能达到和 LRU 算法非常近似的效果。Redis 为实现近似 LRU 算法，它给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳
>
> 上一节提到处理 key 过期方式分为集中处理和懒惰处理，LRU 淘汰不一样，它的处理方式只有懒惰处理。只有当 Redis 执行写操作时，发现内存超出 maxmemory，才会执行一次LRU 淘汰算法。这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于maxmemory 为止
>
> 如何采样就是看 maxmemory-policy 的配置，如果是 allkeys 就是从所有的 key 字典中随机，如果是 volatile 就从带过期时间的 key 字典中随机。每次采样多少个 key 看的是maxmemory_samples 的配置，默认为 5

# 持久化

## RDB

RDB：Redis DataBase

指定时间间隔内将数据集快照（snapshot）写进磁盘，为dump.rdb文件，恢复时将快照文件读入内存

**细节**

Redis会创建（fork）一个子进程来进行持久化，子进程做数据备份操作，主进程继续对外提供服务，所有Redis服务不会阻塞；且不需要把整个内存的数据都复制一份。子进程生成rdb文件之后，会替换之前的rdb文件，子进程退出

> fork：复制一个与当前进程一样的进程，所有数据与原进程一致，但是是一个全新的进程，作为原进程的子进程

### Copy On Write 机制

核心思路：fork一个子进程，只有在父进程发生写操作修改内存数据时，才会真正去分配内存空间，并复制内存数据，而且也只是复制被修改的内存页中的数据，**并不是全部内存数据**

- Redis中执行BGSAVE命令生成RDB文件时，本质就是调用Linux中的fork()命令，Linux下的fork()系统调用实现了copy-on-write写时复制
- fork()是类Unix操作系统上创建线程的主要方法，fork用于创建子进程（等同于当前进程的副本）
- 传统的普通进程复制，会直接将父进程的数据拷贝到子进程中，拷贝完成后，父进程和子进程之间的数据段和堆栈是相互独立的；copy-on-write技术，在fork出子进程后，与父进程共享内存空间，两者只是虚拟空间不同，但是其对应的物理空间是**同一个**

- 整个过程中，主线程是不进行任何IO的，保证了较高的性能

节省了内存空间，提高了性能

### 特点

- 如果需要进行大规模的数据恢复，且对数据恢复的完整性不是非常敏感，则RDB比AOF高效
- 缺点是最后一次持久化之后存入redis的数据或对数据进行的修改可能丢失，且要考虑fork时数据克隆一倍的膨胀性存在

### 相关配置

SNAPSHOTTING

![image-20211018161113070](img\redis\25.png)

**在seconds秒内发生了changes次key的改变，则在seconds秒后存储RDB文件**

> flushall和shutdown也会触发写rdb文件
>
> 或者save指令主动备份
>
> 或者从节点上线时（需要根据rdb文件进行全量复制）

在不设置的情况下，默认配置为：

- 1h内发生至少1次key的改变，则1h后写rdb文件
- 5min内发生至少100次key的改变，则5min后写rdb文件
- 1min内发生至少10000次key的改变，则1min后写rdb文件

#### 基本属性

- stop-writes-on-bgsave-error yes     在最近一次rdb dump时出错了，则会阻止后续的写入操作，防止产生数据不一致的问题
- rdbcompression yes     是否在写dump.rdb时采用LZF压缩算法
- rdbchecksum yes     存储快照后，采用CRC64算法进行数据校验，会增加大约10%的性能消耗

### 创建快照

1. 根据配置文件，冷拷贝*（使用时记得把dump.rdb拷贝到备份机上）*
2. 命令save或者是bgsave
   1. save：只管save，其他全部阻塞
   2. bgsave：后台异步dump，同时可以响应客户端请求，可通过lastsave获取最后一次成功执行快照的时间
3. flushall

## AOF

AOF：Append-Only File

以日志的形式记录每个写操作，只许追加不许改写，恢复数据时将文件从前到后执行一遍

> 当AOF持久化功能打开时，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾

- 优势：灵活配置同步时间
- 缺陷
  - 文件大于rdb文件，恢复速度慢一些
  - 运行效率慢于rdb

![image-20211018173506993](img\redis\27.png)

开启aof后，启动redis会先去访问appendonly.aof文件，再访问dump.rdb

> 当aof文件出现错误导致redis启动不了时，可以使用redis-check-aof程序，`redis-check-aof --fix appendonly.aof`来修复

### 相关配置

APPEND ONLY MODE

![image-20211018172813441](img\redis\26.png)

#### 基本属性

- appendonly no     是否开启，默认为no

- appendfilename

- appendfsync everysec     aof_buf刷盘的模式，有三种：always、everysec、no，默认为everysec，每秒异步写一次。如果一秒内发生宕机，这一秒内的数据将丢失

  - always：每次aof_buf的变化都记录到磁盘，性能较差但数据完整性好
  - no：操作系统来决定什么时候刷盘，linux的操作系统的默认设置下为30s

- no-appendfsync-on-rewrite no     是否能在rewrite时候将数据同步到aof文件，默认为no，表示可以同步

  > 同时在执行bgrewriteaof操作和主进程写aof文件的操作，两者都会操作磁盘，而bgrewriteaof往往会涉及大量磁盘操作，这样就会造成主进程在写aof文件的时候出现阻塞的情形
  >
  > 于是no-appendfsync-on-rewrite参数出场了
  >
  > - 如果该参数设置为no，是最安全的方式，不会丢失数据，因为旧的AOF文件也是完整的，因此rewrite期间挂掉了也没关系，但是要忍受阻塞的问题
  > - 如果设置为yes呢？这就相当于将appendfsync设置为no，这说明并没有执行磁盘操作，只是写入了缓冲区，因此这样并不会造成阻塞（因为没有竞争磁盘），但是如果这个时候redis挂掉，就会丢失数据。丢失多少数据呢？在linux的操作系统的默认设置下，最多会丢失30s的数据

### AOF重写

[https://www.kancloud.cn/chunyu/php_basic_knowledge/1427988](https://www.kancloud.cn/chunyu/php_basic_knowledge/1427988)

[https://blog.csdn.net/qq_31387317/article/details/95315166](https://blog.csdn.net/qq_31387317/article/details/95315166)

**目的**：aof不断扩写，体积会越来越大，需要进行重写。当AOF文件的大小超过所设定的阈值，Redis就会启用AOF的文件压缩，只保留可以恢复数据的最小指令集，可以使用指令bgrewriteaof

**原理**：aof重写同样fork了一份子进程，只不过不会读取旧的aof文件，而是**根据数据库内存中的数据情况**用命令重写了一个aof文件*（比如每条记录使用一个set指令）*，然后替换掉原文件。ps：老版本还要命令合并等各种处理，效率低下

**触发机制**：文件大于上一次rewrite时大小的100%倍*（即为上次的两倍）*且大于64mb

![image-20211018205037836](img\redis\28.png)

在aof重写的过程中，如果有新来的数据或新来的修改，则会被主进程同时写入aof_rewrite_buf，在完成重写后，主进程将aof_rewrite_buf中的内容追加到重写后的aof临时文件中，最后替换旧的aof文件

![img](img\redis\73.png)

注意，在这个过程中有两个IO操作：① 主进程aof_buf通过fsync每秒一次（appendfsync everysec模式下）刷盘；② 子进程写AOF临时文件。磁盘资源紧张时，子进程可能阻塞主进程的写

> Redis经典场景问题：
>
> ![img](img\redis\29.png)
>
> 若no-appendfsync-on-rewrite的策略是 no，即rewrite的时候不允许将aof_buf的数据同步到磁盘的aof文件中，这就会导致在进行rewrite操作时，appendfsync可能会被阻塞。如果当前AOF文件很大，那么相应的rewrite时间会变长，appendfsync被阻塞的时间也会更长
>
> 这不是什么新问题，很多开启AOF的业务场景都会遇到这个问题。解决的办法有这么几个：
>
> - 将no-appendfsync-on-rewrite设置为yes，相当于把appendfsync设置为no，aof_buf中的数据在rewrite期间不会同步到磁盘。这样可以避免与appendfsync争用文件句柄，但是在rewrite期间的AOF有丢失的风险，比如rewrite完成之前redis挂了，aof_buf中的数据就全部丢失，最多可能长达30s
> - 给当前Redis实例添加slave节点，当前节点设置为master, 然后master节点关闭AOF，slave节点开启AOF。这样的方式的风险是如果master挂掉，尚没有同步到slave的数据会丢失

### 持久化实战经验

AOF一般情况下数据完整性好于rdb

RDB适用于备份，因为AOF不断变化不好实现主从备份

- 因为RDB文件只用作后备用途，建议只在Slave上持久化RDB文件，而且15分钟备份一次就够了，只保留save 900 1这条规则
- 如果Enable AOF，好处是在最恶劣情况下也只会丢失不超过两秒数据，启动脚本较简单只load自己的AOF文件就可以了。代价一是带来了持续的IO，二是AOF rewrite的最后将rewrite过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的。只要硬盘许可，应该尽量减少AOF rewrite的频率，AOF重写的基础大小默认值64M太小了，可以设到5G以上。默认超过原大小100%大小时重写可以改到适当的数值
- 如果不Enable AOF，仅靠Master-Slave Replication实现高可用性也可以。能省掉一大笔IO也减少了rewrite时带来的系统波动。代价是如果Master/Slave同时倒掉，会丢失十几分钟的数据，启动脚本也要比较两个Master/Slave中的RDB文件，载入较新的那个。新浪微博就选用了这种架构

## RDB-AOF混合持久化（redis 4.0+提供）

细细想来aofrewrite时也是先写一份全量数据到新AOF文件中再追加增量只不过全量数据是以redis命令的格式写入。那么是否可以先以RDB格式写入全量数据再追加增量日志呢这样既可以提高aofrewrite和恢复速度也可以减少文件大小还可以保证数据的完毕性整合RDB和AOF的优点

4.0实现了这一特性—RDB-AOF混合持久化

综上所述RDB-AOF混合持久化体现在aofrewrite时，即在AOF重写时把fork的那个镜像写成RDB，后续AOF重写缓冲里的数据继续追加到该文件中

 配置为 aof-use-rdb-preamble no  #默认关闭，yes 打开

![image-20211111162136672](img\redis\97.gif)

# 事务

## 事务的本质

事务一次执行多个命令，本质是一组命令的集合，一个事务中的所有命令都会序列化按顺序串行执行，而不会被其他命令插入

**作用**

一个队列中，一次性、顺序性、排他性地执行一系列指令

> mysql的事务靠redo log、undo log来保证，redis没有这些日志，事务实现原理较为简单

## 使用

### 基本命令

- `multi`：标志事务开启，相当于begin
- `exec`：执行事务块内的命令，相当于commit
- `discard`：取消事务内所有指令，相当于rollback
- `watch key`：监视若干个key，如果事务执行前被监视的key被其他命令改动了，则事务将被打断
- `unwatch`：取消对所有key的监视

### 正常执行

![image-20211019173006129](img\redis\30.png)

### 放弃事务

![image-20211019173118746](img\redis\31.png)

### 全体连坐

![image-20211019173313638](img\redis\32.png)

### 冤头债主

![image-20211019173816372](img\redis\33.png)

> 编译时出错“连坐”，运行时出错找“债主”

### Watch监视

![image-20211019210056342](img\redis\34.png)![image-20211019210235690](img\redis\35.png)

在watch key1之后，有其他命令修改了key1的值，后续所有事务失效

**exec或unwatch之后，监视解除**

## Redis的事务

- 单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送的命令请求所打断
- 没有隔离级别的概念：队列中的命令没有提交之前都不会实际的被执行，因为事务提交前任何指令都不会被实际执行，也就不存在”事务内的查询要看到事务里的更新，在事务外查询不能看到”这个让人万分头痛的问题
- 不保证原子性：redis同一个事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚

# 发布订阅

## 传统pub/sub

![image-20211019213255631](img\redis\36.png)

首先subsribe订阅channel1，打开另一个客户端发送`publish channel1 helloWorld`，则会出现如图的效果

> 订阅多个：`psubscribe channel*`，接着任一个以“channel”开头的频道发送`publish channelxxx msg`，都能被收到

## Stream

streamk可看做一个数据结构，模仿kafka实现，用一个长列表存储消息

弥补了pub/sub无法将消息持久化的缺陷

仍然是内存层面的，消费者消费过慢导致消息堆积影响性能，此时会淘汰老的消息

> xadd中有个MAXLEN参数，消息数大于这个值，老的消息就会出队

![image-20211102205329898](img\redis\72.png)

- 每个消息可发布给多个消费群体

- 每个消息有一个唯一标识ID

  > 自动生成的格式为：时间戳-序号，是递增的
  >
  > 支持自定义，但必须为：数字-数字

  > 如果发生时间回拨现象，序号岂不是会产生错误？
  >
  > 实际上，redis内部维护了一个last_generated_id属性，用来记录最后一个消息的id，如果发生回拨现象，则新消息时间戳固定为last_generated_id的时间戳，序号递增，保证了id的递增关系

- 消费群体不消费消息，而是群体中的每个消费者消费

- 消费群体之间存在竞争关系，一个消息只能被消费一次

- 拿到消息的消费者会向消息中间件发送确认应答，该消息就不用再被保留了

  > pending_ids[]：消费到消息但还没有给中间件发送应答，称为PEL，Pending Entry List
  >
  > 一直不应答，这个列表也会越来越大，占用内存，所以要及时应答

- 也支持单消费者

  > 单消费者的目的：保证消费消息的顺序性，消息群组是无法实现顺序性的
  
- 不存在死信队列（无法进入队列或怎么样也无法对消息进行消费的消息集合），可利用xpending查看delivery count，即消息在各个消费者中转发的次数，超过阈值可以xdel→xack

- 不实现消息分区

### 使用

[https://www.jianshu.com/p/8424cb186ef8](https://www.jianshu.com/p/8424cb186ef8)

```sh
# 向stream中添加一条消息，*表示让redis自动生成id
Aliyun2G:0>xadd stream_test * name zhangsan age 18
"1635860848531-0"
Aliyun2G:0>xadd stream_test * name lisi age 20
"1635861096418-0"

# 显示个数
Aliyun2G:0>xlen stream_test
"2"

# 展示消息，-、+分别代表负、正无穷
Aliyun2G:0>xrange stream_test - +
1) 1) "1635860848531-0"
   2) 1) "name"
      2) "zhangsan"
      3) "age"
      4) "18"


2) 1) "1635861096418-0"
   2) 1) "name"
      2) "lisi"
      3) "age"
      4) "20"

#删除，注意：只是从队列中删除，id仍存在于消费者的pengding_ids[]中，需要ack一下
Aliyun2G:0>xdel stream_test "1635861096418-0"
"1"
Aliyun2G:0>xlen stream_test
"1"

#-------------------按单消费者----------------------
#读取1个序号大于0-0的消息
Aliyun2G:0>xread count 1 streams stream_test 0-0
1) 1) "stream_test"
   2) 1) 1) "1635860848531-0"
         2) 1) "name"
            2) "zhangsan"
            3) "age"
            4) "18"

#不管前面的消息，只读取最新一条，$是last entry的意思
#不加block会读到nil，加了之后会阻塞直到另一个客户端向stream中新增一条消息
Aliyun2G:0>xread block 0 count 1 streams stream_test $
1) 1) "stream_test"
   2) 1) 1) "1635861505450-0"
         2) 1) "newkey"
            2) "newmsg"

#-----------------------按消息群组--------------------------
#设置从头读
Aliyun2G:0>xgroup create stream_test group1 0-0
"OK"
#设置不管前面的，从最新一条读
Aliyun2G:0>xgroup create stream_test group2 $
"OK"

#查看stream的信息
#last_delivered_id：用来表示消费者组消费在 Stream 上消费位置的游标信息
#每个消费者组都有一个 Stream 内唯一的名称，消费者组不会自动创建，需要使用 XGROUP CREATE指令来显式创建
# 并且需要指定从哪一个消息 ID 开始消费，用来初始化 last_delivered_id 这个变量

Aliyun2G:0>xinfo stream stream_test
1)  "length"
2)  "2"
3)  "radix-tree-keys"
4)  "1"
5)  "radix-tree-nodes"
6)  "2"
7)  "last-generated-id"
8)  "1635861505450-0"
9)  "groups"
10) "2"
11) "first-entry"
12) 1) "1635860848531-0"
    2) 1) "name"
       2) "zhangsan"
       3) "age"
       4) "18"


13) "last-entry"
14) 1) "1635861505450-0"
    2) 1) "newkey"
       2) "newmsg"

#只查看group的信息
Aliyun2G:0>xinfo groups stream_test
1) 1) "name"
   2) "group1"
   3) "consumers"
   4) "0"
   5) "pending"
   6) "0"
   7) "last-delivered-id"
   8) "0-0"

2) 1) "name"
   2) "group2"
   3) "consumers"
   4) "0"
   5) "pending"
   6) "0"
   7) "last-delivered-id"
   8) "1635861505450-0"
   
#为group1设置一个consumer1并读取一条
# >表示从当前消费组的last_delivered_id后面开始读，并不断推进last_delivered_id   
Aliyun2G:0>xreadgroup group group1 consumer1 count 1 streams stream_test >
1) 1) "stream_test"
   2) 1) 1) "1635860848531-0"
         2) 1) "name"
            2) "zhangsan"
            3) "age"
            4) "18"

#阻塞式读取
Aliyun2G:0>xreadgroup group group1 consumer1 block 0 count 1 streams stream_test >
#陷入阻塞状态，直到有新消息

#-----pending为3，表示消费者没有发送确认应答，当前消息群组的PEL之和为3
Aliyun2G:0>xinfo groups stream_test
1) 1) "name"
   2) "group1"
   3) "consumers"
   4) "1"
   5) "pending"
   6) "3"
   7) "last-delivered-id"
   8) "1635862679780-0"

2) 1) "name"
   2) "group2"
   3) "consumers"
   4) "0"
   5) "pending"
   6) "0"
   7) "last-delivered-id"
   8) "1635861505450-0"

Aliyun2G:0>xinfo consumers stream_test group1
1) 1) "name"
   2) "consumer1"
   3) "pending"
   4) "3"
   5) "idle"
   6) "151419"

#向消息队列发送某ID的消息已被消费的确认应答，消费者的PEL-=1
Aliyun2G:0>xack stream_test group1 "1635860848531-0"
"1"
Aliyun2G:0>xinfo consumers stream_test group1
1) 1) "name"
   2) "consumer1"
   3) "pending"
   4) "2"
   5) "idle"
   6) "221874"
   
#查看PEL
Aliyun2G:0>xpending stream_test group1
1) "2"
2) "1635861505450-0"
3) "1635862679780-0"
4) 1) 1) "consumer1"
      2) "2"
```

# 主从复制

## 基本概念

主机数据更新后根据配置和策略，自动同步到备机的master/slave机制，master以写为主，slave以读为主

**读写分离、容灾恢复**

## 使用

### 配从不配主

更改从机配置，而不更改主机配置

### 从库配置

`slaveof 主库IP 主库端口`

每次与master断开之后，都需要重新连接，除非写在redis.conf里面

> 密码控制也会影响到从库复制，从库必须在配置文件里使用 `masterauth `指令配置相应的密码才可以进行复制操作

## 模拟主从复制

### 使用多个端口开启多个redis服务

先准备多个redis.conf文件，并用这些文件启动redis服务，分别使用6380、6381、6382端口，模拟多台主机

配置如下（以6380为例）

- ![image-20211020164344681](img\redis\37.png)
- ![image-20211020165010644](img\redis\41.png)
- ![image-20211020164514190](img\redis\38.png)
- ![image-20211020164536275](img\redis\39.png)
- ![image-20211020164614128](img\redis\40.png)

![image-20211020165338815](img\redis\42.png)

配置完成，启动三个服务，采用`ps -ef|grep redis`确认进程启动

![image-20211020165531905](img\redis\43.png)

开启多个客户端

![image-20211020170927650](img\redis\44.png)

### 测试是否正常运行

首先使用`info replication`查看：

![image-20211020171411476](img\redis\45.png)

默认都作为master机器

### 一主二仆

![image-20211020173653967](img\redis\46.png)

**slaveof从机会备份主机的全库**

- 读写分离

![image-20211020174127911](img\redis\47.png)

**主机写，从机只能读**

- master挂掉

![image-20211020174935892](img\redis\48.png)

从机保持slave身份，**原地待命**

- slave挂掉

会断开连接，恢复master身份，除非在slave的redis.conf里面设置好对应的master

### 薪火相传

一主多仆模式有一个明显的缺点，**中心化太严重**

可以让上一个slave作为下一个slave的master，slave同样可以接受其他slave的连接和同步请求，那么该slave作为链条中的下一个master，可以有效减轻master的写压力

![image-20211021150129036](img\redis\49.png)

### 反客为主

`slaveof no one`：使当前数据库停止与其他数据库的同步，转成master数据库

![image-20211021152120634](img\redis\50.png)

## 复制原理

Slave启动成功连接到master后会发送一个sync命令，Master接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集的命令，在后台进程执行完毕之后，master将传送整个数据文件到slave，以完成一次完全同步

- 全量复制：而slave服务在接收到数据库文件数据后，将其存盘并加载到内存中

  > **快照同步**
  >
  > 快照同步是一个非常耗费资源的操作，它首先需要在主库上进行一次 bgsave 将当前内存的数据全部快照到磁盘文件中，然后再将快照文件的内容全部传送到从节点。从节点将快照文件接受完毕后，立即执行一次全量加载，加载之前先要将当前内存的数据清空。加载完毕后通知主节点继续进行增量同步
  >
  > ![image-20211112153221873](img\redis\100.png)
  >
  > 在整个快照同步进行的过程中，主节点的增量复制 buffer 还在不停的往前移动，如果快照同步的时间过长或者复制 buffer 太小，都会导致同步期间的增量指令在复制 buffer 中被覆盖，这样就会导致快照同步完成后无法进行增量复制，然后会再次发起快照同步，如此极有可能会陷入快照同步的死循环。
  >
  > 所以务必配置一个合适的复制 buffer 大小参数，避免快照复制的死循环

- 增量复制：Master继续将新的所有收集到的修改命令依次传给slave，完成同步

  > Redis 同步的是指令流，主节点会将那些对自己的状态产生修改性影响的指令记录在本地的内存 buffer 中，然后异步将 buffer 中的指令同步到从节点，从节点一边执行同步的指令流来达到和主节点一样的状态，一遍向主节点反馈自己同步到哪里了 (偏移量)
  >
  > 因为内存的 buffer 是有限的，所以 Redis 主库不能将所有的指令都记录在内存 buffer 中。Redis 的复制内存 buffer 是一个定长的环形数组，如果数组内容满了，就会从头开始覆盖前面的内容
  >
  > ![image-20211112152827898](img\redis\99.png)
  >
  > 如果因为网络状况不好，从节点在短时间内无法和主节点进行同步，那么当网络状况恢复时，Redis 的主节点中那些没有同步的指令在 buffer 中有可能已经被后续的指令覆盖掉了，从节点将无法直接通过指令流来进行同步，这个时候就需要用到更加复杂的同步机制 —— 快照同步

只要是重新连接master，一次完全同步（全量复制）将被自动执行

### 无盘复制

主节点在进行快照同步时，会进行很重的文件 IO 操作，特别是对于非 SSD 磁盘存储时，快照会对系统的负载产生较大影响。特别是当系统正在进行 AOF 的 fsync 操作时如果发生快照，fsync 将会被推迟执行，这就会严重影响主节点的服务效率

所以从 Redis 2.8.18 版开始支持无盘复制。所谓无盘复制是指主服务器直接通过套接字将快照内容发送到从节点，生成快照是一个遍历的过程，主节点会一边遍历内存，一遍将序列化的内容发送到从节点，从节点还是跟之前一样，先将接收到的内容存储到磁盘文件中，再进行一次性加载

### wait指令

**Wait** **指令**

Redis 的复制是异步进行的，wait 指令可以让异步复制变身同步复制，确保系统的强一致性 (不严格)。wait 指令是 Redis3.0 版本以后才出现的

```sh
\> set key value
OK
\> wait 1 0
(integer) 1
```

wait 提供两个参数，第一个参数是从库的数量 N，第二个参数是时间 t，以毫秒为单位。它表示等待 wait 指令之前的所有写操作同步到 N 个从库 (也就是确保 N 个从库的同步没有滞后)，最多等待时间 t。如果时间 t=0，表示无限等待直到 N 个从库同步完成达成一致

假设此时出现了网络分区，wait 指令第二个参数时间 t=0，主从同步无法继续进行，wait 指令会永远阻塞，Redis 服务器将丧失可用性

## 哨兵模式

> “反客为主”的自动版

master主机挂掉之后，剩下的机器以投票的方式选出新的master主机

![image-20211112154023874](img\redis\101.png)

Redis 主从采用异步复制，意味着当主节点挂掉时，从节点可能没有收到全部的同步消息，这部分未同步的消息就丢失了。如果主从延迟特别大，那么丢失的数据就可能会特别多。Sentinel 无法保证消息完全不丢失，但是也尽可能保证消息少丢失。它有两个选项可以限制主从延迟过大

```sh
min-slaves-to-write 1
min-slaves-max-lag 10
```

第一个参数表示主节点必须至少有一个从节点在进行正常复制，否则就停止对外写服务，丧失可用性

何为正常复制，何为异常复制？这个就是由第二个参数控制的，它的单位是秒，表示如果 10s 没有收到从节点的反馈，就意味着从节点同步不正常，要么网络断开了，要么一直没有给反馈

### 使用

> 初始化以6380为master，6381、6382为从机

1. 在根目录建立一个sentinel.conf文件*（根目录没有该文件的情况下，一般都会有的）*

2. 修改sentinel.conf文件

   1. 监控某个master主机

      `sentinel monitor <name> <host> <port> 1`

      ```xml
      sentinel monitor myMonitor 127.0.0.1 6380 1
      ```

      其中，1表示令从机进行投票，从机得到的票数大于1，就变为主机；各得到1票就失败，重新投票

3. 启动哨兵，`redis-sentinel sentinel.conf`

   ![image-20211022213837662](img\redis\51.png)

4. 令主机*（本例中是6380的redis）*shutdown之后，等待一段时间，某个从机被选为新主机

   ![image-20211022215541224](img\redis\52.png)

   经过一系列过程后，6382被选为新主机

   ![image-20211022215645727](img\redis\53.png)

   此时，原主机“复活”，变成了6382的从机

   ![image-20211022220707223](img\redis\54.png)

   > 必须得等到哨兵监控到6380之后

   ![image-20211022220850141](img\redis\55.png)

   可以看到，sentinel文件也发生了改变

   ![image-20211022221059950](img\redis\56.png)

> 一组sentinel可以监控多个master

## 复制延时

由于所有的写操作都是先在master上操作，然后同步更新到slave上，所以从master同步到slave机器有一定的延迟， 当系统很繁
忙的时候，延迟问题会更加严重，slave机器数量的增加也会使这个问题更加严重

## 主从和集群的区别

[https://www.cnblogs.com/chenwenyin/p/13549492.html](https://www.cnblogs.com/chenwenyin/p/13549492.html)

**主从复制是为了数据备份**，**哨兵是为了高可用**，Redis主服务器挂了哨兵可以切换，**集群则是因为单实例能力有限，搞多个分散压力**简短总结如下：**sentinel着眼于高可用，Cluster提高并发量**

# Redis集群

![image-20211022221059950](img\redis\87.png)

在哨兵模式中，仍然只有一个Master节点。当并发写请求较大时，哨兵模式并不能缓解写压力

我们知道只有主节点才具有写能力，那如果在一个集群中，能够配置多个主节点，是不是就可以缓解写压力了呢？

答：是的。这个就是redis-cluster集群模式

## Redis-cluster集群概念

- 由多个Redis服务器组成的分布式网络服务集群
- 集群之中有多个Master主节点，每一个主节点都可读可写；节点的fail是通过集群中超过半数的节点检测失效时才生效
- 节点之间会互相通信，两两相连；所有的redis节点彼此互联（PING-PONG机制），内部使用二进制协议优化传输速度和带宽
- Redis集群无中心节点（去中心化）；客户端与redis节点直连，不需要中间proxy层。客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可

## 集群节点复制

在Redis-Cluster集群中，可以给每一个主节点添加从节点，主节点和从节点之间遵循主从模式的特性

当用户需要处理更多读请求的时候，添加从节点可以扩展系统的读性能

## 故障转移

Redis集群的主节点内置了类似Sentinel的节点故障检测和自动故障转移功能，当集群中的某个主节点下线时，集群中的其他在线主节点会注意到这一点，并对已下线的主节点进行故障转移


集群进行故障转移的方法和Sentinel进行故障转移的方法基本一样，不同的是，在集群里面，故障转移是由集群中其他在线的主节点负责进行的，所以集群不必另外使用Sentinel

## 集群分片策略

[https://zhuanlan.zhihu.com/p/337878020](https://zhuanlan.zhihu.com/p/337878020)

### Redis cluster节点分配

Redis-cluster分片策略，是用来解决key存储位置的

- 集群将db0分为16384个槽位slot，所有key-value数据都存储在这些slot中的某一个上，一个slot可以存放多个数据
- key的槽位计算公式为：slot_number=crc16(key)%16384，其中crc16为16位的循环冗余校验和函数
- 集群中的每个主节点都可以处理0个至16383个槽，当16384个槽都有某个节点在负责处理时，集群进入 上线状态，并开始处理客户端发送的数据命令请求
- 在集群的情况下不支持使用select命令来切换db，因为Redis集群模式下只有一个db0

---

比如：

现在我们是三个主节点分别是：A, B, C 三个节点，它们可以是一台机器上的三个端口，也可以是三台不同的服务器。那么，采用哈希槽 （hash slot）的方式来分配16384个slot的话，它们三个节点分别承担的slot区间是：

- 节点A覆盖0－5460；
- 节点B覆盖5461－10922；
- 节点C覆盖10923－16383

**获取数据：**

如果存入一个值，按照redis cluster哈希槽的算法：CRC16('key') % 16384 = 6782。那么就会把这个key 的存储分配到 B 上了。同样，当我连接(A,B,C)任何一个节点想获取'key'这个key时，也会这样的算法，然后内部跳转到B节点上获取数据

**新增一个主节点：**

新增一个节点D，redis cluster的这种做法是从各个节点的前面各拿取一部分slot到D上，我会在接下来的实践中实验。大致就会变成这样：

- 节点A覆盖1365-5460
- 节点B覆盖6827-10922
- 节点C覆盖12288-16383
- 节点D覆盖0-1364,5461-6826,10923-12287

同样删除一个节点也是类似，移动完成后就可以删除这个节点了

> 感觉有点像一致性哈希，每个redis服务节点对应的虚拟节点是固定的一个范围内，落到每个节点对应的slot范围内的key，就会被存储在该redis服务节点上，实现内存压力的分散

**数据迁移**

- 注意这里的迁移过程是同步的，在目标节点执行 restore 指令到原节点删除 key 之间，原节点的主线程会处于阻塞状态，直到 key 被成功删除

  如果迁移过程中突然出现网络故障，整个 slot 的迁移只进行了一半。这时两个节点依旧处于中间过渡状态。待下次迁移工具重新连上时，会提示用户继续进行迁移

---

在迁移过程中，客户端访问的流程会有很大的变化

首先新旧两个节点对应的槽位都存在部分 key 数据。客户端先尝试访问旧节点，如果对应的数据还在旧节点里面，那么旧节点正常处理。如果对应的数据不在旧节点里面，那么有两种可能，要么该数据在新节点里，要么根本就不存在。旧节点不知道是哪种情况，所以它会：

1. 向客户端返回一个`-ASK targetNodeAddr` 的重定向指令
2. 客户端收到这个重定向指令后，先去目标节点执行一个不带任何参数的 asking 指令
3. 然后在目标节点再重新执行原先的操作指令

>  为什么需要执行一个不带参数的 asking 指令呢？

因为在迁移没有完成之前，按理说这个槽位还是不归新节点管理的，如果这个时候向目标节点发送该槽位的指令，节点是不认的，它会向客户端返回一个`-MOVED `重定向指令告诉它去源节点去执行。如此就会形成 **重定向循环**，`asking` 指令的目标就是打开目标节点的选项，告诉它下一条指令不能不理，而要**必须**当成自己的槽位来处理

### Redis Cluster主从模式

redis cluster 为了保证数据的高可用性，加入了主从模式，一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份，当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉

上面那个例子里, 集群有ABC三个主节点, 如果这3个节点都没有加入从节点，如果B挂掉了，我们就无法访问整个集群了。A和C的slot也无法访问

所以我们在集群建立的时候，**一定要为每个主节点都添加了从节点**，比如像这样，集群包含主节点A、B、C，以及从节点A1、B1、C1, 那么即使B挂掉系统也可以继续正确工作

B1节点替代了B节点，所以Redis集群将会选择B1节点作为新的主节点，集群将会继续正确地提供服务。 当B重新开启后，它就会变成B1的从节点

不过需要注意，如果节点B和B1同时挂了，Redis集群就无法继续正确地提供服务了

> Redis Cluster 可以为每个主节点设置若干个从节点，单主节点故障时，集群会自动将其中某个从节点提升为主节点。如果某个主节点没有从节点，那么当它发生故障时，集群将完全处于不可用状态。不过 Redis 也提供了一个参数 `cluster-require-full-coverage` 可以允许部分节点故障，其它节点可以继续提供对外访问

### 集群redirect转向

由于Redis集群无中心节点，请求会随机发给任意主节点。主节点只会处理自己负责槽位的命令请求，其它槽位的命令请求，该主节点会返回客户端一个转向错误。客户端根据错误中包含的地址和端口重新向正确的负责的主节点发起命令请求

> Cluster 有两个特殊的 error 指令，一个是 moved，一个是 asking
>
> - 第一个 moved 是用来纠正槽位的。如果我们将指令发送到了错误的节点，该节点发现对应的指令槽位不归自己管理，就会将目标节点的地址随同 moved 指令回复给客户端通知客户端去目标节点去访问。这个时候客户端就会刷新自己的槽位关系表，然后重试指令，后续所有打在该槽位的指令都会转到目标节点
> - 第二个 asking 指令和 moved 不一样，它是用来临时纠正槽位的。如果当前槽位正处于迁移中，指令会先被发送到槽位所在的旧节点，如果旧节点存在数据，那就直接返回结果了，如果不存在，那么它可能真的不存在也可能在迁移目标节点上。所以旧节点会通知客户端去新节点尝试一下拿数据，看看新节点有没有。这时候就会给客户端返回一个 asking error携带上目标节点的地址。客户端收到这个 asking error 后，就会去目标节点去尝试。客户端不会刷新槽位映射关系表，因为它只是临时纠正该指令的槽位信息，不影响后续指令

## Redis6集群搭建

[https://blog.csdn.net/wsdc0521/article/details/106765909](https://blog.csdn.net/wsdc0521/article/details/106765909)

> 集群代理：Redis6版本中新增的特性，客户端不需要知道集群中的具体节点个数和主从身份，可以直接通过代理访问集群
>
> ![img](img\redis\88.png)
>
> [https://www.cnblogs.com/wy123/p/12829673.html](https://www.cnblogs.com/wy123/p/12829673.html)

### 准备工作

1. 安装ruby环境

redis集群管理工具redis-trib.rb依赖ruby环境，首先需要安装ruby环境：

```sh
yum -y install ruby
yum -y install rubygems
```

2. 准备节点

创建目录redis-cluster并在此目录下再创建6381 6382 6383 6391 6392 6393共6个目录，在各个目录中创建配置文件redis.conf，内容如下：

```bash
daemonize yes #后台启动
port 6381 #修改端口号，从6381到6393
cluster-enabled yes #开启cluster，去掉注释
cluster-config-file nodes-6381.conf #自动生成
cluster-node-timeout 15000 #节点通信时间
appendonly yes #持久化方式
```

同时把redis.conf复制到其它目录中

> cluster-config-file：每个节点在运行过程中，会维护一份集群配置文件
> 当集群信息发生变化时（如增减节点），集群内所有节点会将最新信息更新到该配置文件
> 节点重启后，会重新读取该配置文件，获取集群信息，可以方便的重新加入到集群中
> 也就是说，当 Redis 节点以集群模式启动时，会首先寻找是否有集群配置文件
> 如果有则使用文件中的配置启动；如果没有，则初始化配置并将配置保存到文件中
>
> 集群配置文件由 Redis 节点维护，不需要人工修改

启动所有节点

![image-20211107220024292](img\redis\89.png)

### 启动集群

```sh
redis-cli --cluster create 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6391 127.0.0.1:6392 127.0.0.1:6393 --cluster-replicas 1
```

![image-20211107220320995](img\redis\90.png)

> cluster-replicas：主/从比例
>
> *redis-cli --cluster*代替了之前的*redis-trib.rb*，我们无需安装ruby环境即可直接使用它附带的所有功能：创建集群、增删节点、槽迁移、完整性检查、数据重平衡等等

### 集群限制

由于Redis集群中数据分布在不同的节点上，因此有些功能会受限：

- db库：单机的Redis默认有16个db数据库，但在集群模式下只有一个db0；
- 复制结构：上面的复制结构有树状结构，但在集群模式下只允许单层复制结构；
- 事务/lua脚本：仅允许操作的key在同一个节点上才可以在集群下使用事务或lua脚本；(使用Hash Tag可以解决)
- key的批量操作：如mget、mset操作，只有当操作的key都在同一个节点上才可以执行；(使用Hash Tag可以解决)
- keys/flushall：只会在该节点之上进行操作，不会对集群的其他节点进行操作

#### hash tag

当key包含{}的时候，不会对整个key做hash，只会对{}包含的部分做hash然后分配槽slot；因此我们可以让不同的key在同一个槽内，这样就可以解决key的批量操作和事务及lua脚本的限制了；

但由于hash tag会将不同的key分配在相同的slot中，如果使用不当，会造成数据分布不均的情况，需要注意

![image-20211107221527585](img\redis\91.png)

redirect到对应节点

![image-20211107221805829](img\redis\92.png)

对myTag进行hash，可以看到，都在同一个slot

### 访问集群

上面介绍了槽的概念，在每个节点存储着不同范围的槽，数据也分布在不同的节点之上，我们在访问集群的时候，如何知道数据在哪个节点或者在哪个槽之上呢？ 下面介绍两种访问连接：

- Dummy客户端

使用redis-cli客户端连接集群被称为dummy客户端，只会在执行命令之后通过MOVED错误重定向找到对应的节点，如图，我们可以使用`redis-cli -c`命令进入集群命令行，当查看或设置key的时候会根据上面提到的CRC16算法计算key的hash值找到对应的槽slot，然后重定向到对应的节点之后才能操作，我们也使用cluster keyslot命令查看key所在的槽slot：

```sh
#使用-c进入集群命令行模式
redis-cli -c -p 6381

#使用命令查看key所在的槽
cluster keyslot key1
```

- Smart客户端

相比于dummy客户端，smart客户端在初始化连接集群时就缓存了槽slot和节点node的对应关系， 也就是在连接任意节点后执行cluster slots，我们使用的JedisCluster就是smart客户端：

```sh
cluster slots
```

### 集群参数优化

- cluster_node_timeout：默认值为15s

  影响ping消息接收节点的选择，值越大对延迟容忍度越高，选择的接收节点就越少，可以降低带宽，但会影响收敛速度。应该根据带宽情况和实际要求具体调整

  影响故障转移的判定，值越大越不容易误判，但完成转移所消耗的时间就越长。应根据网络情况和实际要求具体调整

- cluster-require-full-coverage

  为了保证集群的完整性，只有当16384个槽slot全部分配完毕，集群才可以上线，但同时，若主节点发生故障且故障转移还未完成时，原主节点的槽不在任何节点中，集群会处于下线状态，影响客户端的使用

  该参数可以改变此设定：

  - no:  表示当槽没有完全分配时，集群仍然可以上线；
  - yes: 默认配置，只有槽完全分配，集群才可以上线

> docker搭建，后面再看：[https://www.cnblogs.com/niceyoo/p/13011626.html](https://www.cnblogs.com/niceyoo/p/13011626.html)

# Jedis&Lettuce&Redission

## Jedis

- 基本命令

```java
@Test
void testJedis() {
    Jedis jedis = new Jedis("localhost", 6379);
    jedis.auth("123456");
    // 测试连通
    System.out.println(jedis.ping());
    // 命令
    System.out.println(jedis.keys("*"));
    jedis.set("key1", "value1");
    // 事务
    Transaction transaction = jedis.multi();
    transaction.set("123", "hello");
    transaction.zadd("zset", 100, "haha");
    transaction.zadd("zset", 200, "xixi");
    Response<Long> n = transaction.zcard("zset");
    Response<Set<String>> zset = transaction.zrangeByScore("zset", 100, 200);
    transaction.exec();
    // 注意这些get方法一定要在transaction之后调用，事务是识别不了这些命令的
    System.out.println(n.get());
    System.out.println(zset.get());
}
```

- 主从复制

```java
@Test
void testJedis() {
    Jedis master = new Jedis("localhost", 6380);
    master.auth("123456");
    Jedis slave = new Jedis("localhost", 6381);
    slave.auth("123456");
    slave.slaveof("localhost", 6380);

    master.set("hello", "world");
    System.out.println(slave.get("hello"));
}
```

- JedisPool

```java
static class JedisPoolUtils{
    private static volatile JedisPool jedisPool = null;
    public static JedisPool newInstance(){
        if (jedisPool == null){
            synchronized (JedisPool.class){
                if (jedisPool == null){
                    JedisPoolConfig config = new JedisPoolConfig();
                    // 最大连接数
                    config.setMaxTotal(100);
                    // 最大空闲连接数
                    config.setMaxIdle(30);
                    // 连接前是否先ping测试一下
                    // 注意：，如果设置为true就需要将redis和你的程序放到同一台机器上或者同一局域网上面或者关闭该模式
                    config.setTestOnBorrow(true);
                    jedisPool = new JedisPool(config, "localhost", 6379);
                }
            }
        }
        return jedisPool;
    }
}

@Test
void testJedis() {
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    Jedis aNewConnection = jedisPool.getResource();
    aNewConnection.get("kkk");
    aNewConnection.close();
    jedisPool.close();
}
```

## Lettuce



## 优缺点

> [https://www.zhihu.com/question/53124685](https://www.zhihu.com/question/53124685)

- Jedis

  **优点：**

  - 提供了比较全面的 Redis 操作特性的 API
  - API 基本与 Redis 的指令一一对应，使用简单易理解

  **缺点：**

  - 同步阻塞 IO
  - 不支持异步
  - 线程不安全

- Lettuce

  **优点：**

  - 线程安全
  - 基于 Netty 框架的事件驱动的通信，可异步调用
  - 适用于分布式缓存

  **缺点：**

  - API 更抽象，学习使用成本高

> springboot2.0之后都使用Lettuce了，封装成redisTemplate使用

# Redis6新增特性

## Redis6 ACL

[https://www.icode9.com/content-2-689684.html](https://www.icode9.com/content-2-689684.html)

redis6 ACL命令

`redis-server redis.conf`启动redis服务；`redis-cli`启动redis客户端；`auth default password` ：登录（登录后才能使用各种命令，如果未设置密码就不需要登录）

- `acl list`：查看所有acl规则

![image-20211012101323915](img\redis\8.png)

- `acl setuser user1`：创建用户

![image-20211012101430711](img\redis\9.png)

> off：用户未激活；-@all：不具有任何权限

- `acl setuser user1 on`：激活用户

- `acl setuser user1 on >pass1 >pass2 >pass3 +@all ~*`

> 设置三个密码，每个都可以使用；+@all：使用所有权限；~*：访问所有的key

- `acl whoami/users`：查看当前用户/所有用户

......

# Redis线程模型

## 概述

网络通讯端多线程，内存端单线程

无论有多少个客户端操作，在内存上的操作都是单线程的，也就是每次只有一个客户端进行操作

> 因为每次只有一个客户端，setnx这样的功能才易于实现，也是分布式锁的基础

## 为什么redis性能如此之高？

我们都知道，redis内部处理是单线程的，为什么单线程的性能也可以如此之高？

原因是因为：① 基于内存操作；② 基于epoll实现多路复用；③ 高效的数据存储结构

### 基于epoll的多路复用



# Redis高级数据结构

## Bitmaps

数组的每个元素用一个bit来记录，比int型数组节省32倍的空间，适用于海量用户存储

### 基本命令

- `setbit [key] [value]/getbit [key]`：设置/获取对应位的值

  > 例：用户id为0、1、5、8、18的几个用户在10月1日访问了我们的网站
  >
  > ```sh
  > Aliyun2G:0>setbit visit:1001:user 0 1
  > "0"
  > Aliyun2G:0>setbit visit:1001:user 1 1
  > "0"
  > Aliyun2G:0>setbit visit:1001:user 5 1
  > "0"
  > Aliyun2G:0>setbit visit:1001:user 8 1
  > "0"
  > Aliyun2G:0>setbit visit:1001:user 18 1
  > "0"
  > ```
  >
  > 查看10月1日用户5是否访问了网站
  >
  > ```sh
  > Aliyun2G:0>getbit visit:1001:user 5
  > "1"
  > ```

- `bitcount [key] [start] [end]` ：查看[start, end] bytes中有多少条记录

  > ```sh
  > Aliyun2G:0>bitcount visit:1001:user
  > "5"
  > Aliyun2G:0>bitcount visit:1001:user 0 1
  > "4"
  > Aliyun2G:0>bitcount visit:1001:user 1 1
  > "1"
  > Aliyun2G:0>bitcount visit:1001:user 0 0
  > "3"
  > ```

- `bitop and/not/or... [key1] [key2]`：位图运算

- `bitfield`：多个bit操作

### 使用场景

1. 美团原题：10亿数量的自然数实现排序，限制32位机器，2G内存
   1. 可以采用10亿长度的位图，在自然数对应的索引上置为1，再从前往后遍历整个位图，得到排序，只不过**这个方法会去重**
   2. 不能去重的情况下，可使用范围缩小+堆法：
      - 2G内存，假设有这样一种记录，为：记录{自然数, 出现频次}，假设这样一个记录大小为4字节（自然数）+4字节（频次数据）+8字节（额外数据结构）=16字节，2G内存=2^30字节。也就是说，2G内存，最多可放2 ^ 26次方个这样的记录
      - 自然数范围为0~10亿，约0~2^30，也就是说，可以分成16个范围：0~2^26-1，2^26~2^27-1，2^27~3*2^26-1，...，15\*2^26~2^30-1
      - 遍历16次这些自然数，先判断并处理0~2^26-1范围内的，不断压入小根堆并统计词频，遍历结束后弹出到结果集文件中；再重新遍历这些自然数，处理2^26~2^27-1范围的，以此类推

2. 布隆过滤器

   首先需要一个长度为m的bit数组，和k个不同的哈希函数，接着，以黑名单场景为例

   > 某个包含100亿url的网址黑名单，如何判断用户访问的网站在不在这个黑名单里面？

   - 每添加一个url，使用k个不同的哈希函数计算出对应的$hash_{i(i=0,1,...,k)}$值，再通过$hash_i\%m$散列到bit数组的k个位置，将这k个位置从0改为1
   - 每次判断的时候，通过同一组hash函数，计算k个位置上是不是都是1，如果是的话，则判断为黑名单
   - 因此，有可能出现白名单被判断成黑名单的情况，但不会出现黑名单被判断成白名单，因为只要是黑名单，相应位置一定会是1，不可能是0

Redis使用布隆过滤器：

1. 自己实现操作redis的bitmaps；或google的guava，并存放到redis中

   > 分布式环境下，必须保证不同的机器都可以访问到同一布隆过滤器

2. redission

3. RedisBloom插件+JreBloom包

## HyperLogLog

实际上并不是一种数据结构，底层基于字符串来实现的；和位数组有关

用极小的内存空间实现统计，如：大型网站每个网页每天用户的UV数据统计

> 可以用两个数值标准来统计访问某网站的访客，即“访问次数—PV“ 和 “独立访客（问）数—UV”
>
> 访问次数是总共访问的次数；而独立访问数，对于每个用户，无论出入多少次，都只算作一次，需要去掉重复访问的数据

简单方案：用户ip存入redis的set，scard。但面对大量数据就无能为力了

![image-20211101171644542](img\redis\59.png)

HyperLogLog提供了**不完全精确的去重计数方案**，标准误差为0.81%，能够满足UV统计的需求

### 基本命令

- `pfadd`：添加一次访问记录

  ```sh
  Aliyun2G:0>pfadd UV:20211101 "id001" "id002" "id066" "id001"
  "1"
  ```

- `pfcount`：统计UV数据

  ```sh
  Aliyun2G:0>pfcount UV:20211101
  "3"
  ```

- `pfmerge [key1] [key2]`：将key2的数据融合到key1

  ```sh
  Aliyun2G:0>pfadd UV:20211101 "id001" "id002" "id066" "id001"
  "1"
  Aliyun2G:0>pfcount UV:20211101
  "3"
  Aliyun2G:0>pfadd UV:20211102 "id0019" "id002" "id06" "id001"
  "1"
  Aliyun2G:0>pfcount UV:20211102
  "4"
  Aliyun2G:0>pfmerge UV:20211101 UV:20211102
  "OK"
  Aliyun2G:0>pfcount UV:20211101
  "5"
  ```

### 原理

[原理解析](https://www.cnblogs.com/linguanh/p/10460421.html)

> 可以实现对2 ^ 64个数据的统计，占用大小仅仅2 ^ 14 * 6 / 8 /1024 ≈ 12kb

将用户id hash成64位，由低位到高位取前14位（Redis中是2^14 = 16384个桶）确定用户在哪一个桶，将若干个用户分桶后，每一桶当作一轮n次的（每个用户id相当于一次试验）伯努利试验，由低到高从剩余的50位中找第一个出现1的位置，当作k_max

由于每个桶有6位，id剩下的50位能够得到得最大的k_max，也就是1在最高位，也就是index=50，为110010，即k_max=110010，一个桶肯定能容下

计算出每个桶得k_max，然后取16384个桶的k_max计算调和平均数，再根据 `n = 2 ^ k_max的调和平均数`（极大似然估计），估算出总试验次数（总用户id数）

> 计算过程中还加入了修正因子，这里不再阐述

### 测试

```java
@Test
void test(){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try(Jedis conn = jedisPool.getResource()){
        conn.auth("xsfhHF981123");
        for (int i = 0; i < 1000; i++) {
            conn.pfadd("PAGE1:UV:DAY:20210101", "id" + i);
        }
        System.out.println("真实用户量为: 1000");
        System.out.println("估计用户量为: " + conn.pfcount("PAGE1:UV:DAY:20210101"));
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

1000个数据的统计，大小仅仅1.83k

![image-20211101215705157](img\redis\60.png)

统计结果也是比较准确的

![image-20211101215748120](img\redis\61.png)

## GEO

附近的人、附近的车等等功能；采用GeoHash算法，将二维经纬度映射到一维，这样便更易计算距离，用52位整数来存放编码

底层用zset实现

### GeoHash

GeoHash 算法将二维的经纬度数据映射到一维的整数，这样所有的元素都将在挂载到一条线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。当我们想要计算附近的人时，首先将目标位置映射到这条线上，然后在这个一维的线上获取附近的点就行了

那这个映射算法具体是怎样的呢？它将整个地球看成一个二维平面，然后划分成了一系列正方形的方格，就好比围棋棋盘。所有的地图元素坐标都将放置于唯一的方格中。方格越小，坐标越精确。然后对这些方格进行整数编码，越是靠近的方格编码越是接近。那如何编码呢？一个最简单的方案就是切蛋糕法。设想一个正方形的蛋糕摆在你面前，二刀下去均分分成四块小正方形，这四个小正方形可以分别标记为 00,01,10,11 四个二进制整数。然后对每一个小正方形继续用二刀法切割一下，这时每个小小正方形就可以使用 4bit 的二进制整数予以表示。然后继续切下去，正方形就会越来越小，二进制整数也会越来越长，精确度就会越来越高

上面的例子中使用的是二刀法，真实算法中还会有很多其它刀法，最终编码出来的整数数字也都不一样

编码之后，每个地图元素的坐标都将变成一个整数，通过这个整数可以还原出元素的坐标，整数越长，还原出来的坐标值的损失程度就越小。对于「附近的人」这个功能而言，损失的一点精确度可以忽略不计

GeoHash 算法会继续对这个整数做一次 base32 编码 (0-9,a-z 去掉 a,i,l,o 四个字母) 变成一个字符串。在 Redis 里面，经纬度使用 52 位的整数进行编码，放进了 zset 里面，zset 的 value 是元素的 key，score 是 GeoHash 的 52 位整数值。zset 的 score 虽然是浮点数，但是对于 52 位的整数值，它可以无损存储

在使用 Redis 进行 Geo 查询时，我们要时刻想到它的内部结构实际上只是一个zset(skiplist)。通过 zset 的 score 排序就可以得到坐标附近的其它元素 (实际情况要复杂一些，不过这样理解足够了)，通过将 score 还原成坐标值就可以得到元素的原始坐标

### 基本命令

- `geoadd [key] [Lon] [lat] [member] `：给member设置一个坐标

  ```sh
  Aliyun2G:0>geoadd peoplenearby 100.3 40.1 zhangsan
  "1"
  ```

- `geopos [key] [member] `：获取memberd的坐标

  ```sh
  Aliyun2G:0>geopos peoplenearby zhangsan
  1) 1) "100.30000180006027222"
     2) "40.09999973002739893"
  ```

- `geodist [key] [member1] [member2] [unit]`：两个位置之间的距离

  ```sh
  Aliyun2G:0>geoadd peoplenearby 90.5 45.6 lisi
  "1"
  Aliyun2G:0>geopos peoplenearby lisi
  1) 1) "90.49999862909317017"
     2) "45.59999992371525934"
  Aliyun2G:0>geodist peoplenearby zhangsan lisi
  "1005169.5399"
  Aliyun2G:0>geodist peoplenearby zhangsan lisi m
  "1005169.5399"
  Aliyun2G:0>geodist peoplenearby zhangsan lisi km
  "1005.1695"
  Aliyun2G:0>geodist peoplenearby zhangsan lisi ft
  "3297800.3278"
  Aliyun2G:0>geodist peoplenearby zhangsan lisi mi
  "624.5849"
  ```

- `georadius [key] [Lon] [Lat] [num] [unit]`：某坐标周围半径为num的所有member

  ```sh
  Aliyun2G:0>georadius peoplenearby 100.4 45 1500 km
  1) "lisi"
  2) "zhangsan"
  ```

# Redis慢查询和PipeLine

## 慢查询

与mysql一样，当出现超过最大执行时间阈值的查询时，将会记录下该查询的 `时间 耗时 命令`

![image-20211102151835108](img\redis\62.png)

redis命令生命周期 发送、排队、执行、返回。慢查询只对第三步执行进行统计

### 设置

- `slowlog-log-slower-than`：大于某个时间的查询被记录下来，默认10000，单位为微妙，即千分之一毫秒
- `slow-max-len`：redis的慢查询使用队列记录的，这个参数设置队列最大长度，如果满了之后再有新的记录进入，队列的第一条记录就会出队，默认128

1. 动态设置并持久化到redis.conf

   ```sh
   Aliyun2G:0>config get slowlog-log-slower-than
   1) "slowlog-log-slower-than"
   2) "10000"
   Aliyun2G:0>config set slowlog-log-slower-than 1000
   "OK"
   Aliyun2G:0>config rewrite
   # 只有在启动时指定了redis.conf，rewrite才能生效，否则会报Permission denied
   ```

2. 直接在redis.conf里面修改

### 使用

- `slowlog get n`：查看n个慢查询记录

以第一个为例

![image-20211102154344043](img\redis\63.png)

- `slowlog len`：获取慢查询列表当前的长度
- `slowlog reset` ：对慢查询列表清理（重置）

对于线上slow-max-len配置的建议：

- 线上可加大slow- max-len的值， 记录慢查询存长命令时Redis会做截断，不会占用大量内存，线上可设置1000以上
- 对于线上slowlog-log-slower-than配置的建议：默认为10毫秒，根据Redis并发量来调整，对于高并发建议为1毫秒
- 慢查询是先进先出的队列，早的记录可能出列丢失，需定期执行slowlog get将结果存储到其它设备中（如mysq）

## PipeLine

RTT：数据在网络上的往返传输时间

客户端通过命令操作redis，对于mset、mget这类命令，可以一次性操作多个键，降低了网络通讯次数，减少了RTT。**然而**，大多数命令是没有批量操作的，只能一次次经过发送→排队→执行→返回的过程

![image-20211102155423708](img\redis\64.png)

对于这一类命令，就可以使用PipeLine进行批量操作，一次性的网络传输

![image-20211102155518916](img\redis\65.png)

### 性能测试

```java
@Test
void test(){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try(Jedis conn = jedisPool.getResource()){
        conn.auth("xsfhHF981123");
        long start = System.currentTimeMillis();
        for (int i = 0; i < 500; i++) {
            conn.zadd("test", i, "member:" + i);
        }
        long end = System.currentTimeMillis();
        System.out.println("若干次单次执行耗时: " + (end - start) + "ms");
        /* ----------------- */
        start = System.currentTimeMillis();
        Pipeline pipelined = conn.pipelined();
        for (int i = 0; i < 500; i++) {
            pipelined.zadd("test2", i, "member:" + i);
        }
        pipelined.sync();
        end = System.currentTimeMillis();
        System.out.println("pipeline执行耗时: " + (end - start) + "ms");
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

![image-20211102160922749](img\redis\66.png)

### 和事务的区别

pipeline是客户端的行为，将命令组装成pipeline，对redis是透明的，也就是说，redis并不知道这是pipeline还是一条一条执行的

事务是服务端行为，multi后服务器会将命令执行所在客户端设为一个特殊的状态，后续的命令不会真正执行，而是被服务器缓存了，直到exec

此外，**pipeline无法保证原子性**，如果组装的命令太多，内存超过了TCP传输的一个MTU，势必会被拆包，分为若干组命令执行，而在这若干组命令之间可能会插入其他的命令，导致对用户来说，发生了数据不一致的问题

# LUA脚本

c语言编写

## 好处

- 减少网络开销，多个命令放在一个Lua脚本下
- 原子性，redis会将脚本作为一个整体来执行，中间不会被其他命令插入，无需担心出现竞态条件
- 复用性，脚本当然是可复用的

## 使用

### 本地安装

```sh
wget http://www.lua.org/ftp/lua-5.3.6.tar.gz
tar zxvf lua-5.3.6.tar.gz 
cd lua-5.3.6
# centos下载libreadline相关支持
dnf install readline-devel
make linux
make install
```

输入lua，正确运行，则安装成功

### 基本语法

- 关键字

![image-20211102172044397](img\redis\67.png)

变量无需声明，默认为全局变量，加local才是局部变量

- 数据类型

![image-20211102172209868](img\redis\68.png)

```lua
-- table作数组
> tb1 = {"a","b"}
> print(tb1[0])
nil
> print(tb1[1])
a

-- table作hash
> tb={name="zhangsan", age=18}
> print(tb["name"])
zhangsan

--字符串拼接
> print("str".."ing")
string

--算术操作
> print("str".."ing")
string

```

- 函数

```lua
> function add(num1, num2)
>> a=1 --这里是全局变量
>> return num1+a+num2
>> end

> print(add(1,1))
3
> print(a)
1
```

- 迭代

```lua
> for i=10,1,-1 do -- 10~1，每次-1
>> print(i)
>> end
10
9
8
7
6
5
4
3
2
1
```

```lua
> t={"a","b","c"}
> for i,v in ipairs(t) do -- ipairs: 迭代器
print(i,v)
end
1	a
2	b
3	c
```

java代码嵌入lua脚本：引入luaj-jse包

## Redis+Lua

`eval(sha)`命令、`script命令`

![image-20211102200455404](img\redis\69.png)

### 测试java连通

写一个简单的Lua脚本

![image-20211102202512143](img\redis\70.png)

复制到字符串中并测试运行

```java
@Test
void test(){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try(Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        Pipeline pipelined = conn.pipelined();
        String script = "if redis.call('exists', KEYS[1]) == 1 then\n" +
                "return redis.call('set', KEYS[1], ARGV[1])\n" +
                "else\n" +
                "return redis.call('set', KEYS[1], ARGV[2])\n" +
                "end";
        pipelined.eval(script, 1, "test", "test_exists", "test_not_exists");
        pipelined.get("test");
        pipelined.syncAndReturnAll().forEach(System.out::println);
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

成功运行：

![image-20211102202631393](img\redis\71.png)

# Redis过期策略

Redis的键过期就会删除，于是，考虑这样的问题：

1. 会不会因为同一时间太多的 key 过期，以至于忙不过来
2. Redis 是单线程的，收割的时间也会占用线程的处理时间，如果删除的太过于繁忙，会不会导致线上读写指令出现卡顿

Redis是这样解决的：

## 过期的key集合

redis 会将每个**设置了过期时间的 key** 放入到一个独立的字典中，以后会**定时遍历**这个字典来删除到期的 key。除了定时遍历之外，它还会使用惰性策略来删除过期的 key，所谓惰性策略就是在客户端访问这个 key 的时候，redis 对 key 的过期时间进行检查，如果过期了就立即删除。定时删除是集中处理，惰性删除是零散处理

## 定时扫描策略

Redis 默认会每秒进行十次过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略

1. 从过期字典中随机 20 个 key；
2. 删除这 20 个 key 中已经过期的 key；
3. 如果过期的 key 比率超过 1/4，那就重复步骤 1

同时，为了保证过期扫描不会出现循环过度，导致线程卡死现象，算法还增加了扫描时间的上限，默认不会超过 25ms

## 过期时间随机化

设想一个大型的 Redis 实例中所有的 key 在同一时间过期了，会出现怎样的结果？

毫无疑问，Redis 会持续扫描过期字典 (循环多次)，直到过期字典中过期的 key 变得稀疏，才会停止 (循环次数明显下降)。这就会导致线上读写请求出现明显的卡顿现象。导致这种卡顿的另外一种原因是内存管理器需要频繁回收内存页，这也会产生一定的 CPU 消耗

也许你会争辩说“扫描不是有 25ms 的时间上限了么，怎么会导致卡顿呢”？这里打个比方，假如有 101 个客户端同时将请求发过来了，然后前 100 个请求的执行时间都是25ms，那么第 101 个指令需要等待多久才能执行？2500ms，这个就是客户端的卡顿时间，是由服务器不间断的小卡顿积少成多导致的

所以业务开发人员一定要注意过期时间，如果有大批量的 key 过期，要给过期时间设置一个随机范围，而不能全部在同一时间过期

```sh
# 在目标过期时间上增加一天的随机时间
redis.expire_at(key, random.randint(86400) + expire_ts)
```

> 在一些活动系统中，因为活动是一期一会，下一期活动举办时，前面几期的很多数据都可以丢弃了，所以需要给相关的活动数据设置一个过期时间，以减少不必要的 Redis 内存占用。如果不加注意，你可能会将过期时间设置为活动结束时间再增加一个常量的冗余时间，如果参与活动的人数太多，就会导致大量的 key 同时过期。通过将过期时间随机化总是能很好地解决了这个问题

## 从库的过期策略

从库不会进行过期扫描，从库对过期的处理是被动的。主库在 key 到期时，会在 AOF 文件里增加一条 del 指令，同步到所有的从库，从库通过执行这条 del 指令来删除过期的key

因为指令同步是异步进行的，所以主库过期的 key 的 del 指令没有及时同步到从库的话，会出现主从数据的不一致，主库没有的数据在从库里还存在，比如上一节的集群环境分布式锁的算法漏洞就是因为这个同步延迟产生的

# Redis数据结构细节

## Redis紧凑型数组结构

Redis 是一个非常耗费内存的数据库，它所有的数据都放在内存里。如果我们不注意节约使用内存，Redis 就会因为我们的无节制使用出现内存不足而崩溃。Redis 作者为了优化数据结构的内存占用，也苦心孤诣增加了非常多的优化点，这些优化也是以牺牲代码的可读性为代价的，但是毫无疑问这是非常值得的，尤其像 Redis 这种数据库

**32bit vs 64bit**

Redis 如果使用 32bit 进行编译，内部所有数据结构所使用的指针空间占用会少一半，如果你对 Redis 使用内存不超过 4G，可以考虑使用 32bit 进行编译，可以节约大量内存。4G 的容量作为一些小型站点的缓存数据库是绰绰有余了，如果不足还可以通过增加实例的方式来解决。

### 小对象压缩存储

如果 Redis 内部管理的集合数据结构很小，它会使用紧凑存储形式压缩存储；只有当数据量大于某值后，为了提升查找效率，会转换成标准结构

#### ziplist

Redis 的 ziplist 是一个紧凑的字节数组结构

![image-20211112145818127](img\redis\98.gif)

- 如果它存储的是 hash 结构，那么 key 和 value 会作为两个 entry 相邻存在一起
- 如果它存储的是 zset，那么 value 和 score 会作为两个 entry 相邻存在一起

```sh
Aliyun2G:0>hset hkey f1 v1
"1"
Aliyun2G:0>object encoding hkey
"ziplist"

Aliyun2G:0>zadd zkey 99 xiaoming
"1"
Aliyun2G:0>object encoding zkey
"ziplist"
```

#### intset

Redis 的 intset 是一个紧凑的整数数组结构，它用于存放元素都是整数的并且元素个数较少的 set 集合

如果整数可以用 uint16 表示，那么 intset 的元素就是 16 位的数组，如果新加入的整数超过了 uint16 的表示范围，那么就使用 uint32 表示，如果新加入的元素超过了 uint32 的表示范围，那么就使用 uint64 表示，Redis 支持 set 集合动态从 uint16 升级到 uint32，再升级到 uint64

![image-20211112150139424](img\redis\96.png)

如果 set 里存储的是字符串，那么 sadd 立即升级为 hashtable 结构

```java
Aliyun2G:0>sadd skey 1
"1"
Aliyun2G:0>object encoding skey
"intset"
Aliyun2G:0>sadd skey s
"1"
Aliyun2G:0>object encoding skey
"hashtable"
```

### 存储界限

当集合对象的元素不断增加，或者某个 value 值过大，这种小对象存储也会被升级为标准结构。Redis 规定在小对象存储结构的限制条件如下：

- `hash-max-zipmap-entries 512` # hash 的元素个数超过 512 就必须用标准结构存储
- `hash-max-zipmap-value 64` # hash 的任意元素的 key/value 的长度超过 64 就必须用标准结构存储
- `list-max-ziplist-entries 512` # list 的元素个数超过 512 就必须用标准结构存储
- `list-max-ziplist-value 64` # list 的任意元素的长度超过 64 就必须用标准结构存储
- `zset-max-ziplist-entries 128` # zset 的元素个数超过 128 就必须用标准结构存储
- `zset-max-ziplist-value 64` # zset 的任意元素的长度超过 64 就必须用标准结构存储
- `set-max-intset-entries 512` # set 的整数元素个数超过 512 就必须用标准结构存储

### 内存回收

**内存回收机制**

Redis 并不总是可以将空闲内存立即归还给操作系统

如果当前 Redis 内存有 10G，当你删除了 1GB 的 key 后，再去观察内存，你会发现内存变化不会太大。原因是操作系统回收内存是以页为单位，如果这个页上只要有一个 key 还在使用，那么它就不能被回收。Redis 虽然删除了 1GB 的 key，但是这些 key 分散到了很多页面中，每个页面都还有其它 key 存在，这就导致内存不会立即被回收

不过，如果你执行 flushdb，然后再观察内存会发现内存确实被回收了。原因是所有的key 都干掉了，大部分之前使用的页面都完全干净了，会立即被操作系统回收

Redis 虽然无法保证立即回收已经删除的 key 的内存，但是它会重用那些尚未回收的空闲内存。这就好比电影院里虽然人走了，但是座位还在，下一波观众来了，直接坐就行。而操作系统回收内存就好比把座位都给搬走了

## String

Redis 的字符串叫着「SDS」，也就是 Simple Dynamic String。它的结构是一个带长度信息的字节数组，支持append操作

```c
struct SDS<T> {
    T capacity; // 数组容量
    T len; // 数组长度
    byte flags; // 特殊标识位，不理睬它
    byte[] content; // 数组内容
}
```

上面的 SDS 结构使用了范型 T，为什么不直接用 int 呢，这是因为当字符串比较短时，len 和 capacity 可以使用 byte 和 short 来表示，Redis 为了对内存做极致的优化，不同长度的字符串使用不同的结构体来表示

Redis 规定字符串的长度不得超过 512M 字节。创建字符串时 len 和 capacity 一样长，不会多分配冗余空间

> 因为绝大多数场景下我们不会使用 append 操作来修改字符串

### embstr vs raw

Redis 的字符串有两种存储方式，在长度特别短时，使用 emb 形式存储 (embeded)，当长度超过 44 时，使用 raw 形式存储

一个字符的差别，存储形式就发生了变化。这是为什么呢？为了解释这种现象，我们首先来了解一下 Redis 对象头结构体

#### Redis对象头结构体

```c
struct RedisObject {
    int4 type; // 4bits
    int4 encoding; // 4bits
    int24 lru; // 24bits
    int32 refcount; // 4bytes
    void *ptr; // 8bytes，64-bit system
} robj;
```

- 不同的对象具有不同的类型 type(4bit)
- 同一个类型的 type 会有不同的存储形式encoding(4bit)
- 为了记录对象的 LRU 信息，使用了 24 个 bit 来记录 LRU 信息
- 每个对象都有个引用计数，当引用计数为零时，对象就会被销毁，内存被回收
- ptr 指针将指向对象内容 (body) 的具体存储位置。这样一个 RedisObject 对象头需要占据 16 字节的存储空间

#### 为什么阈值为44字节

接着我们再看 SDS 结构体的大小，在字符串比较小时，SDS 对象头的大小是capacity+3，至少是 3。意味着分配一个字符串的最小空间占用为 19 字节 (16+3)

```c
struct SDS {
    int8 capacity; // 1byte
    int8 len; // 1byte
    int8 flags; 
    // 1byte
    byte[] content; // 内联数组，长度为 capacity
}
```

如图所示，embstr 存储形式是这样一种存储形式，它将 RedisObject 对象头和 SDS 对象连续存在一起，使用 malloc 方法一次分配

![image-20211116221440140](img\redis\103.png)

而 raw 存储形式不一样，它需要两次malloc，两个对象头在内存地址上一般是不连续的

![image-20211116221626233](img\redis\104.png)

而内存分配器 jemalloc/tcmalloc 等分配内存大小的单位都是 2、4、8、16、32、64 等等，为了能容纳一个完整的 embstr 对象，jemalloc 最少会分配 32 （>19）字节的空间，如果字符串再稍微长一点，那就是 64 字节的空间。如果总体超出了 64 字节，Redis 认为它是一个大字符串，不再使用 emdstr 形式存储，而该用 raw 形式

前面我们提到 SDS 结构体中的 content 中的字符串是以字节\0 结尾的字符串，之所以多出这样一个字节，是为了便于直接使用 glibc 的字符串处理函数，以及为了便于字符串的调试打印输出。**在字符串最长长度为一个字节的情况下，一个完整的字符串（RedisObj：16 + SDS：3+1）为20字节，所以长度为44时就达到了64字节**，超出的话就是大字符串，两次malloc，encoding为raw形式

### 扩容策略

字符串在长度小于 1M 之前，扩容空间采用加倍策略，也就是保留 100% 的冗余空间。当长度超过 1M 之后，为了避免加倍后的冗余空间过大而导致浪费，每次扩容只会多分配 1M 大小的冗余空间

## 字典dict

### 概述

dict 是 Redis 服务器中出现最为频繁的复合型数据结构，除了 hash 结构的数据会用到字典外，整个 Redis 数据库的所有 key 和 value 也组成了一个全局字典，还有带过期时间的 key 集合也是一个字典

```c
struct RedisDb {
    dict* dict; // all keys key=>value
    dict* expires; // all expired keys key=>long(timestamp)
}
```

zset 集合中存储 value 和 score 值的映射关系也是通过 dict 结构实现的

```c
struct zset {
    dict *dict; // all values value=>score
    zskiplist *zsl;
}
```

### 内部结构

dict 结构内部包含两个 hashtable，通常情况下只有一个 hashtable 是有值的。但是在dict 扩容缩容时，需要分配新的 hashtable，然后进行渐进式搬迁，这时候两个 hashtable 存储的分别是旧的 hashtable 和新的 hashtable。待搬迁结束后，旧的 hashtable 被删除，新的hashtable 取而代之

![image-20211116223114479](img\redis\105.png)

hashtable的结构，和java的hashmap几乎一致（个人认为是linkedhashmap）

![image-20211116223234243](img\redis\106.png)

```c
// dict结构
struct dict {
    // ...
    dictht ht[2];
}
// hashtable结构
struct dictht {
    dictEntry** table; // 二维
    long size; // 第一维数组的长度
    long used; // hash 表中的元素个数
    // ...
}
// Entry结构
struct dictEntry {
    void* key;
    void* val;
    dictEntry* next; // 链接下一个 entry
}
```

```c
// 查找过程
func get(key) {
    // hash_func 作用等同于 java中的hash()
    let index = hash_func(key) % size;
    let entry = table[index];
    while(entry != NULL) {
        if entry.key == target {
            return entry.value;
        }
        entry = entry.next;
    } 
}
```

> **hash攻击**
>
> 如果 hash 函数存在偏向性，黑客就可能利用这种偏向性对服务器进行攻击。存在偏向性的 hash 函数在特定模式下的输入会导致 hash 第二维链表长度极为不均匀，甚至所有的元素都集中到个别链表中，直接导致查找效率急剧下降，从 O(1)退化到 O(n)。有限的服务器计算能力将会被 hashtable 的查找效率彻底拖垮。这就是所谓 hash 攻击

### 扩容

当 hash 表中元素的个数等于第一维数组（ht[0）的长度时，就会开始扩容，扩容的新数组是原数组大小的 2 倍

不过如果 Redis 正在做 bgsave，为了减少内存页的过多分离 (Copy On Write)，Redis 尽量不去扩容 (dict_can_resize)，但是如果 hash 表已经非常满了，元素的个数已经达到了第一维数组长度的 5 倍 (dict_force_resize_ratio)，说明 hash 表已经过于拥挤了，这个时候就会强制扩容

#### 过程

**具体过程是：**

1. `ht[0]`为旧数组，开辟一个新内存空间，大小为2倍旧数组；令`ht[1]`指向该空间
2. 将`rehashindex`的值设置为`0`，表示rehash工作正式开始（非一次性）
3. 在rehash期间，**每次**对字典执行增删改查操作时，程序除了执行指定的操作以外，还会顺带将`ht[0]`哈希表在`rehashindex`索引上的所有键值对rehash到`ht[1]`，当rehash工作完成以后，`rehashindex`的值`+1`
4. 随着字典操作的不断执行，最终会在某一时间段上`ht[0]`的所有键值对都会被rehash到`ht[1]`，这时将`rehashindex`的值设置为`-1`，表示rehash操作结束；令`ht[0]`=`ht[1]`

#### 渐进式rehash

大字典的扩容是比较耗时间的，需要重新申请新的数组，然后将旧字典所有链表中的元素重新挂接到新的数组下面，这是一个 O(n)级别的操作，作为单线程的 Redis 表示很难承受这样耗时的过程。所以 Redis 使用渐进式rehash 小步搬迁。虽然慢一点，但是肯定可以搬完

它会同时保留旧数组和新数组，然后在定时任务中以及后续对 hash 的指令操作中（来自客户端的 hset/hdel 指令等）渐渐地将旧数组中挂接的元素迁移到新数组上。这意味着**要操作处于 rehash 中的字典，需要同时访问新旧两个数组结构**。如果在旧数组下面找不到元素，还需要去新数组下面去寻找

新增数据都会往新数组里放

> scan 也需要考虑这个问题，对与 rehash 中的字典，它需要同时扫描新旧槽位，然后将结果融合后返回给客户端
>
> scan 指令返回的游标就是第一维数组（dictEntry*，即Entry数组）的位置索引，我们将这个位置索引称为槽 (slot)。如果不考虑字典的扩容缩容，直接按数组下标挨个遍历就行了。limit 参数就表示需要遍历的槽位数，之所以返回的结果可能多可能少，是因为不是所有的槽位上都会挂接链表，有些槽位可能是空的，还有些槽位上挂接的链表上的元素可能会有多个。每一次遍历都会将 limit 数量的槽位上挂接的所有链表元素进行模式匹配过滤后，一次性返回给客户端

### 缩容

当 hash 表因为元素的逐渐删除变得越来越稀疏时，Redis 会对 hash 表进行缩容来减少hash 表的第一维数组空间占用。缩容的条件是元素个数低于数组长度的 10%。缩容不会考虑 Redis 是否正在做 bgsave（不需要申请额外内存，因此不会产生很多冗余内存，再多也不可能超过原数组内存大小）

## Set

Redis 里面 set 的结构底层实现也是字典，只不过所有的 value 都是 NULL，其它的特性和字典一模一样



# Redis应用

## Redis实现消息中间件

> 当没有足够的条件部署消息中间件时，可以通过redis实现消息中间件

先初始化一个jedisPool

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

```java
static class JedisPoolUtils{
    private static volatile JedisPool jedisPool = null;
    public static JedisPool newInstance(){
        if (jedisPool == null){
            synchronized (JedisPool.class){
                if (jedisPool == null){
                    JedisPoolConfig config = new JedisPoolConfig();
                    // 最大连接数
                    config.setMaxTotal(100);
                    // 最大空闲连接数
                    config.setMaxIdle(30);
                    jedisPool = new JedisPool(config, "localhost", 6379);
                }
            }
        }
        return jedisPool;
    }
}
```

### list

生产者向list发送信息，消费者通过brpop来阻塞式拿取消息

```java
@Test
void testPush(){
    pushMsg("redisMQ:list", "hello world");
}
@Test
void testGet(){
    System.out.println(getMessage("redisMQ:list"));
}

void pushMsg(String key, String message) {
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()) {
        conn.lpush(key, message);
    }catch (Exception e){
        e.printStackTrace();
    }
}
List<String> getMessage(String key){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        return conn.brpop(0, key);
    }catch (Exception e){
        e.printStackTrace();
    }
    return null;
}
```

首先运行get的test，发现在阻塞等待，待队列中有消息后，取得消息，退出阻塞

> 注意**空闲连接**的问题
>
> 如果线程一直阻塞在哪里，Redis 的客户端连接就成了闲置连接，闲置过久，服务器一般会主动断开连接，减少闲置资源占用。这个时候 blpop/brpop 会抛出异常来
>
> 所以编写客户端消费者的时候要小心，注意捕获异常，还要重试

### zset

可以用zset实现延时队列，用时间戳作为分数

```java
@Test
void testPush(){
    for (int i = 0; i < 10; i++) {
        pushMsg("redisMQ:zset", "hello world " + i, System.currentTimeMillis() + 2000 * i);
    }
}
@Test
void testGet(){
    getMessage("redisMQ:zset");
}

void pushMsg(String key, String message, double timeout) {
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()) {
        conn.zadd(key, timeout, message);
    }catch (Exception e){
        e.printStackTrace();
    }
}
void getMessage(String key){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        // 由于zset没有阻塞式的指令，只能轮询获取
        while (true){
            Set<String> set = conn.zrangeByScore(key, "-inf", "" + System.currentTimeMillis(), 0, 1);
            if (set.isEmpty()) continue;
            String next = set.iterator().next();
            System.out.println(next);
            conn.zrem(key, next);
        }
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

### pub/sub

```java
@Test
void pub(){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        conn.publish("channel1", "WTF!");
    }catch (Exception e){
        e.printStackTrace();
    }
}

@Test
void sub(){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("xsfhHF981123");
        conn.psubscribe(new MyJedisPubSub(), "channel*");
    }catch (Exception e){
        e.printStackTrace();
    }
}
static class MyJedisPubSub extends JedisPubSub{
    @Override
    public void onPMessage(String pattern, String channel, String message) {
        System.out.println(message);
    }
}
```

### stream

```java
/**
 * stream功能测试
 */
/**
 * 发布消息
 * @param key
 * @param message
 */
void publish(String key, HashMap<String, String> message){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        conn.xadd("streamTest", StreamEntryID.NEW_ENTRY, message);
    }catch (Exception e){
        e.printStackTrace();
    }
}

/**
 * 创建消息群组，游标设置在消息队列的0-0处，也就是可以读取消息队列中已存在的所有的消息
 * ($就是把游标设置在stream尾部，忽略前面所有消息，只从新来的开始消费)
 * @param key
 * @param groupName
 */
void createGroup(String key, String groupName){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        // 最后一个参数makeStream，如果设为false，则stream不存在会报错；设为true则会创建一个
        conn.xgroupCreate(key, groupName, new StreamEntryID(), true);
    }catch (Exception e){
        e.printStackTrace();
    }
}

/**
 * 消费消息
 * @param key
 * @param groupName
 * @param consumer
 */
List<Map.Entry<String, List<StreamEntry>>> consume(String key, String groupName, String consumer){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        HashMap<String, StreamEntryID> map = new HashMap<>();
        map.put(key, StreamEntryID.UNRECEIVED_ENTRY);
        XReadGroupParams params = new XReadGroupParams().count(1).block(0);
        return conn.xreadGroup(groupName, consumer, params, map);
    }catch (Exception e){
        e.printStackTrace();
    }
    return null;
}

/**
 * 发送ack应答
 * @param key
 * @param group
 * @param id
 */
void ack(String key, String group, StreamEntryID id){
    JedisPool jedisPool = JedisPoolUtils.newInstance();
    try (Jedis conn = jedisPool.getResource()){
        conn.auth("123456");
        conn.xack(key, group, id);
    }catch (Exception e){
        e.printStackTrace();
    }
}

static final String KEY = "streamTest";
@Test
void producer(){
    publish(KEY, new HashMap<>(){{
        put("msg1", "hello world");
    }});
}

@Test
void consumer(){
    final String groupName = "group1";
    createGroup(KEY, groupName);
    List<Map.Entry<String, List<StreamEntry>>> res = consume(KEY, groupName, "consumer1");
    for (Map.Entry<String, List<StreamEntry>> entry: res){
        List<StreamEntry> value = entry.getValue();
        for (StreamEntry streamEntry: value){
            System.out.println("消费消息: " + streamEntry.getFields());
            ack(KEY, groupName, streamEntry.getID());
        }
    }
}
```

## 超卖问题

### 问题场景

假设这样一个场景：redis的表中存放着商品数量，每进行一次购买，商品数量减一

```java
@RestController
@RequestMapping("/purchase")
public class PurchaseController {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/buyone")
    public String buy(){
        int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
        if(rest > 0){
            int realRest = rest - 1;
            stringRedisTemplate.opsForValue().set("product-001", realRest + "");
        }else throw new RuntimeException("库存不足");
        return "ok";
    }
}
```

对于该请求，我们都知道会发生并发安全问题：假设有多个请求在其他请求set之前进行了get操作，拿到了相同的数值200，接着减一，再set 199，这样，实际上有多个请求完成了购买操作，但redis中只扣除了一个商品

```java
@RequestMapping("/buyone")
public String buy(){
    synchronized (this) {
        int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
        if (rest > 0) {
            int realRest = rest - 1;
            stringRedisTemplate.opsForValue().set("product-001", realRest + "");
        } else throw new RuntimeException("库存不足");
        return "ok";
    }
}
```

通过上述修改，在单机模式中是可行的，然而在分布式系统中，多个请求可能被分发到不同的tomcat服务器上，再从同一个redis当中取，而synchronized只能保证一台机器上的购买行为正确，对于多台服务器，同样也会产生并发安全问题（可自己用Jmeter测试一下）

### 解决方案之分布式锁

#### 基础使用

**使用redis实现：`setnx`命令，不存在才set**

```java
@RequestMapping("/buyone")
public String buy(){
    String lockKey = "lock";
    boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1");
    if (!result){
        // return or block
    }
    int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
    if (rest > 0) {
        int realRest = rest - 1;
        stringRedisTemplate.opsForValue().set("product-001", realRest + "");
        stringRedisTemplate.delete(lockKey);
    } else {
        stringRedisTemplate.delete(lockKey);
        throw new RuntimeException("库存不足");
    }
    return "ok";
}
```

根据redis单线程处理的特性，只有头一个进入的请求能够set成功，其他的请求都无法成功，可以根据业务场景设置为返回或阻塞。拿到分布式锁的请求完成调用后，删除lockKey，相当于释放分布式锁，这样就实现了分布式系统并发安全性

#### 存在问题

1. 上述代码还存在一个严重的问题：业务代码执行过程中如果抛出异常，分布式锁就会被永远锁上，后续所有请求都会被返回或阻塞，改进方式是try-finally：

```java
@RequestMapping("/buyone")
public String buy(){
    String lockKey = "lock";
    boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1");
    if (!result){
        // return or block
    }
    try {
        int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
        if (rest > 0) {
            int realRest = rest - 1;
            stringRedisTemplate.opsForValue().set("product-001", realRest + "");
        } else {
            throw new RuntimeException("库存不足");
        }
    }finally {
        stringRedisTemplate.delete(lockKey);
    }
    return "ok";
}
```

2. 上述代码依旧存在问题，当业务代码执行到一半机器突然宕机，这个锁也会被永久加上，改进方式是加一个过期时间：

```java
@RequestMapping("/buyone")
public String buy(){
    String lockKey = "lock";
    boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1");
    stringRedisTemplate.expire(lockKey, 10, TimeUnit.SECONDS);
    if (!result){
        // return or block
    }
    try {
        int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
        if (rest > 0) {
            int realRest = rest - 1;
            stringRedisTemplate.opsForValue().set("product-001", realRest + "");
        } else {
            throw new RuntimeException("库存不足");
        }
    }finally {
        stringRedisTemplate.delete(lockKey);
    }
    return "ok";
}
```

3. 上述代码依旧存在问题，针对

   ```java
   boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1");
   stringRedisTemplate.expire(lockKey, 10, TimeUnit.SECONDS);
   ```

   这两条命令不是原子的，也就是说，在两命令之间也可能发生宕机，解决方式是Lua脚本或合并命令

   - lua脚本

   ```java
   static final String lua = "local res = redis.call('setnx', KEYS[1], ARGV[1])\n" +
           "redis.call('expire', KEYS[1], ARGV[2])\n" +
           "return res";
   
   @RequestMapping("/buyone")
   public String buy(){
       String lockKey = "lock";
       boolean result = stringRedisTemplate.execute(new DefaultRedisScript<>(lua, Boolean.class), Arrays.asList(lockKey), "1", "10");
       if (!result){
           // return or block
       }
       try {
           int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
           if (rest > 0) {
               int realRest = rest - 1;
               stringRedisTemplate.opsForValue().set("product-001", realRest + "");
           } else {
               throw new RuntimeException("库存不足");
           }
       }finally {
           stringRedisTemplate.delete(lockKey);
       }
       return "ok";
   }
   ```

   或

   ```lua
   -- test.lua
   local res = redis.call('setnx', KEYS[1], ARGV[1])
   redis.call('expire', KEYS[1], ARGV[2])
   return res
   ```

   ```java
   @RequestMapping("/buyone")
   public String buy(){
       DefaultRedisScript<Boolean> redisScript = new DefaultRedisScript<>();
       // 指定要使用的lua脚本
       redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/test.lua")));
       //指定返回类型
       redisScript.setResultType(Boolean.class);
       String lockKey = "lock";
       boolean result = stringRedisTemplate.execute(redisScript, Arrays.asList(lockKey), "1", "10");
       if (!result){
           // return or block
       }
       try {
           int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
           if (rest > 0) {
               int realRest = rest - 1;
               stringRedisTemplate.opsForValue().set("product-001", realRest + "");
           } else {
               throw new RuntimeException("库存不足");
           }
       }finally {
           stringRedisTemplate.delete(lockKey);
       }
       return "ok";
   }
   ```

   - 合并命令

   ```java
   @RequestMapping("/buyone")
   public String buy(){
       String lockKey = "lock";
       // 一条命令即可搞定
       boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
       if (!result){
           // return or block
       }
       try {
           int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
           if (rest > 0) {
               int realRest = rest - 1;
               stringRedisTemplate.opsForValue().set("product-001", realRest + "");
           } else {
               throw new RuntimeException("库存不足");
           }
       }finally {
           stringRedisTemplate.delete(lockKey);
       }
       return "ok";
   }
   ```

4. 上述代码依旧存在问题，上述的过期时间为10s，假设某次请求1卡顿了，执行了10s以上，在执行过程中键过期了，此时请求2便可以进入，请求2执行了一段时间之后，请求1结束执行，删除锁，但此时删除的是请求2的锁，因此其他请求又可以进来，如此循环，极端情况下永远是前一个请求删除了下一个请求的锁，可能会造成锁的永久失效。可以通过为锁加一个ID来改进，只能自己删除：

   ```java
   @RequestMapping("/buyone")
   public String buy(){
       String lockKey = "lock";
       String clientID = UUID.randomUUID().toString();
       boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, clientID, 10, TimeUnit.SECONDS);
       if (!result){
           // return or block
       }
       try {
           int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
           if (rest > 0) {
               int realRest = rest - 1;
               stringRedisTemplate.opsForValue().set("product-001", realRest + "");
           } else {
               throw new RuntimeException("库存不足");
           }
       }finally {
           if (clientID.equals(stringRedisTemplate.opsForValue().get(lockKey))){
               stringRedisTemplate.delete(lockKey);
           }
       }
       return "ok";
   }
   ```

5. 上述代码依旧存在问题，在finally块中，if逻辑和delete两条语句不是原子的，假设请求1对应的线程1完成对比，正准备删除之前，卡顿了一下，就在这个卡顿的时间点内请求2进入，修改了clientID，然后线程1恢复了，删除了请求2的锁，其他线程又可以进入，导致相同的问题，这里可以用lua脚本来改进

   ```java
   final static String CAS_LUA = "if redis.call('get', KEYS[1]) == ARGV[1] then\n" +
           "redis.call('del', KEYS[1])\n" +
           "end";
   
   @RequestMapping("/buyone")
   public String buy(){
       String lockKey = "lock";
       String clientID = UUID.randomUUID().toString();
       boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, clientID, 10, TimeUnit.SECONDS);
       if (!result){
           // return or block
       }
       try {
           int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
           if (rest > 0) {
               int realRest = rest - 1;
               stringRedisTemplate.opsForValue().set("product-001", realRest + "");
           } else {
               throw new RuntimeException("库存不足");
           }
       }finally {
           stringRedisTemplate.execute(new DefaultRedisScript<>(CAS_LUA), Arrays.asList(lockKey), clientID);
       }
       return "ok";
   }
   ```

6. 上述代码仍然存在问题，假如请求执行时间长，执行时间大于锁的过期时间，同样也容易产生多个请求造成并发问题。解决方式是开启一个后台线程，定时监控主线程是否还在运行，如果还在运行，则延长锁的过期时间；如果不再运行，则自动删除该后台线程，这个操作叫做锁续命。一般不自己实现，要考虑的因素太多，有现成的Redisson分布式锁

### Redisson分布式锁

解决上述一切问题的一个很好的健壮的方案，是采用redission客户端的分布式锁

```java
@Configuration
public class RedissonConfig {
    @Bean
    RedissonClient redisson(){
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        RedissonClient redisson = Redisson.create(config);
        return redisson;
    }
}
```

```java
@Resource
private Redisson redission;

@RequestMapping("/buyone")
public String buy(){
    String lockKey = "lock";
    RLock lock = redission.getLock(lockKey);
    try {
        lock.lock();
        int rest = Integer.parseInt(stringRedisTemplate.opsForValue().get("product-001"));
        if (rest > 0) {
            int realRest = rest - 1;
            stringRedisTemplate.opsForValue().set("product-001", realRest + "");
        } else {
            throw new RuntimeException("库存不足");
        }
    }finally {
        lock.unlock();
    }
    return "ok";
}
```

![image-20211104164438557](img\redis\74.png)

此时，存在两个问题：

1. lock将并发请求串行化，影响性能
2. 线程1成功加分布式锁，结果就在master将lockKey同步到slave之前，master挂了，根据哨兵模式重新推举的master是没有分布式锁的，造成分布式锁失效，同样会产生并发安全问题

#### 问题一

针对问题一，可以尽量缩小加锁的粒度，提升并发度；也可以利用分段锁的思想，将“product-001”分割成“product-001-01”，...，“product-001-10”，再将不同的请求hash到不同的key当中，提升并发度

#### 问题二

1. 方案一：针对问题二，redis属于AP类分布式系统，对于数据一致性要求稍低，可以采用CP类的主从架构，比如Zookeeper

   Zookeeper的主节点拿到锁后，向从节点同步，从节点同步完成之后会向主节点发送ack应答，主节点只有在半数以上的从节点ack之后，才会返回给客户端加锁成功的消息，然后客户端再执行后续逻辑

   如果主节点挂了，Zookeeper底层的ZAB机制能够保证一定能选举出包含分布式锁的从节点作为新的主节点

2. 方案二：RedLock

   ![image-20211104172416669](img\redis\75.png)

   设置多个对等的redis节点，超过半数加锁成功，才能执行客户端业务。这样，以一个三节点的redis对等集群为例，请求1的执行需要两个redis节点加锁成功，如果其中一个加锁成功的节点挂了，新来的请求也无法给超过半数，即两个节点加锁，因为其中一个还有请求1的锁

   原理和Zookeeper类似，但如果其中一个节点没有加锁成功，其他成功了的还要回滚，还有其他一些问题，说白了还不如Zookeeper，不推荐使用

   [Martin Kleppmann对红锁的质疑](https://blog.csdn.net/chen_kkw/article/details/81433470?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-2.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-2.no_search_link)

### 锁冲突处理

客户端在处理请求时加锁没加成功怎么办

一般有 3 种策略来处理加锁失败：

1. 直接抛出异常，通知用户稍后重试； 
2. sleep 一会再重试； 
3. 将请求转移至延时队列，过一会再试

#### **直接抛出特定类型的异常**

这种方式比较适合由用户直接发起的请求，用户看到错误对话框后，会先阅读对话框的内容，再点击重试，这样就可以起到人工延时的效果。如果考虑到用户体验，可以由前端的代码替代用户自己来进行延时重试控制。它本质上是对当前请求的放弃，由用户决定是否重新发起新的请求

#### **sleep**

sleep 会阻塞当前的消息处理线程，会导致队列的后续消息处理出现延迟。如果碰撞的比较频繁或者队列里消息比较多，sleep 可能并不合适。如果因为个别死锁的 key 导致加锁不成功，线程会彻底堵死，导致后续消息永远得不到及时处理

#### **延时队列**

这种方式比较适合异步消息处理，将当前冲突的请求扔到另一个队列延后处理以避开冲突

**延时队列的实现**

延时队列可以通过 Redis 的 zset(有序列表) 来实现。我们将消息序列化成一个字符串作为 zset 的 value，这个消息的到期处理时间作为 score，然后用多个线程轮询 zset 获取到期的任务进行处理，多个线程是为了保障可用性，万一挂了一个线程还有其它线程可以继续处理。因为有多个线程，所以需要考虑并发争抢任务，确保任务不能被多次执行

### 其他解决方案

- redis list

  我们建立product-001为键的list结构，当秒杀时，执行lpop即可。当lpop返回nil时， 即代表库存卖完了-

  - 优点： 实现简单

  - 缺点： 当库存非常大时， 会占用非常多的内存

    > 不过好在用于秒杀的商品，大多不会过万

- redis incr

  当秒杀时，执行DECR 即可，当DECR返回值小于0时，即代表库存卖完了

  - 优点： 节约内存

  - 缺点：decr incr的操作范围都是int64，当decr min_int64时，redis会报告overflow。不过好在考虑到业务实际，几乎不会出现该情况，毕竟库存终究会刷新的，秒杀也不可能一直持续下去

    > 除此之外，如果允许一次买多件的话，库存不够时需要回滚，不加锁会有并发问题。例如库存是4，A要买5件，decrby返回-1，购买失败，需要回滚，把这5件incrby回去；这时候又有B，C各要买2件，库存刚好可以满足，但是在A回滚之前，B，C都已经decrby 2了，两人都购买失败

## 缓存数据库双写不一致问题

### 问题场景

**当多个请求向数据库和缓存同时写入的时候，易产生双写不一致问题，即缓存中的数据和数据库中不一致**

通常，更新缓存和数据库有以下几种顺序：

1. 先更新数据库，再更新缓存
2. 先删缓存，再更新数据库
3. 先更新数据库，再删除缓存

三种方式的优劣来看一下：

**先更新数据库，再更新缓存**

![image-20211105212119780](img\redis\76.png)

可能会出现如图的情况，缓存为10，而数据库最新数据是6

此外，一般也不会采取更新缓存的方式，因为很多时候，在复杂点的缓存场景，缓存不单单是数据库中直接取出来的值，比如可能更新了某个表的一个字段，然后其对应的缓存，是需要查询另外两个表的数据并进行运算，才能计算出缓存最新的值的。还有，更新缓存的代价有时候是很高的，如果你频繁修改一个缓存涉及的多个表，缓存也频繁更新。但是问题在于，这个缓存到底会不会被频繁访问到？如果没有，会产生大量的冷数据，还不如删除操作简单

**先删缓存，再更新数据库**

这么做的问题：如果在删除缓存后，有客户端读数据，将可能读到旧数据，并有可能设置到缓存中，导致缓存中的数据一直是老数据

**先更新数据库，再删除缓存**

这个方案叫 Cache Aside Pattern，是比较经典的方案

（1）读的时候，先读缓存，缓存没有的话，那么就读数据库，然后取出数据后放入缓存，同时返回响应

（2）更新的时候，先删除缓存，然后再更新数据库

但也可能存在问题

![image-20211105215049042](img\redis\77.png)

线程1首先写数据库，然后删除缓存，此时线程3进来要读取该数据并更新到缓存中，但在更新到缓存之前，线程2进来写数据库并删除缓存，然后线程3完成缓存更新，此时缓存中为10而数据库中为6，出现不一致

如何解决？

### 延时双删

![image-20211105220758099](img\redis\78.png)

每次写数据库并删除缓存之后，sleep一段时间之后，再次进行删除缓存的操作，但这种方法

缺点：① 无法确认较好的sleep时间；② 影响性能

### 内存队列

将不同线程的增删改查操作，打包并串行化

缺点：性能受影响较大；如果宕机，队列消失

### 合理解决方案

#### 分布式锁

每次增删改查操作，上分布式锁，串行化解决问题，会有一些性能问题

#### 读写锁

读写锁。大多数web应用是读多写少型的，例如电商网站，浏览时间大部分情况下是大于下订单时间的，因此，可以采取读写锁改进分布式锁的并发度，读读时互不影响

```java
@Resource
private Redisson redission;

@RequestMapping("/buyone")
public String buy(){
    String lockKey = "lock";
    RReadWriteLock lock = redission.getReadWriteLock(lockKey);
    lock.writeLock().lock();
        try {
            /**
             *  1. 更新数据库
             *  2. 删除缓存
             *  (Cache Ahead Pattern)
             */
        }finally {
            lock.writeLock().unlock();
        }
        return "ok";
    }
    @RequestMapping("/watchone")
public String watch(){
    String lockKey = "lock";
    RReadWriteLock lock = redission.getReadWriteLock(lockKey);
    lock.readLock().lock();
    try {
        /**
         *  1. 查询缓存为空
         *  2. 查询数据库
         *  3. 更新缓存
         */
    }finally {
        lock.readLock().unlock();
    }
    return "ok";
}
```

> readLock的lock方法核心源码
>
> ```java
> <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
>     return this.evalWriteAsync(this.getRawName(), LongCodec.INSTANCE, command, "local mode = redis.call('hget', KEYS[1], 'mode'); if (mode == false) then redis.call('hset', KEYS[1], 'mode', 'read'); redis.call('hset', KEYS[1], ARGV[2], 1); redis.call('set', KEYS[2] .. ':1', 1); redis.call('pexpire', KEYS[2] .. ':1', ARGV[1]); redis.call('pexpire', KEYS[1], ARGV[1]); return nil; end; if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then local ind = redis.call('hincrby', KEYS[1], ARGV[2], 1); local key = KEYS[2] .. ':' .. ind;redis.call('set', key, 1); redis.call('pexpire', key, ARGV[1]); local remainTime = redis.call('pttl', KEYS[1]); redis.call('pexpire', KEYS[1], math.max(remainTime, ARGV[1])); return nil; end;return redis.call('pttl', KEYS[1]);", Arrays.asList(this.getRawName(), this.getReadWriteTimeoutNamePrefix(threadId)), new Object[]{unit.toMillis(leaseTime), this.getLockName(threadId), this.getWriteLockName(threadId)});
> }
> ```

封装了lua脚本，通过hset设置分布式锁，key→hash{mode→read}，每个锁包含一个mode字段，每次业务会来判定锁是什么模式的锁

#### TTL

对于某些对缓存和数据库一致性要求并不高的应用或模块，如下订单时库存数量的显示

> 对于读多写多的场景，缓存其实意义不大，因为缓存命中率并不高
>
> 如果是高并发的读多写多场景，请求直接打到DB是不可取的，可以尝试的方案有：
>
> - canal
> - TiDB

## Redis限流

### 为什么要限流？

- 系统的处理能力有限时，阻止计划外的请求继续对系统施压
- 控制用户行为，避免垃圾请求

### 真实场景

调用华为平台的设备历史数据接口时，发现接口报错：`连接被阻断，可能是请求过快？`

查询官方网站，有如下规定：

![image-20211109155914795](img\68.png)

流控值显示，一分钟内该接口仅能被调用100次，于是，自然而然就需要进行限流

### 方法迭代

#### Redis官方-计数器法

Redis官方在INCR命令的介绍里面，给出了两种简单的限流方案：[Redis-INCR](https://redis.io/commands/INCR?utm_source=wechat_session&utm_medium=social&utm_oi=1285003249632886784)

摘录如下：

**Pattern: Rate limiter 1**

```sh
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
MULTI
    INCR(keyname)
    EXPIRE(keyname,10)
EXEC
current = RESPONSE_OF_INCR_WITHIN_MULTI
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    PERFORM_API_CALL()
END
```

每个请求的ip设置一个key为"ip:timestamp"、value为0的记录，过期时间设置为10秒。每次该请求来调用接口，对应记录的value进行incr操作+1，当大于该时间戳内的调用阈值，则返回异常或其他处理

让我们仿照这个方法，延长到100次/60秒，为请求的ip设置一个key为"ip:timestamp/60000"的记录，每次该请求调用，则对应ip incr，但这个方案有明显的问题：① 对于第一分钟的key，在最后n秒内请求了100次，n秒后便会在新一分钟的新key里开始计数，如果在前60-n秒内请求了100次，那这两批请求虽然时间上在1分钟内，实际上却请求了200次；② 需要不断更新key、删除key，每个时间戳还要插入新key，消耗资源

**Pattern: Rate limiter 2**

```sh
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    value = INCR(ip)
    IF value == 1 THEN
        EXPIRE(ip,1)
    END
    PERFORM_API_CALL()
END

##为了防止incr但没有设置incr，弄成原子操作
local current
current = redis.call("incr",KEYS[1])
if current == 1 then
    redis.call("expire",KEYS[1],1)
end
```

单纯用请求ip作为key，存活时间设置为1秒，1秒内请求大于10，就会超阈值

让我们仿照这个方法，延长到100次/60秒，为请求的ip设置一个key为"ip"的记录，存活时间设置为60秒，这个方法同样存在如上面方法中问题①一样的情况

**计数器法的一大问题就是没有很好处理边界情况**

#### 自己的想法

每次请求来，设置一个key为“ip:时间戳”的记录，过期时间设置为60秒

每次新请求来，用keys *统计记录有没有超过100，超过则禁流

这种方法同样存在问题：① 无法处理同一时间戳内大量请求的情况（虽然这种很难发生，也可以加判断语句来避免）；② 每个时间戳都生成key，且要控制他们的超市删除等等，100个请求还好，要是有万级、百万级呢，消耗资源很大；③ keys *统计是O(N)的，效率不高

#### 滑动窗口法

> [不得不谈的限流算法](https://www.sohu.com/a/319155194_663371)

一个例子，将60秒的窗口，分割成6个小窗口，每个窗口过期时间为10秒，第一个窗口包含时间范围为当前第一个请求时间到未来10s，即0s~10s，第二个为10s~20s，依此类推，计数时将6个窗口的计数加起来，超过100则禁流

![image-20211109171150389](img\redis\94.png)

可以考虑用时间戳/10000、时间戳/10000 + 10000、...、时间戳/10000+60000来设置key

同样存在边界问题，只不过缩小了范围，计数器时，第一个60秒的后30s和第二个60s的前30s可能出现200次请求的问题；6个窗口时，在第一个窗口的后5秒内发送了100个请求，5秒后第一个窗口过期，新窗口即第6个窗口的前5秒发送了100个请求，同样产生200个请求的问题，只不过概率被缩小了

> 实际上，计数器法就是1个窗口的滑动窗口法，我的思路就是窗口大小为一个时间戳的滑动窗口
>
> 窗口越多越精确，越消耗redis资源

#### 更科学的滑动窗口

> 这是贴近我自己思路的版本

记录用户每个行为，存成一个zset，key为“用户:行为”，value为时间戳（这个值无所谓，只要保证唯一性），score为时间戳

每次请求来，向zset中添加数据，并删除过期数据，用zcard统计数据，同时给key设置一个expire时间，长时间没有该用户行为，则删除键，时间应设置为时间窗大小，即我的例子中的60秒

实际上，原理差不多贴近我自己的想法，只不过keys *是O(N)的，而zcard，底层有ziplist和skiplist两种实现

> [https://blog.csdn.net/qq920444182/article/details/104655065](https://blog.csdn.net/qq920444182/article/details/104655065)

- 如果是skiplist，即跳表的话，直接获取一个length属性就可以了，复杂度是O(1)
- 如果是用ziplist压缩链表存储的话，就不一样了。如果ziplist的长度小于UINT16_MAX，即16位无符号整型的最大值，65535，就直接返回，否则的话遍历ziplist获取长度

只要在conf文件中配置zset_max_ziplist_entries参数为0或zset-max-ziplist-value参数较小，就可以保证创建的是skiplist而不是ziplist结构，就能保证ZCARD的复杂度一定是O(1)

> 解决了O(N)复杂度的问题，但同样有消耗资源大的问题

可以采用lua脚本实现原子性，pipeline+事务也可以

#### 漏桶算法

上述方法，典型的问题是临界点问题和匀速限流问题，有的方法解决了临界点问题，但都没有解决匀速限流问题

漏桶算法可以

水龙头打开后流下的水（请求）以一定的速率流到漏桶里（限流容器），漏桶以一定的速度出水（接口响应速率），如果水流速度过大（请求过多）就可能会导致漏桶的水溢出（访问频率超过接口响应速率），这时候我们需要关掉水龙头（拒绝请求），下面是经典的漏桶算法图示：

![img](img\redis\95.png)

示例：

```java
public class FunnelRateLimiter {
    static class Funnel {
        int capacity;  // 漏桶容量
        float leakingRate;  // 漏水速率
        int leftQuota;  // 剩余空间
        long leakingTs;  //  上一次漏水的时间
        public Funnel(int capacity, float leakingRate) {
            this.capacity = capacity;
            this.leakingRate = leakingRate;
            this.leftQuota = capacity;
            this.leakingTs = System.currentTimeMillis();
        }
        // 请求过来（流入），首先需要让过期的请求流出，腾出空间
        void makeSpace() {
            long nowTs = System.currentTimeMillis();
            long deltaTs = nowTs - leakingTs;
            // 间隔时间内已经流出的请求数
            int deltaQuota = (int) (deltaTs * leakingRate);
            if (deltaQuota < 0) { // 间隔时间太长，整数数字过大溢出
                this.leftQuota = capacity;
                this.leakingTs = nowTs;
                return;
            }
            if (deltaQuota < 1) { // 腾出空间太小，最小单位是 1
                return;
            }
            // 剩余空间+已经流出的请求数
            this.leftQuota += deltaQuota;
            // 记录此次流出时间
            this.leakingTs = nowTs;
            if (this.leftQuota > this.capacity) {
                this.leftQuota = this.capacity;
            }
        }
        // 是否放行请求
        boolean watering(int quota) {
            makeSpace();
            if (this.leftQuota >= quota) {
                this.leftQuota -= quota;
                return true;
            }
            return false;
        }
    }
    private Map<String, Funnel> funnels = new HashMap<>();
    public boolean isActionAllowed(String userId, String actionKey, int capacity, float leakingRate) {
        String key = String.format("%s:%s", userId, actionKey);
        Funnel funnel = funnels.get(key);
        if (funnel == null) {
            funnel = new Funnel(capacity, leakingRate);
            funnels.put(key, funnel);
        }
        return funnel.watering(1); // 需要 1 个 quota
    } 
}
```

我们观察 Funnel 对象的几个字段，我们发现可以将 Funnel 对象的内容按字段存储到一个 hash 结构中，灌水的时候将 hash 结构的字段取出来进行逻辑运算后，再将新值回填到hash 结构中就完成了一次行为频度的检测

但是有个问题，我们无法保证整个过程的原子性。从 hash 结构中取值，然后在内存里运算，再回填到 hash 结构，这三个过程无法原子化，意味着需要进行适当的加锁控制。而一旦加锁，就意味着会有加锁失败，加锁失败就需要选择重试或者放弃。如果重试的话，就会导致性能下降。如果放弃的话，就会影响用户体验。同时，代码的复杂度也跟着升高很多。这真是个艰难的选择，我们该如何解决这个问题呢？可以使用Redis-Cell

[Redis-Cell](https://blog.csdn.net/yzf279533105/article/details/111310685?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-0.no_search_link&spm=1001.2101.3001.4242.1)

无法放行突发流量

#### 令牌桶算法

[Spring Boot 的接口限流算法优缺点深度分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/145667852)

令牌桶算法，又称token bucket。为了理解该算法，我们再来看一下算法的示意图：

![img](img\redis\96.gif)



从图中我们可以看到，令牌桶算法比漏桶算法稍显复杂。首先，我们有一个固定容量的桶，桶里存放着令牌（token）。桶一开始是空的，token以一个固定的速率r往桶里填充，直到达到桶的容量，多余的令牌将会被丢弃。每当一个请求过来时，就会尝试从桶里移除一个令牌，如果没有令牌的话，请求无法通过

```java
public class TokenBucketDemo {
    public long timeStamp = getNowTime();
    public int capacity; // 桶的容量
    public int rate; // 令牌放入速度
    public Long tokens; // 当前令牌数量
    public boolean grant() {
        long now = getNowTime();
        // 先添加令牌
        tokens = Math.min(capacity, tokens + (now - timeStamp) * rate);
        timeStamp = now;
        if (tokens < 1) {
            // 若不到1个令牌,则拒绝
            return false;
        }
        else {
            // 还有令牌，领取令牌
            tokens -= 1;
            return true;
        }
    }
    private static Long getNowTime(){
        return System.currentTimeMillis();
    }
}
```

> 不要使用定时器，太消耗资源，每次请求来的时候计算就可以了

也可以使用google Guava包中的RateLimiter

我们再来考虑一下临界问题的场景。在0:59秒的时候，由于桶内积满了100个token，所以这100个请求可以瞬间通过。但是由于token是以较低的速率填充的，所以在1:00的时候，桶内的token数量不可能达到100个，那么此时不可能再有100个请求通过。所以令牌桶算法可以很好地解决临界问题*(不是完美实现每个60s内都只有100个请求，但很好地解决了多次突发请求，实际生产中很常用)*

#### 总结

最终，为了完美满足100次/60秒，且100次也不是很多，采用了zset+lua的方案

# 参考

