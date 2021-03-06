# [返回](/)

# 八股文背诵版之—Java基础篇

## final

### :point_right:**讲一下final？**

final是Java的关键字，可以用来修饰类、方法和变量，被final修饰的类是不可以被继承的，被final修饰的方法不可以被重写，被final修饰的变量的值不可以改变。除此之外，final还具有两个语义：第一个是，创建对象时，初始化写final域，与将对象地址赋给一个引用变量，这两个操作之间不能重排序；第二个是，初次读包含final域的对象引用，与读取这个final域，这两个操作之间不能重排序

实现第一个规则是在final域写之后，构造方法结束前插入一个StoreStore屏障，令写入final域对其他cpu可见，且阻止了重排序；第二个规则是在读final域之前插入一个LoadLoad屏障，阻止了在读取对象引用前读取final域的重排序

x86架构下，LoadLoad和StoreStore都是空操作，用到这两种内存屏障的场景中，x86架构下处理器不会进行重排序，因此实际并没有给final插入内存屏障

> JIT和处理器都可能进行重排序；内存屏障的具体实现是在**硬件层面的指令**，一般有lock、mfence、cpuid等

> 更细的：实际上现有cpu无法重排序的，而是storeBuffer和invaildQueue实现的，而x86的stroreBuffer是FIFO的，也没有invalidQueue

### :point_right:**final、finally、finalize的区别？**

final是Java的关键字，用来修饰类时，类不能被继承；用来修饰方法时，方法不能被重写；用来修饰变量时，变量的值不能被修改。finally也是Java的关键字，用在异常捕获处理中，在try-catch-finally中，被finally包住的代码块一定会被执行，可以用来做一些资源清理的工作；在return后的finally会先执行完再return，finally中的return会覆盖前面的return。finalize是Java中的方法，在GC过程中会被低优先级的Finalizer线程调用，作为GC流程的一部分

### :point_right:**为什么方法内部类使用方法的变量时，这个变量一定要是final的？**

两者生命周期不一致，内部类在方法结束后对应栈帧出栈后依然存在，而局部变量在方法结束后就会销毁，如果在内部类使用一个已经销毁的变量，肯定是不可以的，因此将变量令为final的，编译时会传入内部类作为内部类的一个成员变量而存在

## String&StringBuffer&StringBuilder

### :point_right:**String为什么不可变？**

首先，String是被final修饰的，是不能被继承的，因此不能通过继承它来重写覆盖它的方法；其次，String底层的byte数组是用final修饰的，不能指向其他数组；最后是最关键的，String中对底层数组采用private修饰，封装并提供给用户的涉及到修改的方法都会仔细避免修改底层数组，而是返回一个新的String对象，因此说String是不可变的

### :point_right:**String、StringBuffer、StringBuilder的区别？**

1. String底层数组用final修饰，指向的内存地址是不可变的；另外两个是可变的，数组容量不够时会扩容，但对象仍旧可以是之前的StringBuider或StringBuffer对象，而String都会返回一个全新的对象
2. 对String的修改操作都会生成一个新的String，因此在多线程下可以看作是安全的；StringBuffer的操作用synchronized修饰，是线程安全的；StringBuilder不是线程安全的，单线程下使用，效率较高

### :point_right:**new String("123")产生几个对象？**

产生两个对象。首先是字面量“123”，解释器运行到这一行，*将执行ldc指令，目的是将运行时常量池中的对应索引指向的目标推到栈顶，如果发现对应索引位置字面量未完成解析，（字面量是lazy_resolve的，会在ldc时才解析）*则会首先在堆中创建一个“123”的string对象，接着将引用驻留一份在堆中的stringtable里面；接着new String又会生成一个全新的对象

> JDK1.7之前字符串常量池在永久代，是存了一个相同的对象，而不是存引用

## 包装类

### :point_right:**为什么使用包装类？**

因为Java语言是面向对象的语言，而Java基本数据类型不是面向对象的，因此就不具备面向对象的一些特性，不能使用方法，也不能在容器内使用，因此对其进行包装，赋予其面向对象特性

### :point_right:**Integer a = 128、Integer b = 128，a == b是否为true？为什么？**

不为true，Integer a = 128是一个语法糖，实际上会被编译成Integer.valueOf(128)，这个方法在参数处于-128到127的时候会返回Integer.cache数组对应索引位置的值，这是一个静态数组，随着类的加载而加载，并在静态代码块中完成了赋值，因此在-128到127间的数都是从这个数组取，都相等；而在这个范围之外的数，使用new创建了一个对应的Integer对象并返回，因此==比较的是地址，而两个不同对象在堆上的地址肯定是不同的，所以为false

## 泛型

### :point_right:**讲一下泛型是什么？好处是什么？类型擦除知道吗？**

泛型，顾名思义是一种不确定的类型，在使用时根据传入的类型来确定具体是什么类型，比如集合容器中，用E来指代传入的类型，在编写代码时再指定具体的类型，这样这个集合容器就只能存储对应类型的对象。好处在于在编译期就提供了一种类型安全检查，以容器为例，确保运行期间容器中只存放对应类型的数据，就不会出现ClassCastException，取数据时也不用进行强制类型转换。类型擦除是指编写代码时指定的类型，在编译后是被完全擦除的，运行期间只保留原始类型，也就是说，只是提供了编译期的类型检查；这样做的目的是与老版本的API进行兼容，避免重新建立一套全新的泛型API

### :point_right:**限定通配符和非限定通配符？**

?是泛型中的非限定通配符，? extends T，? super T是限定通配符，分别表示只能处理T的子类类型或父类类型

### :point_right:**?和T的区别？**

1. ?不可以用于泛型类的定义，一般是在使用过程中用到?；而T用于泛型类的定义，在泛型类或泛型方法的使用中不可以用
2. T是指一种具体的类型；而?可以通配任意类型
3. 以List\<String> = new ArrayList\<String>()和List<?> = new ArrayList\<String>()为例，前者可以正常进行集合容器操作；后者在编译器捕获String类型后，会用capture#1这样的代号来代替，除了null以外不可以add任何元素

### :point_right:**PECS原则知道吗？**

PECS，也就是Producer Extends Consumer Super，也就是对于List<? extends T>，可以作为生产者，从里面取出来的元素一定是T的子类，反之往里存的话，由于不知道具体应该存T的哪个子类，因此是不可以的；对于List<? super T>，可以作为消费者，往里存T或T的子类一定是没问题的，反之取出来的时候不知道应该取T的哪个父类，用Object会导致类型丢失，因此是不可以的

## 反射

### :point_right:**反射是什么？讲一讲反射原理？**

反射是Java提供的这样一组操作：在运行期间判断一个对象所属的类，在运行期间获取类的内部信息，在运行期间根据类构造一个对象，在运行期间调用一个对象的方法

在运行期间通过对象获取一个类的原理在于，每次类加载时，会在堆空间生成一个对应的Class对象，调用getClass方法，可以通过对象的对象头的klass pointer获取到方法区的类元信息，也就是JVM的instanceKlass实例，其中有个字段\_java\_mirror指向堆中的Class对象，根据Class对象提供的方法就可以判断出所属的类了，并获取类的内部信息

在运行期间根据类构造对象，同样也是根据Class对象，首先通过getConstructor获取对应构造方法的拷贝，接着通过调用返回对象的newInstance方法，真正创建一个对象出来。实际上，newInstance方法中是通过返回对象中的constructorAccessor调用的，一开始调用者对象是DelegatingConstructorAccessorImpl，通过静态代理模式调用，传入的是NativeConstructorAccessorImpl，当反射调用次数大于15次时，会调用generate工厂方法动态生成一个ConstructorAccessorImpl的子类对象作为ca，方便JVM进行优化

运行期间调用的对象同理，首先根据getMethod获取方法对象的拷贝，接着在拷贝对象的invoke方法中通过methodAccessor，以静态代理模式调用NativeMethodAccessorImpl的invoke方法，当调用次数大于15，则动态生成一个MethodAccessImpl的子类实例来调用invoke

###  :point_right:**讲一讲invoke原理？**

invoke调用细节：

分为两部分：getMethod和invoke

- getMethod

  1. 检查安全性和权限*（Java SecurityManager，检查调用者是否安全）*
  2. 通过getMethod0找到对应的方法
     1. getMethod0中调用getMethodsRecursive找方法并加入方法链表
     2. getMethodsRecursive中具体调用privateGetDeclaredMethods来寻找，首先在ReflectionData的缓存里面寻找，找不到调用getDeclaredMethods0本地方法寻找
     3. 还是找不到，则递归调用父类的getMethodsRecursive，找到就加入方法链表，一直找不到就抛出异常
     4. 调用getMostSpecific在方法链表中找到返回值最具体的方法
  3. 返回找到的方法的拷贝，主要操作有将root变量指向原方法，复制methodAccessor的地址等等

- invoke

  1. 首先检查override变量是否为true，是则直接进入下一步，否则检查是否有调用权限，可以通过setAccessible改变这个变量的值

  2. 检查methodAccessor变量是否为null，是的话则通过工厂方法生成一个methodAccessorImpl的实例。一开始是静态模式的，返回的对象是DelegatingMethodAccessorImpl作为代理对象，代理的是NativeMethodAccessorImpl

  3. 通过methodAccessor调用invoke，调用次数大于15，会在generate方法内以动态生成字节码的方式生成一个methodAccessorImpl的实例，记得好像是magicMethodAccessorImpl，便于JVM优化

     > 为什么动态生成的便于优化？因为跨越native边界会对优化有阻碍作用，相当于一个黑箱

### :point_right:**静态代理、JDK代理、cglib代理的区别？手写？**

静态代理：利用多态性，代理类需要和原类继承同一个接口或代理类继承原类，通过传入原对象作为成员变量来间接调用并增强原对象的方法。和动态代理的区别在于：一，对于每一个要被代理的类，都需要写一个对应的代理类；二，对于原类的每一个方法，在代理类中都要重新写一个对应的代理方法，修改方法时也很麻烦；三，代理类在运行前就编译完成，运行时直接加载到内存，缺乏灵活性

JDK代理：是面向接口来实现的，利用Java的反射机制在运行期间生成一个继承同一接口的代理对象，并在invoke周围加上自己的代码逻辑来实现方法增强。相比静态代理，JDK代理不用为每个类单独创建一个代理类，也不用重新写每一个代理方法，在运行期创建代理对象，灵活性更高

cglib代理：面向类实现，不需要接口，通过底层封装的asm工具动态修改类的字节码并生成其子类，通过修改子类继承父类的方法来实现对父类方法的增强。比起JDK动态代理采用的反射，cglib是通过底层的asm动态修改字节码来实现的

> AspectJ是通过使用自带的acj编译器在编译期实现代理，本质上是静态代理，String AOP避免了使用acj编译器，只用了AspectJ的注解实现，底层用的是动态代理

**手写**

- 静态代理

```java
public class Testt {
    public static void main(String[] args) {
        // 原方法
        I i = new O();
        i.foo();
        System.out.println();
        // 代理方法
        I proxy = new Proxy(i);
        proxy.foo();
    }
}
interface I{
    void foo();
}
class O implements I{
    @Override
    public void foo(){
        System.out.println("running......");
    }
}
class Proxy implements I{
    I target;

    public Proxy(I target) {
        this.target = target;
    }

    @Override
    public void foo() {
        System.out.println("-----BEFORE-----");
        target.foo();
        System.out.println("-----AFTER-----");
    }
}
```

- JDK动态代理

```java
public class Testt {
    public static void main(String[] args) {
        // 原方法
        I i = new O();
        i.foo();
        System.out.println();
        // JDK动态代理
        I proxy = (I) Proxy.newProxyInstance(i.getClass().getClassLoader(), i.getClass().getInterfaces(), new JDKProxy(i));
        proxy.foo();
    }
}
interface I{
    void foo();
}
class O implements I{
    @Override
    public void foo(){
        System.out.println("running......");
    }
}
class JDKProxy implements InvocationHandler {
    Object target;

    public JDKProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("-----BEFORE-----");
        method.invoke(target, args);
        System.out.println("-----AFTER-----");
        return null;
    }
}
```

- CGLIB动态代理

```java
public class Testt {
    public static void main(String[] args) {
        // 原方法
        O o = new O();
        o.foo();
        System.out.println();
        // CGLIB动态代理
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(O.class);
        enhancer.setCallback(new CGLIBProxy());
        O proxy = (O) enhancer.create();
        proxy.foo();
    }
}
class O {
    public void foo(){
        System.out.println("running......");
    }
}
class CGLIBProxy implements MethodInterceptor{
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("-----BEFORE-----");
        methodProxy.invokeSuper(o, objects);
        System.out.println("-----AFTER-----");
        return null;
    }
}
```

## 异常

### :point_right:**Error和Exception的区别？**

Error通常是虚拟机运行过程中出现的相关错误，比如系统崩溃、堆栈溢出等等，编译器不对这类错误进行检查，Java应用程序也不应对这类错误进行捕获，一旦这类错误发生，通常应用程序会被终止；Exception类型可以被捕获并处理，应用程序可以继续正常运行

### :point_right:**受检异常和非受检异常的区别？**

非受检异常又称运行时异常，为RuntimeException及其子类，两者区别如下：

- 受检异常会在编译期间进行检查，非受检异常不会
- 受检异常会强制要求处理，非受检异常不要求强制处理





