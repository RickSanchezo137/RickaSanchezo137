# [返回](/)

# 八股文背诵版之—集合容器篇

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
- 从是否释放锁资源来说：同步块中的sleep和join不会释放锁资源，wait会释放

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

### :point_right:**如何在 Windows 和 Linux 上查找哪个线程cpu利用率最高？**

window利用任务管理器查看，主要是Linux：

1. 终端运行top指令，shift+p按cpu占用率从上到下排序，找到对应的pid
2. 使用top -H -p pid进入线程模式，会显示对应进程下的所有线程，终端对应的pid字段为线程号
3. 线程号转换成16进制，记为nid
4. 使用jstack pid > /tmp/info.dat输出所有堆栈信息
5. 打开该文件info.dat，查找nid对应部分的信息

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

3. 自旋超过一定次数，说明竞争比较强烈，则升级成重量级锁，线程会创建一个ObjectMonitor对象，其中的\_header字段赋值为displaced mark word，_owner字段为拥有锁的线程*（lock record→thread）*，\_obj字段指向锁对象，\_recursion表示重入次数，原锁对象的mark word中的锁状态改为10，且贴上指向ObjectMonitor对象的指针。其他线程想要进入同步块时，发现锁被另外的线程持有了，则会被封装成一个ObjectWaiter对象，插入到cxq队列队首，并调用park函数阻塞线程，底层是靠mutex互斥锁；如果线程调用wait，则会插入到WaitSet中；被唤醒或解锁后会从cxq队列中移动到EntryList，并将原有队列的队尾线程作为唤醒候选。如果是重入的话，则会用\_recursions记录重入次数

> 批量重偏向/撤销及epoch，后续跟进

### :point_right:锁消除和锁粗化？

锁消除是指JIT经过逃逸分析发现synchronized包裹的段不会发生数据的逃逸，即不会被其他线程访问到，则会消除这个锁从而获得更好的效率；锁粗化是指JIT编译优化对连续的使用相同锁对象的synchronized段，扩大其范围，合并成一个更大的锁

### :point_right:synchronized可重入吗？

可以重入。在偏向锁情况下，mark word里是线程ID，一个线程重入的话对比线程ID相同直接重入；在轻量级锁情况下，mark word里面指向持有锁线程的栈帧中的lock record，重入一次就在栈帧中添加一个displaced mark word为null的lock record；在重量级锁情况下，mark word指向ObjectMonitor对象，靠其中的_recursions记录重入次数

## AQS

### :point_right:？

## ConcurrentHashMap

### :point_right:？

## 阻塞队列

### :point_right:？

## 线程池

### :point_right:？

## 大厂真题

