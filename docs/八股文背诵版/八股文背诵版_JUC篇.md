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
                    if (counter != target){
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



### :point_right:什么是as-if-serial和happens-before？



## 并发关键字

### :point_right:？



## 大厂真题

