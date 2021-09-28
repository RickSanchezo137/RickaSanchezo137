# 返回](/)

# 《JAVA并发编程实战》读书笔记

> 《JAVA并发编程实战》读书笔记

## 1. 简介

### 1.1. 并发安全问题

#### [竞态条件](#竞态条件-1)

### 1.2. 并发活跃性问题

#### 死锁

#### 饥饿

#### 活锁

### 1.3. 并发性能问题

## 2. 基础：线程安全性

### 2.1. JAVA的同步机制

- synchronized
- volatile
- 显式锁
- 原子变量

### 2.2. 原子性

#### 竞态条件

> 场景
>
> ```java
> class UnSafeSequence{
> 	private int value = 9;
>     public int getValue(){
>         return value++;
>     }
> }
> ```
>
> getValue方法包含一系列操作：读value→将value的值加1→将值写入value
>
> 如果两个线程来调用这个方法，我们想要的结果应当是一个10，一个11；然而若两个线程同时在value==9时读到了value，则最终返回的结果就是两个10，是错误的

线程共享相同的内存地址，这极大方便了多个线程数据共享，然而也为串行操作带来了并行的风险，这种竞态条件就是一个典型的例子

**某个计算的正确性要取决于多个线程交替执行的顺序，这时就会发生竞态条件**，上面是其中最常见的竞态条件操作之一：读取-修改-写入，可能读取到失效的值，并进一步影响后面的操作

> 线程不安全的懒汉式单例模式，也会产生这种竞态条件，是典型的“先检查后操作”，即通过一个可能已经失效的观测结果来决定下一步的动作

#### 复合操作

- “先检查后操作”
- 读取-修改-写入

那么，以上面的场景为例，如何使这种复合操作变成原子性的？

#### 原子变量

```java
class UnSafeSequence{
    private AtomicInteger value = new AtomicInteger(9);
    public int getValue(){
        return value.incrementAndGet();
    }
}
```

在java.util.concurrent.atomic包中包含了一些原子变量类，用于实现在数值和对象引用上的原子状态转换，是由硬件提供原子操作指令实现的

#### 同步代码块

```java
synchronized(共享的内置锁/监视器锁){
    
}
```

```java
class UnSafeSequence{
    // value的值受“this”锁的保护
	private int value = 9;  
    public synchronized int getValue(){
        return value++;
    }
}
```

**锁的可重入性**

同步锁是可重入的，如何解释呢？

```java
class Father{
    synchronized void method(){
        System.out.println("father");
    }
}
class Son extends Father{
    @Override
    synchronized void method() {
        System.out.println("son");
        super.method();
    }
}
```

**注意，这里有一个前置知识：super是存在于子类对象中的一个引用类型的变量，所以使用super.method()调用父类中的方法，锁依旧还是当前子类的实例的锁**

假设当子类Son的对象son在一个线程内调用method方法，将son锁住，如果锁是不可重入的，那么调用super.method()这一句时，由于son已经被持有了，这一句就永远调用不了，陷入死锁。然而事实上并不会这样，说明**锁的操作粒度是“线程”而不是“调用”，即允许一个线程多次调用，而不是只允许一次调用**

**synchronized的局限性**

- 用synchronized修饰的多个原子操作组合起来也可能变成复合操作，产生竞态条件
- 可以保证线程安全，每次只有一个线程来访问或修改由锁保护的共享状态，但由于变成了类似于串行操作，可能会产生**性能问题**

### 2.3. 活跃性与性能

> 按照合理的设计思路，缩减synchronized的作用范围。有些不需要同步的代码应该放在同步代码块外
>
> 所以通常情况下的使用：同步代码块优先于同步方法

## 3. 对象的共享

### 3.1. 可见性

#### 前置知识：内存结构

![image-20210506203943876](imgs\3.png)

1. 线程执行的时候用到某变量，首先要将变量从主内存拷贝回自己的工作内存空间，然后对变量进行操作：读取，修改，赋值等，这些均在工作内存完成，操作完成后再将变量写回主内存
2. 线程的工作内存，物理上就是指寄存器和cpu的cache

#### 线程的不可见问题

> 现在有线程A、线程B，这两个线程共享一个变量x，都可以对x进行访问和操作
>
> 那么，提出这样一个场景，**线程A对x进行修改，同时线程B对x进行读取并打印**，流程是怎样的？用下面的JMM（JAVA Memory Model）结构图表示
>
> ![image-20210506210404476](imgs\1.png)
>
> 首先，每个线程都具有自己的工作内存，相互独立；共享变量x声明后，是放在主内存当中的
>
> ① A读取x：A从主内存读取x的值，将值存到自己工作内存的x的副本中，并从工作内存中读取x的副本的值
>
> ② A修改x：A修改x的副本中的值，写入到主内存的x当中
>
> ③ B读取x：B从主内存读取x的值，将值存到自己工作内存的x的副本中，并从工作内存中读取x的副本的值
>
> **由于A、B同时执行，所以常见的可能出现这样一种情况：**
>
> **A执行到②，还未将新值写入主内存时，B就开始执行③，读取主内存中的旧值，产生错误。这样就产生了不可见问题，也就是，A中共享变量的变化对B是不可见的**

**书中案例**

可见性问题还有一种情况

```java
class NoVisibility{
    private static boolean ready;
    private static int number;
    public static void main(String[] args) throws InterruptedException {
        new ReaderThread().start();
        Thread.sleep(10);
        number = 66;
        ready = true;
        System.out.println("main thread: number == " + number + "     ready == " + ready);
    }
    private static class ReaderThread extends Thread{
        @Override
        public void run() {
            while (!ready){
            }
            System.out.println("reader thread: number == " + number);
        }
    }
}
```

主线程main启动后，调用读线程ReaderThread里面的run方法，会出现这样的情况：读线程一直卡在循环里，产生不可见问题，输出如下图，原因是什么？

![image-20210506202608748](imgs\2.png)

1. 首先，主线程中启动读线程，主线程休息10ms，这时读线程从主内存读取ready==false，开始不断刷新
2. 10ms之后，主线程继续执行，修改number和ready的值，并同步到主内存中
3. 此时读线程仍卡在while循环中，JVM在运行时为了提高效率会将数据加载到cache或寄存器中，读取时直接从寄存器或cache，也就是线程的工作内存中读取。这里while循环里什么都不写，访问ready频率太高，虽然主内存中数据已经改变，但cpu没有时间去更新数据到工作内存中，也就是说，读线程的工作内存中是“脏数据”，便陷入了死循环

在while循环中加入一点操作，让cpu有空余的时间出来来更新工作内存，便不会卡死了

#### 指令重排序问题

还是这个场景，去掉sleep

```java
class NoVisibility{
    private static boolean ready;
    private static int number;
    public static void main(String[] args) throws InterruptedException {
        new ReaderThread().start();
        number = 66;
        ready = true;
        System.out.println("main thread: number == " + number + "     ready == " + ready);
    }
    private static class ReaderThread extends Thread{
        @Override
        public void run() {
            while (!ready){
            }
            System.out.println("reader thread: number == " + number);
        }
    }
}
```

有时可能会出现这样的情况：读线程输出number == 0，这是怎么回事？

这便涉及到JVM的**指令重排序**机制

> **指令重排序**
>
> 在我们的直观感受里，此处main的赋值分为以下的步骤：
>
> ```java
> 1. number = 66;
> 2. ready = true;
> ```
>
> 实际上，由于这两步是各自独立的，JVM认为它们之间没有依赖关系，可能将顺序优化为
>
> ```java
> 1. ready = true;
> 2. number = 66;
> ```

这也是线程不可见问题，ready的值刷新到主存了，但number值还没有刷新到主存时，读线程就读取了ready和number的值，造成错误

**多线程中程序交错执行时，重排序可能会造成内存可见性问题**

#### volatile关键字简介

作用

- **volatile要求直接在主内存操作，跳过工作内存**
- **volatile会防止指令重排序**
- **volatile仅仅用来保证该变量对所有线程的可见性，但不保证原子性**

因此上面的问题，将变量用volatile修饰，就可以解决了

#### 总结

导致共享变量在线程间不可见的原因：

a. 线程交叉执行;

b. 代码重排序结合线程交叉执行;

c. 共享变量更新后的值没有在工作内存与主内存之间及时更新;

解决：

1. syncronized

JMM关于syncronized的两条规定：

  1) 线程解锁前，必须把共享变量的最新值刷新到主内存中；

  2) 线程加锁时，将清空工作内存中共享变量的值，从而使得使用共享变量时需要从主内存中重新读取最新的值（注意，加锁与解锁是同一把锁）

  由于syncronized可以保证原子性及可见性，变量只要被syncronized修饰，就可以放心的使用.

2. volatile

  通过加入内存屏障和禁止重排序优化来实现可见性。

  具体实现过程：

  1）对volatile变量写操作时，会在写操作后加入一条store屏障指令，将本地内存中的共享变量值刷新到主内存;

  2）对volatile变量读操作时，会在读操作前加入一条load屏障指令，从主内存中读取共享变量;

注：volatile不能保证操作的原子性，也就是不能保证线程安全性

### 3.2. 发布与逸出

#### 概念

- 发布：使对象能够在当前作用域之外的代码中使用
- 逸出：不应该被发布的对象被发布

```java
class UnSafeStates{
    private String[] states = new String[]{"this cannot be changed"};
    public String[] getStates(){
        return states;
    }
}
```

这里的states逸出了，因为本来是私有的，现在任何外部代码都能修改states的内容

#### this引用逸出

首先引入场景，在构造函数中创建并发布了一个事件监听器；ThisEscape有一个成员变量a

```java
import java.util.ArrayList;
import java.util.List;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        List<Listener> listeners = new ArrayList<>();
        new Thread(new Runnable() {
            @Override
            public void run() {
                while(listeners.size() == 0){
                    Thread.yield();
                }
                listeners.get(0).onEvent();
            }
        }).start();
        new ThisEscape(listeners);
        listeners.get(0).onEvent();
    }
}
class A{
    int num = 0;
    A(){}
}
interface Listener{
    void onEvent();
}
class ThisEscape{
    private A a;
    public ThisEscape(List<Listener> source) throws InterruptedException {
        source.add(new Listener(){
            public void onEvent(){
                doSomething();
            }
        });
        Thread.sleep(1000);
        a = new A();
    }
    void doSomething(){
        a.num++;
    }
}
```

① 创建一个线程不断刷新listeners列表，发现其中有listener了，就调用它的方法

② 主线程创建ThisEscape的实例，并在构造方法里面发布一个Listener对象

运行结果：![image-20210507153302667](imgs\4.png)

为什么？因为在发布listener的时候，ThisEscape实例还没有初始化完成，而listener中又可以调用ThisEscape实例中的方法或变量，造成空指针

**这就是this逸出造成的，发布的listener中隐含一个this.doSomething，相当于this同listener一起被发布了，然而这个时候this根本还没有构造好，是不应该被发布的**

经常出现this逸出的场景：

- 在构造方法中发布其他对象，且该对象能调用this的方法
- 在构造方法中启动线程

怎么避免？需要采用**安全的对象构造过程**，可以使用私有的构造方法和一个公共的工厂方法

```java
import java.util.ArrayList;
import java.util.List;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        List<Listener> listeners = new ArrayList<>();
        new Thread(new Runnable() {
            @Override
            public void run() {
                while(listeners.size() == 0){
                    Thread.yield();
                }
                listeners.get(0).onEvent();
            }
        }).start();
        ThisEscape.getInstance(listeners);
        listeners.get(0).onEvent();
    }
}
class A{
    int num = 0;
    A(){}
}
interface Listener{
    void onEvent();
}
class ThisEscape{
    private A a;
    final private Listener listener;
    private ThisEscape() throws InterruptedException {
        listener = new Listener(){
            public void onEvent(){
                doSomething();
            }
        };
        Thread.sleep(1000);
        a = new A();
    }
    void doSomething(){
        a.num++;
    }
    public static ThisEscape getInstance(List<Listener> source) throws InterruptedException {
        ThisEscape thisEscape = new ThisEscape();
        // 确认this已经完全构造好了
        // 再把listener和this发布到source中
        source.add(thisEscape.listener);
        return thisEscape;
    }
}
```

先确认this构造好，再发布，避免this逸出

### 3.3. 线程封闭

不共享数据，就算这些数据本身不是线程安全的，由于线程封闭，只有单线程使用，它们也是安全的

- 局部变量
- ThreadLocal

#### Ad-hoc线程封闭

维护线程封闭性的职责完全由程序实现

> 比较脆弱，较少使用

#### 栈封闭

只有通过局部变量才能访问对象，而局部对象在每个线程的工作栈中，是独立的

> 注意不要令局部变量逸出，那样就会破坏栈封闭性了
>
> ```java
> private ArrayList<Animal> animals;  // 成员变量
> public void test() {
>     Animal cat = new Animal();  // 局部变量
>     animals.add(cat); 
>     cat.setAge(1);
> }
> ```
>
> 这里cat变量逸出了，别的线程也可以通过animals来获取cat引用，对cat进行操作，这就破坏了栈封闭

#### ThreadLocal

对于ThreadLocal类的多个线程共享的变量，ThreadLocal对象能为每个线程维护一个该变量的副本，操作时每个线程操作的是自己工作内存里面的变量副本

```java
public class Test {
    static ThreadLocal<String> str = new ThreadLocal<>(){
        @Override
        protected String initialValue() {
            return "init";
        }
    };
    public static void main(String[] args) throws InterruptedException {
        str.set("main");
        new Thread("thread-1"){
            @Override
            public void run() {
                str.set("thread-1");
                System.out.println(str.get());
            }
        }.start();
        new Thread("thread-2"){
            @Override
            public void run() {
                str.set("thread-2");
                System.out.println(str.get());
            }
        }.start();
        System.out.println(str.get());
    }
}
```

运行结果：![image-20210507164133040](imgs\5.png)

可以看到每个线程都有一个str变量的副本

> 记得使用完最好remove()一下，否则容易内存泄漏
>
> 扩展阅读：[https://www.zhihu.com/question/341005993](https://www.zhihu.com/question/341005993)

### 3.4. 不变性

> 前面提到的各种问题，都是多线程试图访问可变状态，如果对象的状态不可改变，这种问题就不存在了

**Final域**

注意，用final修饰的引用地址不变，内容可变，就算内容可变，用final修饰后可变状态减少了，也是值得推荐的

> 除非确实需要更高的可见性，否则应该将所有域都声明为私有域；除非确实需要某个域是可变的，否则应该声明为final域。这是良好的编程习惯
>
> final的一个语义有限制重排序，jvm在优化时会注意这一点
>
> 1. 在构造函数内对一个final域的写入,与随后把这个被构造对象的引用赋值给一个引用
>
> 变量,这两个操作之间不能重排序
>
> 2. 初次读一个包含final域的对象的引用,与随后初次读这个final域,这两个操作之间不能
>
> 重排序

#### 安全发布的常用模式

- 在静态初始化方法中初始化一个对象引用

  > 由JVM内部的同步机制确保安全

- 将对象的引用保存到volatile类型的域或者AtomicReferance对象中

- 将对象的引用保存到某个正确构造对象的final类型域中

- 将对象的引用保存到一个由锁保护的域中

#### 实用策略

- 线程封闭
- 只读共享
- 线程安全共享
- 加锁保护对象

## 4. 线程安全的对象

### 4.1. 设计线程安全的类

#### 基本要素

1. 找出构成对象状态的所有变量

```java
class Basic{
    private int num;
    private List<Ref> list = new ArrayList<>(){{
        add(new Ref());
    }};
}
class Ref{}
```

Basic的状态包括num的状态 + list的状态 + list中Ref的状态

2. 找出约束状态变量的不变性条件
3. 建立对象状态的并发访问管理策略

#### 同步策略

1. 收集同步需求

```java
class Counter{
    private int count;
    final private String name = "counter";
}
```

如何收集同步需求？需要判断对象的状态空间，对于Counter的count，除了[Integer.MIN_VALUE, Integer.MAX_VALUE] 这个状态空间之外，还需要不断递增，在进行同步设计时，也需要考虑进这些约束

final 显著减少了name的状态空间，在多线程开发过程中，学会final的使用是一个很好的习惯

2. 基于状态的先验条件

在单线程中，若某个操作无法满足某个先验条件，会失败；而在多线程中，先验条件可能由于其他线程的操作而变成真，产生“先检查后操作”的竞态条件。因此在多线程开发中，需要注意先满足先验条件，再执行操作

3. 状态的所有权

对象的状态是它里面的对象包含的域的子集

### 4.2. 实例封闭

#### 监视器模式

> 把对象的所有可变状态都封装起来，并由对象自己的内置锁来保护

例如下面的例子，对可变状态value的所有访问，都必须经过同步方法；value是没有逸出的

```java
class Monitor{
    private int value = 0;
    public synchronized void add(){
        value++;
    }
    public synchronized void subtract(){
        value--;
    }
}
```

更灵活的写法

```java
class Monitor{
    private int value = 0;
    private final Object obj = new Object();
    public void add(){
        synchronized (obj) {
            value++;
        }
    }
    public void subtract(){
        synchronized (obj){
            value--;
        }
    }
}
```

#### 从零设计一个线程安全的类

**示例：飞机及坐标**

> 飞机具有名字和坐标，get线程获取各个飞机的名称和坐标，update线程更新各个飞机的坐标，飞机是一个线程不安全的类
>
> ```java
> class Plane{
>     public String name;
>     public Float x;
>     public Float y;
>     public Plane(String name, Float x, Float y) {
>         this.name = name;
>         this.x = x;
>         this.y = y;
>     }
> }
> ```
>
> 设计一个线程安全的飞机组监控系统

首先是直观实现

```java
class PlaneMonitor{
    private final Map<String, Plane> locations;
    PlaneMonitor(Map<String, Plane> locations){
        this.locations = locations;
    }
    public synchronized Map<String, Plane> getLocations(){
        return locations;
    }
    public synchronized Plane getLocation(String name){
        return locations.get(name);
    }
    public synchronized void updateLocations(String name, Float x, Float y){
        Plane plane = locations.get("name");
        plane.x = x;
        plane.y = y;
    }
}
```

这样是线程不安全的，locations其中的对象被错误地逸出了，其他线程也可以任意修改，需要进行改进

```java
class PlaneMonitor{
    private final Map<String, Plane> locations;
    PlaneMonitor(Map<String, Plane> locations){
        this.locations = locations;
    }
    public synchronized Map<String, Plane> getLocations(){
        return deepcopy(locations);
    }
    public Plane getLocation(String name){
        Plane source = locations.get(name);
        return null == source ? null : new Plane(source.name, source.x, source.y);
    }
    public void updateLocations(String name, Float x, Float y){
        Plane plane = locations.get("name");
        plane.x = x;
        plane.y = y;
    }
    private static Map<String, Plane> deepcopy(Map<String, Plane> locations){
        Map<String, Plane> map = new HashMap<>();
        for(Map.Entry<String, Plane> temp: locations.entrySet()){
            map.put(temp.getKey(), new Plane(temp.getKey(), temp.getValue().x, temp.getValue().y));
        }
        return Collections.unmodifiableMap(map);
    }
}
```

- 对于getLocations方法，返回一个值完全相同的map的深拷贝，map中的plane对象也全部改成值一致的深拷贝，线程获取不到之前的对象，不会出现map或plane对象错误逸出的问题
- 对于getLocation方法，返回一个深拷贝，外部线程接触不到plane对象
- map设为unmodifiableMap，设为只读的

实现了线程安全，缺点是update之后不能马上更新，需要相应的飞机坐标被get了之后才会更新，也就是说，每次get可能获得的是更新之前的坐标，这一点根据实际需求，有利有弊

#### 线程安全性的委托

> 将线程安全性委托给JAVA已经实现好的一些线程安全结构来保证

```java
class PlaneMonitor{
    private final ConcurrentHashMap<String, Plane> map;
    private final Map<String, Plane> unmodifiableMap;
    PlaneMonitor(Map<String, Plane> locations){
        map = new ConcurrentHashMap<>(locations);
        unmodifiableMap = Collections.unmodifiableMap(map);
    }
    public Map<String, Plane> getLocations(){
        return unmodifiableMap;
    }
    public Plane getLocation(String name){
        return unmodifiableMap.get(name);
    }
    public void updateLocations(String name, Float x, Float y){
        map.replace(name, new Plane(name, x, y));
    }
}
```

不需要自己加锁，把线程安全性委托给ConcurrentHashMap来实现，此外，由于getLocation方法会导致plane逸出，所以可以把plane改成不可变的；这样也同时实现了get实时数据

```java
class Plane{
    public final String name;
    public final Float x;
    public final Float y;
    public Plane(String name, Float x, Float y) {
        this.name = name;
        this.x = x;
        this.y = y;
    }
}
```

> 有时也会发生委托失效，譬如：将线程安全委托给Java的原子变量，但原子变量之间存在复合操作，造成线程不安全，这时候需要进一步进行加锁或其他处理

> 上面的例子中，plane具有“坐标不能变”的不变性条件，所以不能随便让它逸出，要是没有任何不变性条件，那它当然可以随便逸出，甚至不用加final，让其他线程能够改变它的值

#### 利用好现有的线程安全类

> 已有的线程安全类，具有良好的健壮性，经过“千锤百炼”

- 直接在已有的基础上改（*大多数情况不行*）

- 继承线程安全类

  > 风险在于，若父类改变同步策略，则子类的安全性可能悄无声息被破坏了

  > 例子：扩展Vector并增加一个“若没有则添加”方法
  >
  > 因为Vector的同步锁使用同步方法实现，因此扩展比较容易
  >
  > ```java
  > class ExtendedVector<E> extends Vector<E>{
  >     public synchronized boolean putIfAbsent(E x){
  >         boolean absent = !contains(x);
  >         if(absent){
  >             add(x);
  >         }
  >         return absent;
  >     }
  > }
  > ```

- 客户端加锁

  > 对于Collection.synchronizedXXX获取的同步容器，既无法在原有基础上修改，也无法通过子类继承来扩展它的用途，这时就需要用到客户端加锁，也就是，**将使用的对象本身作为锁**
  >
  > ```java
  > class ListHelper<E> {  // 辅助类
  >     private final List<E> list = Collections.synchronizedList(new ArrayList<>());
  >     public boolean putIfAbsent(E x){
  >         synchronized (list) {
  >             boolean absent = list.contains(x);
  >             if (absent) {
  >                 list.add(x);
  >             }
  >             return absent;
  >         }
  >     }
  > }
  > ```
  >
  > 注意，此时不能采用上面的用synchronized修饰方法，因为这样锁住的只是ListHelper，list仍可以被不期望地修改，必须采用客户端加锁
  >
  > 辅助类和客户端对象是完全无关的两个类，会破坏封装性

- 组合

  > 将操作委托给底层的list实例，自己完全控制同步策略的实现，是一种监视器模式，也类似于一种装饰器模式
  >
  > ```java
  > class ImprovedList<E> implements List<E>{
  >     final List<E> list;
  > 
  >     public ImprovedList(List<E> list) {
  >         this.list = list;
  >     }
  > 
  >     public synchronized boolean putIfAbsent(E x){
  >         boolean absent = list.contains(x);
  >         if(absent){
  >             list.add(x);
  >         }
  >         return absent;
  >     }
  >     @Override
  >     public synchronized boolean contains(Object o) {
  >         return list.contains(o);
  >     }
  > 
  >     @Override
  >     public synchronized Iterator<E> iterator() {
  >         return list.iterator();
  >     }
  >     //  ......其他需要实现的方法
  > }
  > ```
  >
  > 跟Collections.synchronizedXXX这些方法的思想比较接近

## 5. JAVA并发基础模块

### 5.1. 同步容器类

#### 存在的问题

早期JDK中使用较多的是同步容器，在大部分方法前加上synchronized关键字，采用同步的方式调用方法

- Vector

稍微读下源码看看，扩容策略，扩容调用的是grow()方法

![image-20210518104129861](D:\笔记\秋招\docs\J.U.C\imgs\7.png)

调用newLength方法获取新长度

![image-20210518103727872](imgs\6.png)

minGrowth为minCapacity-oldCapacity，也就是将要增加的长度；prefGrowth当capacityIncrement>0时即为capacityIncrement，可以通过构造器来设置，capacityIncrement==0时即为旧容量

newLength取minGrowth和prefGrowth中的最大值，再加上旧长度，也就是说，扩容长度为 ①旧容量+将要增加的长度和 ②旧容量+capacityIncrement或旧容量乘2，这两者之间的较大值

- Hashtable
- SynchronizedCollection

> 是SynchronizedList、SynchronizedMap等类的父类，通过Collections.synchronizedXXX方法返回对应的同步容器，以同步List为例
>
> ![image-20210518105300025](imgs\8.png)

**虽然同步容器类是线程安全的，但在复合操作下仍可能出现线程不安全的情况，也需要加锁使之变成原子操作**

常见的复合操作包括：① 迭代，例如对一个容器迭代时，另一个线程在删除容器的元素；② 跳转；③ 条件运算，例如“若没有则添加”（*实际上就是一种“先检查后操作”*）

**这些复合操作的情况，都需要通过额外的同步处理**

#### 迭代器与ConcurrentModificationException

容器在迭代时，可以使用迭代器

```java
public class Test {
    public static void main(String[] args) {
        Vector<Integer> vector = new Vector();
        vector.add(1);
        vector.add(2);
        vector.add(3);
        Iterator<Integer> iterator = vector.iterator();
        while (iterator.hasNext()){
            System.out.println(iterator.next());
        }
    }
}
```

当采用迭代器迭代的过程中，另一个线程删除元素导致线程不安全问题，则会抛出ConcurrentModificationException

> 同步容器的迭代器发现被修改，就抛该异常，而不是等到遍历到错误的地方再抛，这是一种“及时失败”的策略（fail-fast）

```java
public class Test {
    public static void main(String[] args) {
        Vector<Integer> vector = new Vector();
        vector.add(1);
        vector.add(2);
        vector.add(3);
        Iterator<Integer> iterator = vector.iterator();
        new Thread(){
            @Override
            public void run() {
                while (iterator.hasNext()){
                    System.out.println(iterator.next());
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                vector.remove(2);
            }
        }.start();
    }
}
```

![image-20210520095213117](imgs\9.png)

实现方式是将计数器与容器关联起来，迭代期间检查计数器，如果被修改则抛出异常，然而，这个检查过程是没有经过同步处理的，因此对于计数器的检查也可能是错误的。这是一种设计上的平衡，是为了避免检查计数器时开销太大影响程序性能

如果不想对容器加锁，一种可以考虑的替代方法是克隆容器，并在容器的副本上进行迭代（*克隆的过程中仍然是要加锁的*）

#### 隐式迭代

**我们需要注意隐式的迭代**

```java
class HiddenTraversal{
    private final Set<Integer> set = new HashSet();
    public synchronized void add(Integer e){
        set.add(e);
    }
    public void method(){
        for(int i = 0; i < 10; i++){
            add(i);
        }
        System.out.println(set);
    }
}
```

多线程调用method方法，虽然add是安全的，但注意println，这一句也会遍历set，是一个不容易被发现的迭代

> 还有很多其他例子，比如Vector的hashCode
>
> ![image-20210520163135767](imgs\10.png)
>
> 对于这些迭代，都需要注意并加锁或采取其他处理

### 5.2. 并发容器

> 由于同步容器的开销大，因此 JDK 5.0提供了多种并发容器来改进性能

#### ConcurrentHashMap

**概述**

并不是采用同步方法来实现同步，而是使用粒度更细的**分段锁**

- 任意数量的读取线程可以并发地访问Map
- 读取和写入线程可以并发地访问Map
- 一定数量的写入线程可以并发地修改Map

在并发环境下实现更高的吞吐量，而在单线程环境下只损失非常小的性能

**迭代器**

- 提供的迭代器不会抛出ConcurrentModificationException，不需要在迭代过程中对容器加锁
- 提供的迭代器具有弱一致性（Weakly Consistent）而不是“及时失败”
  - 可以容忍并发的修改
  - 可以（但不保证）在迭代器被构造后将修改操作反映给容器

```java
public class Test {
    public static void main(String[] args) {
        Map<String,String> map = Collections.synchronizedMap(new HashMap<String, String>(){{
            put("1", "test1");
            put("2", "test2");
        }});
        Iterator iterator = map.entrySet().iterator();
        new Thread(){
            @Override
            public void run() {
                while (iterator.hasNext()){
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(iterator.next());
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                map.remove("2");
            }
        }.start();
    }
}
```

一个线程遍历、一个线程删除，根据fast-fail策略，抛出ConcurrentModificationException

改成ConcurrentHashMap

```java
public class Test {
    public static void main(String[] args) {
        Map<String,String> map = new ConcurrentHashMap<>() {{
            put("1", "test1");
            put("2", "test2");
        }};
        Iterator iterator = map.entrySet().iterator();
        new Thread(){
            @Override
            public void run() {
                while (iterator.hasNext()){
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(iterator.next());
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                map.remove("2");
            }
        }.start();
    }
}
```

发现正常运行了

> 很容易理解，在这种弱一致性的策略下，size和isEmpty等方法的语义被减弱了，因为结果可能是已经过期的；事实上，size和isEmpty等方法在并发场景下作用不大，因为它们经常会变，所以这样是可接受的

#### ConcurrentMap

对于Vector这种类来说，采用客户端加锁可以实现对Vector的独占式访问，创建原子操作

> 场景：一个写入线程，一个打印线程，目标是把写入构建成原子操作，只有写入全部完成之后，再打印

```java
public class Test {
    public static void main(String[] args) {
        Vector<String> vector = new Vector<>();
        //  一个写入线程，一个打印线程，要求写完所有的再打印
        new Thread(){
            @Override
            public void run() {
                synchronized (vector){
                    for(int i = 0; i < 10; i++){
                        vector.add("test " + i);
                        try {
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                System.out.println(vector);  // toString方法被客户端锁锁住
            }
        }.start();
    }
}
```

因为它的toString方法（实际上是所有方法）都需要获取对象锁，所以用一个客户端锁就能实现独占访问，让操作变成原子操作，但ConcurrentHashMap不一样

```java
public class Test {
    public static void main(String[] args) {
        Map<String, String> map = new ConcurrentHashMap<>();
        new Thread(){
            @Override
            public void run() {
                synchronized (map){
                    for(int i = 0; i < 10; i++){
                        map.put("" + i, "test" + i);
                        try {
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }.start();
        new Thread(){
            @Override
            public void run() {
                System.out.println(map);
            }
        }.start();
    }
}
```

ConcurrentHashMap的方法没有对对象加锁，所以采用客户端加锁，无法将写入变成原子操作，其他的线程依旧可以在写入的时候访问甚至修改map的值

幸运的是，**ConcurrentHashMap提供了一系列常用的复合操作**，因此我们不需要再手动实现将它们变成原子操作了，这些方法是在ConcurrentMap接口中被声明的

#### CopyOnWriteArrayList/Set

> 替代同步容器，获得更好的并发性能，迭代过程中不用加锁或复制

**Copy-On-Write：写入时复制**

- 访问对象时不需要进一步的同步

- 每次修改时，都会创建并重新发布一个新的容器副本

  > 以add为例
  >
  > ![image-20210521112124886](imgs\11.png)
  >
  > 调用了getArray和setArray，array是什么？点进去看看
  >
  > ![image-20210521112247045](imgs\12.png)
  >
  > 就是底层数组，也就是说，add方法首先拿到数组引用array，通过Arrays.copyOf创建新数组并加值，再令array指向这个新数组

- 迭代器保留一个指向底层数组的引用，这个数组是不会被修改的（每次修改都会创建一个新数组，并将底层数组的引用指向这个新数组），所以多个线程可以同时对其迭代，不会彼此干扰，也不会干扰写入线程

  > ![image-20210521112519927](imgs\13.png)

- 迭代器不会抛出ConcurrentModificationException，返回的元素与迭代器创建时一致

  > 调用的是getArray方法，返回的是当时的引用，修改之后的值当然获取不到了

**在迭代操作远多于修改操作时使用**

### 5.3. 阻塞队列

> put/take方法：当队列满/空时阻塞
>
> offer/poll：当队列满/空时返回错误状态

#### FIFO策略

- LinkedBlockingQueue：链表实现
- ArrayBlockingQueue：数组实现

#### 优先级策略

- PriorityBlockingQueue

  - put/offer方法在队列满时不会阻塞/返回错误状态，而是会自己扩容，通过tryGrow方法

  ![image-20210525100713319](imgs\14.png)

  ![image-20210525101332923](imgs\15.png)

  > 小于64，2 * oldCap + 2；大于64，1.5 * oldCap

  - 根据类的compareTo方法或自定义的Comparator比较器确定优先级

    ```java
    public class Test {
        public static void main(String[] args) throws InterruptedException {
            PriorityBlockingQueue<String> pbq = new PriorityBlockingQueue(1);
            String s1 = "element-1", s2 = "element-2", s3 = "element-3", s4 = "element-4", s5 = "element-5";
            pbq.put(s3);
            pbq.put(s5);
            pbq.put(s1);
            pbq.put(s4);
            pbq.put(s2);
            System.out.println(pbq);
            while(!pbq.isEmpty()){
                System.out.print(pbq.take() + " ");
            }
        }
    }
    ```

    实验结果

    ![image-20210525104348403](imgs\16.png)

    > 可以看到，在队列中的顺序不是按优先级来的，但take拿出来的时候却是按优先级拿出来的，为什么会是这样？来看源码
    >
    > 既然在队列中的顺序不对劲，首先看插入数据的put方法
    >
    > ![image-20210525104559282](imgs\17.png)
    >
    > 调用的是offer方法
    >
    > ![image-20210525105532405](imgs\18.png)
    >
    > 如果comparator（成员变量，可通过构造器指定）为null，用类自己实现的Comparable接口来比较；不为空则用比较起来比较。这里以siftUpComparable为例，n为当前未插入新元素时的size，e是将要插入的元素，es指向底层数组
    >
    > ```java
    > private static <T> void siftUpComparable(int k, T x, Object[] es) {
    >     //  key看作x，即要插入的元素
    >     Comparable<? super T> key = (Comparable<? super T>) x;
    >     while (k > 0) {
    >         /*
    >         	实际上，底层数组排列是堆实现，那么，parent就是当前要插入元素的父节点的下标
    >         */
    >         int parent = (k - 1) >>> 1;
    >         Object e = es[parent];
    >         //  与父节点下标比较，如果小于，则上浮
    >         if (key.compareTo((T) e) >= 0)
    >             break;
    >         es[k] = e;
    >         k = parent;
    >     }
    >     es[k] = key;
    > }
    > ```
    >
    > 我们发现，插入过程，就是一个二叉堆的上浮构建过程，若当前节点小于父节点，则上浮；最后，构成一个小顶堆
    >
    > 以刚刚的例子，依次插入element-3、5、1、4、2，过程如下
    >
    > ![image-20210525111441964](imgs\19.png)
    >
    > 自然是1、2、3、5、4
    >
    > take时：
    >
    > ![image-20210525112425272](imgs\20.png)
    >
    > 调用dequeue
    >
    > ![image-20210525113108037](imgs\21.png)
    >
    > 调用siftDownComparable，k=0，x为数组末尾的元素的值（数组末尾元素已置为null），es为底层数组，n为删除末尾元素后的size
    >
    > ```java
    > private static <T> void siftDownComparable(int k, T x, Object[] es, int n) {
    >     // assert n > 0;
    >     //  令key为末尾元素的值
    >     Comparable<? super T> key = (Comparable<? super T>)x;
    >     //  2 * half + 1 >= n
    >     //  所以如果k大于等于half，就没有子结点了，所以计算k < half时就可以
    >     int half = n >>> 1;           // loop while a non-leaf
    >     while (k < half) {
    >         //  左子节点
    >         int child = (k << 1) + 1; // assume left child is least
    >         Object c = es[child];
    >         int right = child + 1;
    >         //  比较左、右子结点，如果左大于右，则指向右子结点
    >         if (right < n &&
    >             ((Comparable<? super T>) c).compareTo((T) es[right]) > 0)
    >             c = es[child = right];
    >         //  如果末尾元素的值大于k节点较小子结点的值，则子结点上浮
    >         if (key.compareTo((T) c) <= 0)
    >             break;
    >         es[k] = c;
    >         k = child;
    >     }
    >     es[k] = key;
    > }
    > ```
    >
    > take 1
    >
    > ![image-20210525153247127](imgs\22.png)
    >
    > take 2
    >
    > ![image-20210525154008381](imgs\23.png)
    >
    > 后面同理
    >
    > 整体思路就是，末尾元素的值用一个临时变量key保存，末尾处置为null；从根节点开始，如果它的较小子节点小于key，则下沉，如果它的较小子节点大于key，则将它所在处的值置为key，此时
    >
    > 父节点 < key < 较小子节点，满足最小堆的定义
    >
    > 其实就是一个根的下沉的思路，只不过写法和常用的（我个人常用的）不太一样

#### 没有队列的队列

- SynchronousQueue

  > 不存储数据，put也不在满的时候阻塞，而是在被take之前一直保持阻塞
  >
  > 打个比方：服务员是生产者，食客是消费者，桌子的大小就是队列的容量，每盘菜就是队列中的元素；服务员可以一直往桌子端菜，直到桌子满了，食客在桌子上有菜的情况下可以一直吃（假定吃完的菜盘子直接消失），直到所有都吃完，这是一个典型的生产者-消费者模式
  >
  > 然而，使用SynchronousQueue，相当于没有这个桌子，服务员一直端着菜等着，直到食客来吃这盘菜

  ```java
  public class Test {
      static int count = 0;
      public static void main(String[] args) throws InterruptedException {
          SynchronousQueue<String> syncQueue = new SynchronousQueue<>();
          new Thread(){
              @Override
              public void run() {
                  try {
                      Thread.sleep(10000);
                      while(true) {
                          System.out.println("取线程拿了" + syncQueue.take());
                          ++count;
                          if(count >= 2) break;
                      }
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
          }.start();
          System.out.println("主线程put了e1");
          syncQueue.put("e1");
          Thread.sleep(100);
          System.out.println("主线程put了e2");
          syncQueue.put("e2");
      }
  }
  ```

#### 双端队列及工作窃取算法

- Linked/Array(Blocking)Deque

工作窃取算法（work-stealing）：

1. 一个大任务分割为若干个互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应

2. 但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务待处理。干完活的线程与其等着，不如帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行

3. 而在这时它们可能会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务线程永远从双端队列的尾部拿任务执行

```java
import java.util.NoSuchElementException;
import java.util.concurrent.LinkedBlockingDeque;
public class Test {
    static Integer count = 1;
    static LinkedBlockingDeque<String>[] works = new LinkedBlockingDeque[3];
    //  初始化任务
    static {
        for(int i = 0; i < 3; i++){
            works[i] = new LinkedBlockingDeque<>();
            for (int j = 0; j < 5; j++){
                works[i].offer("task-" + count++ + " from work " + (i + 1));
            }
        }
    }
    public static void main(String[] args) {
        Thread t1 = new Thread(new Worker1());
        Thread t2 = new Thread(new Worker2());
        Thread t3 = new Thread(new Worker3());
        Long start = System.currentTimeMillis();
        t1.start();
        t2.start();
        t3.start();
        while(true) {
            if (!t1.isAlive() && !t2.isAlive() && !t3.isAlive()) {
                Long end = System.currentTimeMillis();
                System.out.println("time consuming: " + (end - start) + "ms");
                break;
            }
        }
    }
}
class Worker{
    void doWork(int index, int sleepTime){
        while(!Test.works[index].isEmpty()){
            try {
                Thread.sleep(sleepTime);
                System.out.println("worker-" + (index + 1) + " complete " + Test.works[index].remove());
            } catch (InterruptedException | NoSuchElementException e) {
            }
        }
        System.out.println("work-" + (index + 1) + " complete");
    }
}
class Worker1 extends Worker implements Runnable {
    @Override
    public void run() {
        doWork(0, 80);
    }
}
class Worker2 extends Worker implements Runnable{
    @Override
    public void run() {
        doWork(1, 50);
    }
}
class Worker3 extends Worker implements Runnable{
    @Override
    public void run() {
        doWork(2, 30);
    }
}
```

不采用work-stealing的结果，每个线程只完成自己的工作，3、2、1依次完成：

![24](imgs\24.png)

加入work-stealing判断

```java
import java.util.NoSuchElementException;
import java.util.concurrent.LinkedBlockingDeque;
public class Test {
    static Integer count = 1;
    static LinkedBlockingDeque<String>[] works = new LinkedBlockingDeque[3];
    //  初始化任务
    static {
        for(int i = 0; i < 3; i++){
            works[i] = new LinkedBlockingDeque<>();
            for (int j = 0; j < 5; j++){
                works[i].offer("task-" + count++ + " from work " + (i + 1));
            }
        }
    }
    public static void main(String[] args) {
        Thread t1 = new Thread(new Worker1());
        Thread t2 = new Thread(new Worker2());
        Thread t3 = new Thread(new Worker3());
        Long start = System.currentTimeMillis();
        t1.start();
        t2.start();
        t3.start();
        while(true) {
            if (!t1.isAlive() && !t2.isAlive() && !t3.isAlive()) {
                Long end = System.currentTimeMillis();
                System.out.println("time consuming: " + (end - start) + "ms");
                break;
            }
        }
    }
}
class Worker{
    void workStealing(int index, int sleepTime){
        while(true){
            for(int i = 0; i < 3 && i != index; i++){
                if(!Test.works[i].isEmpty()){
                    try {
                        Thread.sleep(sleepTime);
                        System.out.println("worker-" + (index + 1) + " complete " + Test.works[i].removeLast());
                    }catch (InterruptedException | NoSuchElementException e){
                    }
                }
            }
            if(Test.works[0].isEmpty() && Test.works[1].isEmpty() && Test.works[2].isEmpty()) break;
        }
    }
    void doWork(int index, int sleepTime){
        while(!Test.works[index].isEmpty()){
            try {
                Thread.sleep(sleepTime);
                System.out.println("worker-" + (index + 1) + " complete " + Test.works[index].remove());
            } catch (InterruptedException | NoSuchElementException e) {
            }
        }
        System.out.println("work-" + (index + 1) + " complete");
        workStealing(index, sleepTime);
    }
}
class Worker1 extends Worker implements Runnable {
    @Override
    public void run() {
        doWork(0, 80);
    }
}
class Worker2 extends Worker implements Runnable{
    @Override
    public void run() {
        doWork(1, 50);
    }
}
class Worker3 extends Worker implements Runnable{
    @Override
    public void run() {
        doWork(2, 30);
    }
}
```

实验结果：

![image-20210525223231245](imgs\25.png)

3完成后，去帮1、2，总体耗时减少了

#### 中断方法

> 场景，一个线程从0打印到99，另一个线程在打印线程打印到66时，告诉它需要停止运行

```java
public class Test {
    static int i;
    public static void main(String[] args) {
        Task task = new Task();
        task.start();
        while(true) {
            if(i == 66) {
                task.stop();
                break;
            }
        }
    }
    static class Task extends Thread{
        @Override
        public void run() {
            for(i = 0; i < 100; i++){
                System.out.println(i);
            }
        }
    }
}
```

这样写乍一看是对的，然而结果是：

![image-20210526161823946](imgs\26.png)

到68的时候才停止。这是由于stop操作过程中，打印线程的数据改变了，产生了数据不同步

**这种中断方法是不科学的**

1. 暴力且不安全
2. 一个线程去停止另一个线程，导致被停止的线程没有办法完整完成资源清理过程
3. 被清理线程直接释放锁，导致数据不同步

需要摒弃stop方法，**合理的方式是：通知线程“你该停止了”，让线程自己控制自己停止**

- interrupt：使线程中断标志置为1

  ```java
  Thread A = new Thread();
  A.interrupt();
  ```

- isInterruped：返回线程中断状态

  ```java
  Thread A = new Thread();
  System.out.println(A.isInterrupted());
  ```

- interrupted：返回线程中断状态，并清除线程中断标志（置为false）

  ```java
  Thread.interrupted();
  ```

```java
public class Test {
    static volatile int i;
    public static void main(String[] args) {
        Task task = new Task();
        task.start();
        while(true) {
            if(i == 66) {
                task.interrupt();
                break;
            }
        }
    }
    static class Task extends Thread{
        @Override
        public void run() {
            for(i = 0; i < 100; i++){
                if (!Thread.interrupted()) {
                    System.out.println(i);
                } else {
                    System.out.println("clean......");
                    return;
                }
            }
        }
    }
}
```

![image-20210526164023582](imgs\27.png)

这样的中断方式比较合理，注意用volatile修饰 i，保证共享变量可见性

此外：

- 阻塞的线程，检查到中断标志位为true，则抛出InterruptedException，**并清除中断标志**

  > 可以用来中断阻塞，即结束阻塞状态

常用的使用InterruptedException的方法：

- 传递InterruptedException，更高层来处理中断
- 恢复中断，进行某些处理后，将中断位再次置为true

### 5.4. 同步工具类

> 特点：封装一些状态，通过状态决定线程是等待还是继续执行；提供了一些操作状态的方法

- Semaphore：信号量
- Barrier：栅栏
- Latch：闭锁

#### 闭锁

可以延迟线程的进度直到闭锁到达终止状态；相当于一扇门，只有在某个特定条件（闭锁终止）下才开

常用场景：

- 确保某个运算在所有需要的资源都初始化之后再进行
- 确保某个服务在它的所有前置服务完成之后再启动
- 等待操作者全部就绪再进行某操作

经典实现：CountDownLatch

- countDown：计数器减一
- await：等待计数器到0

> 场景：实现5人开黑，每个人网速随机（1~5s进入游戏），所有人就绪后才能开始游戏

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(5);
        System.out.println("进入游戏");
        for(int i = 0; i < 5; i++){
            new Thread(new PlayerTask(1 + i, latch)).start();
        }
        latch.await();
        System.out.println("开始游戏");
    }
}
class PlayerTask implements Runnable{
    private int count;
    private CountDownLatch latch;
    PlayerTask(int count, CountDownLatch latch){
        this.count = count;
        this.latch = latch;
    }
    void ready(){
        try {
            Thread.sleep(1000 * (1 + new Random().nextInt(5)));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("玩家" + count + "准备就绪！");
    }
    @Override
    public void run() {
        ready();
        //  准备事件完成，countDown闭锁
        latch.countDown();
    }
}
```

这么写貌似更科学...

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch startGate = new CountDownLatch(1), endGate = new CountDownLatch(5);
        for(int i = 0; i < 5; i++){
            new Thread(new PlayerTask(1 + i, startGate, endGate)).start();
        }
        System.out.println("游戏资源初始化......");
        Thread.sleep(5000);
        //  告诉玩家，初始化完成了
        startGate.countDown();
        System.out.println("进入游戏");
        endGate.await();
        System.out.println("开始游戏");
    }
}
class PlayerTask implements Runnable{
    private int count;
    private CountDownLatch startGate, endGate;
    public PlayerTask(int count, CountDownLatch startGate, CountDownLatch endGate) {
        this.count = count;
        this.startGate = startGate;
        this.endGate = endGate;
    }
    void ready(){
        try {
            Thread.sleep(1000 * (1 + new Random().nextInt(5)));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("玩家" + count + "准备就绪！");
    }
    @Override
    public void run() {
        //  等待资源初始化
        try {
            System.out.println("玩家" + count + "等待......");
            startGate.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //  进入游戏并准备
        ready();
        //  准备事件完成，countDown闭锁
        endGate.countDown();
    }
}
```

#### FutureTask

FutureTask相当于可生成结果的Runnable，也可实现闭锁

Future.get的结果取决于任务的状态，如果完成，立刻返回结果；否则阻塞至结果完成

- 实现闭锁

```java
import java.util.Random;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        FutureTask<Integer>[] tasks = new FutureTask[5];
        FutureTask maingame = new FutureTask(new MainGame());
        for(int i = 0; i < 5; i++){
            tasks[i] = new FutureTask(new PlayerTask(1 + i, maingame));
            new Thread(tasks[i]).start();
        }
        new Thread(maingame).start();
        //  锁住主线程
        maingame.get();
        for(int i = 0; i < 5; i++){
            //  所有玩家准备就绪后才能开始游戏
            tasks[i].get();
        }
        System.out.println("开始游戏");
    }
}
class MainGame implements Callable<Integer>{
    void init() throws InterruptedException {
        System.out.println("游戏资源初始化......");
        Thread.sleep(5000);
        System.out.println("进入游戏");
    }
    @Override
    public Integer call() throws Exception {
        init();
        return 1;
    }
}
class PlayerTask implements Callable<Integer> {
    private int count;
    private FutureTask<Integer> maingame;
    public PlayerTask(int count, FutureTask maingame) {
        this.count = count;
        this.maingame = maingame;
    }
    void ready(){
        try {
            Thread.sleep(1000 * (1 + new Random().nextInt(5)));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("玩家" + count + "准备就绪！");
    }
    @Override
    public Integer call() throws ExecutionException, InterruptedException {
        //  用maingame.get锁住玩家线程
        maingame.get();
        System.out.println("玩家" + count + "等待......");
        ready();
        return 1;
    }
}
```

- 实现异步

```java
import java.util.Random;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask futureTask = new FutureTask(new Task());
        new Thread(futureTask).start();
        Thread.sleep(new Random().nextInt(1000));
        System.out.println("其他任务进行中");
        //  异步执行两个任务
        futureTask.get();
        System.out.println("所有任务完成");
    }
}
class Task implements Callable<Integer> {
    @Override
    public Integer call() throws RuntimeException, InterruptedException {
        Thread.sleep(new Random().nextInt(1000));
        System.out.println("睡眠任务已完成");
        return 1;
    }
}
```

#### 信号量

Semaphore：可用于实现不可重入的互斥锁；可用于实现资源池；可用于实现有界阻塞容器

> 通过信号量自己构建一个阻塞队列，只实现插入取出两个方法

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        MyArrayBlockingQueue<Integer> queue = new MyArrayBlockingQueue(3);
        new Thread(){
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                    System.out.println(queue.poll());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
        queue.offer(1);
        queue.offer(2);
        queue.offer(3);
        //  此处会阻塞，直到另一个线程取走一个数据
        queue.offer(4);
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        System.out.println(queue.poll());
    }
}
class MyArrayBlockingQueue<E>{
    final private Lock lock = new ReentrantLock();
    final private Semaphore semaphore;
    private E[] array;
    final private Integer capacity;
    private Integer size = 0;
    MyArrayBlockingQueue(Integer capacity){
        this.capacity = capacity;
        array = (E[]) new Object[capacity];
        semaphore = new Semaphore(capacity);
    }
    public E poll(){
        lock.lock();
        if(isEmpty()){
            lock.unlock();
            throw new RuntimeException("队列为空");
        }
        lock.unlock();
        semaphore.release();
        lock.lock();
        E result = array[0];
        E[] arr = (E[]) new Object[capacity];
        System.arraycopy(array, 1, arr, 0, --size);
        array = arr;
        lock.unlock();
        return result;
    }
    public void offer(E val) throws InterruptedException {
        semaphore.acquire();
        lock.lock();
        array[size++] = val;
        lock.unlock();
    }
    public boolean isEmpty(){
        return size == 0;
    }
    public int size(){
        return size;
    }
}
```

> release不会阻塞

#### 栅栏

CyclicBarrier：所有线程都到达栅栏处，栅栏才会打开；中断或者设置超时时间可以结束await的阻塞状态并抛出BrokenBarrierException

> 场景：等餐桌上所有人吃完之后，商家计算每个人吃了多少，计算总价格，然后吃的最慢的那个人付钱

```java
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicInteger;

public class Test {
    public static void main(String[] args) {
        AtomicInteger sum = new AtomicInteger(0);
        CyclicBarrier barrier = new CyclicBarrier(5, new Runnable() {
            //  打开栅栏后的操作
            @Override
            public void run() {
                for(int i = 0; i < 5; i++){
                    sum.addAndGet(Consumer.all[i]);
                }         
                System.out.println(Thread.currentThread().getName() + "结账：共" + sum + "元");
            }
        });
        for(int i = 0; i < 5; i++){
            new Thread(new Consumer(i, barrier)).start();
        }
    }
}
class Consumer implements Runnable{
    static Integer[] all = new Integer[5];
    private Integer index;
    final private CyclicBarrier barrier;
    public Consumer(Integer index, CyclicBarrier barrier) {
        this.index = index;
        this.barrier = barrier;
    }
    @Override
    public void run() {
        Thread.currentThread().setName("食客" + index);
        all[index] = new Random().nextInt(100);
        System.out.println("食客" + index + "吃完了");
        //  等待其他食客吃完
        try {
            barrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

和CountDownLatch的区别：

- CountDownLatch等待某指定的事件发生，CyclicBarrier等待所有线程完成任务
- CountDownLatch通过cutDown手动减计数器，CyclicBarrier是通过线程完成自动的减
- CountDownLatch是一次性的，CyclicBarrier是循环利用的

#### Exchanger

在exchanger点，等两个线程都到达该点时交换线程数据，如果一个到了另一个没到，则到的那一个会等另一个到达

> 场景：一手交钱，一手交货

```java
import java.util.Random;
import java.util.concurrent.Exchanger;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        Exchanger<String> exchanger = new Exchanger<>();
        Buyer buyer = new Buyer(exchanger);
        Seller seller = new Seller(exchanger);
        new Thread(buyer).start();
        new Thread(seller).start();
    }
}
class Business{
    final Exchanger<String> exchanger;
    String data = "";
    public Business(Exchanger<String> exchanger) {
        this.exchanger = exchanger;
    }
    void doBusiness(){
        System.out.println(Thread.currentThread().getName() + data + "已准备好");
        try {
            Thread.sleep(new Random().nextInt(3000));
            data = exchanger.exchange(data);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "拿到" + data);
    }
}
class Seller extends Business implements Runnable{
    public Seller(Exchanger<String> exchanger) {
        super(exchanger);
        data = "货";
    }
    @Override
    public void run() {
        doBusiness();
    }
}
class Buyer extends Business implements Runnable{
    public Buyer(Exchanger<String> exchanger) {
        super(exchanger);
        data = "钱";
    }
    @Override
    public void run() {
        doBusiness();
    }
}
```

### 5.5. 构建高效且可伸缩的缓存

> 场景：有这样一个复杂计算方法
>
> ```java
> interface Computable<K, V>{
>     V compute(K args) throws InterruptedException;
> }
> class ExpensiveFunction implements Computable<String, BigInteger>{
>     @Override
>     public BigInteger compute(String args) throws InterruptedException {
>         //  用sleep模拟一个需要耗时很久的计算
>         Thread.sleep(new Random().nextInt(1000));
>         return new BigInteger(args);
>     }
> }
> ```
>
> 考虑该计算耗时很久，加入缓存技术

1. 同步方法和hashmap构建的缓存

```java
import java.math.BigInteger;
import java.util.HashMap;
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        Computable calculator = new ExpensiveFunction(), cache1 = new Cache1(calculator);
        Long start = System.currentTimeMillis(), end;
        CountDownLatch latch = new CountDownLatch(5);
        //  多个线程同时进行计算
        for(int i = 0; i < 5; i++){
            new Thread("任务" + i){
                @Override
                public void run() {
                    try {
                        String arg = String.valueOf(new Random().nextInt(5));
                        System.out.println(getName() + "针对参数" + arg + "的计算结果：" + cache1.compute(arg));
                        latch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
        latch.await();
        end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start) + "ms");
    }
}
interface Computable<K, V>{
    V compute(K args) throws InterruptedException;
}
class ExpensiveFunction implements Computable<String, BigInteger>{
    @Override
    public BigInteger compute(String args) throws InterruptedException {
        //  用sleep模拟一个需要耗时很久的计算(2s)
        Thread.sleep(2000);
        return new BigInteger(args);
    }
}
class Cache1<K, V> implements Computable<K, V> {
    final private Computable<K, V> calculator;
    final private HashMap<K, V> cache = new HashMap<>();
    public Cache1(Computable<K, V> calculator) {
        this.calculator = calculator;
    }
    @Override
    public synchronized V compute(K args) throws InterruptedException {
        //  如果缓存中有，直接返回
        if(cache.containsKey(args)){
            return cache.get(args);
        }
        //  如果缓存中没有，则通过代理模式计算结果
        V result = this.calculator.compute(args);
        cache.put(args, result);
        return result;
    }
}
```

缺点在于，compute方法整个被锁住，如果一个线程正在计算，另一个线程就无法调用compute方法，必须等计算完成

![image-20210601091944292](imgs\28.png)

2. 使用ConcurrentHashMap作为缓存，由于它是线程安全的，因此多个线程可并发地访问它而不用加锁

```java
import java.math.BigInteger;
import java.util.HashMap;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        Computable calculator = new ExpensiveFunction(), cache2 = new Cache2(calculator);
        Long start = System.currentTimeMillis(), end;
        CountDownLatch latch = new CountDownLatch(5);
        //  多个线程同时进行计算
        for(int i = 0; i < 5; i++){
            new Thread("任务" + i){
                @Override
                public void run() {
                    try {
                        String arg = String.valueOf(new Random().nextInt(5));
                        System.out.println(getName() + "针对参数" + arg + "的计算结果：" + cache2.compute(arg));
                        latch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
        latch.await();
        end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start) + "ms");
    }
}
interface Computable<K, V>{
    V compute(K args) throws InterruptedException;
}
class ExpensiveFunction implements Computable<String, BigInteger>{
    @Override
    public BigInteger compute(String args) throws InterruptedException {
        //  用sleep模拟一个需要耗时很久的计算(2s)
        Thread.sleep(2000);
        return new BigInteger(args);
    }
}
class Cache2<K, V> implements Computable<K, V> {
    final private Computable<K, V> calculator;
    final private ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
    public Cache2(Computable<K, V> calculator) {
        this.calculator = calculator;
    }
    @Override
    public V compute(K args) throws InterruptedException {
        //  如果缓存中有，直接返回
        if(cache.containsKey(args)){
            return cache.get(args);
        }
        //  如果缓存中没有，则通过代理模式计算结果
        V result = this.calculator.compute(args);
        cache.put(args, result);
        return result;
    }
}
```

缺点在于如果某个线程正在进行对于某个参数的计算，而另一个线程不知道，又重新进行对于相同参数的计算过程，会大大降低效率，有没有方法可以告诉其他线程，“我正在进行这个计算，你等着拿结果就可以了”

3. 使用FutureTask来存储缓存的Value，如果缓存没有则等待计算结果，如果有则直接返回

```java
import java.math.BigInteger;
import java.util.HashMap;
import java.util.Random;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        Computable calculator = new ExpensiveFunction(), cache3 = new Cache3(calculator);
        Long start = System.currentTimeMillis(), end;
        CountDownLatch latch = new CountDownLatch(5);
        //  多个线程同时进行计算
        for(int i = 0; i < 5; i++){
            new Thread("任务" + i){
                @Override
                public void run() {
                    try {
                        String arg = String.valueOf(new Random().nextInt(5));
                        System.out.println(getName() + "针对参数" + arg + "的计算结果：" + cache3.compute(arg));
                        latch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
        latch.await();
        end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start) + "ms");
    }
}
interface Computable<K, V>{
    V compute(K args) throws InterruptedException;
}
class ExpensiveFunction implements Computable<String, BigInteger>{
    @Override
    public BigInteger compute(String args) throws InterruptedException {
        //  用sleep模拟一个需要耗时很久的计算(2s)
        Thread.sleep(2000);
        return new BigInteger(args);
    }
}
class Cache3<K, V> implements Computable<K, V> {
    final private Computable<K, V> calculator;
    final private ConcurrentHashMap<K, Future<V>> cache = new ConcurrentHashMap<>();
    public Cache3(Computable<K, V> calculator) {
        this.calculator = calculator;
    }
    @Override
    public V compute(K args) throws InterruptedException {
        //  如果缓存中没有，先把一个FutureTask的任务放进缓存，再通过代理模式计算结果
        Future<V> future = cache.get(args);
        if(future == null){
            FutureTask<V> futureTask = new FutureTask<V>(new Callable<V>() {
                @Override
                public V call() throws Exception {
                    return calculator.compute(args);
                }
            });
            future = futureTask;
            cache.put(args, future);
            futureTask.run();
        }
        try {
            return future.get();
        } catch (ExecutionException e) {
            e.printStackTrace();
            throw new RuntimeException("出现错误");
        }
    }
}
```

这种方法已经比较完美了

![image-20210601093754256](imgs\29.png)

缺点在于compute中的操作是非原子性的，存在“先检查后操作”的过程，还是会出现同时计算

```java
import java.math.BigInteger;
import java.util.HashMap;
import java.util.Random;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws InterruptedException {
        Computable calculator = new ExpensiveFunction(), cache4 = new Cache4(calculator);
        Long start = System.currentTimeMillis(), end;
        CountDownLatch latch = new CountDownLatch(5);
        //  多个线程同时进行计算
        for(int i = 0; i < 5; i++){
            new Thread("任务" + i){
                @Override
                public void run() {
                    try {
                        String arg = String.valueOf(new Random().nextInt(5));
                        System.out.println(getName() + "针对参数" + arg + "的计算结果：" + cache4.compute(arg));
                        latch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
        latch.await();
        end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start) + "ms");
    }
}
interface Computable<K, V>{
    V compute(K args) throws InterruptedException;
}
class ExpensiveFunction implements Computable<String, BigInteger>{
    @Override
    public BigInteger compute(String args) throws InterruptedException {
        //  用sleep模拟一个需要耗时很久的计算(2s)
        Thread.sleep(2000);
        return new BigInteger(args);
    }
}
class Cache4<K, V> implements Computable<K, V> {
    final private Computable<K, V> calculator;
    final private ConcurrentHashMap<K, Future<V>> cache = new ConcurrentHashMap<>();
    public Cache4(Computable<K, V> calculator) {
        this.calculator = calculator;
    }
    @Override
    public V compute(K args) throws InterruptedException {
        while (true) {
            //  如果缓存中没有，先把一个FutureTask的任务放进缓存，再通过代理模式计算结果
            Future<V> future = cache.get(args);
            if (future == null) {
                FutureTask<V> futureTask = new FutureTask<V>(new Callable<V>() {
                    @Override
                    public V call() throws Exception {
                        return calculator.compute(args);
                    }
                });
                future = cache.putIfAbsent(args, futureTask);
                if (future == null) {
                    future = futureTask;
                    futureTask.run();
                }
                futureTask.run();
            }
            try {
                return future.get();
            } catch (CancellationException e) {
                //  计算出现错误后，移除，进入while循环重新计算
                cache.remove(args, future);
            } catch (ExecutionException e) {
                e.printStackTrace();
                throw new RuntimeException("出现错误");
            }
        }
    }
}
```

采用putIfAbsent的原子操作，确保不会同时计算

## 6. 任务执行

### 6.1. 在线程中执行任务

> 任务边界：常用的Web服务器、邮件服务器等，都将独立的客户请求作为边界

#### 串行执行任务

```java
public class Test {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true){
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    }
}
```

对每个请求单独串行地处理，服务器资源利用率非常低

#### 显式为任务创建线程

```java
public class Test {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true){
            final Socket connection = socket.accept();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    handleRequest(connection);
                }
            }).start();
        }
    }
}
```

**无限制创建线程的不足**

- 线程生命周期的开销巨大：销毁与创建等
- 资源消耗：线程数大于cpu处理能力，会产生大量闲置线程
- 稳定性：破话JVM的限制，导致OOM错误等等

### 6.2. Executor框架

![image-20210601110859593](imgs\30.png)

> 将任务的提交与执行解耦

#### 基于Executor的Web服务器实现

```java
public class Test {
    static final int NTHREAD = 10;
    public static void main(String[] args) throws IOException {
        Executor executor = Executors.newFixedThreadPool(NTHREAD);
        ServerSocket socket = new ServerSocket(80);
        while (true){
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                @Override
                public void run() {
                    handleRequest(connection);
                }
            };
            executor.execute(task);
        }
    }
}
```

#### 自定义Executor

```java
class ThreadPerTaskExecutor implements Executor{
    @Override
    public void execute(Runnable command) {
        new Thread(command).start();
    }
}
```

```java
class WithinThreadExecutor implements Executor{
    @Override
    public void execute(Runnable command) {
        command.run();
    }
}
```

#### 线程池

> 可以调用Executors的静态工厂方法之一来获得不同的线程池，通过阻塞队列

- newFixedThreadPool：阻塞队列LinkedBlockingQueue实现，固定长度的线程池
- newCachedThreadPool：SynchronousQueue实现，线程池规模不存在限制
- newSingleThreadExecutor：阻塞队列LinkedBlockingQueue实现，创建单个工作线程来执行任务，如果异常，则创建另一个来替代
- newScheduledThreadPool：添加延时功能

#### Executor的生命周期

> 所有任务线程（非守护线程）执行完毕之后才会退出
>
> Executor的生命周期是不可见的，可能正常退出也可能异常退出

为了解决生命周期的问题，Executor扩展了ExecutorService接口，包含运行、关闭和已终止状态

- shutdown：不再接受新任务，同时等待已有的线程完成后关闭
- shutdownNow：直接关闭
- awaitTermination：等待ExecutorService关闭
- isTerminated：判断是否关闭

> ExecutorService关闭后，新提交的任务会由“拒绝执行处理器（Rejected Execution Handler）”来处理，会抛弃任务，或通过execute抛出一个RejectedExecutionException

#### 延迟任务与周期任务

Timer执行任务时只能运行一个线程，且并不捕获异常，当定时任务抛出未检查异常时，Timer会错误地认为整个Timer都被取消了，也不会恢复线程的执行，新的任务也不能被调度，称之为“线程泄漏”

![image-20210602093255568](imgs\31.png)

> 第二个任务并未执行，而是抛出“Timer already cancelled”

可以使用ScheduledThreadExecutor，是基于DelayQueue实现的，DelayQueue中包含Delayed对象，有自己相应的延迟时间，逾期才能从队列中take

```java
public class Test {
    public static void main(String[] args) {
        ScheduledThreadPoolExecutor scheduledFuture = new ScheduledThreadPoolExecutor(5);
        scheduledFuture.schedule(new Runnable() {
            @Override
            public void run() {
                System.out.println("hahaha");
                //  这时就不会取消整个Timer了
                throw new RuntimeException();
            }
        }, 1, TimeUnit.SECONDS);
        scheduledFuture.schedule(new Runnable() {
            @Override
            public void run() {
                System.out.println("hahaha");
                throw new RuntimeException();
            }
        }, 5, TimeUnit.SECONDS);
    }
}
```

- scheduleAtFixedRate()

- scheduleWithFixedDelay()

> 注意：都是基于相对时间的

#### Future和Callable

Future和Callable能够实现对线程结果的获取和线程生命周期的控制，Future的默认实现为FutureTask，以ThreadPoolExecutor为例

```java
public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(5);
        Task task = new Task();
        Future future = service.submit(task);
        future.get();
        System.out.println(future.getClass());
        service.shutdown();
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        return 1;
    }
}
```

ThreadPoolExecutor没有submit方法，所以submit方法是父类AbstractExecutorService的方法

![image-20210602214726123](imgs\32.png)

首先调用newTaskFor方法，返回一个继承RunnableFuture接口的对象

![image-20210602215007209](imgs\33.png)

这是一个钩子方法，默认返回FutureTask；可以自己重写子类修改

> 钩子方法：父类钩子可以在定义时就挂着东西（父类可有实现），可以在后来看情况挂上别的东西（子类可重写），也可以总是不挂任何东西（父类中无实现，并且子类中未重写或重写无实现）

对于Runnable，也可以submit，只不过会先通过newTaskFor(Runnable command)及一系列方法包装成Callable类，execute调用的是子类ThreadPoolExecutor中的方法，最终会调用到call或run方法

#### CompletionServiceExecutor和BlockingQueue

> 场景：多个线程等待若干秒后各生成一个数字，将返回的数字用于下一步任务

1. 利用Future.get

```java
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(5);
        Future<Integer>[] futures = new Future[5];
        Long start = System.currentTimeMillis(), end;
        //  数字等待的时间，可以看到，最耗时的任务需要等待5000ms
        int[] waitTime = new int[]{5000, 800, 500, 200, 50};
        for(int i = 0; i < 5; i++){
            futures[i] = service.submit(new Task(waitTime[i]));
        }
        int[] result = new int[5];
        //  新开一个线程，开始复杂计算
        new FutureTask<Void>(new HardCompute(result)).run();
        //  轮询get的话，必须等到第一个任务，也就是耗时最长的5000的任务执行完之后，再获得后面的结果
        for(int i = 0; i < 5; i++){
            result[i] = futures[i].get();
        }
        service.shutdown();
        end = System.currentTimeMillis();
        System.out.println("耗时: " + (end - start) + "ms");
    }
}
class Task implements Callable<Integer>{
    final private int num;
    public Task(int num) {
        this.num = num;
    }
    @Override
    public Integer call() throws Exception {
        Thread.sleep(num);
        System.out.println(Thread.currentThread().getName() + "的数字: " + num);
        return num;
    }
}
class HardCompute implements Callable<Void>{
    final private int[] result;
    public HardCompute(int[] result) {
        this.result = result;
    }
    @Override
    public Void call() throws Exception {
        Set<Integer> set = new HashSet<>();
        while(true){
            Thread.sleep(20);
            for(int i = 0; i < 5; i++){
                if(result[i] != 0 && !set.contains(result[i])){
                    set.add(result[i]);
                    System.out.println(result[i] + "开始下一步");
                }
            }
            if(set.size() == 5) break;
        }
        return null;
    }
}
```

![image-20210603005824768](imgs\35.png)

分析结果，只有等待耗时最长的任务的结果出来以后，才能获得其他任务的结果，并开始下一步任务

现在用CompletionService包装一下

```java
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(10);
        CompletionService<Integer> ecs = new ExecutorCompletionService(service);
        Future<Integer>[] futures = new Future[5];
        Long start = System.currentTimeMillis(), end;

        //  生成数字等待的时间，可以看到，最耗时的任务需要等待5000ms
        int[] waitTime = new int[]{5000, 800, 500, 200, 50};
        //  开始生成数字
        for(int i = 0; i < 5; i++){
            futures[i] = ecs.submit(new Task(waitTime[i]));
        }

        int[] result = new int[5];
        //  新开一个线程，开始将生成好的数字用于下一步任务
        FutureTask nextMission = new FutureTask<Void>(new NextMission(result));
        service.submit(nextMission);
        //  轮询get的话，必须等到第一个任务，也就是耗时最长的5000的任务执行完之后，再获得后面的结果
        for(int i = 0; i < 5; i++){
            result[i] = ecs.take().get();
        }
        nextMission.get();
        service.shutdown();
        end = System.currentTimeMillis();
        System.out.println("耗时: " + (end - start) + "ms");
    }
}
class Task implements Callable<Integer>{
    final private int num;
    public Task(int num) {
        this.num = num;
    }
    @Override
    public Integer call() throws Exception {
        Thread.sleep(num);
        System.out.println(Thread.currentThread().getName() + "生成数字: " + num);
        return num;
    }
}
class NextMission implements Callable<Void>{
    final private int[] result;
    public NextMission(int[] result) {
        this.result = result;
    }
    @Override
    public Void call() throws Exception {
        Set<Integer> set = new HashSet<>();
        while(true){
            Thread.sleep(20);
            for(int i = 0; i < 5; i++){
                if(result[i] != 0 && !set.contains(result[i])){
                    set.add(result[i]);
                    System.out.println(result[i] + "开始下一步");
                }
            }
            if(set.size() == 5) break;
        }
        return null;
    }
}
```

先完成的任务可以提前进入下一步，大大提升了资源利用率和效率

> 分析下源码，ExecutorCompletionService会用一个阻塞队列QueueingFuture来存储submit的任务，并重写FutureTask的done方法，当任务完成后，加入到completionQueue里面，它是一个BlockingQueue

![image-20210603011240844](D:\笔记\秋招\docs\J.U.C\imgs\36.png)

#### 给任务设置时限

```java
import java.util.concurrent.*;

import static java.util.concurrent.TimeUnit.SECONDS;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        ExecutorService service = Executors.newFixedThreadPool(5);
        Future f = service.submit(new FutureTask<Integer>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                Thread.sleep(3000);
                return 123;
            }
        }));
        try {
            System.out.println(f.get(1, SECONDS));
        }catch (TimeoutException e){
            System.out.println("超时，任务取消");
            f.cancel(true);
        }
        service.shutdown();
    }
}
```

- invokeAll：为所有线程设置统一的超时参数，并保存结果

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

import static java.util.concurrent.TimeUnit.SECONDS;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        ExecutorService service = Executors.newFixedThreadPool(5);
        List<Task> tasks = new ArrayList<>();
        tasks.add(new Task(3000));
        tasks.add(new Task(1000));
        List<Future<Integer>> results = service.invokeAll(tasks, 2, SECONDS);
        for(Future result: results){
            if(!result.isCancelled()){
                System.out.println(result.get());
            }
        }
        service.shutdown();
    }
}
class Task implements Callable<Integer>{
    final private Integer waitTime;
    public Task(Integer waitTime) {
        this.waitTime = waitTime;
    }
    @Override
    public Integer call() throws Exception {
        Thread.sleep(waitTime);
        return 123;
    }
}
```

## 7. 取消与关闭



## 参考

[1] JAVA并发编程实战

[2] [https://blog.csdn.net/weixin_38106322/article/details/105745555——绅士jiejie](https://blog.csdn.net/weixin_38106322/article/details/105745555)

[3] [https://zhuanlan.zhihu.com/p/52853855——辞慾](https://zhuanlan.zhihu.com/p/52853855)

[4] [https://www.cnblogs.com/jiading/articles/12589275.html——别再闹了](https://www.cnblogs.com/jiading/articles/12589275.html)

[5] [https://www.cnblogs.com/daxin/p/3364014.html——大新博客](https://www.cnblogs.com/daxin/p/3364014.html)

[6] [https://segmentfault.com/q/1010000007900854——逆则贤](https://segmentfault.com/q/1010000007900854)

[7] [https://www.cnblogs.com/fsmly/p/11020641.html——风沙迷了眼](https://www.cnblogs.com/fsmly/p/11020641.html)

[8] [https://www.zhihu.com/question/341005993](https://www.zhihu.com/question/341005993)

[9] [https://blog.csdn.net/qq_27469549/article/details/80718931——坑铿吭](https://blog.csdn.net/qq_27469549/article/details/80718931)

[10] [https://www.jianshu.com/p/23e5866f308e——aworker](https://www.jianshu.com/p/23e5866f308e)

[11] [https://blog.csdn.net/pange1991/article/details/80944797——潘建南](https://blog.csdn.net/pange1991/article/details/80944797)