## [返回](/)

# 《MySQL技术内幕——InnoDB存储引擎》读书笔记

# 1. MySQL体系结构和存储引擎

MySQL是单进程多线程的架构

> 配置文件：my.ini
>
> 可通过`mysql --help`或`mysql --help|findstr(win)/grep(linux) my.ini `查看`Default options are read from the following files in the given order:`
>
> 有多个配置文件，读取顺序为上述查看到的，后面会覆盖前面，也就是说不一样时以最后一个为准

## database和instance

- database：物理操作系统文件或其他形式文件类型的集合。在MySQL数据库中，数据
  库文件可以是frm、MYD、MYI、ibd 结尾的文件
- 实例：MySQL数据库实例由后台线程以及一个共享内存区组成。共享内存可以被运行
  的后台线程所共享。需要牢记的是，数据库实例才是真正用于操作数据库文件的

## 存储引擎

### InnoDB

- InnoDB采用了聚集(clustered) 的方式，每张表的存储都是按主键的顺序进行存放

  > 如果没有显式地在表定义时指定主键，InnoDB 存储引擎会为每一行生成一个 6字节的ROWID（隐藏字段），并以此作为主键

### MyISAM

- 不支持事务、表锁
- 支持全文索引
- 只缓存索引，不缓存数据

### Memory

- 将数据存放在内存中，速度快
- 用于存临时表
- hash索引，而非B+树

## 连接

- TCP/IP套接字方式是MySQL数据库在任何平台下都提供的连接方式，也是网络中
  使用得最多的一种方式
- 命名管道和共享内存
- UNIX套接字

# 2. InnoDB

## InnoDB体系架构

多个内存块组成一个内存池，用于缓存磁盘数据、维护内部数据结构、redo log buffer等等

![image-20211127174234959](img\jsnm\1.png)

### 后台线程

- Master Thread：非常核心的线程，主要负责将缓存中的数据异步刷新到磁盘

  - 刷脏
  - change buffer的merge操作
  - 回收undo log
  - ......

- IO Thread：InnoDB中使用了大量的AIO（Async IO），这些IO Thread主要用于处理AIO请求的回调等等

- Purge Thread：回收undo log

  - InnoDB 1.1 之前，purge只能在master thread中进行；1.1后可以用独立的purge thread，需要在配置文件中指定

    ```sh
    [mysqld]
    innodb_purge_threads=1
    ```
  
- Page Cleaner Thread：刷脏页；减轻一部分master thread的工作压力；异步刷脏

### 内存

InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术来提高数据库的整体性能

通过`innodb_buffer_pool_size`配置缓冲池大小

主要内容：

![image-20211127175616931](img\jsnm\2.png)

#### 内存管理之LRU List、Free List和Flush List

**LRU List**

**管理内存中已有的数据页**

数据库中的缓冲池是通过LRU（LatestRecentUsed，最近最少使用）算法来进行管理的。即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页

InnoDB使用改进的LRU算法，InnoDB的存储引擎中，LRU列表中还加入了midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放人到LRU列表的首部，而是放人到LRU列表的midpoint
位置

> 为什么不采用朴素的LRU算法，直接将读取的页放入到LRU列表的首部呢？
>
> 有些操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说又仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放人LRU列表的首部，那么非常可能将所需要的热点数据页从LRU列表中移除，而在下一次需要读取该页时，InnoDB存储引擎需要再次访问磁盘

**Free List**

LRU列表用来管理已经读取的页，但当数据库刚启动时，LRU列表是空的

- 需要读取数据到内存时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，放入到LRU列表中
- 否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页

> InnoDB 1.0.x 后还支持页压缩，即另外用unzip_LRU列表进行管理，存放小于16KB的页
>
> 例如对需要从缓冲池中申请页为4KB的大小，其过程如下（伙伴算法）：
>
> 1. 检查4KB的unzip_ LRU列表，检查是否有可用的空闲页；若有，则直接使用
> 2. 否则，检查8KB的unzip_ LRU列表；若能够得到空闲页，将页分成2个4KB页，存放到4KB的unzip_ LRU列表
> 3. 若不能得到空闲页，从LRU列表中申请一个16KB的页，将页分为1个8KB的页、2个4KB的页，分别存放到对应的unzip_ LRU 列表中
>
> 即不断从小到大找，找到了后再分裂并分别管理

**Flush List**

在LRU列表中的页被修改后，称该页为脏页(dirty page)，即缓冲池中的页和磁盘上的页的数据产生了不一致。这时数据库会通过CHECKPOINT机制将脏页刷新回磁盘

而Flush列表中的页即为脏页列表。需要注意的是，脏页既存在于LRU列表中，也存在于Flush列表中。LRU 列表用来管理缓冲池中页的可用性，Flush 列表用来管理将页刷新回磁盘，二者互不影响

#### redo log buffer

三种模式：`innodb_flush_log_at_trx_commit`

- =0，master thread每秒一次将redo log buffer中的内容复制到os buffer（文件系统都是有缓冲的），同时通知操作系统调用fsync()即os buffer同步到磁盘的行动

  > MySQL crash，可能丢失一秒的数据

- =1，默认情况，每次提交事务的时候刷到os buffer并通知fsync()，数据安全性最高

- =2，每次提交事务时刷到os buffer（不经过redo log buffer，master thread也就无法每隔一秒进行fsync调用了），但不通知fsync()，这样系统调用fsync()的时间就完全无法掌握了

  > MySQL crash不怕，但在fsync()前若发生os crash可能会丢失若干秒的数据

#### 额外的缓存池

某些操作过程中需要的数据结构的内存是从这里分配的，如果这里的不够，再在缓冲池上分配

## checkpoint机制

如果每次直接刷盘，不存redo log

- 倘若每次一个页发生变化，就刷盘，那么这个开销是非常大的，因为是随机写；若热点数据集中在某几个页中，那么数据库的性能将变得非常差
- 同时，如果在刷盘时发生了宕机，那么数据就不能恢复了；如果裸写磁盘，一个需要操作多个磁盘块的操作执行到一半了，程序崩溃，数据回滚是比较难搞的

### WAL：Write Ahead Log

修改数据的事务提交时，**在刷盘之前，首先写入redo log之中**

这样就可以避免上面两个问题了：

1. 采用顺序写，避免随机写磁盘，提高效率
2. 刷盘时宕机，重启时能够根据redo log进行重做

**两阶段提交**：redo log prepare→写binlog→redo log commit

### checkpoint

Checkpoint（检查点）技术的目的是解决以下几个问题：

- 缩短数据库的恢复时间

  > 当数据库发生宕机时，数据库不需要重做所有的日志，因为Checkpoint之前的页都已经刷新回磁盘。故数据库只需对Checkpoint后的重做日志进行恢复。这样就大大缩短了恢复的时间

- 缓冲池不够用时，将脏页刷新到磁盘

  > 当缓冲池不够用时，根据LRU算法会溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘

- 重做日志不可用时，刷新脏页

  > 强制产生Checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置

#### 几种模式

- Sharp Checkpoint

  > 数据库关闭时，强制进行刷盘

- Fuzzy Checkpoint

  - Master Thread Checkpoint

    > master thread选择合适的时间异步将一部分的数据页刷盘，通常速度为每1秒或10秒，用户线程不会阻塞

  - FLUSH_LRU_LIST Checkpoint

    > 用户查询的数据不在内存中，需要从磁盘载入内存，但发现缓冲池可用内存页不足，就要将LRU List的尾端页淘汰，如果这些页中有脏页，就需要进行checkpoint，使用Page Cleaner线程
    >
    > - InnoDB 1.2.x（MySQL 5.6）之前，会阻塞用户线程
    > - InnoDB 1.2.x之后，采用Page Cleaner Thread来异步进行该checkpoint，不阻塞
    >
    > PS：通过`innode_lru_scan_depth`设置每次扫描LRU列表的长度，默认1024，即如果出现可用页不足的情况，从LRU尾端扫描1024个页

  - Async/Sync Checkpoint

    > redo log不可用，即快写满时，不可覆盖的（已刷盘的可直接被覆盖，循环写的特点）redo log比例超过75%，会触发Async Checkpoint；不可覆盖的redo log比例超过90%，会触发Sync Checkpoint。都是在Page Cleaner线程中
    >
    > - InnoDB 1.2.x之前，Async会阻塞发现问题的查询，Sync会阻塞所有查询

  - Dirty Page too much Checkpoint

    > 脏页太多导致用户的查询可能需要一次性淘汰很多脏页，影响效率
    >
    > `innodb_max_dirty_pages_pct`=75%，若脏页比例超过该值，则强制Checkpoint

PS：刷脏速度和`innodb_io_capacity`、`innodb_max_dirty_pages_pct`和write_pos - checkpoint来确定：根据后两个算出一个R，用`innodb_io_capacity` * R%定义刷盘速度，因此条件允许情况下，可以适当调高`innodb_io_capacity`

## InnoDB关键特性

### change buffer

InnoDB 1.0.x 后，对insert buffer进行了升级，支持所有DML操作，命名change buffer。其中包含insert buffer、delete buffer、purge buffer（对应update）

> update分两个过程：① 标记为删除，用delete buffer缓存；② 真正删除，用purge buffer缓存

对非唯一索引的插入，由于是按主键顺序插入的，如果按二级索引访问，会造成离散的访问数据页，所以会造成性能下降

change buffer在进行修改操作时，首先判断所要的数据页在不在内存中，如果不在不会将它缓存到内存中（避免离散访问），而是将操作写到change buffer中，待到后面需要读取这一页时，再进行merge操作将对应数据页更改为脏页，这样就大大提高了效率

> 何时merge？
>
> 1. 访问到该数据页
> 2. 发现插入记录后数据页可用空间会降低到1/32以下
> 3. master thread每个1或10秒进行merge

### double write

[https://www.cnblogs.com/nandi001/p/11662992.html](https://www.cnblogs.com/nandi001/p/11662992.html)

#### partial page write

将内存中的脏页进行刷脏时，可能会出现这样的问题：数据库的数据页是16KB，而操作系统一般是2KB或者4KB，将某一个16KB的脏页刷盘时，可能刚刷了一部分（2或4KB的倍数），数据库系统就挂了

再次重启时，数据库会对数据页进行校验，检查page的checksum，checksum就是page的最后事务号，发现无法恢复，这一页就“坏了”，永远丢失了

> redo log是物理日志，记录的是某一完整的数据页，不同偏移量上面发生的变更记录，并不记录整个页的情况*（这要记录得占用多大呀）*，对于不完整的数据页自然是没有办法恢复的

这就是partial page write，即部分写失效问题

#### 解决

可用double write来解决，在磁盘的ibdbata文件中划出2M连续的空间，称为共享表空间

![img](img\jsnm\3.png)

1. 每次产生脏页时，首先自然是写redo log，之后在刷脏时，会首先通过memcpy将脏页复制到double write buffer，接着迅速分两次，每次1MB进行fsync写到磁盘上的共享表空间。这个过程是针对连续内存的顺序写，因此比较快

2. 写完double write后，再将double write buffer中的脏页写到表空间的数据页中，这个过程是离散写

   如果这个过程发生了崩溃，恢复时：

   1. 首先发现该页checksum不对劲，去共享表空间尝试恢复该脏页
      1. 如果共享表空间也没有，说明刷盘这个行为没有发生，磁盘中没有坏数据页，直接根据原始数据和redo log恢复就可以了
      2. 如果共享表空间有，说明是在刷盘阶段出现了问题，先根据共享表空间的副本恢复数据页完整性，再根据redo log恢复

![img](img\jsnm\4.png)

> 设置`skip_innodb_doublewrite`可取消该机制

### 自适应哈希

Auto Hash Index

对于频繁访问且访问模式一致的二级索引页，InnoDB在判断哈希的收益较大时，会给该页建立一个哈希表，直接定位到数据页，而不需要再在索引树上面查找

`set global innodb_adaptive_hash_index=off/on`

缺陷：① 占用buffer pool；② 可能哈希冲突； ③ 无法排序；④ 只适用于等值查询；⑤ 无法人为管理

默认开启，但一般不建议使用

### 异步IO

Asynchronous IO

- sync IO：发出一个IO请求，停等到数据返回
- AIO：可以发出多个请求，再等待IO操作完成

IO merge：访问连续的页，合并成一个访问

目前版本都是Native AIO，调用内核级别的AIO（除Mac OS，还是使用InnoDB自己的实现）

### 刷新邻接页

刷新某个脏页时，发现所在分区（extend）有其他脏页，则一起刷脏，可以将可能出现的多次IO合并成一次IO

如果磁盘（比如固态硬盘）IOPS非常高，建议关闭该功能，因为可能将没那么脏的脏页刷脏，但它又很快重新变脏了

`innodb_flush_neighbors`控制

## InnoDB启动与关闭

> 实际上是指mysql对InnoDB的处理过程

根据`innodb_fast_shutdown`控制，默认为1

- =0：需要完成所有的full purge（删除所有无用undo）、merge change buffer、刷脏。InnoDB升级时，必须先把这个设置为0
- =1：刷脏
- =2：不刷脏也不purge、merge，而是将所有日志都写入日志文件。下次启动时进行recovery机制（还有异常关闭数据库实例时，也会进行该操作，比如：kill进程、重启机器等，会回滚操作并记录错误日志）

> 不能进行有效恢复时，比如数据页出现corruption，则数据库可能宕机（crash）并记录错误日志

# 3. 文件

[https://www.cnblogs.com/wy123/p/7102128.html](https://www.cnblogs.com/wy123/p/7102128.html)

1. MySQL（server）创建并管理的数据库文件：
   - .frm文件：存储数据表的框架结构，文件名与表名相同，每个表对应一个同名frm文件，与操作系统和存储引擎无关，即不管MySQL运行在何种操作系统上，使用何种存储引擎，都有这个文件
2. MyISAM数据库表文件：
   - .MYD文件：即MY Data，表数据文件
   - .MYI文件：即MY Index，索引文件
   - .log文件：日志文件

3. InnoDB采用表空间（tablespace）来管理数据，存储表数据和索引，InnoDB数据库文件（即InnoDB文件集，ib-file set）：

- ibdata1、ibdata2等：系统表空间文件，存储InnoDB系统信息和用户数据库表数据和索引，所有表共用
- .ibd文件：单表表空间文件，每个表使用一个表空间文件（file per table），存放用户数据库表数据和索引
- 日志文件： ib_logfile1、ib_logfile2

# 4. 表

