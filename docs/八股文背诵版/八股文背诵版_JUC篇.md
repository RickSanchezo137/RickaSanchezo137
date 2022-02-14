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

可见性是指被volatile关键字修饰的变量，多个线程在对它进行读取、修改等操作时，不会出现在多个线程中、或主内存和工作内存中不一致的情况。具体是线程在写入volatile变量后，会令其他cpu的线程工作内存中的volatile变量副本失效，并将自己的volatile变量从工作内存强制刷新进主内存

有序性是指volatile和它前后的读写操作之间的有序性。具体是volatile变量禁止了volatile前后的某些读写操作与volatile变量之间进行重排序

使用javap反解析后可以发现变量带了ACC_VOLATILE的标志，为这一行指令前后插入了内存屏障，内存屏障相当于一个抽象的语义，实际底层是调用了lock addl &0, 0，真正起作用的是lock指令，它能够使其他cpu中对应的缓存行失效，并将自己缓存行的值刷新到主内存中，同时保证前后某些指令无法越过lock这一指令

缺陷在于无法保证原子性，即无法保证多个线程

> lock的缓存一致性是借助MESI协议实现，它是一个黑盒协议，不同种类CPU有自己相应硬件层面上的设计实现；而将自己缓存行的值刷新到主存是lock自己额外的功能

> MESI：保证不同工作内存中同一位置缓存行内变量值的一致性

### :point_right:知道伪共享问题吗？

知道。工作内存的读取刷新等等操作，也就是缓存的一系列操作，是以缓存行为单位的。以64字节的缓存行为例，如果两个用volatile修饰的变量在一个缓存行内，两个线程分别修改这两个变量，根据MESI协议，其中一个线程完成某一变量的修改之后，另一个线程的对应缓存行会失效，对另一个变量的修改也需要重新从主存读取，明明是不相干的两个变量，只因为同在一个缓存行，加大了时间消耗

解决方法是不使用volatile修饰，或者字节填充，让两个变量位于两个不同缓存行，就可以各自在自己的工作内存中修改，互不影响

## synchronized

### :point_right:？

## AQS

### :point_right:？

## 大厂真题

