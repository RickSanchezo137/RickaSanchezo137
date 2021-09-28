# [返回](/)

# JVM调参合集

## Java启动参数分类

1. 标准参数（-），所有的JVM实现都必须实现这些参数的功能，而且向后兼容；通过`java -help`查看
2. 非标准参数（-X），默认jvm实现这些参数的功能，但是并不保证所有jvm实现都满足，且不保证向后兼容；通过`java -X`查看
3. 非Stable参数（-XX），此类参数各jvm实现有所不同，将来可能随时取消，需慎重使用；通过`java -XX:+PrintFlagsInitial`查看

## 参数显示

显示JVM启动默认参数：`-XX:+PrintCommandLineFlags`

运行中显示进程的某个参数：`jinfo -flag xx(指令) pid`

显示某个-XX参数：`java -XX:+PrintFlagsInitial`，默认值；`java -XX:+PrintFlagsFinal`，最终值

## 执行引擎

### 执行模式

- `-Xint`：完全采用解释器
- `-Xcomp`：完全采用JIT，出现问题时解释器才介入
- `-Xmixed`：两者混合，默认是这个

## 运行时数据区

### 堆栈

#### 栈大小

- `-Xss<size>`(不加k、m等的话，就默认以字节为单位)

#### 堆大小

- 堆初始大小：`-Xms<size>`，默认为PC的$\frac{1}{64}$​​​

- 堆最大大小：`-Xmx<size>`，默认为PC的$\frac{1}{4}$​​

> 设置的是新生代+老年代的大小
>
> -X，非标准参数；m，memory；s，start；x，max

#### 堆分配

- `-XX:NewRatio`=\<size>，即$\frac{老年代空间}{新生代空间}=x$​​​，默认为2
- `-XX:SurvivorRatio`=\<size>，即 $Eden空间:Survivor0空间:Survivor1空间=x:1:1$，默认为8
- Survivor区垃圾的最大年龄：`-XX:MaxTenuringThreshold`=\<age>
- 新生代最大大小：`-Xmn<size>`
- 是否开启TLAB：`-XX:+/-UseTLAB`
- TLAB占Eden的百分比：`-XX:TLABWasteTargetPercent=x`
- 开启标量替换：`-XX:+EliminateAllocations`，默认开启

### 方法区

#### JDK7及以前：PermGen

- 永久代初始分配空间：`-XX:PermSize`=\<size>，默认20.75m

- 永久代最大可分配空间：`-XX:MaxPermSize`=\<size>，32位机器默认64m，64位默认82m

#### JDK8及以后：Metaspace

- 元空间初始分配空间：`-XX:MetaspaceSize`=\<size>，默认值依赖于平台，约21m

  > - 对于64位的服务端JVM来说，其默认值又看作是初始的高水位线，一旦触及，便触发Full GC并卸载没用的类（即这些类对应的类加载器不再存活），然后这个高水位线将会重置，新的值取决于GC时释放了多少元空间。如果释放的空间不足，在低于MaxMetaspace的基础上适当提高该值；如果释放过多，则适当降低该值
  >
  > - 如果这个值设置过低，容易频繁调整高水位线，频繁FGC，所以建议设高一些

- 元空间最大可分配空间：`-XX:MaxMetaspaceSize`=\<size>，默认-1，即无限制，取决于本地内存

#### 运行时常量池

- 设置字符串常量池大小：`-XX:StringTableSize <size>`

  > JDK 6默认是1009；JDK 7、8默认60013，JDK 8中，1009是可设置的最小长度

### 对象内存

#### 指针压缩

- `-XX:+/-UseCompressedOops`

> OOP：Ordinary Object Pointer，普通对象指针，指向对象的指针，当然包括类对象
>
> 专门针对klass pointer的：`-XX:+UseCompressedClassPointers`

## JUC

### synchronized

#### 偏向锁

```shell
输入：java -XX:+PrintFlagsFinal -version | findstr(Linux shell下是grep指令) BiasedLocking
     intx BiasedLockingBulkRebiasThreshold         = 20                                        {product} {default}
     intx BiasedLockingBulkRevokeThreshold         = 40                                        {product} {default}
     intx BiasedLockingDecayTime                   = 25000                                     {product} {default}
     intx BiasedLockingStartupDelay                = 0                                         {product} {default}
     bool UseBiasedLocking                         = true                                      {product} {default}
java version "13.0.2" 2020-01-14
Java(TM) SE Runtime Environment (build 13.0.2+8)
Java HotSpot(TM) 64-Bit Server VM (build 13.0.2+8, mixed mode, sharing)
```

- 偏向锁开关：`-XX:+/-UseBiasedLocking`

- 关闭偏向锁延迟：`-XX:BiasedLockingStartupDelay=0`

  > *JDK10之后默认为此，之前是-XX:BiasedLockingStartupDelay=4000，延迟4秒*

#### 锁消除

- 锁消除需满足下面几个条件

`-server `

`-XX:+DoEscapeAnalysis`(JDK6u23之后默认开启) 

`-XX:+EliminateLocks`

其中+DoEscapeAnalysis表示开启逃逸分析，+EliminateLocks表示锁消除

## GC

### 小工具

- 打印GC简要信息：`-XX:+PrintGC`
- 打印gc细节：`-XX:+PrintGCDetails`
- 运行中显示进程的GC信息：`jstat -gc pid`

### 垃圾收集器

#### Serial

- 可以使用`-XX:+UseSerialGC`，令新生代采用Serial，老年代采用Serial Old

#### ParNew

- 可以使用`-XX:+UseParNewGC`，令新生代采用ParNew，不影响老年代
- 可以使用`-XX:ParallelThreads`，限制线程数量，默认为cpu数

#### Parallel

- 可以使用`-XX:+UseParallelGC`，令年轻代使用Parallel Scavenge

- 可以使用`-XX:+UseParallelOldGC`，令老年代代使用Parallel Old

  > 默认开启一个，另一个也激活

- 可以使用`-XX:ParallelThreads`，限制线程数量，8以下，默认为cpu数；8以上，为3+5*cpu/8

- STW时间：`-XX:MaxGCPauseMillis=<n ms>`

  > 为了尽可能满足该参数的需求，会自适应调正堆大小以及其他一些参数
  >
  > 该参数使用需谨慎

- GC时间的比例：`-XX:GCTimeRatio`，取值范围0-100，默认99，表示用户线程时间：GC时间=99：1

  > 与上面的参数具有一定矛盾性

- 自适应策略：`-XX:+UseAdaptiveSizePolicy`

  > 堆中各个部分的大小等参数自适应调正，以达到堆大小、吞吐量和STW时间的平衡点。手动调优比较困难的场合，可以仅指定最大堆、STW时间、GC时间的比例，并采用这一参数，让虚拟机自适应调整

#### CMS

- `-XX:+UseConcMarkSweepGC`，手动使用CMS

  > 会自动打开`-XX:+UseParNewGC`，最终是ParNew+CMS+Serial Old的组合

- `-XX:CMSInitiatingOccupanyFraction`，设置堆内存老年代使用率的阈值，超过这个阈值开始CMS GC

  > JDK5及之前默认为68%，之后为92%。如果内存增长率低，可以将阈值设高一点，避免触发CMS，减少老年代GC可以改善性能；反之可以设低一点，避免频繁进入Serial Old GC。由此，根据这个参数可以有效降低FGC的次数

- `-XX:+UseCMSCompactAtFullCollection`，FGC之后对内存空间进行压缩整理

- `-XX:CMSFullGCsBeforeCompaction`，执行多少次FGC后进行压缩整理

- `-XX:ParallelCMSThreads`，设置CMS的线程数量

  > 默认为(ParallelGCThreads+3)/4，ParallelGCThreads是年轻代并行收集器的线程数
  >
  > 当cpu资源紧张时，在并发清除阶段，应用程序的性能可能比较糟糕

#### G1

- `-XX:+UseG1GC`：手动指定使用G1GC
- `-XX:G1HeapRegionSize`：设置每个region的大小，为2的幂，范围1MB—32MB，目标是根据最小的java堆内存大小划分出2048个区域。默认是堆内存的1/2000
- `-XX:MaxGCPauseMillis`：期望达到的最大停顿时间阈值，默认为200ms
- `-XX:ParallelGCThreads`：STW时并行的GC线程数，默认为8
- `-XX:ConcGCThreads`：并发标记时GC线程数，可以设置为ParallelGCThreads的1/4左右
- `-XX:InitiatingHeapOccupanyPercent`：超过该堆内存占用率值就触发GC，默认为45



# 参考

