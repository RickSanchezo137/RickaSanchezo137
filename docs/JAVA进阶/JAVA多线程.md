# [返回](/)

# JAVA多线程

## 程序、进程、线程

- 程序：程序是为了完成特定任务，用某种语言编写的一组指令的集合，**是一段静态的代码**
- 进程：程序的一次执行过程，或正在运行的程序，具有生命周期。**进程是系统资源分配的单位**
- 线程：进程的进一步细化，程序内部的一条执行路径。**线程是调度和执行的单位**，具有自己独立的运行栈和程序计数器（pc）。一个进程中的多个线程共享相同的内存单元/内存空间地址，从同一堆中分配对象，可以访问相同的对象和变量

![image-20210420163531660](imgs\JAVA多线程\1-1.png)

> 对于多个线程，每个线程有自己独立的虚拟机栈空间和程序计数器，而方法区和堆是被多个线程所共享的
>
> **对于JAVA来说，运行java.exe，至少有三个线程，主线程main()，垃圾回收线程gc()以及异常处理线程**

- 用户线程和守护线程
  - 守护线程服务用户线程，在start前调用，如gc()
  - JVM若只有守护线程，则JVM将退出

## 单核/多核CPU

- 单核：单个CPU单位时间执行一个任务，是并发的

  > 注意：单核也可以实现多线程，单核多线程指的是单核CPU轮流执行多个线程，通过给每个线程分配CPU时间片来实现，只是因为这个时间片非常短，所以在用户角度上感觉是多个线程同时执行；线程间的切换叫做上下文切换，需要保存线程运行状态等等，可能有些情况下这种开销会大于串行的开销

- 多核：多个CPU单位时间执行多个任务，是并行的

## 线程的创建和使用（四种）

### Thread

#### 线程的创建

1. 创建一个继承Thread的子类

2. 重写Thread类的run()

3. 创建子类对象并调用start()

   **start()功能**

   1. 启动当前线程     *注意：run()是无法启动线程的*

   2. 调用当前线程的run()

   3. 已经调用start()的对象不能再次调用start()  

      > ```java
      > if (threadStatus != 0)
      >     throw new IllegalThreadStateException();
      > // Thread源码中有这样一句，第一次调用时threadStatus为0，结束后就不为0了，再调用便会报错
      > ```

```java
public class Test {
    public static void main(String[] args) {
        new MyThread().start();
        new Thread(){
            @Override
            public void run(){
                System.out.println("这里是匿名子类......");
            }
        }.start();
    }
}
class MyThread extends Thread{
    @Override
    public void run() {
        System.out.println("mythread......");
    }
}
```

#### 常用方法

- start()：启动当前线程；调用当前线程的run()
- run()：一般需要重写，将创建的线程要执行的操作声明在此方法中
- currentThread()：静态方法，返回执行当前代码的线程
- getName()：获取当前线程的名字
- setName()：设置当前线程的名字
- yield()：释放当前cpu的执行权（public static native void yield()，是本地方法），但是不能保证在当前线程调用yield()之后，其他线程就一定能获得执行权，也有可能是当前线程又回到运行状态继续运行，yield不会释放锁
- join()：在线程a中调用线程b的join方法，线程a进入阻塞状态，直到b完全结束之后a才结束阻塞状态（a中调用，a就阻塞）
- sleep()：让当前线程阻塞指定的时间，会让出cpu，但不会释放锁

### Runnable

#### 线程的创建

1. 创建一个实现Runnable接口的实现类
2. 实现Runnable接口的run()
3. 创建实现类对象并作为参数传入Thread的构造器，创建Thread类的对象
4. Thread类的对象调用start()

```java
public class Test {
    public static void main(String[] args) {
        new Thread(new MyThread()).start();
    }
}
class MyThread implements Runnable{
    @Override
    public void run() {
        System.out.println("mythread...");
    }
}
```

> 这里实际上用到了代理模式，Thread和MyThread都继承于Runnable接口，Thread相当于MyThread的代理，控制对MyThread的run方法的调用
>
> ![image-20210427213555484](imgs\JAVA多线程\3-1.png)![image-20210427213648179](imgs\JAVA多线程\3-2.png)
>
> target相当于被代理的对象，也就是这里的MyThread

### 对比

一般优先选择Runnable

- Thread的方式只能单继承Thread类，扩展性不如Runnable
- Runnable实现可以满足实现类的对象的共享（作为Thread构造器的参数），多个线程可以共享一些数据而不用声明static变量

### JDK5.0新增

- Callable接口
- 线程池

[链接](#JDK50新增的线程创建方式)

## 线程的调度

### JAVA的调度方法

1. 同优先级的线程组成先进先出队列（先到先服务），使用时间片策略
2. 对高优先级，采用优先调度的抢占式策略

#### 优先级

- MAX_PRIORITY：10
- MIN_PRIORITY：1
- NORM_PRIORITY：5

**方法**：getPriority()，setPriority()

> 1. 线程创建时继承父线程的优先级
> 2. 低优先级只代表获得调度的概率低，并非一定在高优先级后面调用

## 生命周期

- 新建：新建一个线程
- 就绪：调用start()，等待cpu时间片
- 运行：就绪的线程获得cpu资源，开始执行run()中的方法
- 阻塞：让出cpu并临时中止自己的运行
- 死亡：完成工作或被强制性中止或出现异常导致线程结束

![image-20210428112534510](imgs\JAVA多线程\5-1.png)

**JDK中使用Thread.State枚举来定义线程的状态**

```java
public enum State {
    NEW,  // 新建，没有start
    RUNNABLE,  // 运行
    BLOCKED,  // 阻塞
    WAITING,  // 空参方法导致的停等
    TIMED_WAITING,  // 带时间参数方法的停等
    TERMINATED;  // 中止
}
```

## 线程的同步

> 问题：取钱问题，A、B两人同时取钱2000
>
> ```java
> public class Test {
>     public static void main(String[] args) {
>         TakeMoney takeMoney = new TakeMoney();
>         new Thread(takeMoney).start();
>         new Thread(takeMoney).start();
>     }
> }
> class TakeMoney implements Runnable{
>     int money = 3000;
>     @Override
>     public void run() {
>         if(money > 2000){
>             // ①
>             money -= 2000;
>         }
>     }
> }
> ```
>
> 可能出现这样的情况：A取钱，刚进入if，在①处，还没取到时，B也进入了if语句，这样的话两人同时取了2000，银行默默亏了1000......
>
> 一个线程操作未完成时，其他若干个线程参与进来，这时候就需要线程同步

### 同步代码块

```java
synchronized(同步监视器){
	// 需要被同步的代码（操作共享数据的代码）
}
```

> 同步监视器，俗称：锁。任何一个类的对象都可以充当锁。要求多个线程必须共用一把锁

1. 针对于Runnable接口创建线程

```java
public class Test {
    public static void main(String[] args) {
        TakeMoney takeMoney = new TakeMoney();
        new Thread(takeMoney).start();
        new Thread(takeMoney).start();
    }
}
class TakeMoney implements Runnable{
    int money = 3000;
    Object obj = new Object();  // 锁，是两个线程共享的
    @Override
    public void run() {
        synchronized (obj) {
       	// 这里也可以写成如下，this是TakeMoney的对象，也是两个线程共享的
       	// synchronized(this){
            if (money > 2000) {
                money -= 2000;
            }
        }
    }
}
```

2. 针对于继承Thread类创建线程方法

```java
public class Test {
    public static void main(String[] args) {
        new TakeMoney().start();
        new TakeMoney().start();
    }
}
class TakeMoney extends Thread{
    static int money = 3000;
    static Object obj = new Object();  // 需要加上static，保证锁的共享
    @Override
    public void run() {
        // 这里也可以也成如下，类锁，类对象只会加载一次
       	// synchronized(TakeMoney.class){
        synchronized (obj) {
            if (money > 2000) {
                money -= 2000;
            }
        }
    }
}
```

> 同步的好处：解决了线程的安全问题
>
> 局限性：操作同步代码时只有一个线程，相当于单线程

### 同步方法

```java
权限修饰符 synchronized 返回值 method(args){
    ……
} 
```

> 同步方法中同步监视器为this

1. 针对于继承Thread类创建线程方法

```java
// Thread类
public class Test {
    public static void main(String[] args) {
        new TakeMoney().start();
        new TakeMoney().start();
    }
}
class TakeMoney extends Thread{
    static int money = 3000;
    @Override
    public synchronized void run() {
        if (money > 2000) {
            try {
                sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            money -= 2000;
        }
    }
}
```

这样写是无法同步的，因为锁不是共享的，而是每个对象自己独立的this引用，需要改写成静态方法

```java
public class Test {
    public static void main(String[] args) {
        new TakeMoney().start();
        new TakeMoney().start();
    }
}
class TakeMoney extends Thread{
    static int money = 3000;
    @Override
    public void run() {
        take();
    }
    public synchronized static void take(){
        if (money > 2000) {
            try {
                sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            money -= 2000;
        }
    }
}
```

**这时的同步监视器是TakeMoney.class的类对象**

### Lock

[链接](#Lock锁机制)

## 单例模式懒汉式线程同步

[单例模式回顾](/JAVA基础/JAVA面向对象?id=单例设计模式（singleton）)

### synchronized

1. 把方法改成同步方法

   ```java
   class Singleton{
       private Singleton(){}
       private static Singleton instance = null;
       // 锁是Singleton.class
       public static synchronized Singleton getInstance(){
           if(instance == null){
               instance = new Singleton();
           }
           return instance;
       }
   }
   ```

2. 加入同步代码块

   ```java
   class Singleton{
       private Singleton(){}
       private static Singleton instance = null;
       public static Singleton getInstance(){
           // 同样是类锁
           synchronized (Singleton.class) {
               if (instance == null) {
                   instance = new Singleton();
               }
               return instance;
           }
       }
   }
   ```

效率比较差，instance创建出来以后，实际上可以同时判断不为null，然后return就可以了，但在同步代码块中，只能一个一个判断，效率就下降了

### 双检锁

**双检锁（错误版本）**

```java
class Singleton{
    private Singleton(){}
    private static Singleton instance = null;
    public static Singleton getInstance(){
        if(instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

效率得到了提升，若不为null直接return instance，而不必等前面的线程拿完才能拿

**但事实上，只采用双检锁，可能会出现”无序写入“的问题，这是由于JMM的指令重排序机制导致的**

> **指令重排序**
>
> 在我们的直观感受里，可能此处instance的初始化分为以下的步骤：
>
> 	1. 内存中分配一块内存空间
> 	2. 在该空间上实例化一个Singleton类的对象
> 	3. 使用instance指向该对象
>
> 实际上，由于第2、3步之间没有依赖关系，根据指令重排序机制，顺序可能调整为
>
>  	1. 内存中分配一块内存空间
>  	2. 使用instance指向该内存空间
>  	3. 在该空间上实例化一个Singleton类的对象
>
> 也就是说，在进行到第二步时，**instance可能已经不为null了，但实际上还没有构造完成**。举一个例子，线程A首先判断instance为null，在A初始化instance的过程中，进行到第二步时，B进来了，它判断不为null，直接跳到return instance这一步了，这样的话，B拿到的instance实际上是没有构造好的，如何解决这个问题呢？见下文

**双检锁（正确版本）**

**volatile：使用[volatile](#volatile关键字)关键字就能避免这个问题**

```java
class Singleton{
    private Singleton(){}
    // 使用volatile修饰instance
    private static volatile Singleton instance = null;
    public static Singleton getInstance(){
        if(instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

## 线程的死锁

### 基本概念

> **所有线程处于阻塞状态，等待资源的释放**

### 产生死锁的条件

1. **互斥条件**：线/进程抢夺的资源必须是临界资源，一段时间内，该资源只能同时被一个线/进程所占有
2. **请求和保持条件**：当一个线/进程持有了一个（或者更多）资源，申请另外的资源的时候发现申请的资源被其他线/进程所持有，当前线/进程阻塞，但不会释放自己所持有的资源
3. **不可抢占条件**：进程已经获得的资源在未使用完毕的情况下不可被其他进程所抢占
4. **循环等待条件**：存在一个封闭的进程链，使得每个进程至少占有此链中下一个进程所需要的一个资源

解决方法：死锁预防和死锁避免

#### 死锁预防

需要较大开销

1. 破坏请求和保持条件
   1. 破坏请求条件：线/进程运行之前，一次性拿运行过程中所需的所有资源，拿不到，就释放自身所有资源
   2. 破坏保持条件：线/进程运行之前只获取初始需要的资源，运行过程中请求新的资源，同时不断释放自己用过的资源
2. 破坏不可抢占条件：当一个线/进程申请一个资源，然而却申请不到的时候，必须释放已经申请到的所有资源
3. 破坏循环等待条件：线/进程申请资源按照一定的顺序（银行家算法等）

#### 死锁避免

资源有序分配，银行家算法等等

### 死锁的简单例子

```java
public class Test {
    public static void main(String[] args) {
        Object lock1 = new Object(), lock2 = new Object();
        new Thread(){
            @Override
            public void run() {
                synchronized (lock1) {
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    // 请求锁2的资源，但不释放锁1
                    synchronized (lock2){
                    }
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                synchronized (lock2) {
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    // 请求锁1的资源，但不释放锁2，这样两个线程就卡死了
                    synchronized (lock1){
                    }
                }
            }
        }.start();
    }
}
```

我们使用同步时，要注意避免死锁的产生，尽量避免嵌套同步

## Lock锁机制

> JDK 5.0开始，提供了更强大的Lock充当同步锁

```java
ReentrantLock lock = new ReentrantLock();
```

![image-20210428145439291](imgs\JAVA多线程\9-1.png)

> 当构造器传入参数fair==true时，return的是FairSync()，也就是公平锁，能够让线程”公平“地执行，即有排队等待的线程就优先让它获取锁，不会出现线程饥饿的现象

```java
import java.util.concurrent.locks.ReentrantLock;

public class Test {
    public static void main(String[] args) {
        TakeMoney takeMoney = new TakeMoney();
        new Thread(takeMoney).start();
        new Thread(takeMoney).start();
    }
}
class TakeMoney implements Runnable{
    int money = 3000;
    private ReentrantLock lock = new ReentrantLock();
    @Override
    public void run() {
        try {
            lock.lock();  // 加锁
            if (money > 2000) {
                money -= 2000;
            }
        }finally {
            lock.unlock();  // 解锁
        }
    }
}
```

> 注意：lock需作为线程共享对象

### 面试题：synchronized与Lock的异同？

- 相同：解决线程安全问题

- 不同：lock需要手动的启动/结束同步，比较灵活；synchronized是自动的

开发中的使用顺序建议：Lock→同步代码块→同步方法

## 线程通信

### notify()和wait()

> **注意：wait()会释放锁**

场景：两个线程交替打印1~10

```java
public class Test {
    public static void main(String[] args) {
        Print print = new Print();
        new Thread(print).start();
        new Thread(print).start();
    }
}
class Print implements Runnable{
    private int number;
    @Override
    public void run() {
        while(number <= 10) {
            System.out.println(Thread.currentThread().getName() + "打印了" + number);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            number++;
        }
    }
}
```

这是线程不安全的，打印结果：

![](D:\笔记\秋招\docs\JAVA进阶\imgs\JAVA多线程\11-1.png)

添加同步

```java
public class Test {
    public static void main(String[] args) {
        Print print = new Print();
        new Thread(print).start();
        new Thread(print).start();
    }
}
class Print implements Runnable{
    private int number;
    @Override
    public synchronized void run() {
        while(number <= 10) {
            System.out.println(Thread.currentThread().getName() + "打印了" + number);
            number++;
        }
    }
}
```

![image-20210428162646811](imgs\JAVA多线程\11-2.png)

很难实现交替打印

添加wait()、notify()机制（都是native方法，必须在同步代码块或同步方法内调用）

- wait()：当前线程进入阻塞并释放对象锁
- notify()：解除另一个线程（按优先级选择，若只有两个，就选择另一个）的阻塞状态，**不是释放对象锁**

```java
public class Test {
    public static void main(String[] args) {
        Print print = new Print();
        new Thread(print).start();
        new Thread(print).start();
    }
}
class Print implements Runnable{
    private int number;
    @Override
    public synchronized void run() {
        while(number <= 10) {
            notify();  // 解除另一个线程的阻塞，但当前线程已经拿到print的对象锁了，所以另一个还要等
            System.out.println(Thread.currentThread().getName() + "打印了" + number);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            number++;
            try {
                wait();  // 当前线程阻塞并释放print对象锁，另一个线程拿到锁，开始执行
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        notify();  
        /*
        最后一次，打印10的线程B进入wait阻塞状态并释放对象锁，之前被对象锁阻塞的打印9的线程A从wait处接着
        运行，发现number>10，跳出循环并解除B的阻塞，没有这一句的话B会一直卡在阻塞状态
        */
    }
}
```

![image-20210428171440133](imgs\JAVA多线程\11-3.png)

**a.wait()的使用前提是a是被上了对象锁，即a为同步监视器，但等待的是wait出现的地方的线程，而不是a（a是一个线程类对象的情况下）**

#### sleep()和wait()的异同

- 相同：都可以使当前线程进入阻塞状态
- 不同：
  1. sleep是Thread类中声明的；wait是Object类中声明的
  2. sleep可以在任何需要的场景下调用；wait只能通过同步监视器调用
  3. 如果两个方法都使用在同步代码块/方法中，sleep不会释放锁，wait会释放

### 生产者/消费者问题

```java
public class Test {
    public static void main(String[] args) {
        Product product = new Product();
        Producer producer = new Producer(product);
        Consumer consumer = new Consumer(product);
        Thread pThread = new Thread(producer);
        pThread.setName("生产者1");
        Thread cThread = new Thread(consumer);
        cThread.setName("消费者1");
        pThread.start();
        cThread.start();
    }
}
class Product{
    private int num = 0;
    public synchronized void produce() {
        if(num < 20){
            num++;
            System.out.println(Thread.currentThread().getName() + "生产第" + num + "个产品");
            notify();
        }else {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public synchronized void consume() {
        if(num > 0){
            System.out.println(Thread.currentThread().getName() + "消费第" + num + "个产品");
            num--;
            notify();
        }else {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
class Producer implements Runnable{
    private Product product;

    public Producer(Product product) {
        this.product = product;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "开始生产产品");
        while (true){
            product.produce();
        }
    }
}
class Consumer implements Runnable{
    private Product product;

    public Consumer(Product product) {
        this.product = product;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "开始消费产品");
        while (true){
            product.consume();
        }
    }
}
```

## JDK5.0新增的线程创建方式

### 实现Callable接口

**实现call方法**

相比于run的特点

- 可以有返回值
- 可以抛出异常
- 支持泛型的返回值
- 需要借助FutureTask类

**Future接口**

- 可以对Callable、Runnable任务的执行结果进行取消、查询是否完成、获取结果等
- 有唯一的实现类FutureTask
- FutureTask同时实现了Runnable、Future接口。也就是说，它既可以用作Runnable被线程执行，也可以作为Future用作获取Callable的返回值

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Task task = new Task();
        FutureTask futureTask = new FutureTask(task);
        new Thread(futureTask).start();
        // get返回值即为call方法的返回值
        System.out.println("sum == " + futureTask.get());
    }
}
class Task implements Callable{
    @Override
    public Object call() throws Exception {
        int sum = 0;
        for (int i = 1; i <= 100; i++) {
            if(i % 2 == 0){
                System.out.println(i);
                sum += i;
            }
        }
        return sum;
    }
}
```

#### FutureTask只能执行一次

看源码

```java
/**
	执行结果(全局变量), 有2种情况:
	1. 顺利完成返回的结果
	2. 执行run()代码块过程中抛出的异常
*/
private Object outcome; 

//正在执行run()的线程, 内存可被其他线程可见
private volatile Thread runner;

public void run() {
	/**
		FutureTask的run()仅执行一次的原因：
		1. state != NEW表示任务正在被执行或已经完成, 直接return
		2. 若state==NEW, 则尝试CAS将当前线程设置为执行run()的线程,如果失败,说明已经有其他线程 先行一步执行了run(),则当前线程return退出
	*/
	if (state != NEW ||!UNSAFE.compareAndSwapObject(this, runnerOffset,null, Thread.currentThread()))
		return;
	try {
		//持有Callable的实例,后续会执行该实例的call()方法
		Callable<V> c = callable;
		if (c != null && state == NEW) {
			V result;
			boolean ran;
			try {
				result = c.call();
				ran = true;
			}catch (Throwable ex) {
				result = null;
				ran = false;
				//执行中抛的异常会放入outcome中保存
				setException(ex);
			}
			if (ran)
				//若无异常, 顺利完成的执行结果会放入outcome保存
				set(result);
		}
	}finally {
		// help GC 
		runner = null;
		int s = state;
		if (s >= INTERRUPTING)
			handlePossibleCancellationInterrupt(s);
	}
}

```

所以要使用FutureTask执行多个任务，得new多个FutureTask

```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

class Task implements Callable {

    private int sum = 0;

    @Override
    public Object call() {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(1000);
                sum+=i;
                System.out.println(Thread.currentThread().getName() + "运行  :  " + sum);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return sum;
    }
}
public class Test {
    public static void main(String[] args) {
        //1.new一个实现callable接口的对象
        Task task = new Task();
        //2.通过futureTask对象的get方法来接收futureTask的值
        FutureTask f1 = new FutureTask(task);
        FutureTask f2 = new FutureTask(task);
        Thread t1 = new Thread(f1,"task1");
        Thread t2 = new Thread(f2,"task2");
        t1.start();
        t2.start();
        // 这样是不行的
        /*        
        FutureTask futureTask = new FutureTask(task);
        Thread t1 = new Thread(futureTask,"task1");
        Thread t2 = new Thread(futureTask,"task2");
        */
    }
}
```

### 实现线程池

**提前创建好多个线程放入池中，使用时直接获取，使用完放回池中，可以实现重复利用，避免频繁创建销毁**

好处

- 提高响应速度
- 降低资源消耗
- 便于线程管理

**Executors工具类，ExecutorService接口**

```java
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(10);
        System.out.println(service.getClass());  // 这里是ThreadPoolExecutor
        // 接口无法改属性，所以向下转型成ThreadPoolExecutor再改
        ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) service;
        threadPoolExecutor.setKeepAliveTime(60, TimeUnit.MINUTES);
        threadPoolExecutor.setMaximumPoolSize(10);
        
        // 适用于Runnable
        service.execute(new NumberThread());
        // 适用于Callable
        Future future = service.submit(new NumberThread2());
        System.out.println("奇数和: " + future.get());
        // 中止线程池
        service.shutdown();
    }
}
class NumberThread implements Runnable{
    @Override
    public void run() {
        for(int i = 0; i <= 100; i++){
            if(i % 2 == 0){
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        }
    }
}
class NumberThread2 implements Callable {
    @Override
    public Object call() throws Exception {
        int sum = 0;
        for (int i = 0; i <= 100; i++) {
            if(i % 2 != 0){
                sum += i;
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        }
        return sum;
    }
}
```

## volatile关键字

- 防止指令重排
- 防止线程不可见性

## 锁理论

#### 不同种类的锁

**独占/共享锁**

- 独占锁：被一个线程独占的锁，比如 `ReentrantLock、synchronized、ReentrantReadWriteLock.WriteLock`
- 共享锁：可多个线程共同持有的锁，比如 `ReentrantReadWriteLock.ReadLock`

**互斥/读写锁**

- 互斥锁：加锁后阻塞其他线程，即与其他线程互斥
- 读写锁：读时共享，写时互斥

**公平锁/非公平锁**

- 公平锁：按照等待锁的优先级，先申请锁的线程先获取到，`new ReentrantLock(false)`
- 非公平锁：不按申请锁的顺序，随机获取，部分线程可能饥饿，`new ReentrantLock(true)`

**可重入/不可重入锁**

- 可重入锁：锁的操作粒度是“线程”而不是“调用”，即允许一个线程多次调用，而不是只允许一次调用

  ```java
  public class Test {
      public static void main(String[] args) {
          synchronized (Test.class) {
              method1();  //  正常调用，说明类锁是可重入的，否则就死锁了
          }
      }
      static synchronized void method1(){
          System.out.println("method1");
          method2();
      }
      static synchronized void method2(){
          System.out.println("method2");
      }
  }
  ```

- 不可重入锁：容易造成死锁

**乐观锁/悲观锁**

- 乐观锁：乐观锁假设认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则返回错误的信息，让用户决定如何去做，比如java的CAS
- 悲观锁：认为存在很多并发更新操作，必须加锁

**分段锁**

- 一种设计锁的思想，例如 `ConcurrentHashMap`

**自旋锁**

- 尝试获取锁的线程不会阻塞，而是会不断循环来申请，好处是减少上下文切换，缺点是占用CPU资源

**偏向锁/轻量级锁/重量级锁**

> synchronized的锁升级机制
>
> 获取哪个锁通过对象头中的字段来确定

- 偏向锁：一段同步代码一直被一个线程访问，那么该线程会自动获取锁，降低获取锁的代价
- 轻量级锁：当锁是偏向锁时，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，提高性能
- 重量级锁：当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁

## CAS

[CAS——lixinkuan](https://lixinkuan.blog.csdn.net/article/details/94319775?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-7.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-7.control)

[CAS底层原理——Viper的程序员修炼手册](https://blog.csdn.net/weixin_43314519/article/details/110195569)

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

[2] [https://baijiahao.baidu.com/s?id=1663045221235771554&wfr=spider&for=pc——程序猿的内心独白](https://baijiahao.baidu.com/s?id=1663045221235771554&wfr=spider&for=pc)

[3] [https://blog.csdn.net/zy13608089849/article/details/82703192——进击的小宇宙](https://blog.csdn.net/zy13608089849/article/details/82703192)

[4] [https://blog.csdn.net/qq_44384533/article/details/111920875——zhongh Jim](https://blog.csdn.net/qq_44384533/article/details/111920875)

[5] [https://www.cnblogs.com/shiyun32/p/12623783.html——一帘幽梦&nn](https://www.cnblogs.com/shiyun32/p/12623783.html)

[6] [https://lixinkuan.blog.csdn.net/article/details/94319775?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-7.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-7.control——lixinkuan](https://lixinkuan.blog.csdn.net/article/details/94319775?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-7.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-7.control)

[7] [https://blog.csdn.net/weixin_43314519/article/details/110195569](https://blog.csdn.net/weixin_43314519/article/details/110195569)
