# [返回](/)

# 八股文背诵版之—JUC篇

## 线程进程基础知识

### :point_right:**并发编程三要素？**

原子性、有序性和可见性。原子性是指一个或多个操作是不可割裂的整体，要么全部成功、要么全部失败；有序性是指某一段程序按顺序依次执行，不会被打乱顺序；可见性是指某线程对共享变量的修改，在其他线程中也是可见的

### :point_right:**进程、线程的区别？**

- 进程是系统资源分配的基本单位、线程是cpu调度和任务执行的基本单位，一个进程可以包含有多个线程
- 每一个进程有自己独立的代码和数据空间，切换时开销较大，而同一进程下的多个线程共享进程的地址空间和资源，有自己独立的PC计数器和运行栈，切换开销较小
- 多个进程之间互不影响；而一个线程挂掉会导致所在的整个进程崩溃

### :point_right:**Java线程的几种状态？**

创建（NEW）、就绪（RUNNABLE）、运行（RUNNING）、等待（WAITING）、定时等待（TIMED_WAITING）、阻塞（BLOCK）、终止（TERMINATED）

- 创建：分配资源并调用start()启动后进入就绪态
- 就绪：分配到CPU后进入运行态
- 运行：
  - 遇到自己不持有的锁时或IO阻塞时，进入阻塞状态
  - 线程持有锁的情况下用锁对象调用wait()，进入等待状态；或调用Thread对象.join()，自身进入等待；或LockSupport的park
  - 调用带时间参数的wait(T)、sleep(T)、join(T)、parkNanos(T)，进入定时等待状态
  - 异常中断或运行完成，进入终止状态
- 阻塞、等待、定时等待：出来之后会进入就绪态

> 挂起：OS三级调度中的中级调度，也就是内存调度，通过交换技术把暂时不能运行的进程调出内存，有就绪挂起和阻塞挂起，提高内存利用率

### :point_right:**sleep、wait、join区别？**

- 从方法定义来说：sleep、join是Thread类的方法、wait是Object类的方法
- 从方法调用来说：sleep由Thread类调用、join由thread对象来调用，它们没有调用地点的限制、wait由锁对象来调用，且必须调用在同步块中
- 从是否释放cpu来说：sleep、wait都是调用这个方法处的线程暂停执行，自然会释放cpu，join底层是wait实现的，自然也会释放
- 从是否释放锁资源来说：同步块中的sleep和join不会释放同步块对应的锁资源，wait会释放*（join是synchronized方法，释放的是调用者t）*

> t.join()底层由wait实现，不是native方法，目的是令调用这个方法的线程等待t线程执行完成，牢记

> wait会令对象锁膨胀成重量级锁，因为它是与ObjectMonitor绑定的

### :point_right:**死锁是什么？怎么形成的？如何避免或预防？如何检测或杀死？**

多个线程同时抢占一个共享资源，每个线程持有资源的一部分且不释放，同时去请求其他部分的资源，造成线程之间互相卡死的现象，叫做死锁

形成需要四个条件，一是互斥，即多线程互斥访问资源，二是不可抢占，即线程不能抢占其他线程的资源，三是请求与保持，即线程因请求资源而阻塞时，对于自己已经请求到的资源不释放，四是循环等待，即每个线程循环等待其他线程持有的资源

避免的方式只需要破环其中一种。互斥是不可破坏的，否则会有并发安全问题；不可抢占可以破坏，即线程抢占不到其他资源时会主动释放自己持有的；请求与保持可破坏，即一次性申请所有资源；循环等待可靠顺序资源申请破坏，每个线程只能申请编号大于自己持有资源的资源

预防可以靠合理的调度算法，比如银行家算法

检测可以靠死锁定理

杀死可以靠资源剥夺、线程回退、杀死线程、重启系统

> 避免是靠破坏死锁可能产生的条件，死锁便不可能出现；预防是在可能出现死锁的情况下，采取合理的策略

### :point_right:**生产者消费者实现？**

可从synchronized-wait/notify、Condition-await/signal、阻塞队列、LockSupport-park/unpark这几种去实现

> 使用wait时要注意虚伪唤醒问题，用while包裹住而不是if，否则下次就不会进行条件判断，而是直接执行wait后面的代码

### :point_right:**什么是上下文切换？**

一般来说，运行的线程数大于cpu个数，而同一时刻一个cpu中只能执行一个线程，为了让多个线程都能够得到有效地执行，一般采用时间片轮转的方式，也就是每个线程执行一个时间片后让出cpu给其他线程，这个过程涉及到线程执行状态的保存以及拿到cpu之后的恢复执行，这样的过程就叫做上下文切换

### :point_right:**实现一下三个线程分别打印A、B、C、最后有序打印出五个ABC？**

自己不应该打印时就wait，释放锁，唤醒其他所有；该打印时就持有锁打印

```java
class Testt {
    static int counter = 0;
    static int sum = 0;
    static char[] abc = {'A', 'B', 'C'};
    static Object lock = new Object();
    public static void main( String [] args ) {
        Thread A = new Thread(new Task(0));
        Thread B = new Thread(new Task(1));
        Thread C = new Thread(new Task(2));
        A.start();
        B.start();
        C.start();
    }
    static class Task implements Runnable{
        int target;
        public Task(int target) {
            this.target = target;
        }
        @Override
        public void run() {
            for (int i = 0; i < 5; i++) {
                synchronized (lock){
                    while (counter != target){
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println(abc[counter]);
                    counter = (counter + 1) % 3;
                    ++sum;
                    lock.notifyAll();
                    try {
                        if (sum == 15){
                            return;
                        }
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

> 还有Condition（不用唤醒全部）、Semophore等写法，后面练练

### :point_right:Java中interrupt、interrupted、isInterrupted方法的区别？

- interrupt：Java中建议主动处理中断而不是外界强行中断，这个方法用来设置中断标志位
- interrupted：如果中断了则返回true，并将中断标志位reset
- isInterrupted：返回是否中断

### :point_right:手动创建线程？

继承Thread、实现Runnable、实现Callable、线程池

## 并发模型及规则

### :point_right:讲一讲JMM？

JMM即为Java Memory Model，是Java规范定义的一种抽象的内存模型，用于屏蔽底层硬件和操作系统在内存访问具体方式上的差异性，使得Java程序在不同的平台都能实现一致的内存访问效果

具体在于，Java的各种变量都存放在主内存中，而各个线程有自己的工作内存，线程对于变量的读取、修改等操作必须在工作内存中进行，每次读取时需要从主内存拷贝到工作内存当中，每次写入需要将写入后的结果刷新到主内存中；线程的工作内存之间是不可见的，只能通过主内存来实现变量值的传递

### :point_right:什么是as-if-serial和happens-before？

as-if-serial是Java规范定义的一个规则，即在单线程程序中，程序最终执行的结果始终与程序按指令顺序执行的结果保持一致。具体一点，以程序员视角来说，在单线程程序中可以放心地将程序结果看作是顺序执行的结果，可以忽略底层的实现；而面向底层来说，程序并不一定按指令顺序执行，但无论编译器或处理器怎样进行重排序，都不能改变程序的执行结果

happens-before同样是Java规范定义的规则，即满足happens-before规则的多线程程序，不需要额外的同步措施，也不会出现并发安全问题。具体一点，以程序员的视角来说，按照happens-before规则编写的多线程程序，可以放心地认为是没有并发安全问题的，而可以忽略底层具体的实现；而面向底层来说，不限制编译器和处理器重排序的具体方式，但无论是怎样实现的，必须确保符合happens-before的程序不出现并发安全问题，不能做改变执行结果的重排序*（通过内存屏障实现）*

## volatile

### :point_right:volatile的作用？

> 作用是什么→具体→怎么实现→缺陷

主要有两个作用，一是保证可见性，二是保证一定的有序性

可见性是指被volatile关键字修饰的变量，多个线程在对它进行读取、修改等操作时，不会出现在多个线程中、或主内存和工作内存中不一致的情况。具体是线程在写入volatile变量后，会令其他cpu的线程工作内存中的volatile变量副本失效，并将自己的从工作内存强制刷新进主内存

有序性是指volatile和它前后的读写操作之间的有序性。具体是volatile变量禁止了volatile前后的某些读写操作与volatile变量之间进行重排序

使用javap恢复成字节码指令后可以发现变量带了ACC_VOLATILE的标志，Java规范中规定需要为这一行前后插入内存屏障。内存屏障相当于一个抽象的语义，可以有各种不同的实现，而hotspot实际是使用了lock add，add对应一个空操作，真正起作用的是lock指令，它能够起到内存屏障的效果，使其他cpu中对应的缓存行失效，并将自己缓存行的值刷新到主内存中，同时保证前后某些指令无法越过lock这一指令

缺陷在于无法保证原子性，因为对volatile变量的修改不是原子操作，是由多个指令构成的，多个线程同时修改时可能会覆盖操作，造成结果的错误

> lock的缓存一致性是借助MESI协议实现，它是一个黑盒协议，不同种类CPU有自己相应硬件层面上的设计实现；而将自己缓存行的值刷新到主存、防止重排序是lock自己额外的功能

> MESI：保证不同工作内存中同一位置缓存行内变量值的一致性

> *lock锁缓存，于是线程可以独占cpu缓存行进行操作，那为什么还会出现原子性问题？关键在于缓存上面还有一层寄存器，寄存器存储中间值进行运算，即已经把缓存的值取走了，即使后面缓存失效也影响不到寄存器已经拿走的值，造成了线程不安全问题*
>
> [volatile为什么不能保证原子性？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/329746124)

### :point_right:知道伪共享问题吗？

知道。工作内存的读取刷新等等操作，也就是缓存的一系列操作，是以缓存行为单位的。以64字节的缓存行为例，如果两个用volatile修饰的变量在一个缓存行内，两个线程分别修改这两个变量，根据MESI协议，其中一个线程完成某一变量的修改之后，另一个线程的对应缓存行会失效，因此另一个线程对另一个变量的修改也需要重新从主存读取，明明是不相干的两个变量，只因为同在一个缓存行，加大了时间消耗

解决方法是不使用volatile修饰，或者字节填充，让两个变量位于两个不同缓存行，就可以各自在自己的工作内存中修改，互不影响

## CAS

### :point_right:CAS是什么？

CAS是一个cpu并发原语，意思是Compare And Swap，比较并交换。概念上来说有三个操作数，A、B和V，V是指内存地址，A是V处对应的期望的原数据，B为要更新成的数据，CAS操作就是如果对V处的数据进行修改更新，首先要判断V处的数据是否与期望的A值相同，如果不同，则修改失败，如果相同，才能够成功更新成B。它的JVM源码里对应的是Atomic::cmpxchg方法，对应的底层指令是cmpxchg，多处理器架构下会多一个lock前缀。CAS的作用主要是用于实现乐观锁，因为悲观锁往往需要阻塞其他线程，就意味着要进行多次上下文切换，乐观锁可以节省这部分上下文切换的开销，一定程度上提高效率

### :point_right:CAS有什么不足？

有三个问题，一是CAS往往用在自旋锁当中实现并发安全，CAS失败会不断循环，导致cpu资源消耗；二是CAS只能针对单个变量；三是可能会出现ABA问题，就是比如三个线程，第一个将A改成C，正要CAS之前线程阻塞了，第二个将A改成了B，第三个将B改成了A，此时第一个线程重新得到CPU，会认为这期间没有作出任何修改，成功修改成C。解决方法就是给变量添加一个版本号，比较值的同时也需要计算版本号是否正确

## synchronized

### :point_right:synchronized作用​？

synchronized是Java的关键字，可以用于修饰代码块，方法，用于保证原子性、可见性和有序性

原子性在于被synchronized包装的代码块、方法是线程安全的，因为只有成功获取monitor的线程才可以运行，其他线程都无法进入同步代码块或同步方法，单线程下的原子性是可以保证的

可见性在于Java规范规定某线程进入synchronized前需要令其他线程对应的共享变量失效，出来时需要令共享变量刷到主存，具体实现是通过给synchronized对应的字节码指令monitorenter和moniterexit处插入内存屏障实现的

> 个人猜测待实证（汇编）：可见性具体实现可能是因为monitorenter和moniterexit在JVM源码中用到了底层的lock cmpxchg指令，而lock具有保证可见性和内存屏障的作用，因此是可见的

有序性在于monitorenter和monitorexit处插入的内存屏障，防止synchronized内部指令与外部指令重排序，同时as-if-serial保证了synchronized内部不会发生影响结果的重排序，可以看作是有序执行的

还有一点，一个monitorenter对应两个monitorexit，第二个monitorexit是对应方法异常退出后的解锁

### :point_right:写一个单例？除了你的这种写法，还有其他的吗？

> 先从简单懒汉式写起，面试官十有八九会问DCL

DCL：

```java
class Singleton{
    private static volatile Singleton instance = null;
    private Singleton(){}
    public static Singleton newInstance(){
        if (instance == null){
            synchronized (Singleton.class){
                if (instance == null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

几个问题：

1. 为什么要synchronized？

   不加synchronized会出现在new之前，多个线程判断为null，并创建了多个对象的情况

2. 为什么要两个if？

   第二个if自然是因为如果判断为null，就要创建；第一个是为了提高效率，如果没有的话，即使对象被创建好了，每个时间只有一个线程能成功返回instance，而我们期望的肯定是只要创建好了，多个线程同时拿到instance也是可以的

3. 为什么要加volatile？

   针对线程对象的创建和引用的赋值，我们期望是这样的顺序，就是在对象完全创建之后，再将地址值传递给引用，然而创建对象不止一条指令，如果其中某条指令与给引用赋值这个操作发生了重排序，就会出现在对象还未完全创建好的时候，就将地址值传递给了引用，此时引用指向的是一个不完整的对象，但也不是null，直接就这样返回的话，可能出现问题

   volatile禁止了对象完成创建和给引用赋值这两个操作之间的重排序，保证了对象创建完成之后才能将地址赋给引用，避免上述情况发生

其他写法：饿汉式、枚举、静态内部类（利用其懒加载特性）等等

> ```java
> class SingletonHolder{
>     static class Singleton{
>         private static Singleton instance = new Singleton();
>         private Singleton(){}
>         public static Singleton newInstance(){
>             return instance;
>         }
>     }
> }
> ```

### :point_right:synchronized保证可见性吗​？

保证。首先，JMM规范中要求synchronized保证可见性，因此，在hotspot的实现中，monitorenter和monitorexit指令插入了内存屏障，可以保证可见性

### :point_right:synchronized在代码块和方法中的区别？

代码块必须显式地将锁对象包裹在synchronized后面的括号中，方法会将this作为锁对象。代码块中看javap指令可以看到javac编译成了monitorenter和monitorexit两个字节码指令，对于方法会添加一个ACC_SYNCHRONIZED标志位，加锁都是差不多的流程

### :point_right:锁升级？

以前的synchronized是重量级锁，涉及到各种上下文切换等，开销比较大，是比较“重”的锁。后面对synchronized进行了改进，有一个从偏向锁升级成轻量级锁升级成重量级锁的过程，大大提高了synchronized的效率

具体细节是这样的，在Java中每一个对象都可以作为锁对象，结合synchronized将代码块或方法锁住。Java的每个对象都有一个对象头，决定是否为锁对象的关键就在于对象头中的markword

1. 普通对象的在没有调用native的hashcode方法之前，markword由分代年龄和后三位的锁状态信息构成。如果在 ①没有开启偏向锁；②开启了偏向锁UseBiasedLocking同时开启了偏向锁启动延迟BiasedLockingStartupDelay，但还没有超出延迟时间；③调用了native的hashcode方法，这三种情况下，即为不可偏向的无锁态，后三位为001；上述三种情况之外，为匿名偏向状态，后三位为101

2. 普通对象结合synchronized使用，变成锁对象。此时线程想要进入synchronized，会首先判断锁状态，如果是无锁不可偏向状态，则直接升级成轻量级锁；如果是匿名偏向状态，则线程通过CAS将指向自己的指针贴到markword中，并执行同步块内的代码；如果是已偏向状态，线程检查自己的线程ID和锁对象的markword中线程ID是否一致，如果是一致则表明发生了重入则继续执行，如果不一致表明发生了线程竞争，需要等到安全点时，从偏向锁状态撤销到无锁状态，并升级成轻量级锁

   升级成轻量级锁的过程是这样的，首先持有锁的线程在自己的栈帧中创建一个结构叫做lock record，具有两个成员属性，一个是displaced mark word，一个是owner指针。首先将锁对象中的mark word拷贝到lock record作为displaced mark word，接着尝试通过CAS改变锁对象中的mark word，需要贴入一个指向自己栈帧中的lock record的指针，接着将owner指向锁对象中的mark word，成功则表明占有锁，修改锁标志位后两位为00，不成功则进入自旋*（JDK 7之前可以通过PreSpinLock参数调整自旋次数阈值，后面就是JVM自适应自旋，会根据时间局部性原理，认为刚刚成功进行自旋并完成CAS的很有可能再次成功，从而增加要进入锁对象的线程的自旋次数阈值）*。如果是重入的话，则会在线程中添加一个displaced mark word为null，owner依旧指向锁对象的mark word，根据lock record的数量来记录重入次数

3. 自旋超过一定次数，说明竞争比较强烈，则升级成重量级锁，线程会创建一个ObjectMonitor对象，其中的\_header字段赋值为displaced mark word，_owner字段为拥有锁的线程*（lock record→thread）*，\_obj字段指向锁对象，\_recursion表示重入次数，原锁对象的mark word中的锁状态改为10，且贴上指向ObjectMonitor对象的指针。其他线程想要进入同步块时，发现锁被另外的线程持有了，则会被封装成一个ObjectWaiter对象，插入到\_cxq队列队首，并调用park函数阻塞线程，底层是靠mutex互斥锁；如果线程调用wait，则会插入到\_waitSet中；被唤醒或解锁后会从cxq队列中移动到\_EntryList，并将原有队列的队尾线程作为唤醒候选。如果是重入的话，则会用\_recursions记录重入次数

> 批量重偏向/撤销及epoch，后续跟进

### :point_right:锁消除和锁粗化？

锁消除是指JIT经过逃逸分析发现synchronized包裹的段不会发生数据的逃逸，即不会被其他线程访问到，则会消除这个锁从而获得更好的效率；锁粗化是指JIT编译优化对连续的使用相同锁对象的synchronized段，扩大其范围，合并成一个更大的锁

### :point_right:synchronized可重入吗？

可以重入。在偏向锁情况下，mark word里是线程ID，一个线程重入的话对比线程ID相同直接重入；在轻量级锁情况下，mark word里面指向持有锁线程的栈帧中的lock record，重入一次就在栈帧中添加一个displaced mark word为null的lock record；在重量级锁情况下，mark word指向ObjectMonitor对象，靠其中的_recursions记录重入次数

## AQS

### :point_right:介绍一下AQS？

> 是什么？什么用？技术细节及流程？缺陷？

> STAR法则：
>
> - ① situation：情景，为什么要这样？场景是什么？
> - ② task：任务，要完成的任务是什么？
> - ③ action：行动，为了完成任务，是怎么行动的？
> - ④ result：实现的结果如何

① AQS的全称是AbstractQueuedSynchronizer，它是一个抽象类，是一个用于构建锁、各种同步器等工具类的框架类，通过继承他能够方便地编写出高效率、线程安全的线程同步工具类

② 它的任务是保证持有锁的线程运行时是线程安全的，同时合理令竞争锁的线程进行自旋、阻塞，实现高效

③ AQS有一套阻塞并唤醒线程以及分配锁的机制，底层主要是靠一个由Node节点构成的双向队列组成的

- Node是AQS的静态内部类，它用于封装线程并构成双向队列，Node中有这样几个比较重要的构成：thread引用指向它所包装的线程；prev和next属性指向前后节点，int型变量waitStatus表达自己所包装线程的状态，有CANCELED=1，SIGNAL=-1，CONDITION=-2，PROPAGATE=-3；用引用SHARED=new Object()表达共享模式，对应CountDownLatch、读锁、CyclicBarrier等等，EXCLUSIVE=null表达独占模式，对应有ReentrantLock、写锁等等

- 对于AQS本身来说，有这样几个重要的构成：Node head引用指向双向队列的头节点，这是一个哨兵节点，不包装任何线程；Node tail指向尾节点；int state是最为关键的共享资源变量，由AQS扩展的工具类都要靠state来决定线程持有锁的状态，比如在ReentrantLock中，线程独占锁之后令state+1，重入也+1，只有等原线程释放锁并令state减为0，其他线程才有机会占有锁，又比如在使用CountDownLatch的时候，初始化setState令state为一个大于0的值，只有其他线程countDown令state重新等于0，原调用await的线程才能被唤醒，所以说我认为这是最关键的一个变量

  AQS中比较常见且重要的方法，我认为是acquire/release以及以他们为前缀的包括acquireShared/releaseShared等方法，因为AQS的子类工具类的主要功能加解锁基本都是靠调用这几个方法来实现，它们都是final修饰的是不能重写的，也就表明继承AQS的工具类只能根据这一套逻辑来处理

  **以acquire方法为例**，这是针对独占锁的方法

  - 首先是在if逻辑`if(!tryAcquire(arg)&&acquireQueued(addWaiter(Node.Exclusive), arg))`里面尝试调用tryAcquire方法，这是一个钩子方法，本身的实现是抛出UnsupportedOperationException，子类可以通过重写它构建自己获取锁的逻辑，以ReentrantLock为例，就是重写了tryAcquire方法，如果是非公平锁，就尝试令线程通过CAS让state变成1，并设置AQS的父类AbstractOwnableSynchronizer的exclusiveOwnerThread为当前线程，用于重入时候的判断

  - 如果tryAcquire拿锁失败，就会进入if逻辑中的第二个方法`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`

    - addWaiter的主要作用是创建一个新Node，令其thread字段指向当前线程，并令nextWaiter为null，同时通过CAS将这个Node接到上面提到的双向队列的队尾，如果队列不存在则创建一个，最后将这个Node返回

    - acquireQueued：*两个if*

      1. 主要作用是对刚刚返回的Node中的线程进行阻塞，首先会令一个临时变量interrupted = false，接着进入一个死循环，首先进入第一个if逻辑`if(p==head&&tryAcquire(arg))`判断Node的前驱节点p是否为head，如果是的话，则尝试一次CAS拿锁，拿锁成功的话则令这个Node成为新的head，令其中的thread指向null，并令acquireQueued方法返回interrupted变量，此时为false
      2. 如果前驱不为头节点或拿锁失败的话则进入第二个if逻辑`if(shouldParkAfterFailedAcquire(p, node))`，顾名思义，就是是否应该在加锁失败之后阻塞线程，进入方法可以看到，首先需要判断前驱节点的waitStatus，如果为SIGNAL即-1，返回true；如果>0表明前驱是CANCELED，则将所有前面CANCELED的节点全部从队列中移除，最后返回false；其他情况下，将前驱节点的waitStatus置为SIGNAL并返回false。对于刚刚返回true的结果，进入parkAndCheckInterrupt方法使用LockSupport.park来进行阻塞；对于返回false的结果，也就是前驱节点状态不为SIGNAL的结果，再进行一次判断前驱节点是否为head并CAS拿锁的操作，如果不满足条件，则再次进入shouldParkAfterFailedAcquire，这时由于上一轮已经把前驱节点的状态设置为SIGNAL了，就一定会进入parkAndCheckInterrupt用park阻塞自己，也就是说，在前驱节点为SIGNAL的情况下会尝试拿锁一次，不是SIGNAL的情况下会尝试拿锁两次

      此外，LockSupport.park时，如果线程的中断标志位为true，则不会阻塞而直接执行后面的代码，在parkAndCheckInterrupt中会直接返回Thread.interrupted，也就是返回true并令中断位恢复。返回true后，根据`interrupted |= parkAndCheckInterrupt()`，刚刚提到的interrupted局部变量就会一直为true，但此时还不能进行处理，因为线程还没能拿到锁，等到线程拿到锁执行后acquireQueued就会返回true，调用selfInterrupt()方法令自己的中断位为true，并在自己的逻辑中处理中断。总结起来，也就是线程在拿锁过程中被申请中断了，也可以正确执行AQS的逻辑进行阻塞*（需要通过局部变量记录中断位并令中断位为false）*，**否则如果不令中断位为false，park就对这个线程一直起不到阻塞作用了，它会一直循环拿锁**

  **接着是release方法**

  - 这个方法比较简单，首先会去调用tryRelease，以ReentrantLock重写的为例，会直接修改state的值减一*（不需要CAS，因为已经持有锁了）*，如果state不等于0的时候返回false，等于0的时候返回true
  - 返回true之后，判断head不为空且waitStatus不为0，我们知道在加锁的时候已经被改成了SIGNAL，所以成功的话，进入unparkSuccessor方法进行线程唤醒，也就是，靠头节点唤醒它后面的节点中的线程
    - 在unparkSuccessor中，首先通过CAS尝试将head的waitStatus改成0，这一步允许失败
    - 接着判断其后继节点，如果为null或者waitStatus大于0，则从队列尾部向前找到最接近队首的waitStatus小于0的节点并唤醒它，不为null且waitStatus不大于0则直接唤醒它；此时卡在LockSupport.park处的线程苏醒，由于没有中断，返回false，于是在外部的`if(!tryAcquire(arg)&&acquireQueued(addWaiter(Node.Exclusive), arg))`逻辑中，也就不会调用selfInterrupt方法了

> propagate：共享锁具有传播性，当若干个线程等待在队列中，有共享资源的时候，可能此时是出现了多个共享资源。在独占锁中只有持有锁的线程release的时候需要唤醒头节点后面的节点，获取锁时不需要；而在共享锁中当线程**获得**共享资源时，也要通过setHeadAndPropagate去唤醒后面的线程可以来拿共享资源了，避免等待太久，唤醒过程在老版本JUC中可能出现有的线程永远无法唤醒的bug，因此propagete状态的引入就是为了避免这个状态

> 读写锁：state高16位用作读锁*（>>>16后操作）*，共享模式；低16位用作写锁，独占模式

> 共享锁里面可以调用tryLock来非公平地拿锁，不过是sync.nonFairTryAcquire，只CAS获取一次，不会有后面acquireQueued那些操作

### :point_right:为什么要从后向前找线程解锁？

假设有这样一个场景，首先A线程拿到锁正常执行，此时B线程第一次tryAcquire失败，进入入队逻辑，我们都知道会再tryAcquire一次，如果是这样的流程：

1. A tryRelease，将state减为0，同时用一个局部变量h等于head
2. A卡顿
3. B tryAcquire成功，由于B节点已经入队，需要令对应的节点为新的head节点，并令之前的head.next指向null方便gc
4. A恢复，此时A的局部变量h指向的是原头节点，但它的next已经为null了，所以无法通过next来查找了，只能从后往前找

### :point_right:讲讲公平锁？

公平锁顾名思义就是为了实现线程按顺序“公平”的获取锁，具体实现在其他方面比如入队等和非公平锁基本一致，只有刚开始调用tryAcquire的时候，会在if逻辑中**首先**用hasQueuedPredecessors判断是否有前驱节点且其中的线程不是本线程，如果有的话，tryAcquire返回false，进入入队逻辑；没有的话，才能在if逻辑中走到CAS自旋，尝试拿锁

### :point_right:讲讲Condition？

Condition提供了一种精确唤醒的办法，以ReenTrantLock为例，调用newCondition会返回一个ConditionObject对象，每个ConditionObject对象中对应有一个条件队列，头节点是firstWaiter，尾节点是lastWaiter。某个Condition调用await时，会首先将自己封装成一个Node节点插入到对应Condition对象的条件队列当中。封装成节点时会首先判断自己有没有持有锁，即通过exclusiveThread来判断，不持有则抛出IllegalMonitorException异常；如果持有锁，则封装成节点并返回，然后挂起自己。此后，如果有其他线程调用了这个Condition对象的signal方法，则会从条件队列中移出，进入AQS队列，并且unpark刚刚所说的在条件队列中的线程，然后调用acquireQueued方法参与竞争state锁以及在AQS队列中阻塞的逻辑

> Condition实现轮流打印
>
> ```java
> import lombok.SneakyThrows;
> import java.util.concurrent.locks.*;
> class Testt {
>     public static void main(String[] args) {
>         ReentrantLock lock = new ReentrantLock();
>         Condition A = lock.newCondition(), B = lock.newCondition(), C = lock.newCondition();
>         new Thread(new Runnable() {
>             @SneakyThrows
>             @Override
>             public void run() {
>                 for (int i = 0; i < 5; i++) {
>                     lock.lock();
>                     A.await();
>                     System.out.println("A");
>                     B.signal();
>                     lock.unlock();
>                 }
>             }
>         }, "A").start();
>         new Thread(new Runnable() {
>             @SneakyThrows
>             @Override
>             public void run() {
>                 for (int i = 0; i < 5; i++) {
>                     lock.lock();
>                     B.await();
>                     System.out.println("B");
>                     C.signal();
>                     lock.unlock();
>                 }
>             }
>         }, "B").start();
>         new Thread(new Runnable() {
>             @SneakyThrows
>             @Override
>             public void run() {
>                 for (int i = 0; i < 5; i++) {
>                     lock.lock();
>                     C.await();
>                     System.out.println("C");
>                     A.signal();
>                     lock.unlock();
>                 }
> 
>             }
>         }, "c").start();
>         lock.lock();
>         A.signal();
>         lock.unlock();
>         while (Thread.activeCount() > 2){
>             Thread.yield();
>         }
>     }
> }
> ```

### :point_right:ReentrantLock和Condition怎么绑定的？

Condition是AQS的非静态内部类，当非静态内部类实例化时，会隐式持有外部类实例的引用this$0。ReentrantLock实例化时，实际还令内部的sync=new NonFairSync()，而Sync是AQS的子类，因此此时AQS对应的实例就是这个sync；接着调用newCondition方法，实际上是调用sync.newCondition，此时的Condition对象已经持有指向这个sync实例的引用this\$0了，因此后面在await方法中，调用的是自己的方法，而调用acquireQueued时，在自己内部找不到这个方法，于是调用的是这个sync对象的方法。也就是说，通过这个sync和ReentrantLock实现了绑定

## ConcurrentHashMap

### :point_right:介绍一下ConcurrentHashMap？

> 依旧是Situation→Task→Action→Result

ConcurrentHashMap是一个线程安全的Map，它的出现是为了解决在多线程下使用HashMap可能出现的线程不安全的情况，比如循环链表、数据丢失、数据覆盖等问题；同时保证一定的并发性，而不是像Hashtable那样都用synchronized包装，造成效率底下

**1.7**

ConcurrentHashMap的具体实现在JDK 1.7及1.8的版本中有所不同，在1.7中的实现是靠分段锁的思想，一个ConcurrentHashMap对象底层是一个Segment数组，Segment是ConcurrentHashMap的内部类，它继承了ReentrantLock，可以调用lock等方法进行加解锁。每个Segment内部有一个HashEntry数组table，HashEntry跟HashMap 1.7版本的Entry比较类似，是一个链表节点的结构，是具体存放键值对的数据结构。总体看下来，是一个数组+数组+链表的结构，每次某线程操作一个Segment的时候，会进行加锁，此时其他的线程就无法对这个Segment进行操作，但不影响对其他Segment的操作，这样，就保证一定并发性的基础下，也是线程安全的，具体的细节是这样的：

- 初始化：首先初始化的时候可以指定initialCapacity、loadFactory、concurrencyLevel，会首先初始化一个二次幂大小的Segment数组segments，且只初始化第一个Segment对象segments[0]以及其中的HashEntry数组，作为一个模板待后面使用，并且计算**segmentShift**和**segmentMask**，便于后面快速定位时使用

- put：*Segment位置的确认→调用segments[x]的put*

  接着以put方法为例，会根据hash方法计算哈希值，将计算到的哈希值右移segmentShift位，与segmentMask相与去确定选择哪一个Segment槽*（选择HashEntry槽位的时候根据hash直接计算，选择Segment槽位时右移hash计算，这样，就一个用的是低位信息，一个用的是高位信息：下面get方法中有例子）*，如果这个槽位的Segment为null，则先调用ensureSegment方法，先根据初始化时确定好的模板的一系列参数来生成对应的HashEntry数组，接着创建一个Segment并令table指向刚刚生成的HashEntry数组，同时通过CAS加自旋将这个Segment对象插入刚刚的Segment槽位，接着调用Segment对象的put方法

  在Segment对象的put方法中，首先尝试tryLock，也就是nonFairTryAcquire拿锁一次，如果获取到锁，就创建一个HashEntry节点并头插法插入槽位；如果失败，则会进入scanAndLockForPut方法中，首先直接用hash和table长度减一**直接相与**确定HashEntry槽位，接着初始化retries变量为-1，接着进入`while(!tryLock())`自旋：当retries<0时，有三种情况，首先会判断对应槽位是否为null，如果是就**预创建**一个HashEntry节点，令retries=0；如果不为null，则遍历链表，遍历到尾部的时候再预创建一个节点，令retries=0；如果途中遇见满足hashcode相同和equals的key，就令retries=0。上述三种情况之外都不会进入遍历链表的逻辑了，会不断自旋尝试获取锁，如果超过阈值就调用lock()方法，如果自旋过程中其他线程插入了节点，则重置retries并重新遍历链表。scanAndLockForPut获取到锁，用预创建的节点头插法插入槽位。如果put后，count+1大于threshold，进入扩容逻辑

- 扩容：遍历table，对于不为null的槽位，遍历链表并移到新数组上，用到了lastRun的方法*（具体看源码，很清晰）*，以lastRun获取到的是高位节点为例，lastRun及后面的节点直接移动到新数组对应位置，其他的节点通过头插法逐个插到新位置。这样就导致，lastRun及后面的节点顺序不变，前面的节点和原数组相比倒序了

- get：get操作两步走，先确定Segment索引，将hash右移后与上segmentMask，再确定HashEntry索引，将hash原封不动与上table.length-1

**1.8**

1.8的主要思想和1.7的版本不同，主要是靠cas和synchronized来实现的。底层实现是Node数组table，用hash值标识Node节点的状态，大于0为普通节点，MOVED=-1为正处于扩容过程中的节点，TREEBIN=-2为指向红黑树的节点，其中的first引用指向树，表明这个桶已经被树化了；sizeCtl可以用来记录各种信息，大于等于0表示存放的是初始容量或扩容阈值，等于-1表示正在初始化，<0且不等于-1表示正在进行扩容，高16位用于记录数组的容量，低16位用于记录参与扩容的线程数。需要对其中的某个桶进行修改操作时，会用synchronized将这个桶锁住，此外，如果在访问节点时发现节点为MOVED节点的时候，还可以参与数组的扩容操作，这是一个较高优先级的操作，通过transferIndex变量和计算出的stride确定每个线程的工作范围。具体的细节是这样的：

- 初始化：初始化的时候可以指定initialCapacity、loadFactory和concurrencyLevel，这里不会首先初始化Node数组，而是在put的时候进行延迟初始化。初始化的时候用sizeCtl来记录初始大小，无参构造方法中为0，其他构造方法中为二次幂

- put：*spread+自旋中的四个if+addCount*

  - 首先会计算hash值，这里调用的是spread方法，在进行哈希扰动后又与上了一个`HASH_BITS=0x7FFFFFFF`，令为正数，防止与其他类型节点混淆

  - 然后进入自旋

    - 首先判断table是否为null，如果是，则进入initTable方法，首先进入一个while自旋，如果判断sizeCtl<0表明此时有其他线程正在操作，则yield放弃cpu，否则通过cas将sizeCtl改成-1，表明自己正在进行初始化，初始化完成之后，令sizeCtl等于数组扩容阈值*（这里判断tab==null有个double check，针对有可能出现的ABA问题，防止重复初始化）*
    - 接着会判断对应桶是否为null，如果为null则通过cas给对应桶的位置创建一个节点并break跳出循环
    - 接着判断节点是否为forwarding节点，也就是哈希值是否为-1，如果是则先调用helpTransfer辅助扩容
    - 接着真正进行put操作，首先synchronized锁住节点，判断头节点哈希值是否大于等于0，如果是则进行普通节点的遍历并记录链表长度，遍历完过后通过尾插法插入新节点并break；如果找到重复key则根据onlyIfAbsent判断是否需要覆盖并break；如果哈希值为-2，表示为TreeBin节点，指向红黑树，就调用树节点的putTreeVal方法

  - 跳出自旋后，如果binCount大于阈值且节点总数目大于指定阈值，对这个桶进行树化，第一个节点不存值，是一个TreeBin节点，哈希值为-2，first指向真正的红黑树。接着调用addCount方法：*计算节点数→扩容*

    - 首先是计算Map的节点数，ConcurrentHashMap 1.8的节点数是靠volatile的baseCount变量和CounterCell[]数组来实现的，和LongAdder采用了完全相同的算法。首先判断CounterCell数组是否为null，为null的话就首先通过cas给baseCount加，如果失败的话则**根据线程哈希探针**找CounterCell数组中某个桶，如果桶不为空，通过cas令对应CounterCell节点的value加，否则进入fullAddCount方法自旋

      如果是数组或桶为null的情况，则利用cellsBusy变量结合cas作为锁，竞争锁并初始化数组或桶，一直竞争不到锁，则只能重新通过cas处理baseCount；如果是对CounterCell桶cas失败的情况，首先会尝试rehash一次，对应桶再次失败，说明竞争比较激烈，则尝试给CounterCell数组扩容

      最后，将baseCount和CounterCell数组中所有CounterCell对象的value相加即为最终结果

    - 接着判断节点数是否大于阈值，如果是则需要扩容。扩容会首先调用resizeStamp方法，具体是`Integer.numberOfLeadingZeros(n)|((1<<RESIZE_STAMP_BITS-1))`，接着通过cas将sizeCtl或上这个值后左移`RESIZE_STEMP_SHIFT`位，再加上2，这里的`RESIZE_STAMP_BITS`+`RESIZE_STEMP_SHIFT`=32，也就是说，扩容中的sizeCtl最高位一定是1，且高`RESIZE_STAMP_BITS`位可以用来记录numberOfLeadingZeros，可以通过这个值计算出数组容量，低`RESIZE_STEMP_SHIFT`位用于记录线程个数；后续的线程到来时，发现sizeCtl<0，就知道已经有线程正在扩容，于是通过cas给sizeCtl加一，进入辅助扩容

- 扩容：线程发现forwarding节点、发现sizeCtl<0但数组是已经初始化的等等情况，都会进入扩容流程，volatile的变量transferIndex减stride，就是自己负责迁移的范围，然后通过cas更新transferIndex。每处理完原数组的某个节点，都会将节点标记为forwarding节点，里面有个nextTable指针指向新table，这样后续的线程可以根据这个forwarding节点去参与辅助扩容或在新table中找数据。线程完成扩容后，会令sizeCtl减一，全部完成后才能更新table的指向，并且令sizeCtl为新table的扩容阈值

  链表的迁移在1.7版本上更进了一步，在遍历过程中记录hn high node和ln low node，并记录runBit和lastRun，runBit是最后一次发生变化的`哈希值&原table长度的结果`，首先是将lastRun之外的节点通过尾插法连成hn链表、ln链表，然后将lastRun接上对应的链表，最后将这hn链表放到新table的节点旧索引+旧长度的位置，ln放到新table旧索引的位置

  ![1](imgs\JUC\1.png)

- get：get的时候，如果碰到forwarding节点，就会调用Node的find方法，实际上调用的是它的子类ForwardingNode的find方法

### :point_right:ConcurrentHashMap和Hashtable的区别？

HashTable是同步容器，用synchronized修饰所有方法，每个操作都只能有一个线程在进行，效率不高；ConcurrentHashMap是并发容器，在1.7版本中利用了分段锁的思想，在1.8版本中摈弃了分段锁，采用synchronized只锁住一个桶的思想，并通过其他巧妙的设计，在保证了线程安全的情况下，保持了并发性，大大提高了效率

### :point_right:ConcurrentHashMap的size怎么计算的？

ConcurrentHashMap 1.7是通过在自选中遍历segments数组并统计每个segment的count之和来实现的，会先不加锁的计算三遍，如果其中连续两次结果一致，就直接break并返回，否则超过了retry次数阈值，会用lock去给每个segment加锁，再计算一次；在1.8中是调用sumCount()方法，也就是baseCount加上CounterCell[]数组的所有非空对象的value之和

### :point_right:1.7和1.8的区别？

锁的思想；是否有树化；哈希值的作用；头插法尾插法等

## CopyOnWriteArrayList

### :point_right:简单介绍一下CopyOnWriteArrayList？

CopyOnWriteArrayList，顾名思义，是“写时复制”列表，它的出现是为了解决传统列表在多线程情况下可能出现的线程不安全问题，实现一个并发安全的容器。它的底层是一个Object数组array，用于存放列表的元素，读取元素的方法不做任何加锁操作，可以多线程同时读取；而修改或添加操作会用synchronized+lock变量将方法内代码块锁住并且先对原始数组进行拷贝，并在拷贝的数组上进行操作，拷贝完成之后再调用setArray令array指向这个拷贝的数组。这种方式避免了多线程进行修改时的线程不安全情况，但弊端在于会产生大量的拷贝的新数组，占用内存空间，可能频繁触发GC，加上操作用synchronized锁只能有一个线程进行操作，同样也降低了效率

## ThreadLocal

### :point_right:了解ThreadLocal吗？简单介绍一下应用场景

ThreadLocal是一个多线程变量副本工具类，对于同一个变量的操作，每个线程在内部有一个副本，线程对这个变量的操作相当于是对线程内部的变量进行操作，与其他线程隔离开来，不会出现线程安全问题，是加锁之外的一种线程安全机制。具体的实现原理是在每个线程对象内部有个ThreadLocalMap的成员变量threadLocals，它是一个map的结构，底层是Entry数组table，每次对ThreadLocal变量进行set操作的时候，就以WeakReference包装ThreadLocal变量，即令其中的referent引用指向这个变量，并将这一个WeakReference作为key，要set的参数作为value，封装成一个Entry插入到map当中，如果出现hash冲突则通过线性再散列方法处理；当get的时候，首先从对应线程中拿到对应的threadLocals，也就是那个map，接着调用map.get方法，通过计算到的hash值和table.length-1相与，确定所在的位置并返回值。threadLocals允许有键为null的Entry存在

为什么要使用弱引用封装Entry的key，是因为用强引用的话，如果使用ThreadLocal变量的某个线程长时间不中止，就算这个ThreadLocal对象超过了它应当的存在的生命周期，还是不能被回收，出现短生命周期对象被长生命周期引用所指向的情况，出现内存泄漏，因此要使用弱引用，在GC时一定被回收

应用场景有：① 管理不同的数据库连接，用ThreadLocal包装Connection对象，这样每个线程调用getConnection不会每次都拿不同的连接，不同线程之间互不影响；② ThreadLocal结合AOP，实现日志打印，每个线程负责的请求完成之后再一起打印，多个线程之间互不影响；③ 多个线程同时使用SimpleDateFormat的时候，可能线程不安全，因为要操作同一个pattern成员变量，这时就可以将SimpleDateFormat用ThreadLocal封装；④ 使用ThreadLocal保存用户session信息

### :point_right:ThreadLocal会出现内存泄漏吗？分析一下

会出现，因为虽然Entry的key是弱引用，但value是强引用，而且ThreadLocalMap允许null键的存在，因此可能导致键为null的Entry的value指向的对象迟迟无法回收，造成内存泄漏。ThreadLocal在set、get方法的末尾调用cleanSomeSlots，去除null键的Entry，一定程度上缓解了这个问题，但如果是在set一次之后发生了gc，且后面没有后续操作这种情况呢，因此使用ThreadLocal要注意去手动调用remove方法，清除null键

## 阻塞队列

### :point_right:简单介绍一下阻塞队列？

阻塞队列是一个可以实现线程安全的阻塞取和阻塞添加的队列，当队列满时，put操作会被阻塞住；当队列空时，take操作会被阻塞住。阻塞是靠两个ConditionObject实现的，一个是notFull，一个是notEmpty，当put时，首先lock住，如果队列满则调用notFull.await()阻塞线程并释放锁，后续的线程再想进来，也会进入notFull对应的条件队列，此时只有其他线程完成了take操作，会调用notFull的signal，才会唤醒刚刚的线程继续在await方法中竞争锁，并进行后续的put操作。Condition底层是靠LockSupport实现的阻塞及唤醒

### :point_right:手写一个生产者消费者

```java
class Producer{
    ArrayBlockingQueue<Object> q;
    public Producer(ArrayBlockingQueue<Object> q) {
        this.q = q;
    }
    public Object take() throws InterruptedException {
        return q.take();
    }
}
class Consumer{
    ArrayBlockingQueue<Object> q;
    public Consumer(ArrayBlockingQueue<Object> q) {
        this.q = q;
    }
    public void put(Object o) throws InterruptedException {
        q.put(o);
    }
}
```

### :point_right:手写一个阻塞队列？

```java
class MyBlockingQueue<T>{
    private final List<T> list;
    private final int bound;
    private final ReentrantLock lock = new ReentrantLock();
    Condition notFull = lock.newCondition(), notEmpty = lock.newCondition();
    public MyBlockingQueue(int bound) {
        this.list = new ArrayList<>(bound + 1);
        this.bound = bound;
    }
    public T take(){
        lock.lock();
        try{
            while (list.size() == 0){
                notEmpty.await();
            }
            return list.remove(0);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        throw new RuntimeException("error!");
    }
    public void put(T t){
        lock.lock();
        try{
            while (list.size() == bound){
                notFull.await();
            }
            list.add(t);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

## 线程池

### :point_right:线程池几种状态？

- RUNNING：运行中
- SHUTDOWN：处理已有任务，不添加新任务；调用shutdown进入
- STOP：中断处理的任务，不添加新任务；调用shutDownNow进入
- TIDYING：死亡进行时；SHUTDOWN状态下所有已有任务完成会进入
- TERMINATED：线程彻底终止；TIDYING状态会执行terminated

### :point_right:拒绝策略？

线程池首先会有coorPoolSize个持续活跃的线程用于处理任务，任务数超过coorPoolSize能处理的之后，会将任务放进阻塞队列中等待执行，如果队列装满之后则会创建新线程，扩充到maximumPoolSize，如果任务再次超过maximumPoolSize，就会根据RejectedExecutionHandler参数指定的拒绝策略对新任务进行处理。拒绝策略有：AbortPolicy，会直接拒绝任务并抛出RejectExecutionException；CallerRunsPolicy，调用线程池的线程来处理新任务；DiscardOldestPolicy，抛弃队列中最老的任务并令新任务加入；DiscardPolicy，直接抛弃任务，不抛出异常

### :point_right:为什么有个corePoolSize，还要有个maximumPoolSize？

线程池的构建有七个参数：coorPoolSize，maximumPoolSize，keepAliveTime，unit，workQueue，threadFactory以及handler。线程池一般是任务来时才创建线程，除非调用了prestartCoreThread或者prestartAllCoreThreads，然后任务数到达coorPoolSize后，所有核心线程都在处理任务；此时若有新任务，则会进入workQueue排队等待执行，如果是有界队列，当队列满之后，maximumPoolSize就是在这个时候起到作用的，可以新创建一些线程来处理新任务，阈值为maximumPoolSize。当空闲下来后，keepAliveTime之后这些超出核心线程数的线程会销毁，如果设置了allowCoreThreadTimeout为true，则核心线程也会销毁

### :point_right:为什么不推荐用Executors创建线程池？

以几个常见的通过Executors创建线程池的方法为例：

newFixedThreadPool和newSingleThreadExecutor分别设置若干个和一个核心线程，且只让核心线程工作，它们的workQueue是无参构造的LinkedBlockingQueue，是长度为Integer.MAX_VALUE的队列，新任务来会一直扩充，造成OOM，且会一直等待，造成响应时间无限延长

newCachedThreadPool的队列是SychronousQueue，向它put之后，没有消费者线程会自旋然后阻塞住，它和take操作是一一配对的，因此，每次新来一个任务，就需要创建一个新线程去消费这个任务，它的线程数可以扩充到Integer.MAX_VALUE，并且空闲线程60s后才会销毁，因此可能因为线程太多产生OOM，线程太多也会造成较大开销

因此建议根据实际情况，自己new ThreadPoolExecutor对象并自己指定参数

### :point_right:你是依据什么设置线程池参数的？

有一个通用的策略是，对于IO密集型任务，设置corePoolSize为cpu核数*2，因为IO过程cpu空闲，可以多创建几个线程；对于cpu密集型任务，则创建较少量线程，corePoolSize为cpu核数+1，充分利用cpu的计算能力

还可以靠自己计算来实现，比如每秒来的任务是n个，每个任务需要处理t秒，允许的响应时间为s，对于corePoolSize，能够处理每秒来的所有任务，需要n/(1/t)个线程；对于maximumPoolSize，可以设置为每秒到达的最大任务数/(1/t)；对于workQueue，不要采用无界队列，可以考虑使用ArrayBlockingQueue或者PriorityBlockingQueue，队列长度可以设置为每秒能处理的线程数*响应时间，也就是n\*s

也可以在运行过程中通过setCorePoolSize和setMaximumPoolSize动态调整参数*（源码注释有提到）*

> [如何设置线程池参数？美团给出了一个让面试官虎躯一震的回答](https://mp.weixin.qq.com/s?__biz=Mzg3NjU3NTkwMQ==&mid=2247505103&idx=1&sn=a041dbec689cec4f1bbc99220baa7219&source=41#wechat_redirect)

### :point_right:Java的线程池怎么实现的？

线程池中有两个比较重要的容器，一个是HashSet\<Worker> workers，一个是workQueue，其中Worker继承了AQS；一个重要的原子变量ctl，它的高三位用于记录线程池状态，一开始为RUNNING，即-1左移COUNT_BITS=29位，后面想要再获取状态，可通过&~COUNT_BITS来实现；通过将低29位可以用来存放线程个数，即与COUNT_MASK相与来实现*（COUNT_MASK=(1<<29)-1，即COUNT_MASK的低29位全部是1，高3位全是0）*

- *execute：三个if*
  - 调用execute方法后，首先会通过ctl计算workCount，也就是正在运行的线程数，如果小于corePoolSize，则调用addWorker(command, true)创建核心线程并加入workers执行任务，并return
  - 如果大于corePoolSize或addWorker失败，则进入第二个if，首先判断线程池状态，接着尝试将任务加入workQueue当中，如果入队成功会先recheck一次线程池状态，如果不为RUNNING则从workQueue移除并拒绝任务，如果是RUNNING但工作线程为0则会调用addWorker(null, false)创建线程从队列中取任务执行
  - 如果入队失败则调用addWorker(command, false)创建线程执行任务，如果失败则拒绝任务
- addWorker*：两个作用（ps：第一个参数为firstTask，null表明从队列中取任务，第二个参数为core，true表示不能大于corePoolSize，false表示不能大于maximumPoolSize）*
  - 增加工作线程数workCount，并创建新线程加入workers执行任务
- Worker：
  - addWorker会调用worker.run会调用runWorker方法，里面会调用firstTask.run，如果firstTask为null，则调用getTask并调用所返回对象的run方法

### :point_right:线程池中的线程是一开始就有的吗？如何保证核心线程不被销毁？

不是，传入任务之后才会创建线程，除非调用prestartCoreThread或prestartAllCoreThreads

保证核心线程不销毁在于runWorker方法中，如果工作全部完成，Worker里面的firstTask为null，会调用getTask方法，如果当前活动线程数大于核心线程数，就会调用workQueue.poll，否则调用workQueue.take阻塞住线程，就不会销毁了

### :point_right:手写线程池

> 目前无能为力

## CountDownLatch&CyclicBarrier

### :point_right:CountDownLatch和CyclicBarrier的区别？

对于CyclicBarrier

1. 阻塞等待的是所有参与线程，参与线程全部到达barrier时，进行终极任务
2. 具有**重复利用性**，可以执行多次终极任务

而对于CountDownLatch

1. 参与线程不会阻塞，而是终极任务的线程阻塞，等待其他所有线程手动countDown，**不影响自身执行**
2. 终极任务只能执行一次

