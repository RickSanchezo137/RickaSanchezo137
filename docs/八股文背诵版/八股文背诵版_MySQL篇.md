# [返回](/)

# 八股文背诵版之—MySQL篇

## InnoDB基础

### :point_right:InnoDB和MyISAM的区别？

MyISAM是旧版本MySQL的默认存储引擎，后面被InnoDB所替代，区别有：

1. InnoDB支持事务，MyISAM不支持事务
2. InnoDB支持外键，MyISAM不支持外键
3. InnoDB的锁粒度可以到达行锁，而MyISAM只支持表级锁
4. InnoDB和MyISAM都是采用B+树作为索引结构，只不过具体的实现方式不同。InnoDB是采用聚集索引，它的数据记录就存储在主键索引树的叶子节点上*（表数据文件就是按照B+树组织的）*，通过主键可以直接查找到数据，辅助索引指向的是主键，查找需要返回主键索引，也就是回表；MyISAM采用的是非聚集索引，它的叶子节点不存放数据文件，而是存放指向数据文件的地址指针，辅助索引的叶子节点不存放主键，而是也存放数据文件指针，与主键索引是分离的，每次查找先找到文件指针，再定位到数据文件。因此，InnoDB必须有唯一索引*（主键存在则主键，不存在则隐藏字段DB_ROW_ID）*，而MyISAM可以没有
5. InnoDB不存储表的具体行数，count(*)需要全表扫描，这是由于不同的一致性视图下count(\*)的结果不同；而MyISAM不支持事务，所以行数直接用一个字段保存
6. InnoDB在redo log的支持下，是crash safe的；而MyISAM恢复起来比较困难

### :point_right:你觉得为什么InnoDB可以替代MyISAM？

如果是只读数据，那MyISAM当然可以完成；但大多数应用会对数据库进行修改，而InnoDB提供了并发安全保障，更细的锁粒度以及提供事务，同时还具备crash safe的能力，替代MyISAM也就不奇怪了

### :point_right:知道redo log吗？讲一下WAL？

redo log是InnoDB存储引擎特有的一种日志，叫做重做日志，它是一种物理日志，记录了某某数据页的某某位置做了某某修改，它是四个文件循环存储的。目的是在MySQL异常终止之后，可以根据redo log去恢复。具体的实现就是靠WAL，即Write Ahead Logging，同时它也能提高MySQL的性能。MySQL对数据的修改是这样的流程：首先在内存中进行修改并commit，接着在redo log中记录修改，记录完标记为prepare状态，接着写binlog，写完记录xid，最后标记redo log commit，等到空闲时刻再将修改更新到数据库中，这就是WAL的思想。由于WAL是顺序存储，而刷脏是随机IO，因此MySQL性能被提高了

> 记住两个参数：innodb_flush_log_at_trx_commit、sync_binlog
>
> 双1原则

### :point_right:讲一下两阶段提交？作用？

两阶段提交是这样一个流程：先写redo log并标记prepare，再写bin log并标记xid，最后标记redo log commit

作用是为了保证redo log和bin log在逻辑上的一致性，避免从库根据bin log重做了之后，内容与主库不一致

### :point_right:InnoDB四大特性？

1. change buffer：不在内存的普通索引记录的修改，可以先不从磁盘读，而是在内存的change buffer中修改，读取对应数据页时再merge
2. double write：数据页在刷脏时系统宕掉，可能会刷不完整的数据页。因此每次数据页刷脏的时候，会首先在共享表空间，也就是ibdata的空间开辟一个2mb大小的double write buffer，刷脏时先迅速分两次刷进这个buffer*（顺序IO）*。恢复的时候会首先检查数据页checksum，如果不完整，则去double write buffer里面重新将页面恢复，并结合redo log重做
3. adaptive hash index：只能用于普通索引，系统判断对它的访问是有规律的定值查询，会生成hash index提高效率
4. read ahead：异步地预先将部分数据页从磁盘取到内存，期望它能被用到

### :point_right:讲一讲order by原理？

order by根据参数`max_length_for_sort_data`的设置，可能采用全字段排序或row id排序。全字段排序的流程是，首先根据聚簇索引树找到所需要的记录字段，并加入sort buffer；接着在sort buffer中根据order by后面的字段进行排序，最后返回结果集。row id的流程是，首先在聚簇索引找到order by后面的字段，将row id和字段加入sort buffer；接着进行排序；最后回表，找到结果集并返回

内存足够情况下，优先选择全字段；不够情况下全字段可能采用外部排序，所以可以使用row id；还有一种办法是创建索引时就创建相应顺序的

### :point_right:设置xx字段时，如何选择设置成唯一索引还是普通索引？

读取字段时，普通索引可能比唯一索引多扫描几个记录，相差不大；修改字段时，如果在相应数据页在内存内也相差不大，但如果不在，唯一索引因为需要进行唯一性判断，所以要先从磁盘读出来，而普通索引可以直接在change buffer里面修改，等读到相应数据页的时候再merge上去，所以对于修改频繁的字段，可设置为普通索引

### :point_right:未提交的事务可能被持久化吗？

可能的。redo log和bin log每个事务都有独立的binlog cache不同，redo log只有一个redo log buffer，而redo log的中间状态也会进入redo log buffer，所以刷盘时这一部分也可能被刷

### :point_right:刷脏时机？

1. 系统在空闲的时间片进行刷脏*（master thread）*
2. redo log buffer写满，必须阻塞一切操作，全力刷脏，推进check point
3. 系统进行某个操作时向内存读取数据页，然而内存空间不足，要淘汰脏页，此时需要刷脏
4. 系统关闭之前，会刷脏所有脏页
5. InnoDB一个刷脏策略，领接刷脏，刷脏某个页的时候顺带把旁边的数据页也刷脏了

### :point_right:bin log格式？区别？

有raw格式、statement格式和mixed格式。raw格式记录了完整的信息，但内存消耗较大；statement记录了sql语句的信息，但在有些情况下会导致主从不一致的情况；mixed格式会判断哪些语句不会导致主从不一致，采用statement格式，其他采用raw格式

## 索引

### :point_right:简单讲讲索引？

索引是一种能够对数据进行合理组织，提高查询效率的结构。它一方面能够提高查找效率，一方面维护和存储索引需要空间和时间上的开销，总体来说，是一种空间换时间的行为。以MySQL为例，有全文索引*（用于模糊查询效率低问题）*、R树索引*（空间数据索引，用于地理信息）*、哈希索引和B+树索引，默认采用的索引是B+树索引。对于B+树索引，又有聚簇索引和非聚簇索引这两种形式

### :point_right:创建索引的三种写法？



### :point_right:Hash索引和B+树索引的区别？

1. Hash索引只能用于精确的等值查找，可以很高的查询效率，而对于范围查找只有靠B+树实现，因为Hash索引是无序的，同时模糊查询、最左前缀匹配也都是靠B+树索引
2. Hash索引由于Hash冲突存在的可能性，性能可能不稳定；B+树索引每次都查询到叶子节点，查询性能较稳定
3. Hash索引必须回表；B+树索引不一定需要回表，因为有索引覆盖

### :point_right:B树和B+树的区别？为什么使用B+树作为默认索引​？时间复杂度？

B树的内部节点和叶子节点都存放键值对，而B+树内部节点只存放键，叶子节点存放数据；B+树的叶子节点层，每个叶子节点都和它右边的叶子节点相连，便于进行遍历和顺序查找

如果是读取不在内存Buffer Pool里面的数据，或者将数据写回磁盘，需要首先在磁盘上进行查找并读到内存，在磁盘上的随机寻址是一个比较耗时的工作，因此需要做到尽量少的磁盘寻址的步骤，就要求树的高度比较低。B+树由于每个内部节点只存储键，对于同样大的一个磁盘块，能存储比B树更多的节点，因此寻址次数较少，**查找效率提高**；此外B+树查询**性能比较稳定**，每次都是走到叶子节点层；最后，B+树由于叶子节点之间的顺序性，在**范围查找的效率**上比B树优越很多

N节点的M阶B+树的时间复杂度可以看作是以M为底logN。计算过程是这样的：对于树，时间复杂度即为树高，有N个节点，每个节点最多能连向M个路径，假设树高h，M^h=N，所以h=以M为底的logN

### :point_right:什么是聚簇/非聚簇索引？

聚簇索引：索引、数据放一起存；非聚簇索引：索引、数据分开存，索引的叶子节点存储指向数据存放的地址

### :point_right:非聚簇索引一定会回表吗？

不一定，由于索引覆盖的存在，如果非聚簇索引上本身包含了所需查询的所有信息，这样就可以不用回表

### :point_right:对于xx字段，怎么构建索引？

原则：① 优先选择高区分度的；② 选择经常作为查询条件的而不是经常更新的字段作为索引；③ 多利用最左前缀，减小索引大小；④ 不要太多索引

### :point_right:使用索引效率一定提升吗？

不一定，索引的创建和维护是需要代价的，不合理的使用反而会影响效率

### :point_right:什么是前缀索引？最左前缀原则知道吗？

前缀索引是指选取字符串索引的长度稍短的前缀子串作为索引，能够在保证一定区分度的情况下，减小索引树的大小。最左前缀原则是指对于联合索引的前若干个字段，或字符串索引的前若干个字符，也可以在索引树上匹配查找

### :point_right:索引下推知道吗？

知道。在辅助索引回表的时候，如果索引中包含了where条件中所需要的信息，则可以进行筛选，只对符合where条件的辅助索引进行回表查找

### :point_right:哪些情况会出现索引失效？

1. 不符合最左前缀匹配原则
2. 条件中含有or
3. 对索引字段进行计算，或调用函数
4. 索引字段用了不等于*（!=、<>）*，或is null、is not null

## 事务

### :point_right:事务四大特性？

- Atomicity：原子性，事务的执行是一个原子的过程，要么全部成功、要么全部失败
- Consistency：一致性，事务在执行前后状态是一致的
- Isolation：隔离性，事务之间互相隔离，互不影响
- Duration：持久性，提交后的事务具有持久性

### :point_right:数据库事务的作用？

可以用于解决数据库的并发一致性问题

- 脏读：读取到了其他事务还没有提交的修改

- 不可重复读：事务中对一条记录多次读取的结果不一致

- 幻读：事务中以当前读的形式读取某数据时，其他事务插入数据，导致事务读取到的前后数目不一致

  > 问题：导致加锁语义被破坏；session A set→session B insert→session A commit，导致在备库中显示为session B insert→session A set，B insert的记录被修改了，主备逻辑不一致

### :point_right:数据库的隔离级别？

- READ UNCOMMITTED：读未提交
- READ COMMITTED：读提交，可阻止脏读
- REPEATABLE READ：可重复读，可阻止脏读、不可重复读，InnoDB下可阻止幻读
- SERIALIZABLE：串行化读，加锁单线程读

通过MVCC和间隙锁可以实现中间两种，通过加锁机制可以实现串行化

### :point_right:什么是MVCC？

> MVCC出现场景→是什么→怎么实现→解决了什么问题

多线程访问下数据库可能出现并发一致性问题，如果加锁营造单线程环境，但会很大程度影响运行效率

因此出现了MVCC，即多版本并发控制，是InnoDB提出的一种在避免加锁的情况下，解决并发一致性问题的方案

在MVCC机制下，InnoDB的每个记录都有三个隐藏字段，DB_ROW_ID、DB_TRX_ID以及DB_ROLL_PTR，在事务开始的时候，会给数据库生成一个当前的快照，这个快照不是记录的拷贝，而是通过DB_TRX_ID即事务版本，以及undo log来实现的。事务开始后，每条记录的DB_TRX_ID字段会写入当前的事务ID，事务进行中对每条记录进行访问时，会比对DB_TRX_ID的版本对自己是否可见：如果是在自己之前提交的，则可见；如果是未提交的或在自己创建后提交的，则不可见。对于不可见的记录，会通过DB_ROLL_PTR即回滚指针找到undo log，并根据undo log将记录恢复成上一个版本，再进行可见性的比对，不行的话继续顺着版本链往上找

根据MVCC机制，可以实现事务内的快照读，实现读提交和可重复读，但在当前读的时候，读到的都是最新的记录，此时可能出现幻读的问题，需要靠间隙锁来解决

## 锁

### :point_right:什么是间隙锁？行锁怎么实现的？

间隙锁是一种加载索引之间的间隙上的锁，间隙锁只与插入这个行为互斥。它是InnoDB为了解决幻读问题提出的锁，一般来说，InnoDB在更新时或当前读情况下，加锁的单位是next-key lock，会对扫描到索引的范围加上next-key lock，因此对于这些加上锁的索引间隙，其他线程就无法在这些间隙插入记录，会被阻塞住

行锁是加载索引上面的，可以通过update、lock in share mode，for update加行锁，lock in share mode两两不互斥，如果没有加在索引上，则会变成全表锁。此外，加锁只加扫描的部分，如果lock in share mode是加在二级索引上的，则查询被索引覆盖了，那么也会只锁二级索引，可以正常在主键索引上进行修改；其他两者在这种情况下，默认也会给主键索引加锁

### :point_right:两阶段锁协议是什么？

行锁不是等不需要的时候就解锁，而是在事务提交后才解锁

### :point_right:意向锁是什么？

对记录加行锁后，如果另一个session过来想要给这个表加表锁是不行的，所以加表锁就必须遍历表上有没有加行锁，这样太麻烦了，于是在加行锁之后会自动给表添加一个意向锁，这样再想加表锁的时候，发现加上了意向排他锁，就不用遍历也知道不能加锁了。总的来说，意向锁有点一个中间层的意思，省去了加表锁需要的遍历过程

### :point_right:乐观锁和悲观锁？sql实现一个？

乐观锁是指总期望认为在多线程下是安全的，没有加锁措施，直接通过CAS去修改值；悲观锁总期望认为多线程下是不安全的，因此会主动加锁避免这种情况发生

乐观锁思路：给表添加一个version字段，每次修改某记录时，先读取version，假设旧值为a，将修改version的操作和本来要做的修改操作写在一个sql语句里面，如果版本比对一致则修改成功，否则失败，例如：

`update table t set x = y, version = version + 1 where id = xxx and version = a `

悲观锁思路：flush tables with read lock，mdl锁，lock table t with read/write lock，set global readonly = true，update，lock in share mode，for update等等

### :point_right:死锁是什么？sql实现一个？怎么解决？

若干个线程共同请求某共享资源，分别各自持有一部分，同时又等待另一部分，造成多个线程互相卡死的状况

`session A: begin; select x from t where id = 1 for update; select y from t where id = 2 for update; (block by A)`

`session B: begin; select y from t where id = 2 for update; select x from t where id = 1 for update; (block by A) `

1. 超时后全部回滚，通过`innodb_lock_wait_timeout`设置
2. 死锁检测并回滚某一线程，`innodb_deadlock_detect`设置为on，并控制并发度
3. 修改业务sql逻辑

## 应用

### :point_right:如何定位sql问题？

使用explain命令，查看并分析sql语句的执行计划，包括扫描的行数、索引的选择等

### :point_right:MySQL分页？超大分页？

MySQL可以通过limit语句来进行分页查询。超大分页问题，就是对于这样一个查询：

`select * from t where age > 20 limit 1000000, 10`

MySQL的执行流程是这样的：首先加载1000010条记录，接着取后十条，丢弃前面所有。这样效率是非常低下的，开销也很大，优化方式有：

1. 优化这样的需求，即立即跳转到1000000条记录之后，可以分段来，或者提前缓存一部分内容
2. 阿里开发手册：优化查询语句，`select * from t where id in (select id from t where age > 20 limit 1000000, 10)`，覆盖索引大大提高了效率，减少了开销

### :point_right:存储用户的密码散列，应该用什么字段？

散列值为固定值，应当使用char()而不是varchar，提高检索效率

### :point_right:大表查询如何优化？

1. 优化sql语句、索引
2. 添加缓存机制
3. 读写分离
4. 垂直分表/水平分表

### :point_right:了解慢查询日志吗？统计过慢查询吗？如何优化？



### :point_right:为什么要设置主键？设置为UUID还是自增主键？



### :point_right:字段为什么设置成not null？

null值会占用更多字节，且会在程序中造成很多与预期不符的情况

### :point_right:如何优化查询过程中的数据访问？



### :point_right:长难语句如何优化？



### :point_right:limit/union/where如何优化？



### :point_right:分表分库讲一下？



### :point_right:主从复制讲一下？具体流程？



### :point_right:了解读写分离吗？



### :point_right:MySQL cpu飙升，怎么处理？



### :point_right:数据表损坏如何修复？