# [返回](/)

# JVM概述

![在这里插入图片描述](imgs\0.png)

## 架构模型

**基于栈的指令架构**，无地址指令，精简指令集

相比于基于寄存器的指令，优点在于：

- 跨平台
- 指令集小
- 编译器易实现

缺点在于：

- 性能下降
- 同样的功能需要更多的指令

## 生命周期

### 启动

JVM的启动是通过引导类加载器（BootstrapClassLoader）创建一个初始类（initial class）来完成的，这个类是由JVM的具体实现指定的

### 执行

执行Java程序时，真正执行的是JVM进程

程序开始，就启动；程序结束，就停止

### 退出

- 程序正常执行结束
- 程序执行过程中出现异常或错误导致异常终止
- 操作系统出现错误导致JVM进程终止
- 某线程调用Runtime或System类的exit方法，或Runtime的halt方法，并且Java安全管理器也允许这次操作

# 1. 类加载子系统概述

## 类加载器子系统的作用

- 从文件系统或网络中加载Class文件，Class文件在文件开头有特定标识
- ClassLoader只负责类的加载，至于是否可运行，则由执行引擎（Execution Engine）决定
- 加载的类的信息存放在方法区，除了类的信息，方法区还会存放运行时常量池信息，可能还包括字符串字面量和数字常量

## 类的加载过程

### 装载（loading）

1. 根据类的全限定类名获取类的二进制字节流
2. 将字节流对应的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个相应的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

### 链接（linking）

#### 验证（verify）

1. 确保Class文件的字节流包含的信息符合虚拟机的要求，保证被加载类的正确性，不会危害虚拟机自身安全
2. 主要包括四种验证
   1. 文件格式验证
   2. 元数据验证
   3. 字节码验证
   4. 符号引用验证

![image-20210730184033046](imgs\1.png)

> class文件头的“CAFEBABE”

#### 准备（prepare）

为**类变量**在方法区中分配内存并设置该类变量的默认初始值，即零值

> 1. 不包括final修饰的类变量，这种会当作常量，编译期时就分配好，准备阶段会直接显式初始化
> 2. 不包括类成员变量，实例的变量会随着对象实例一起分配到java堆中，而类变量是在方法区

#### 解析（resolve）

将常量池内的符号引用转换成直接引用

### 初始化（initialization）

执行类构造器方法\<clinit>()，完成类变量的初始化以及静态代码块的执行

> 无需定义，javac编译器自动收集类中类变量的赋值动作和静态代码块中的动作并合并生成的
>
> 可以用idea的jclasslib插件查看
>
> ![image-20210802163824327](imgs\2.png)
>
> 如果没有这样的动作（没有类变量/类变量没有初始化/没有静态代码块），则不会生成clinit方法

> 一个小例子
>
> ```java
> public class Testt {
>     static {
>         num = 2;
>     }
>     private static int num = 1;
>     public static void main(String[] args) {
>         System.out.println(Testt.num);
>     }
> }
> ```
>
> 为什么可以这样写？num明明定义在静态代码块之下呀
>
> 回溯类加载过程就明白了：在linking的prepare过程中，类变量已经在方法区分配了内存并完成零值初始化
>
> 接着会在initialization的过程中，先调用静态代码块赋值成2，再显式初始化为1
>
> ![image-20210802164958500](imgs\3.png)
>
> 看字节码也是这样
>
> **注意**：这里不可以声明前使用该变量，报非法的前向引用，这是java规范中决定的，[链接](https://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.3.2.3)
>
> ![image-20210802165212685](imgs\4.png)

- **注意：**clinit是生成的方法，且只负责类变量的初始化和静态代码块的执行，类中定义的构造器方法对应的字节码方法是init

![image-20210802171955824](imgs\5.png)

- 如果有父类，会先执行父类的clinit
- 虚拟机保证多线程下clinit的同步加锁

## 类加载器

> 根据java官方规范，分为引导类加载器和自定义加载器两大类，其中，自定义加载器是指所有继承于ClassLoader类的加载器，包括AppClassLoader、ExtClassLoader*（JDK 9之后，PlatformClassLoader）*
>
> **注意：**Bootstrap ClassLoader、ExtClassLoader和AppClassLoader之间并不是继承关系，而是一种包含关系，可以用getParent方法获取
>
> 分别打印
>
> ```java
> ClassLoader classLoader = Test.class.getClassLoader();
> System.out.println(classLoader);
> System.out.println(classLoader.getParent());
> System.out.println(classLoader.getParent().getParent());
> 
> 结果：
> jdk.internal.loader.ClassLoaders$AppClassLoader@3fee733d
> jdk.internal.loader.ClassLoaders$PlatformClassLoader@b4c966a
> null  // 获取不到，是native的
> ```

### 引导类加载器

引导类加载器/启动类加载器/Bootstrap ClassLoader

- 使用C/C++实现，是JVM的一部分

- 用来加载Java的核心类库

  > JAVA_HOME/jre/lib/rt.jar、resources.jar或sun.boot.class.path路径下面的内容，用于提供JVM自身需要的类

- 不继承自ClassLoader，没有父加载器

- 加载扩展/平台类加载器和应用类加载器，并指定为他们的父加载器

- 出于安全考虑，只能加载包名以java、javax、sun等开头的类

### 扩展类加载器

扩展类加载器/Extension ClassLoader：ExtClassLoader

- Java实现
- 派生于ClassLoader类，父加载器为引导类加载器
- 从java.ext.dirs系统属性所指定的位置加载类库，或从JDK的安装目录/jre/lib/ext子目录加载
- 用户创建的jar若放在此目录下，也会由扩展类加载器加载

> JDK 9之后是平台类加载器，PlatformClassLoader
>
> 在JDK1.8及以前的版本中提供的加载器为“ ExtClassLoader”，因为在JDK的安装目录中提供了一个ext的目录，开发者可以将*.jar文件拷贝到此目录中，这样就可以直接执行了，但是这样的处理开发并不安全，最初的时候也是不提倡使用的，所以从JDK1.9开始将其彻底废除了，同时为了与系统类加载器和应用类加载器之间保持设计的平衡，提供有平台类加载器

### 应用程序类加载器

应用程序类加载器/系统类加载器：AppClassLoader

- Java实现
- 派生于ClassLoader类，父加载器为扩展/平台类加载器
- 从环境变量classpath所指定的位置加载类库，或从系统属性java.class.path加载
- 是程序中默认的类加载器
- 可通过ClassLoader.getSystemClassLoader()获取

### 自定义类加载器

为什么要自定义类加载器?

- 隔离加载类

  > 比如说，不同中间件在一个工程时，有可能路径和类名都一致，需要进行**类的仲裁**，这些中间件就会自己实现类加载器

- 修改类加载的方式

- 扩展加载源

- 防止源码泄漏

## 双亲委派机制

### 工作机制

- 当一个类加载器收到了加载类的请求，并不会自己去加载，而是把这个请求委托给它的父加载器
- 如果父加载器还有父加载器，则进一步向上委托，直到最顶层的引导类加载器
- 如果父加载器能够完成加载，则返回；如果加载不了，子加载器才会去尝试加载

```java
// JDK 11实现
protected Class<?> loadClassOrNull(String cn, boolean resolve) {
    synchronized (getClassLoadingLock(cn)) {
        // check if already loaded
        Class<?> c = findLoadedClass(cn);

        if (c == null) {

            // find the candidate module for this class
            LoadedModule loadedModule = findLoadedModule(cn);
            if (loadedModule != null) {

                // package is in a module
                BuiltinClassLoader loader = loadedModule.loader();
                if (loader == this) {
                    if (VM.isModuleSystemInited()) {
                        c = findClassInModuleOrNull(loadedModule, cn);
                    }
                } else {
                    // delegate to the other loader
                    c = loader.loadClassOrNull(cn);
                }

            } else {

                // check parent
                if (parent != null) {
                    c = parent.loadClassOrNull(cn);
                }

                // check class path
                if (c == null && hasClassPath() && VM.isModuleSystemInited()) {
                    c = findClassOnClassPathOrNull(cn);
                }
            }

        }
    }
}
```

### 优势

- 避免类的重复加载

- 保护程序安全，防止核心API被随意篡改

  > 沙箱安全机制

> JVM如何判断两个class对象是否是同一个类的？
>
> 1. 类的完整名必须一致，包括包名
> 2. 加载该类的类加载器必须一致

JVM必须知道一个类型是否由启动加载器加载的，如果不是，那么JVM会将这个类加载器的一个引用作为类型信息的一部分保存在方法区中*（其实很好理解，启动类加载器的引用是null）*。当解析一个类型到另一个类型的引用的时候，JVM需要保证这两个类型的类加载器是相同的

## 类的主动使用和被动使用

### 主动使用

- 创建类的实例

- 访问某个类或接口的静态变量，或赋值

- 调用类的静态方法

- 反射

  > 比如：Class.forName()

- 初始化一个类的子类

- Java虚拟机启动时被标明为启动类的类

- JDK 7开始提供的动态语言支持：java. lang. invoke .MethodHandle实例的解析结果REF_ getstatic. REF_ putstatic. REF_invokeStatic句柄对应的类没有初始化，则初始化

除上述外，都是被动使用，**不会进行类的加载中的初始化过程**

# 2. 运行时数据区

![image-20210802224111728](imgs\6.png)

红框部分为线程共享的部分（堆、堆外内存/非堆（方法区、代码缓存）），一个JVM进程一份；其他为线程独占的部分，一个线程一份

![image-20210802124111728](imgs\28.png)

> 绝大部分垃圾回收集中在堆区，少部分在方法区

> Runtime：运行时环境，饿汉式单例

## 线程

### 基本概念

- 线程是一个程序里的运行单元。JVM允许一个应用有多个线程并行的执行
- 在Hotspot JVM里，每个线程都与操作系统的本地线程直接映射。当一个Java线程准备好执行以后，此时一个操作系统的本地线程也同时创建。Java线程执行终止后，本地线程也会回收
- 操作系统负责所有线程的安排调度到任何一个可用的CPU上。一旦本地线程初始化成功，它就会调用Java线程中的run()方法

### 后台线程

- 虚拟机线程：这种线程的操作是需要JVM达到安全点才会出现。这些操作必须在不同的线程中发生的原因是他们都需要JVM达到安全点，这样堆才不会变化。这种线程的执行类型包括"stop-the-world"的垃圾收集，线程栈收集，线程挂起以及偏向锁撤销
- 周期任务线程：这种线程是时间周期事件的体现(比如中断)，他们一般用于周期性操作的调度执行
- GC线程：这种线程对在JVM里不同种类的垃圾收集行为提供了支持
- 编译线程：这种线程在运行时会将字节码编译成到本地代码
- 信号调度线程：这种线程接收信号并发送给JVM，在它内部通过调用适当的方法进行处理

## 2.1. 程序计数器（PC寄存器）

> 全称：程序计数寄存器，Program Counter Register

> 是线程独占的

以一个简单的main方法为例

```java
static int num;
public static void main(String[] args) {
    System.out.println(num);
}
```

利用javap -v xxx.class查看反解析（二进制字节流→可以看到的字节码指令）结果

![image-20210803213023097](imgs\7.png)

红框对应的就是指令地址，也叫偏移地址，存放在PC计数器中

执行引擎根据PC计数器所指向的地址，读取相应的指令并运行

> #对应常量池的内容
>
> ![image-20210803213323841](imgs\8.png)

### 作用

- 用来存储指向下一条指令的地址，由执行引擎去读取下一条指令
- 程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成
- 字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令

### 特点

- 是一块很小的内存空间，也是运行最快的存储空间
- 线程私有，声明周期与线程保持一致
- 任何时间一个线程只有一个方法在执行，即当前方法。程序计数器会存储当前线程正在执行的 Java方法的 JVM指令地址；如果是native方法，则为未指定值（undefined）
- 唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域

### 两个问题

#### 为什么要设置PC计数器

1. CPU需要不停的切换各个线程，这时候切换回来以后，就得知道接着从哪开始继续执行
2. JVM的字节码解释器需要通过改变PC寄存器的值来明确下一条应该执行什么样的字节码指令

所以，众多线程在并发执行过程中，任何一个确定的时刻，一个处理器或者多核处理器中的一个内核，只会执行某个线程中的一条指令。这样必然导致经常中断或恢复，如何保证分毫无差呢？每个线程在创建后，都会产生自己的程序计数器和栈帧，程序计数器在各个线程之间互不影响

#### 为什么它是线程私有的

1. CPU会不停做任务切换，这样必然会导致经常中断或恢复
2. 为了能够准确地记录各个线程正在执行的当前字节码指令地址，最好的办法自然是为每一个线程都分配一个PC寄存器，这样一来各个线程之间便可以进行独立计算，从而不会出现相互干扰的情况

## 2.2. 虚拟机栈

### 栈和堆

- 栈：处理程序的执行问题
- 堆：处理数据的存储问题

### 概述

- 线程私有的，每个线程创建时都会创建一个虚拟机栈，生命周期和线程一致
- 虚拟机栈内部保存一个个栈帧，对应的是一个个方法

#### 作用

- 主要负责Java程序的运行
  - 保存方法的局部变量、部分结果
  - 参与方法的调用和返回

#### 特点

- 快速有效，访问速度仅次于PC计数器
- JVM对栈的操作只有两个：①方法执行，伴随着进栈；②方法执行之后的出栈
- 不存在垃圾回收问题

#### 可能出现的异常

Java虚拟机规范允许Java栈的大小是动态的或者是固定不变的

- 如果采用固定大小的Java虚拟机栈，那每个线程的Java虚拟机栈容量可以在线程创建的时候独立选定。如果线程请求分配的栈容量超过Java虚拟机栈允许的最大容量，Java虚拟机将会抛出一个StackOverflowError异常
- 如果Java虚拟机栈可以动态扩展，并且在尝试扩展的时候无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的虚拟机栈，那Java虚拟机将会抛出一个OutOfMemoryError 异常

#### 运行原理

<img src="imgs\9.png" alt="image-20210803225118609" style="zoom: 50%;" />

每个活动线程中，每个时间点只有一个栈帧在活动，称为当前栈帧，对应的方法为当前方法

它是处于栈顶的。如果运行过程中调用了其他方法，则会新生成一个栈帧并压栈，成为新的当前栈帧

return指令和抛出异常都会导致栈被弹出

### 2.2.1. 局部变量表（local variables）

定义为一个数字数组，主要用于存储方法参数和定义在方法体内的局部变量，包括各类基本数据类型、对象引用以及returnAddress

> returnAddress 类型目前已经很少见了，它是为字节码指令 jsr、jsr_w 和 ret 服务的，指向了一条字节码指令的地址，很古老的 Java 虚拟机曾经使用这几条指令来实现异常处理，现在已经由异常表代替

- 建立在虚拟机栈上，线程私有，所以不存在线程安全问题
- 所需容量大小在编译期确定，保存在方法的Code属性的maximum local variables数据段，方法运行期间大小不会改变

以一个方法method()为例

```java
public int method(int num){
    int ans = 1 << num;
    return ans;
}
```

![image-20210804210215150](imgs\10.png)

在方法的Code属性中有个 locals=3，表示最大容量是3，局部变量表中有三个元素，分别是该对象的this引用、方法参数和局部变量

> 如果当前帧是由构造方法或者实例方法创建的，那么**该对象引用this将会存放在index为0的slot处**，其余的参数按照参数表顺序排列

> 参数过多导致方法栈帧的局部变量表膨胀，最终最大递归深度降低

#### 基于jclasslib剖析方法结构

同样以method()为例

method

![image-20210804212922227](imgs\11.png)

Code

![image-20210804213116610](imgs\12.png)

![image-20210804213425926](imgs\13.png)

LineNumberTable

![image-20210804213836847](imgs\14.png)

LocalVariableTable

![image-20210804214446222](imgs\15.png)

#### Slot

Slot(槽)是局部变量表最基本的存储单元

32位以内的类型只占用一个slot（包括returnAddress类型）；64位的类型（long和double）占用两个slot，采用起始索引作为索引

> byte、short、char、boolean都被转为int

> 可重复利用，如果一个局部变量过了其作用域，之后声明的新局部变量可能会复用过期的slot，节省资源

> 局部变量表是栈帧中与性能调优关系密切的部分，其中的变量是重要的垃圾回收根节点，被局部变量表中直接或间接引用的对象都不会被回收

> 一个slot对应多大的内存，虚拟机规范中没有说

### 2.2.2. 操作数栈（operand stack）

操作数据的栈，执行引擎通过操作数栈来进行运算

由于是数组实现的，因此最大深度在编译期就确定好，保存在方法的Code属性中，为max_stack

32位以内的类型只占用一个栈单位深度；64位的类型占用两个

> 下面看一个例子：

以方法method()为例

```java
public int method(int num){
    byte b = 8;
    int i = 15;
    int k = b + i;
    return k;
}
```

![image-20210805162645217](imgs\16.png)

- 注意：这些操作都是由执行引擎调度cpu来完成的
- 为什么b、c存放在slot 2、3？因为0存放的是this，1存放的是方法参数
- add、store、push前面的b、i、l等代表数据类型

以method2()为例

```java
public void method2(){
    method();
}
```

![image-20210805164051139](imgs\17.png)

#### 栈顶缓存技术

栈顶缓存技术（ToS，Top-of-Stack Caching）：由于Hotspot JVM是基于栈的指令架构，完成一个操作所需要的指令更多，因此操作数在内存中的读/写次数也比较多

为了提高速度，Hotspot JVM将栈顶元素全部缓存到CPU寄存器中，降低对内存的读写次数，提升执行引擎的执行效率

### 2.2.3. 动态链接（dynamic linking）

#### 前置知识：符号引用和直接引用

> 基于R神的知乎回答：[https://www.zhihu.com/question/30300585/answer/51335493](https://www.zhihu.com/question/30300585/answer/51335493)
>
> 以及木大：[https://www.zhihu.com/question/55994121](https://www.zhihu.com/question/55994121)
>
> 还有这位大佬：[https://www.cnblogs.com/thdjw230/p/14777050.html](https://www.cnblogs.com/thdjw230/p/14777050.html)

**1. 符号引用**

以上述的method2为例

![image-20210805172632398](imgs\19.png)

注意看invokevirtual指令，指向常量池中的#10，也就是说，方法调用到这里时，会从常量池下标为#10的索引处取出方法对应字节码的内存地址，那么，方法的内存地址是如何被定位到的呢？接着往下

我们就以Methodref和Utf8两个tag为例，在类文件的常量池中，这里的Methodref对应CONSTANT_Methodref_info结构，Utf8对应CONSTANT_Utf8_info结构*（用 jclasslib可以看到）*，这两个结构的 jvm源码分别是这样

```c++
CONSTANT_Utf8_info {  
   u1 tag;  // 类型标签，说明这个是Utf8
   u2 length;  // 数组长度
   u1 bytes[length];  // byte型数组 
} 

CONSTANT_Methodref_info { 
    u1 tag; // 类型标签，说明这个是Methodref
    u2 class_index; // 类索引，对应#7
    u2 name_and_type_index; // 名称和类型索引，对应#11
}
```

由此，根据Methedref里面的信息，我们可以定位到#7.#11→#8.#11→JVM.#12:#13→JVM.method:()I，也就是说，最终都由Utf8的字符串组成

![image-20210805173157980](imgs\20.png)

**这个能够准确定位方法的位置的字符串，就被看作是这个方法的符号引用**

主要包含：① 类和接口的全限定名(Full Qualified Name)；② 字段的名称和描述符(Descriptor)；③ 方法的名称和描述符

> 在类加载的装载（Loading）阶段，会将类的静态存储结构转化成运行时的结构，此时，CONSTANT_Utf8_info会被转换成Symbol*类型的指针，指向方法区中的Symbol类型的对象，并放入方法区的SymbolTable中统一进行管理，其中Symbol的内容跟Class文件的CONSTANT_Utf8_info描述的内容大体一致，都是同样格式的UTF-8编码的字符串。CONSTANT_Methodref_info以及其他结构也会被转化成一个个运行时常量池中的结构，这里用符号STRUCT\*代替，并最终指向不同的Symbol。也就是说，类文件常量池中的各个索引`#x`指向的是类似于CONSTANT_Utf8_info、CONSTANT_Methodref_info这样的静态结构；而在运行时常量池中，就指向了被创建出来的Symbol\*、STRUCT\*这样的结构，**注意：CONSTANT_String_info除外，它是lazy_resolve的，在resolve之前，跟类文件里一致，以索引的格式存在，称为JVM_CONSTANT_UnresolvedString**

**2. 直接引用**

是JVM（或其它运行时环境）所能直接使用的形式。它既可以表现为直接指针，也可能是其它形式。关键点不在于形式是否为“直接指针”，而是在于JVM是否能“直接使用”这种形式的数据。**相当于内存区域中方法字节码的地址偏移量**，通过偏移量虚拟机可以直接在该类的内存区域中找到方法字节码的起始位置

#### 动态链接

大部分字节码指令在执行过程中都要进行对常量池的访问

运行时常量池中存放着对当前方法的引用，而动态链接就是指向常量池中这个引用的，目的是为了支持当前方法的代码**实现**动态链接，比如：invokedynamic指令（解析是为了将符号引用转换成直接引用，执行引擎需要通过动态链接获取到方法的入口并调用它）

它的作用是在运行期间在方法调用时，将指向常量池中方法的符号引用替换成方法的直接引用（动态解析，类加载过程中linking时的resolve是静态解析）

![image-20210805172403096](imgs\18.png)

目的？栈帧里不能直接存放方法及各种属性，很占用空间，通过引用能够节省空间，同时实现很好地复用

#### 方法的调用

![preview](imgs\21.png)

- 静态分派：方法在编译期就可知，且运行期保持不变，就可以直接在编译期替换成直接引用，叫做静态分派

  > 对应早期绑定

  - 静态方法
  - 父类方法
  - 构造方法
  - 私有方法
  - final修饰的方法

- 动态分派：由于java的多态性等，方法在编译期无法确定，只能在运行期间调用时通过栈帧里面的“dynamic linking”将运行时常量池中的符号引用替换成直接引用

  > 对应晚期绑定

> 最新理解：invokexxx的时候，符号引用动态解析为直接引用，赋值给动态链接，使执行引擎能够通过这个动态链接来**调用**方法

**方法调用指令**

- 普通调用指令：
  - invokestatic：调用静态方法，解析阶段确定方法唯一版本，是非虚的
  - invokespecial：调用构造方法、私有方法及父类方法，解析阶段确定方法唯一版本，是非虚的
  - invokevirtual：调用所有虚方法
  - invokeinterface：调用接口方法
- 动态调用指令：
  - invokedynamic：动态解析出要调用的方法，然后执行

> java中所有普通方法都具有虚拟方法的特性，用final修饰的就是非虚拟的

> ![image-20210805221502674](imgs\22.png)

**虚方法表**

JVM在类加载的链接阶段会生成虚方法表vtable（virtual method table），在类变量的零值初始化之后。在invokevirtual、invokeinterface指令的时候，如果子类中没有找到，子类虚方法表就会一直往上指向父类虚方法表中存在的这个方法*（对于描述相同的方法，在子、父类的方法表中的索引位置是一致的）*

对于invokevirtual、invokeinterface指令，会结合栈顶对象的虚方法表和MethodRef引用来确定方法的直接引用，而对于invokespecial就会忽略栈顶对象，直接通过MethodRef来找了，相当于静态链接

#### invokedynamic

> java是静态语言
>
> 对类型的检查在编译期，为静态语言；在运行期，为动态语言

lambda表达式中会用到

![image-20210805224403293](imgs\23.png)

### 2.2.4. 方法返回地址（return address）

方法正常退出时，栈帧中会保存主调方法的PC计数器的值作为返回地址；异常退出时，就需要根据异常处理表来确定返回地址了

> 方法调用时，存下调用者的PC寄存器值在returnAddress，接着PC进入被调用的方法并不断移动，方法正常退出时，PC的值设置成returnAddress的值，就能回到方法被调用时的位置

### 一些附加信息

程序调试提供支持的信息等等

> 有些资料中，动态链接、方法返回地址和附加信息统称为帧数据区

## 2.3. 本地方法栈

### 本地方法接口

通过这一段代码块注册native方法，通过c++实现

```java
private static native void registerNatives();
static {
    registerNatives();
}
```

作用：

- java环境外交互
- 操作系统交互
- Sun‘s Java

### 本地方法栈

- 线程私有的，每个线程创建时都会创建一个本地方法栈，生命周期和线程一致
- 不允许栈动态扩展，可能出现SOF；支持，可能出现OOM
- Hotspot将本地方法栈和虚拟机栈合二为一

## 2.4. 堆区

### 概述

- 一个JVM实例只存在一个堆内存，堆也是Java内存管理的核心区域

- 堆区在JVM启动时即被创建，空间大小也就确定了，是JVM管理的内存最大的一块空间

  > 可调节

- 《Java虚拟机规范》规定，堆在物理上可以处于不连续的内存空间，但逻辑上应当被视为连续的

- 所有的线程共享堆，不过可以划分线程私有的缓冲区（Thread Local Allocation Buffer，TLAB）

**几乎所有对象实例和数组都应当在运行时分配在堆上**

- 帧栈的局部变量表中存放的数组和引用类型都指向堆中的数据
- 方法结束后，堆中的对象不会马上被移除，仅仅在垃圾收集时才会被移除
- 堆，是GC（Garbage Collection）的重点区域

### 堆的内存划分

> 注意：是**”逻辑上“**的划分

JDK 7及之前：

- young/new generation space：新生代/新生区/年轻代     Young/New
  - Eden
  - Survivor
- Tenure/old generation space：老年代/养老区/老年区     Old/Tenure
- permanent space：永久代/永久区     Perm

JDK 8之后：

- young/new generation space：新生代/新生区/年轻代     Young/New
  - Eden
  - Survivor
- Tenure/old generation space：老年代/养老区/老年区     Old/Tenure
- meta space：元空间     Meta

> -XX:+PrintGCDetails，打印gc细节
>
> 或者通过jps、jstat -gc pid来查看

> 堆初始大小：-Xms \<size>，默认为PC物理内存1/64；堆最大大小：-Xmx \<size>，默认为PC物理内存1/4
>
> ​	设置的是新生代+老年代的大小
>
> ​	-X，非标准参数；m，memory；s，start；x，max
>
> 开发时，建议设成一样的值，防止不断扩容、内存抖动

注意，在JDK9之前，如1.8，使用的是ParallelGC，当我们分配初始内存20m，用jstat查看往往会小于20m，这是因为计算堆内存时会**只计算一个survivor的大小**，G1GC后就不会有这种情况

#### 年轻代与老年代

> 默认情况

![image-20210809215624889](imgs\24.png)

> 默认：
>
> ​	-XX:NewRatio=2，即$\frac{老年代空间}{新生代空间}=2$
>
> ​	-XX:SurvivorRatio=8，即 $Eden空间:Survivor0空间:Survivor1空间=8:1:1$​
>
> ​		*虽然官方文档和参数是这样，但由于JVM在不同版本会自适应优化，所以比例不一定是 8:1:1*，比如：
>
> ![image-20210809221904156](imgs\25.png)

- 几乎所有的对象都是在Eden区new出来的

- 绝大多数的 Java对象的销毁都在新生代进行

  > IBM论文里说据他们统计95%的对象朝生夕死一样存活时间极短，为了保险默认实际使用了90%；经过统计，每次gc会有90%的对象被回收，所以要预留空间去保存剩下的10%；设置eden 和survivor（两个）为8:1:1，每次GC都将Eden和其中一个survivor（from）中的存活对象复制到剩余的survivor（to）中

  > 可以使用`-Xmn`设置新生代最大内存大小

#### 堆分代思想

便于优化GC性能，如果不分代，需要扫描整个堆，找出哪些需要回收哪些不需要

### 对象分配过程

**标准流程**

<img src="imgs\26.png" alt="image-20210809224403109" style="zoom: 67%;" />

除开特殊情况，一般的流程是：

1. 对象首先在Eden区被创建，随着Eden区逐渐被填满后，JVM会进行YGC(Young GC、Minor GC)，会回收绝大部分的对象，幸存的部分会进入一个空的Survivor区（如果都为空，进入Survivor0区），并标记分代年龄为1，这个Survivor区又称为from区

2. Eden区在经历垃圾回收之后，后续又会逐渐被填满，继续被YGC，Eden区的幸存对象再和from区的幸存对象一起，被放到空的Survivor区（to区），且对象分代年龄自增1

   > 如果这时候to区满了，怎么办？
   >
   > 直接把to区的对象放入老年代；如果这之后to区还是放不下，就直接把新对象放入老年代

   > 为什么要两个Survivor区？
   >
   > 防止内存碎片化。设想一下，如果只有一个survivor区，eden中的垃圾被放到survivor中，如果survivor中本身已经有垃圾了，那么这两部分内存不是紧挨着的，中间会产生空隙，而这部分空隙是无法利用的，造成堆中产生大量内存碎片，严重影响效率。而复制的话会主动令eden和from区来的垃圾内存地址连续，就不会产生碎片了

3. 如果对象的分代年龄大于阈值15，则会被放到老年区

   > 为什么分代年龄的阈值是15？
   >
   > 可以看一看java对象内存布局的markword，里面记录有关分代年龄的字段占4个bit（无锁态、偏向锁态），也就是最大值为15

   大概流程如图：

<img src="imgs\27.png" alt="image-20210810205822610" style="zoom:67%;" />

#### 垃圾回收

1. 部分收集（Partial GC）

   1. 新生代收集（Minor GC / Young GC）：只收集新生代的垃圾

      - 由Eden区满触发
      - 会触发STW，Stop The World，暂停用户线程，JVM处于没有任何其他正在执行的字节码的状态

   2. 老年代收集（Major GC / Old GC）：只收集老年代的垃圾

      - 由老年代满触发
      - 一般来说，Major GC前会伴随着YGC，但Parallel Scavenge收集器可能会直接进行Major GC
      - 速度一般比YGC慢十倍以上，STW时间更长

      > 目前，只有CMS GC有单独收集老年代的行为

   3. 混合收集（Mixed GC）：收集整个新生代和部分老年代的垃圾

      > 目前，只有G1 GC有这样的行为

2. 整堆收集（Full GC）：收集整个堆区和方法区的垃圾

   - 触发机制
     - System.gc()时，系统建议执行Full GC，但不必然执行
     - 老年代空间不足
     - 方法区空间不足

   > 开发中要尽量避免Full GC，减少STW时间

   > 方法区的垃圾回收主要在于废弃常量和不再使用的类型的回收（校验这个比较复杂，比如回收一个类，还要看他派生的子类会不会用到等等，所以Full GC比较耗时耗力）

#### 内存分配策略（对象提升(Promotion)规则）

- 优先分配到Eden

- 大对象直接分配到老年代

  > 开发中尽量避免大对象

- 长期存活的对象分配到老年代

- 动态对象年龄判断：Survivor区中某相同年龄的所有对象大小总和大于Survivor区的一半，则大于或等于该年龄的对象可以直接进入老年代，无需达到MaxTenuringThreshold中要求的年龄

#### TLAB

TLAB：Thread Local Allocation Buffer

由于对象的创建在JVM中非常频繁，因此在并发环境下从堆中划分内存空间是线程不安全的；为了避免多个线程同时操作同一地址，需要进行同步等机制，影响内存分配速度

因此，从内存模型的角度而不是垃圾收集的角度，在Eden区为每个线程划分了一份私有缓存区域，避免线程安全问题，同时提升内存分配吞吐量，称为**快速分配策略**，一般是分配的首选策略

> 从OpenJDK衍生出来的JVM都提供了TLAB的设计

> 默认情况下开启，通过`-XX:+/-UseTLAB`来设置；默认占Eden区的1%，通过`-XX:TLABWasteTargetPercent`来设置

一旦对象在TLAB分配失败，JVM就会尝试使用加锁机制确保数据操作的原子性，从而直接在Eden区分配内存

#### 堆是分配对象内存的唯一选择吗？

随着JIT编译器的发展和逃逸技术分析逐渐成熟，栈上分配、标量替换优化技术将会导致不再绝对是堆上分配了

- 如果经过逃逸分析后，一个对象并没有逃逸出方法的话，可能被优化成栈上分配

  > 分析对象动态作用域，JVM分析引用的作用范围决定是否要分配到堆上
  >
  > - 对象在方法内定义，且只在方法内使用，则认为没有发生逃逸
  >
  > ```java
  > public void method(){
  >     JVM jvm = new JVM();
  >     jvm = null;
  > }
  > ```
  >
  > - 对象在方法内定义，被外部方法引用，则认为发生了逃逸

  开启逃逸分析：`-XX:+DoEscapeAnalysis`(JDK6u23之后默认开启) 


#### 基于逃逸分析的代码优化

- 栈上分配

JIT编译器在编译期间根据逃逸分析的结果，可能将一个没有逃逸出方法的对象优化成栈上分配

- 锁消除（同步省略）

锁消除：JIT编译器在运行时，通过逃逸分析，如果判断一段代码中，堆上的所有数据不会逃逸出去从来被其他线程访问到，就可以去除这些锁。比如说，在单线程或线程私有栈使用同步容器

> 前提是java必须运行在server模式（server模式会比client模式作更多的优化），同时必须开启逃逸分析:
>
> `-server `
>
> `-XX:+DoEscapeAnalysis`(JDK6u23之后默认开启) 
>
> `-XX:+EliminateLocks`
>
> 其中+DoEscapeAnalysis表示开启逃逸分析，+EliminateLocks表示锁消除

- 标量替换

标量（Scalar）是指一个无法再分解成更小数据的数据，比如Java中的原始数据类型

相对的，那些还可以分解的数据叫做聚合量（Aggregate），比如Java中的对象

在JIT阶段，经过逃逸分析发现一个对象不会被外界访问，就可能把对象通过其中的成员变量来代替，并以局部变量的形式放在局部变量表中，这就是标量替换

```java
public void alloc(){
    Point point = new Point(1, 2);
    System.out.println("x: " + point.x + "  y: " + point.y);
}
@Data
@AllArgsConstructor
class Point{
    private int x;
    private int y;
}
```

这样的代码，可能被优化成：

```java
public void alloc(){
    // 原point的变量
    int x = 1, y = 2;
    System.out.println("x: " + x + "  y: " + y);
}
```

`-XX:+EliminateAllocations`：开启标量替换，默认打开

**栈上分配和这里的标量替换不一样**，栈上分配是指狭义的把对象分配到栈上，hotspot还未实现这一功能

## 2.5. 方法区

### 堆、栈、方法区的交互关系

<img src="imgs\29.png" alt="image-20210811211312486" style="zoom: 50%;" />

### 基本概念

#### 方法区的位置

逻辑上属于堆的一部分，但一些简单的实现可能不会选择去进行垃圾回收或进行压缩

对于Hotspot JVM，方法区和代码缓存合起来有个别名叫**Non-heap**，为了和堆区分开来

因此，**方法区看作是一块独立于Java堆的内存空间**

#### 特点

- 多个线程共享
- 实际物理内存和堆一样可以不连续
- 方法区的大小决定系统可以存放多少个类

#### hotspot中方法区的演进

**方法区（method area）**只是**JVM规范**中定义的一个概念，用于存储类信息、常量池、静态变量、JIT编译后的代码等数据，具体放在哪里，不同的实现可以放在不同的地方。而**永久代**是**Hotspot**虚拟机特有的概念，是方法区的一种实现，别的JVM都没有这个东西

- 在Java 6中，方法区中包含的数据，除了JIT编译生成的代码存放在native memory的CodeCache区域，其他都存放在永久代

- 在Java 7中，Symbol的存储从PermGen移动到了native memory，并且把静态变量（变量本身，并非它指向的内容）从instanceKlass末尾（位于PermGen内）移动到了java.lang.Class对象的末尾（位于普通Java heap内），字符串常量池也移到堆中。逐步开始”去永久代“

  > 为什么去永久代？
  >
  > - 收购并融合JRockit
  > - 永久代中的数据存放时间都比较久，如果动态加载较多的类，很容易造成PermGen的OOM，而为永久代设置大小是很难预测的很困难的
  > - 永久代难以调优

  > 为什么移动字符串常量池？
  >
  > 开发中会有大量的字符串被创建，永久代回收效率低，导致永久代内存不足，放到堆里，能及时回收内存

- 在Java 8中，永久代被彻底移除，取而代之的是另一块与堆不相连的本地内存——元空间（**Metaspace**）,‑XX:MaxPermSize 参数失去了意义，取而代之的是-XX:MaxMetaspaceSize。字符串常量池、静态变量仍在堆

#### 方法区大小

- JDK7及以前

  - `-XX:PermSize`设置永久代初始分配空间，默认20.75m
  - `-XX:MaxPermSize`设置永久代最大可分配空间，32位机器默认64m，64位默认82m

- JDK8以后

  - `-XX:MetaspaceSize`设置元空间初始分配空间，默认值依赖于平台，约21m

    > - 对于64位的服务端JVM来说，其默认值又看作是初始的高水位线，一旦触及，便触发Full GC并卸载没用的类（即这些类对应的类加载器不再存活），然后这个高水位线将会重置，新的值取决于GC时释放了多少元空间。如果释放的空间不足，在低于MaxMetaspace的基础上适当提高该值；如果释放过多，则适当降低该值
    >
    > - 如果这个值设置过低，容易频繁调整高水位线，频繁FGC，所以建议设高一些

  - `-XX:MaxMetaspaceSize`设置元空间最大可分配空间，默认-1，即无限制，取决于本地内存

### 内部结构

#### 类型信息

类型的相关信息，包括：

- 类型的完整有效名称（全名，即包名.类名）
- 父类的完整有效名称
- 类型的修饰符（public、final、abstract的某个子集）
- 类的直接接口的一个有序列表

#### 域信息

类型的所有域（Field）的相关信息以及声明顺序，包括：

- 域名称
- 域类型
- 域修饰符

#### 方法信息

方法的相关信息以及声明顺序，包括：

- 方法名称
- 返回类型
- 参数的数量和类型（按顺序）
- 方法修饰符
- 方法的字节码、操作数栈、局部变量表及大小
- 异常表

#### 静态变量

#### CodeCache

JIT编译后的代码缓存

#### 运行时常量池

字节码中的常量池加载到内存中，变成运行时常量池

![image-20210822112008003](imgs\30.png)

存放字面量和各种信息的符号引用，用于类加载的链接阶段的静态解析和方法运行时的动态解析，有效避免字节码文件过大

> 一个简单的类，里面的结构也可能包含各种类和方法，如果直接放直接引用的话，字节码文件就太大了，符号引用有效避免了这一点

**具备动态性**，里面的内容可能动态变化

## 对象实例化

### 对象实例化的6个步骤

1. 判断对应的类是否被加载、链接、初始化

   > JVM遇到new指令，会去检查这个指令的参数在Metaspace的常量池中是否有该类的符号引用，并判断是否完成类加载

2. 为对象分配内存

   - 内存规整（Serial、ParNew等，标记压缩算法），采用指针碰撞法
   - 内存不规整（CMS，标记清除算法），采用空闲列表

3. 处理并发问题

   - CAS
   - TLAB

4. 初始化分配到的空间

   > 成员变量等等的零值初始化*（注意，类加载的链接-准备阶段会进行的是类变量的零值初始化）*

5. 设置对象头

6. 执行\<init>方法进行初始化

   > 包括属性显式初始化、代码块、构造器初始化

### 对象内存布局

![image-20210822222656796](imgs\31.png)

针对于堆中的对象，分成几个部分，以下面为例（64位系统）

```java
class Student{
    int age;
    String name;
}
```

new Student()

<img src="imgs\8.png" alt="image-20210713165149246" style="zoom:50%;" />

- 对象头

  - markword：8字节
    - 锁信息
    - GC信息
    - hashCode
  - klass pointer（instanceOopDesc）：指向所属类的方法区内的类元信息（instanceKlass）地址
    - 开启指针压缩：4字节（默认开启）
    - 不开启指针压缩：8字节
  - length：4字节，数组特有

- 实例数据

  - instance data：类成员变量等，本例中为age的4字节 + name引用的4个字节（开启指针压缩：-XX:+UseCompressedOops）

- 对齐填充

  - padding：64位虚拟机下，填充成8个字节的倍数

    > HotSpot虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍

### 对象访问定位

#### 如何定位

<img src="imgs\34.png" alt="image-20210822224018256" style="zoom: 50%;" />

#### 访问方式

1. 句柄访问

<img src="imgs\32.png" alt="image-20210822223548563" style="zoom: 50%;" />

优点：垃圾回收等情况下对象发生移动时，reference是稳定的，只需要改变到对象实例的指针

2. 直接访问（Hotspot采用的）

<img src="imgs\33.png" alt="image-20210822223903944" style="zoom:50%;" />

### 直接内存

> 在之前进行网络传输时，JVM发送数据到网络，要经历：数据→JVM→复制到JVM堆（用户态）→复制到操作系统内存（堆外内存）（内核态）→网卡
>
> Java NIO提供操作直接内存的对象，常用的有DirectByteBuffer
>
> ```java
> ByteBuffer buffer = ByteBuffer.allocateDirect(xxx);
> ```

Java堆和直接内存的总和受系统内存的限制

可通过`MaxDirectMemorySize设置`；若不指定，默认与堆大小一致

# 3. 执行引擎

## Java是半编译半解释的语言

字节码通过JVM逐行解释的方式执行，与此同时，热点代码块被JIT编译器直接编译成机器码存放在CodeCache（后端编译），加速执行。所以说Java是半编译半解释执行的

> `-Xint`：完全采用解释器；`-Xcomp`：完全采用JIT，出现问题时解释器才介入；`-Xmixed`：两者混合，默认是这个

## 解释器分类

在Java的发展历史里，共有两套解释执行器，即古老的**字节码解释器**、现在普
遍使用的**模板解释器**

- 字节码解释器在执行时通过纯软件代码模拟字节码的执行，效率非常低下

- 而模板解释器将每条字节码和一个模板函数相关联，模板函数中直接产生这
  条字节码执行时的机器码，从而很大程度上提高了解释器的性能

  > 在HotSpot VM中，解释器主要由Interpreter模块和Code模块构成
  >
  > - Interpreter模块：实现了解释器的核心功能
  > - Code模块：用于管理HotSpot VM在运行时生成的本地机器指令

## 编译器

Java语言的“编译期” 其实一段“不确定”的操作过程，因为它可能是指一个前端编译器把 .java文件转变成.class文件的过程；也可能是指虚拟机的后端运行期编译器（JIT编译器，Just In Time Compiler）把字节码转变成机器码的过程；还可能是指使用静态提前编译器(AOT编译器，Ahead Of Time Compiler)直接把. java文件编译成本地机器代码的过程

> - 前端编译器：Sun的Javac、 Eclipse JDT中的增量式编译器（ECJ）
> - JIT编译器：HotSpot VM的C1、C2编译器
> - AOT编译器：GNU Compiler for the Java（GCJ）、Excelsior JET

### 热点代码及探测方式

#### 热点代码

① 被多次调用的方法；② 方法体内循环次数较多的循环体

热点代码被JIT编译的过程发生在方法执行过程中，因此也被称为栈上替换，又称OSR（On Stack Replacement）编译

#### 热点探测

探测热点代码的方式：Hotspot VM采用的是基于计数器的热点探测，为每一个方法建立两种计数器：

1. 方法调用计数器（Invocation Counter）：统计方法调用次数

   - 默认阈值在Client模式下为1500，Server模式下为10000，可通过`-XX:CompileThreshold`来设置

   - 方法被调用时，首先检测codecache中是否有对应的机器码，如果有则执行；否则会令方法调用计数器+1，并判断方法调用计数器和回边计数器的总和是否大于方法调用计数器的阈值，大于的话向JIT编译器提交该代码的编译请求

     <img src="imgs\35.png" alt="image-20210823211834600" style="zoom:67%;" />

2. 回边计数器（Back Edge Counter）：统计循环体的循环次数

#### 热度衰减

实际上，方法调用计数器记录的并不是调用的绝对次数，而是一段时间内的调用频率，如果这段时间内调用次数并没有达到提交JIT的阈值，计数器值会减半，叫做热度衰减（Counter Decay），可通过`-XX:+/-UseCounterDecay设置`；这段时间称为半衰周期，可通过    `-XX:CounterHalfLifeTime`设置，单位是秒

### 分类

#### 两种模式

- server compiler：`-server`模式（64位只有server模式）下采用，又称C2编译器，进行激进的优化和耗时更长的编译（启动慢，但稳定执行后效率远好于C1）

  > 主要在全局层面优化，包括标量替换、同步消除等

- client compiler：`-client`模式下采用，又称C1编译器，进行简单可靠的优化和更快的编译

  > 优化方法：
  >
  > - 方法内联：引用的方法代码编译到引用点处，减少栈帧的生成、参数的传递和地址跳转
  > - 去虚拟化：对唯一的实现类进行内联
  > - 冗余消除：不会使用的代码折叠掉

#### 补充：AOT

JDK9之后引入AOT（Ahead Of Time）编译器，借助了Graal编译器，将字节码文件编译成机器码，并存储在动态共享库中，是在**运行之前**的

# 补充：OOP-Klass体系

> 参考
>
> - [https://blog.csdn.net/linxdcn/article/details/73287490——linxdcn](https://blog.csdn.net/linxdcn/article/details/73287490)
> - [https://blog.csdn.net/qq_32950237/article/details/111397432——JPENDOW](https://blog.csdn.net/qq_32950237/article/details/111397432)
> - [https://blog.csdn.net/u010365717/article/details/107063001——阿萨德执行](https://blog.csdn.net/u010365717/article/details/107063001)
> - R神的回答：[https://www.zhihu.com/question/50258991](https://www.zhihu.com/question/50258991)

## 前言

HotSpot是基于c++实现，而c++是一门面向对象的语言，本身具备面向对象基本特征，所以Java中的对象表示，最简单的做法是为每个Java类生成一个c++类与之对应。但HotSpot JVM并没有这么做，而是设计了一个OOP-Klass Model

- 这里的 OOP 指的是 Ordinary Object Pointer （普通对象指针），它用来表示对象的实例信息，看起来像个指针，实际上是藏在指针里的对象（是一个指针，作为一个java对象头存在）
-  Klass 则包含元数据和方法信息，用来描述Java类，是“类元信息”

之所以采用这个模型是因为HotSopt JVM的设计者不想让每个对象中都含有一个vtable（虚方法表），所以就把对象模型拆成klass和oop，其中oop中不含有任何虚方法，而Klass就含有虚函数表，可以进行method dispatch

```shell
• Klass表示Java类在JVM中的存在形式
    • InstanceKlass表示类的元信息
		• InstanceMirrorKlass表示类的Class对象对应的类元信息
		• InstanceRefKlass表示强软弱虚等引用对象对应的类元信息
	• ArrayKlass表示数组类的元信息
		• TypeArrayKlass表示基本数组的类元信息
		• ObjArrayKlass表示引用数组的类元信息
• oopDesc表示JAVA对象在JVM中的存在形式
	• instanceOopDesc表示普通类对象
	• arrayOopDesc表示数组类对象
		• typeArrayOopDesc表示基本数组类对象
		• objArrayOopDesc表示引用数组类对象
```

## 内容

### OOP

oops，也就是jvm中的对象指针，有一个共同基类，叫做oopDesc，**存储在堆中**

```c++
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
}

class instanceOopDesc : public oopDesc {
}
 
class arrayOopDesc : public oopDesc {
}
```

instanceOopDesc和arrayOopDesc，都继承了oopDesc中的内容，分别用于普通对象的创建和数组对象的创建

包含两个部分：_mark，也就是我们常说的markword；以及\_metadata，包含压缩和未压缩的指针，都指向了方法区中的instanceKlass，也就是类元信息

### Klass

Klass及其子类对象用于表示java中类的类元信息，**存储在方法区中**

### InstanceKlass

JVM在运行时，需要一种用来标识Java内部类型的机制。在HotSpot中的解决方案是：为每一个已加载的Java类创建一个instanceKlass对象，用来在JVM层表示Java类

InstanceKlass包含了Java运行时环境中类所需的所有数据信息，在类加载这一步，类加载器会将.class文件读入类加载器的class content，然后以InstanceKlass的形式写入JVM内存区域的方法区中

#### InstanceMirrorKlass

InstanceMirrorKlass是类所对应的Class对象（java.lang.Class）的InstanceKlass，是InstanceKlass的子类

**提出一个问题，了解了上述之后，那么，JAVA对象是如何与Class对象建立映射的？**

以下列图示来理解会简单很多

以`Test test = new Test()`为例

![image-20210906114346234](imgs\41.png)

这就可以回答上面的问题了，实例对象通过指向类元信息的指针找到方法区的InstanceKlass，再通过里面的_java_mirror字段定位到Class对象

> JDK6及以前，静态变量放在InstanceKlass对象的末尾；JDK7及以后，变到了Class对象，也就是mirror对象的末尾

> JVM实际实现中，InstanceKlass会用klassOopDesc包装，为方便理解，就不改了，直接称为InstanceKlass

> 注意：在JDK 7及后续版本中，Klassklass体系已不再使用，这里就不做介绍了

# String

## 字符串常量池

String Pool是一个固定长度的HashTable*（注意，可不是util包下的Hashtable，而是jvm源码实现的一个类似hashtable的结构）*，key是根据String实例的值和长度计算生成的hash，value存放了对String实例的**引用**

如果放入的string太多，造成很多hash冲突，链表过长，会导致性能下降。使用`-XX:StringTableSize`可以设置，JDK6默认1009；JDK7、8默认60013，JDK8可设置的最小值是1009

字面量会自动放入stringpool，或调用intern方法

> 字符串常量池这个HashTable，当存放的数据足够多需要rehash时，其rehash的行为和Java中使用的HashMap有所不同。字符串常量池底层的HashTable在rehash时不会扩容，即rehash前后bucket数量是一样的。它的rehash过程只是通过一个新的seed重新计算一遍来尝试摊平每个bucket中LinkedList的长度（不保证rehash后的性能有很大的提升）

## String拼接

### 案例一

```java
public class Test {
    public static void main(String[] args) {
        String s1 = "s1";
        String s2 = "s2";
        String s3 = "s1s2";
        String s4 = "s1" + "s2";
        String s5 = s1 + "s2";
        String s6 = "s1" + s2;
        System.out.println(s3 == s4);  // true
        System.out.println(s3 == s5);  // false
        System.out.println(s3 == s6);  // false
        System.out.println(s5 == s6);  // false
    }
}
```

对s3、s4、s5、s6逐一分析

- s3：字面量初始化，在堆中生成了一个对象”s1s2“，将指向该对象的引用赋给s3，同时将这个引用在字符串常量池中也驻留一份

- s4：JVM对于这种拼接方式，会自动编译成”s1s2“，反解析一下可以证明，这是编译期优化

  ![image-20210508153434567](D:\笔记\秋招\docs\JAVA进阶\imgs\JAVA常用类\4.png)

- s5：JDK9之前是这样实现的

  ![image-20210508154642340](D:\笔记\秋招\docs\JAVA进阶\imgs\JAVA常用类\6.png)

  过程：

  ```java
  // 对于String s5 = s1 + "s2";
  // ①
  StringBuffer sb = new StringBuffer(String.valueof(s1));
  // ②
  sb.append("s2");
  // ③
  s5 = sb.toString();
  // 这一步的toString会调用String对象的构造器，在堆中新生成一个String对象
  // 由于这个过程没有"s1s2"字面量产生，也没有调用intern()方法，所以常量池中的引用和该新对象的地址不同
  
  /**
  public synchronized String toString() {
      if (toStringCache == null) {
          toStringCache = Arrays.copyOfRange(value, 0, count);
      }
      return new String(toStringCache, true);
  }
  **/
  ```

  JDK9之后，源码不再将拼接过程编译成字节码，而是通过动态编译，调用的是StringConcatFactory的makeConcatWithConstants方法

  ![image-20210508154339899](D:\笔记\秋招\docs\JAVA进阶\imgs\JAVA常用类\5.png)

  有六种拼接策略，其中前五种也是通过StringBuffer实现的

  参考[https://zhuanlan.zhihu.com/p/259254321——君慕贤](https://zhuanlan.zhihu.com/p/259254321)

- s6：基本同s5，也在堆中产生一个新的String对象

这样的话，结果就不言而喻了，s3、s4都等同于常量池中的引用，而s5、s6各自在堆中new了一个新的String

> 此时堆里有三个”s1s2“的对像，一个是s5指向的，一个是s6指向的，一个是常量池中的引用s3、s4指向的

**注意**

> 上题中，将s1改为**final**，对于String s5 = s1 + "s2"，JVM会优化成等同于"s1"+"s2"，那么s3==s5就为true了

> 用+拼接会不停new StringBuffer，所以拼接时建议自己new一个StringBuffer

### 案例二（重点：intern难题）

[美团技术博客：深入解析String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)

①

```java
public class Test {
    public static void main(String[] args) {
        String s1 = "s1";
        String s2 = s1 + "s2";
        String s3 = "s1s2";
        System.out.println(s2 == s2.intern());
    }
}
```

②

```java
public class Test {
    public static void main(String[] args) {
        String s1 = "s1";
        String s2 = s1 + "s2";
        System.out.println(s2 == s2.intern());
        String s3 = "s1s2";
    }
}
```

①的结果为false，②为true，为何？

1. 对于①

   a. String s2 = s1 + "s2"的过程中，在堆中创建了一个"s1s2"的对象，并把引用赋给s2

   b. String s3 = "s1s2"过程中，又在堆中创建了一个新的"s1s2"的对象，并把引用赋给s3，由于是字面量创建，因此指向该对象的引用驻留了一份在字符串常量池中，也就是，字符串常量池中指向"s1s2"的引用值==s3

   c. 此时堆中有两个"s1s2"对象，s2.intern()返回的是字符串常量池中指向“s1s2”的对象，这个对象是由s3指向的，所以s3==s2.intern()，s2 != s2.intern()

2. 对于②

   a. 这里可以不管输出语句后面的那句，考虑s2.intern()，s2.intern()返回的是字符串常量池中指向“s1s2”的对象，发现没有，于是将堆中指向"s1s2"的引用，也就是s2，直接放一份在字符串常量池中。也就是说，此时有两个引用指向堆中的"s1s2"，一个是s2，一个是字符串常量池中的引用，它们的值是相等的

   b. 所以 s2 == s2.intern()

   > 注意，这是JDK 7之后才会有的结果，JDK7之前stringpool是放在永久代的，会复制一个相同新对象，并把**新对象**放入常量池中（不是引用），所以结果会是false；JDK7、8 stringpool移动到堆，会直接将指向堆（非stringpool）的**引用**直接复制给stringpool
   >
   > **在JDK 6中，字面量形式的字符串会在永久代中的stringpool中新创建一个对象，而JDK 7中会在堆中产生，且把引用复制到stringpool**

## 字面量是如何进入字符串常量池的？

[前置知识](#前置知识：符号引用和直接引用)

[相关链接](https://www.zhihu.com/question/55994121)

> Class文件中的常量池主要存放两大类常量：
>
> ①字面量(Literal)：文本字符串等，CONSTANT_Utf8以及一些用final修饰的基本类型
>
> ②符号引用(Symbolic References)

定义一个字面量

```java
public static void main(String[] args) {
    String str = "hello";
}
```

javap看字节码反解析结果，发现有：

![image-20210825121024953](imgs\36.png)

在类加载过程中的装载过程中，会将静态的CONSTANT_xxx等等的存储结构转换成其他的运行时常量池里的结构比如Symbol*等，而CONSTANT_String_info是lazy resolve的，只有调用ldc指令时，会去尝试解析，如果此时还只是一个索引，跟类文件中的静态结构一致（称为JVM_CONSTANT_UnresolvedString），则会在堆中创建一个对应内容的字符串，并把引用驻留在StringTable，并将索引替换成这个引用，接下来运行时常量池的这一项就为字符串的引用；如果StringTable中已经有该引用，则返回该引用，常量池中该索引处的值变为该引用，后续ldc指令就可以直接把它推到栈顶进行操作

**也就是说，在ldc指令后字面量才进入stringpool，之前都是在运行时常量池，也就是Metaspace当中**

> ldc #x指令：将常量池中的索引为#x的项推到栈顶

![image-20210825130444303](imgs\37.png)

## G1的string去重操作

据统计，堆上有大量的重复string对象，G1的操作是这样的

- 当垃圾收集器工作的时候，会访问堆上存活的对象，**这时会检查每一个对象是否为候选的需要去重的String对象**

- 如果是，把这个对象的一个引用插入到队列中等待后续处理。一个去重的线程在后台运行，处理这个队列，处理方式是从队列中删除这个元素，并去重它引用的String对象

  如何去重？

  - 使用一个hashtable来记录所有的被String对象使用的char[]/byte[]数组，去重的时候，会查询这个hashtable，看看堆上是否已存在一样的数组
    - 如果存在，String对象内部的引用会被调整为该数组，原指向的数组被回收
    - 如果没有，则插入hashtable

# GC

> 什么是垃圾？在程序运行过程没有被任何指针指向的对象。就是需要被回收的垃圾

> 垃圾回收：
>
> - 释放没用的对象
> - 清理内存碎片

## 相关概念

### System.gc()

通过`System.gc()`或`Runtime.getRuntime().gc()`，会显式触发Full GC

**System.gc()附带一个免责说明，并不保证对垃圾收集器的调用**

提醒JAVA虚拟机，“希望”它进行一次Full GC

> 一个值得注意的例子
>
> ```java
> public void method() {
>     {
>         byte[] buffer = new byte[10 * 1024 * 1024];
>     }
>     System.gc();
> }
> ```
>
> 对于这个例子，gc后的信息如下：
>
> ![image-20210907155038582](imgs\44.png)
>
> 可以发现，就算超出了buffer的作用域，同样没有被GC
>
> 来看看字节码
>
> ![image-20210907160057506](imgs\45.png)
>
> 给局部变量表分配了两个槽位，但实际上局部变量表中只有一个this，证明buffer的引用并没有被切断
>
> ![image-20210907213800274](imgs\46.png)
>
> 改成：
>
> ```java
> public void method(){
>     {
>         byte[] buffer = new byte[10 * 1024 * 1024];
>     }
>     int a = 1;
>     System.gc();
> }
> ```
>
> ![image-20210907214105412](imgs\47.png)
>
> 成功gc了
>
> ![image-20210907214207025](imgs\48.png)
>
> slot被复用了
>
> 覆盖了之前buffer的引用，成功gc
>
> R大的解释是这样的：HotSpot VM并不会在局部变量离开作用域之后对其做显式的清理动作
>
> [https://www.zhihu.com/question/34341582/answer/58444959](https://www.zhihu.com/question/34341582/answer/58444959)

### OOM

没有空闲内存，且垃圾回收器也无法提供更多内存

- 堆OOM

  - JVM堆内存设置不够

  - 创建大量大对象，长时间不能被垃圾收集器回收

    > 当创建的对象过大，JVM判断出垃圾收集无法解决这个问题，直接抛出OOM

- 栈OOM

  - 单个栈帧过大（往往是局部变量表过大）
  - 如果栈深度支持动态扩展，则可能由于内存不足而OOM，hotspot不支持栈帧的动态扩展，因此栈深度大于阈值只会有SOF

- 方法区

  - 对于JDK7之前，String.intern()会不断往PermGen的字符串常量池中复制字符串的实例，可能出现PermGen的OOM
  - JDK7之后，字符串常量池移到堆中，可以通过动态生成类来撑爆方法区

### 内存泄漏

对象不会被程序使用了，但GC又无法回收

- 单例模式中单例有指向外部的引用
- 数据库连接、网络连接和io连接等未关闭

### Stop The World

GC时会产生一个Stop-The-World，即应用程序的停顿

**原因**：GC可达性分析时，根节点枚举必须要STW，因为必须满足GC Roots的准确性，所以必须在一个能保证一致性的快照中进行，如果一致性无法保证，则分析结果不一定准确

> 所有的GC都无法避免STW

#### 安全点（safe point）

程序并非在所有地方都能够马上停下来，需要找到一个安全点（safe point）

**如何选择？**

安全点的选择很重要，如果太少，则GC得等到运行到安全点，等待时间过久；如果太频繁，影响程序性能

选择的原则是那些**让程序较长时间执行的指令**，如：方法调用、循环跳转、异常跳转

**如何在GC发生时，令所有线程都运行到最近的安全点停顿下来呢？**

- 抢先式中断：JVM中断所有线程，如果有线程不在安全点，就恢复它，然后让它继续运行到安全点，基本没有vm采用这种方法
- 主动式中断：给线程设置中断标志，线程运行到safe point时，主动轮询中断标志，如果中断标志为真，则将自己挂起

#### 安全区域（safe region）

上述提到通过主动式中断来进行停顿，然而线程在sleep或blocked的时候是无法响应中断的，这时候如何处理？

Hotspot VM提出了安全区域的概念，即在运行的一段过程中，能够确保线程中**对象的各个引用关系不会发生改变，也就是具有一致性**，这一段过程便是安全区域。sleep或blocked状态的线程，都是安全区域

线程在进入安全区域后，会标识自己进入了安全区域，此后GC遇到安全区域便会自动忽略；线程出安全区域时，会检查GC是否已经完成了STW的事件（例如根节点枚举或垃圾收集期间其他导致STW的事件），如果发生了，就继续执行，没有发生，则必须等待至GC完成会导致STW的事件

相当于将安全点“拉长”了

### 垃圾收集的串/并行和并发

- 并行（Parallel）垃圾回收器：主要关注垃圾回收线程之间的并行，此时默认用户线程是被STW暂停了的

- 串行（Serial）垃圾回收器：主要是垃圾回收线程的串行，也就是STW期间只有一个垃圾回收线程

- 并发（Concurrent）垃圾回收器：主要关注用户线程和垃圾回收线程之间可以同时执行（不一定是并行，也可能是交替执行）

  > 由于用户线程并未被冻结，所以程序仍然能响应服务请求，但由于垃圾收集器线程占用了一部分系统资源，此时应用程序的处理的吞吐量将受到一定影响

## 相关算法

### GC Roots

#### 基本概念

**GC Roots：一组必须活跃的引用**，主要包含：

- 虚拟机栈中引用的对象：比如方法中的参数、局部变量等
- 本地方法栈内JNI（本地方法）引用的对象
- 类静态属性引用的对象
- 运行时常量池中所引用的对象
- 被synchronized持有的对象
- JVM内部的引用
  - 基本数据类型对应的Class对象
  - 一些常驻的异常对象
  - 系统类加载器
- 反映JVM内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等

**补充知识**

- GC Roots 本身是没有所谓的存储位置，他们都是字节码加载运行过程中加入JVM中的一些普通对象，只不过被认为是GC Roots
- 虚拟机会回收GC Roots吗？不会但不保证绝对不会，因为JVM有几十种，无法保证以后会不会加入回收GC Roots 的机制，但就HotSpot而言是不会的
- 除了这些固定的GC Roots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“临时性”地加入，共同构成完整GC Roots集合。比如：分代收集和局部回收(Partial GC)。如果只针对Java堆中的某一块区域进行垃圾回收（比如:典型的只针对新生代)，必须考虑到内存区域是虚拟机自己的实现细节，更不是孤立封闭的，这个区域的对象完全有可能被其他区域的对象所引用，这时候就需要一并将关联的区域对象也加入GC Roots集合中去考虑，才能保证可达性分析的准确性
- 小技巧：
  Root 采用栈方式存放变量和指针，所以如果一个指针，它保存了堆内存里面的对象，但是自己又不存放在堆内存里面，那它就是一个Root

#### 根节点枚举

根节点枚举，就是找到GC Roots的过程，这个过程是需要STW的，必须保持一致性，以保持后续引用链分析结果的准确性

**Oopmap**

随着java应用的不断扩展，找GC Roots现在变得越来越困难，固定可作为GC Roots的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）中，尽管目标明确，但查找过程要做到高效并非一件容易的事情，现在Java应用越做越庞大，光是方法区的大小就常有数百上千兆，里面的类、常量等更是恒河沙数，若要逐个检查以这里为起源的引用肯定得消耗不少时间

由于目前主流Java虚拟机使用的都是准确式垃圾收集，所以当用户线程停顿下来之后，其实并不需要一个不漏地检查完所有执行上下文和全局的引用位置，虚拟机应当是有办法直接得到哪些地方存放着对象引用的。在HotSpot的解决方案里，是使用一组称为OopMap的数据结构来达到这个目的。一旦类加载动作完成的时候， HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译过程中，也会在特定的位置记录下栈里和寄存器里哪些位置是引用。这样收集器在扫描时就可以直接得知这些信息了，并不需要真正一个不漏地从方法区等GC Roots开始查找其他GC Roots

每个被JIT编译过后的方法也会在safepoint的位置记录下OopMap，记录了执行到该方法的某条指令的时候，栈上和寄存器里哪些位置是引用。这样GC在扫描栈的时候就会查询这些OopMap就知道哪里是引用了。之所以要选择安全点来记录OopMap，是因为造成引用关系变化的指令很多，如果对每条指令（的位置）都记录OopMap的话，这些记录就会比较大，那么空间开销会显得不值得。选用一些比较关键的点来记录就能有效的缩小需要记录的数据量，但仍然能达到区分引用的目的。仍然在解释器中执行的方法则可以通过解释器里的功能自动生成出OopMap出来给GC用

#### 记忆集、卡表和写屏障

前面提到过，除了固定的GC Roots以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“临时性”地加入，共同构成完整GC Roots集合，比如对新生代进行回收时，老年代的一部分指向新生代的引用可能会被加入GC Roots，产生跨代引用

如果扫描整个老年代并选择性加入GC Roots，未免开销过大。Hotspot的解决方式是引入记忆集（Remember Set）

记忆集是一种用于记录**从非收集区域指向收集区域的指针集合**的抽象数据结构，采用“卡表”（Card Table）的方式去实现记忆集，它定义了记忆集的记录精度、与堆内存的映射关系等（关于卡表与记忆集的关系，不妨按照Java语言中HashMap与Map的关系来类比理解）。 卡表最简单的形式可以只是一个字节数组，而HotSpot虚拟机确实也是这样做的。以下这行代码是HotSpot默认的卡表标记逻辑：

```c++
CARD_TABLE [this address >> 9] = 0;
```

字节数组CARD_TABLE的**每一个元素都对应着其标识的内存区域中一块特定大小的内存块**，这个内存块被称作“卡页”（Card Page）。一般来说，卡页大小都是以2的N次幂的字节数，通过上面代码可以看出HotSpot中使用的卡页是2的9次幂，即512字节（地址右移9位，相当于用地址除以512）。那如果卡表标识内存区域的起始地址是0x0000的话，数组CARD_TABLE的第0、1、2号元素，分别对应了地址范围为0x0000～0x01FF、0x0200～0x03FF、0x0400～0x05FF的卡页内存块

![image-20210916095918339](imgs\59.png)

一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或更多）对象的字段存在着跨代指针，那就将对应卡表的数组元素的值标识为1，称为这个元素变脏（Dirty），没有则标识为0。在垃圾收集发生时，只要筛选出卡表中变脏的元素，就能轻易得出哪些卡页内存块中包含跨代指针，把它们加入GC Roots中一并扫描

> 以下面G1的region之间的跨代引用为例，对于region2，它的记忆集就存放region1和region3指向自己的指针所在的卡页，表明这两个卡页中存在跨代引用，是“脏的”
>
> 对于新生代而言，对应的记忆集就存放了老年代指向自己的指针；在G1中，每个region都有一个RSet

![image-20210915201928614](imgs\57.png)

**写屏障**

我们已经解决了如何使用记忆集来缩减GC Roots扫描范围的问题，但还没有解决卡表元素如何维护的问题，例如它们何时变脏、谁来把它们变脏等

卡表元素何时变脏的答案是很明确的——有其他分代区域中对象引用了本区域对象时，其对应的卡表元素就应该变脏，变脏时间点原则上应该发生在引用类型字段赋值的那一刻。但问题是如何变脏，即如何在对象赋值的那一刻去更新维护卡表呢？假如是解释执行的字节码，那相对好处理，虚拟机负责每条字节码指令的执行，有充分的介入空间；但在编译执行的场景中呢？经过即时编译后的代码已经是纯粹的机器指令流了，这就必须找到一个在机器码层面的手段，把维护卡表的动作放到每一个赋值操作之中

在HotSpot虚拟机里是通过写屏障（Write Barrier）技术维护卡表状态的。注意将这里提到的“写屏障”，以及后面在低延迟收集器中会提到的“读屏障”与解决并发乱序执行问题中的“内存屏障”区分开来，避免混淆。写屏障可以看作在虚拟机层面**对“引用类型字段赋值”这个动作的AOP切面**，在引用对象赋值时会产生一个环形（Around）通知，供程序执行额外的动作，也就是说赋值的前后都在写屏障的覆盖范畴内。在赋值前的部分的写屏障叫作写前屏障（Pre-Write Barrier），在赋值后的则叫作写后屏障（Post-Write Barrier）。HotSpot虚拟机的许多收集器中都有使用到写屏障，但直至G1收集器出现之前，其他收集器都只用到了写后屏障。下面这段代码清单是一段更新卡表状态的简化逻辑：

![image-20210916100803428](imgs\60.png)

除了写屏障的开销外，卡表在高并发场景下还面临着“伪共享”（False Sharing）问题。假设处理器的缓存行大小为64字节，由于一个卡表元素占1个字节，64个卡表元素将共享同一个缓存行。这64个卡表元素对应的卡页总的内存为32KB（64×512字节），也就是说如果不同线程更新的对象正好处于这32KB的内存区域内，就会导致更新卡表时正好写入同一个缓存行而影响性能。为了避免伪共享问题，一种简单的解决方案是不采用无条件的写屏障，而是先检查卡表标记，只有当该卡表元素未被标记过时才将其标记为变脏，即将卡表更新的逻辑变为以下代码所示：

![image-20210916103131804](imgs\61.png)

只有判断没标记时，才去标记，而不是每次写屏障时都去更改，减少发生伪共享的次数

> 在JDK 7之后，HotSpot虚拟机增加了一个新的参数-XX：+UseCondCardMark，用来决定是否开启卡表更新的条件判断。开启会增加一次额外判断的开销，但能够避免伪共享问题，两者各有性能损耗，是否打开要根据应用实际运行情况来进行测试权衡

### 标记阶段

需要区分出内存中哪些是存活对象，哪些是已经死亡的对象。只有被标记为死亡的对象才会在GC阶段被回收。当一个对象已经不再被任何存活的对象所引用的时候，就可以宣判死亡

#### 引用计数算法

对每个对象保存一个引用计数器属性，用于记录对象被引用的情况，被引用了就加一，引用失效就减一，值为0就表示不被使用，可以回收

**优点**

- 实现简单
- 垃圾对象易于辨识、判定效率高
- 回收没有延迟，不需要等内存不够，只需要发现0就回收

**缺点**

- 对象需要单独的字段存储计数器，增加了存储空间的开销
- 每次赋值都需要更新计数器，增加时间开销
- 无法处理循环引用（比如循环链表等等），这是致命的缺陷，导致Java的垃圾回收器没有使用该算法

> 可达性分析算法能在效率较高的情况下，解决循环引用问题

#### 可达性分析算法

这种类型的垃圾回收通常也叫做追踪性垃圾收集（Tracing Garbage Collection）

**思路**

1. 以根对象集合（GC Roots）作为起点，从上到下搜索被根对象集合所连接的对象是否可达

2. 使用可达性分析算法后，内存中的对象都会被根对象集合直接或间接连接着，搜索的路径称为引用链（Reference Chain）

   ![image-20210826210448867](imgs\38.png)

3. 如果对象没有被引用链相连，说明是不可达的，意味着该对象已经死亡，可以标记为垃圾对象

> 进行这种分析时，不能是动态的，也就是说必须在当前一个能保证一致性的快照中进行分析，这就是GC时必须进行“Stop The World”的原因

**OopMap**

当用户线程停顿下来之后，其实并不需要一个不漏地检查完所有执行上下文和全局的引用位置，虚拟机应当是有办法直接得到哪些地方存放着对象引用的。在HotSpot的解决方案里，是使用一组称为OopMap的数据结构来达到这个目的。一旦类加载动作完成的时候，HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译过程中，也会在特定的位置记录下栈里和寄存器里哪些位置是引用。这样收集器在扫描时就可以直接得知这些信息了，并不需要真正一个不漏地从方法区等GC Roots开始查找

#### finalize机制

- Java语言提供了对象终止（finalization）机制来允许开发人员提供对象被销毁之前的自定义处理逻辑
- 当垃圾回收器发现没有引用指向一个对象，即：垃圾回收此对象之前，总会先调用这个对象的finalize()方法
- finalize()方法允许在子类中被重写，用于在对象被回收时进行资源释放。通常在这个方法中进行一些资源释放和清理的工作，比如关闭文件、套接字和数据库连接等
- **永远不要主动调用finalize方法，应当交给垃圾回收器调用**
  - finalize可能导致对象复活
  - finalize方法的执行时间是没有保证的，完全由GC线程决定，极端情况下，若不发生GC，则finalize方法将没有执行机会
  - 一个糟糕的finalize方法会严重影响GC性能

由于finalize机制的存在，虚拟机中的对象一般处于三种可能的状态：

- 可触及的：从GC Roots开始，可以到达这个对象

- 可复活的：对象的所有引用都被释放，但可能在finalize中复活

  > ```java
  > public class JVM {
  >     private static Rebirth obj;
  >     public static void main(String[] args) throws InterruptedException {
  >         // 令GC Roots obj指向new Rebirth()出来的对象，称为objR
  >         obj = new Rebirth();
  >         // objR没有引用指向了
  >         obj = null;
  >         // gc，objR被第一次标记
  >         System.gc();
  >         // 由于Finalizer优先级很低，所以等两秒让它运行
  >         // 这个过程中，Finalizer调用objR的finalize方法
  >         Thread.sleep(2000);
  >         if(obj == null){
  >             System.out.println("raw rebirth is dead");
  >         }else {
  >             System.out.println("raw rebirth is alive");
  >         }
  >     }
  >     static class Rebirth{
  >         Rebirth(){
  >             System.out.println("I'm object " + this);
  >         }
  >     }
  > }
  > ```
  >
  > 结果1，未重写finalize()，直接被二次标记成垃圾，被回收：
  >
  > ![image-20210827113212185](imgs\39.png)
  >
  > 修改Rebirth为：
  >
  > ```java
  > static class Rebirth{
  >     Rebirth(){
  >         System.out.println("I'm object " + this);
  >     }
  > 
  >     @Override
  >     protected void finalize() throws Throwable {
  >         // 此时在F-Queue中，被Finalizer调用
  >         // 令外部的obj重新指向自己
  >         obj = this;
  >         System.out.println("I'm " + obj + ", I rebirth!");
  >         // 从“即将回收”集合中移除了
  >     }
  > }
  > ```
  >
  > 结果2，逃生成功：
  >
  > ![image-20210827113726687](imgs\40.png)
  >
  > > 一般用于资源回收关闭什么的，不能像上述这么写

- 不可触及的：对象的finalize被调用，并且没有复活，就会进入不可触及状态。不可触及的状态不可能被复活，因为finalize方法只会被调用一次

#### 标记阶段总结

判断一个对象objA是否可回收，至少要经历两次标记过程：

1. 如果objA到GC Roots没有引用链，则进行第一次标记

2. 进行筛选，判断该对象是否有必要执行finalize方法

   - 如果objA没有重写finalize方法或者已经执行过finalize方法，直接被标记为不可触及的

   - 如果objA重写了finalize方法且没有执行，则会插入到F-Queue中，由一个JVM创建的低优先级的Finalizer线程负责调用它的finalize方法，这是对象最后的逃脱机会

   - 之后，GC对F-Queue中的对象进行第二次标记，objA如果在finalize中与引用链建立了联系，则会被移出“即将回收”集合

     > 如果之后，该对象再次出现没有引用的状况，则一定会判定为不可触及的，因为finalize只能调用一次

### 清除阶段

#### 标记-清除算法（Mark-Sweep）

- 执行过程

  当堆中的有效内存空间（available memory）被耗尽的时候，会停止整个程序（Stop The World），然后进行两项工作，一是标记，二是清除

  - 标记：Collector根据GC Roots，标记出所有被引用的对象，并在对象的Header中记录为可达对象
  - 清除：Collector对堆空间进行线性遍历，如果发现某个对象的Header中没有记录为可达对象，则回收

- 缺点

  - 效率不高
  - 需要STW，用户体验差
  - 清理出来的内存空间不连续，有内存碎片，需要维护一个空闲列表

> 何为清除？
>
> 并不是真的置空，而是把需要清除的对象的地址保存在空闲列表中，下次有新对象存放时，判断空闲列表的空间是否够，如果够，直接放下覆盖即可

#### 标记-复制算法（Copying）

为了解决标记清除算法的效率和碎片化缺陷，提出标记-复制算法，一般简称为复制算法

- 核心思想

  将活着的内存空间分为两块，每次只使用其中一块，垃圾回收时将该空间内所有存活对象复制到未使用的另一块空间，然后回收全部该空间的内存，并交换两块内存的角色

- 优点

  - 复制过去后保持空间的连续性，不会产生内存碎片

- 缺点

  - 需要消耗内存空间
  - 对象的复制会导致内存地址的改变，对于G1这种分拆成大量region的GC，需要维护region之间的对象引用关系，也就是指向该对象的各种引用需要不断更改，造成时间空间开销大

> 如果系统中垃圾对象很少，要复制的对象很多，就比较不理想
>
> 而在新生代中，一次回收可以回收70%-99%的垃圾，存活对象较少，所以效果很好，回收性价比高，因此目前的商用虚拟机一般都在新生代采用这种算法

#### 标记-压缩/整理算法（Mark-Compact）

对于新生代，需要复制的对象较少，因此可以采用复制算法；而对于老年代这种大部分都是存活对象的，复制成本很高，需采用其他算法

标记清除算法确实可以应用在老年代中，但由于效率低下、内存碎片化，所以要做出改进，标记压缩算法由此而生

> 标记清除算法在回收阶段效率不低，但因内存分配和访问相比垃圾收集频率要高得多，这部分的耗时增加，总吞吐量仍然是下降的

- 执行过程

  - 标记：从GC Roots开始标记所有的被引用对象
  - 压缩：将所有被标记对象压缩到内存一端，按顺序排放；之后，清理边界外所有的空间

  清理时，JVM只需持有一个起始内存地址即可，比空闲列表好多了

- 优点
  - 解决内存碎片问题
  - 消除了复制算法中内存减半的高额代价
- 缺点
  - 效率低于复制算法
  - 移动对象时需要调整指向它的引用
  - 整理碎片的移动过程中，仍会有STW的过程

#### 效率对比

![image-20210906213802008](imgs\42.png)

## 分代收集

**不同生命周期的对象可以采取不同的收集方式，以便提高回收效率**，比如Hotspot VM的新生代、老年代

### 不同阶段的开销

- Mark阶段的开销和存活对象的数量成正比
- Sweep阶段的开销与所管理的内存大小呈正相关
- Compact阶段与存活数据的大小呈正相关

#### 结论

因此，对于新生代，采取复制算法复制少量的存活对象，效率高，没有内存碎片，同时，hotspot采用eden区和两个较小的survivor区的设计优化了复制算法，使得内存利用率提高了；而对于老年代，存活对象较多，明显不适合采用复制算法，则需要结合标记清除和标记整理算法

几乎目前所有的GC都是采用分代收集（Generational Collecting）算法执行垃圾回收的

## 增量收集

垃圾回收过程中，有一个重要的问题就是STW，即Stop The World，需要停止所有字节码的执行，等待垃圾收集的完成，影响效率，由此诞生了增量收集算法

### 基本思想

让垃圾收集线程和应用程序线程交替执行。每次垃圾收集只收集一小片区域内存空间，接着切换到应用程序，如此循环直到垃圾回收完成

**处理收集线程和应用程序之间的冲突，允许收集线程分小阶段完成标记、清除、复制或整理工作**

- 优点：减少系统停顿
- 缺点：线程切换以及上下文转换的消耗，导致系统吞吐量下降，总体成本上升

## 分区算法

### 基本思想

将一块大的内存区域分割成若干的小区间，根据目标的停顿时间，每次合理回收若干个小区间的内存，而不是整个堆空间，从而减少GC停顿时间

**堆内存分割成若干小区间（region），每个小区间独立回收**

![image-20210907111345904](imgs\43.png)

### 区别

- 与分代收集的区别：分代收集按对象生命周期长短划分为两个不同的部分；分区收集是将堆内存划分成连续的不同小区间
- 与增量收集的区别：增量收集过程发生线程交替切换；分区收集每次都是完整的过程，只不过区域较小

> 目前的GC都是复合算法，且并发并行都有

## 垃圾回收器

### 分类

- 按线程数分

  - 串行回收器：同一时间段内只允许有一个cpu进行垃圾回收操作，此时工作线程被暂停，直到垃圾收集工作结束

    > 在诸如单cpu处理器或者较小的应用内存等硬件平台不是特别流畅的场合，串行收集器的性能表现可以超过并行和并发
    >
    > 默认应用在client模式下

  - 并行回收器：多个cpu执行垃圾回收，提升了应用的吞吐量，不过仍需要停止用户线程（STW），采用独占式回收

- 按工作模式分

  - 独占式：停止用户线程，直到垃圾回收完全结束
  - 并发式：垃圾回收与用户线程交替工作，减少应用停顿的时间

- 按碎片处理方式

  - 压缩式
  - 非压缩式

- 按工作区间分

  - 年轻代
  - 老年代

### 性能指标

- **吞吐量：运行用户代码的时间占总运行时间的比例**
- 垃圾收集开销：垃圾回收的时间占总运行时间的比例
- **暂停时间：STW的时间**
- 收集频率：相对于应用程序的执行，收集操作发生的频率
- **内存占用：java堆区占用内存大小**
- 快速：一个对象从被创建到被回收的时间

三个加粗的内容，共同构成一个“不可能三角”。三者总体的表现会随技术进步越来越好，但目前优秀的垃圾收集器最多同时满足其中两项

主要关注吞吐量和暂停时间

#### 吞吐量

关注系统应用程序连续运行，不频繁切换，具有较高吞吐量，在这种情况下不需STW的绝对时间短，只需在总体过程中所占比例小即可

#### 暂停时间

关注的是STW的暂停绝对时间，每次暂停时间短，势必会导致内存回收频繁，从而吞吐量下降，在追求低延迟的应用中会用到

**这两者一定程度上是矛盾的**，追求吞吐量的话，势必会要求内存回收频率的降低，为了充分回收，那么每次回收的时间就需要更长，这与暂停时间是矛盾的。追求暂停时间的话，势必会导致内存回收频繁，线程切换频繁，切换的时间开销和暂停的时间开销导致吞吐量下降

一般标准：在最大吞吐量有限的情况下，尽量降低暂停时间。是一个相对折中的方案

### 发展历史

![image-20210914152656360](imgs\50.png)

**7款经典的垃圾收集器**

- 串行回收器：Serial、Serial Old
- 并行回收器：ParNew、Parallel Scavenge、Parallel Old
- 并发回收器：CMS、G1

![image-20210914154339542](imgs\51.png)

![image-20210914152834395](imgs\49.png)

> 图中是新生代和老年代收集器的搭配
>
> CMS GC和Serial Old GC之间的实线比较特殊，会在老年代满之前提前回收，因为用户线程不会暂停，仍会产生垃圾，所以得在满之前回收。可能出现CMS失败的情况（Current Mode Failure），比如垃圾产生速度大于回收的速度，这时候需要以Serial Old GC作为后备方案
>
> 红线是在JDK8中被deprecated，且在9中被彻底移除的配对
>
> 绿线是在JDK14中被deprecated的配对，CMS被删掉了

### Serial收集器：串行回收

#### Serial

最基本、历史最悠久的垃圾收集器，JDK1.3之前回收新生代的唯一选择

是hotspot中client模式下的新生代回收默认选择

采用复制算法、串行回收，有Stop-The-World

#### Serial Old

执行老年代垃圾收集，使用标记-压缩算法

是client模式下默认的老年代收集器

server模式下有两个用途：

1. 与新生代的Parallel Scavenge配合使用
2. 作为老年代CMS收集器的后备方案

> 可以使用`-XX:+UseSerialGC`，令新生代采用Serial，老年代采用Serial Old

#### 优势

- 相比其他收集器的单线程相比，简单而高效，专心进行垃圾回收，不考虑线程切换

- client模式下是不错的选择

> 现在几乎都不采用了，在交互性强的应用中，单线程处理时很长的STW是不可接受的

### ParNew收集器：并行回收

Par：Parallel，并行的；New：新生代

相当于Serial的多线程版本，采用的也是复制算法，STW，由于多个cpu加快了垃圾回收，STW时间缩短

很多JVM server模式下的默认的新生代回收选择

> 新生代回收频繁，采用并行能够提高处理效率；老年代回收少，利用串行可以减少切换的开销

ParNew一定比Serial高效吗？不是这样的，在单cpu的场景下，反而会增加线程切换的开销

> 除Serial外只有ParNew可以和CMS配合工作

> 可以使用`-XX:+UseParNewGC`，令新生代采用ParNew，不影响老年代
>
> 可以使用`-XX:ParallelThreads`，限制线程数量，默认为cpu数

### Parallel收集器：吞吐量优先

全称Parallel Scavenge，同样采用复制算法、并行回收、STW，性能差别也不是特别大

主要差别在于它的目标是**达到一个可控制的吞吐量**，此外还具有自适应调节策略

> 高吞吐量的任务适合在后台运行而不需要太多交互的任务
>
> 在JDK1.6提供了Parallel Old，用来代替Serial Old
>
> Parallel Old采用标记压缩算法，但是基于并行的（标记清除算法由于内存分配和访问效率低，因此吞吐量低，Parallel中不被采用）

在server下性能比较不错，对于用户来讲，停顿越少越好；但对于服务器，有高并发等需求，注重整体的吞吐量，所以服务器端适合Parallel

在java8中，默认是此收集器

> 可以使用`-XX:+UseParallelGC`，令年轻代使用Parallel Scavenge
>
> 可以使用`-XX:+UseParallelOldGC`，令老年代代使用Parallel Old
>
> 默认开启一个，另一个也激活
>
> **其他设置**
>
> - STW时间：`-XX:MaxGCPauseMillis=<n ms>`
>
>   为了尽可能满足该参数的需求，会自适应调正堆大小以及其他一些参数
>
>   该参数使用需谨慎
>
> - GC时间的比例：`-XX:GCTimeRatio`，取值范围0-100，默认99，表示用户线程时间：GC时间=99：1
>
>   与上面的参数具有一定矛盾性
>
> - 自适应调节策略：`-XX:+UseAdaptiveSizePolicy`
>
>   堆中各个部分的大小等参数自适应调正，以达到堆大小、吞吐量和STW时间的平衡点
>
>   手动调优比较困难的场合，可以仅指定最大堆、STW时间、GC时间的比例，并采用这一参数，让虚拟机自适应调整

### CMS收集器：低延迟

JDK1.5，hotspot推出CMS（Concurrent Mark Sweep）收集器，在强交互应用中具有划时代意义，真正意义上的并发收集器，第一次实现了让垃圾收集线程和用户线程同时工作

采用标记清除算法，同样也会STW

无法与Parallel Scavenge配合，只能使用Serial或ParNew来配合

#### 并发的可达性分析

在CMS中，有并发的可达性分析

在根节点枚举步骤中，由于GC Roots相比起整个Java堆中全部的对象毕竟还算是极少数，且在各种优化技巧（如OopMap）的加持下，它带来的停顿已经是非常短暂且相对固定（不随堆容量而增长）的了。可从GC Roots再继续往下遍历对象，对象图结构越复杂，要标记更多对象而产生的停顿时间自然就更长

**想解决或者降低用户线程的停顿，就要先搞清楚为什么必须在一个能保障一致性的快照上才能进行对象图的遍历？**

为了能解释清楚这个问题，我们引入三色标记（Tri-color Marking）作为工具来辅助推导，把遍历对象图过程中遇到的对象，按照“是否访问过”这个条件标记成以下三种颜色： 

- 白色：表示对象尚未被垃圾收集器访问过。显然在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达
- 黑色：表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经扫描过。黑色的对象代表已经扫描过，它是安全存活的，如果有其他对象引用指向了黑色对象，无须重新扫描一遍。黑色对象不可能直接（不经过灰色对象）指向某个白色对象
- 灰色：表示对象已经被垃圾收集器访问过，但这个对象上至少存在一个引用还没有被扫描过

关于可达性分析的扫描过程，把它看作对象图上一股以灰色为波峰的波纹从黑向白推进的过程，如果用户线程此时是冻结的，只有收集器线程在工作，那不会有任何问题。但如果用户线程与收集器是并发工作呢？收集器在对象图上标记颜色，同时用户线程在修改引用关系，即修改对象图的结构，这样可能出现两种后果。一种是把原本消亡的对象错误标记为存活，产生浮动垃圾；另一种是把原本存活的对象错误标记为已消亡，这就是**非常致命的后果**了，程序肯定会因此发生错误，下面演示了这样的致命错误具体是如何产生的

![image-20210914224037028](imgs\54.png)

Wilson于1994年在理论上证明了，当且仅当以下两个条件同时满足时，会产生“对象消失”的问题，即原本应该是黑色的对象被误标为白色：① 插入了一条或多条从黑色对象到白色对象的新引用； ② 删除了全部从灰色对象到该白色对象的直接或间接引用

因此，我们要解决并发扫描时的对象消失问题，只需破坏这两个条件的任意一个即可。由此分别产生了两种解决方案：增量更新（Incremental Update）和原始快照（Snapshot At The Beginning，SATB）

- 增量更新要破坏的是第一个条件，当黑色对象插入新的指向白色对象的引用关系时，就将这个新插入的引用记录下来，等并发扫描结束之后，再将这些记录过的引用关系中的黑色对象为根，重新扫描一次。这可以简化理解为，黑色对象一旦新插入了指向白色对象的引用之后，它就变回灰色对象了
- 原始快照要破坏的是第二个条件，当灰色对象要删除指向白色对象的引用关系时，就将这个要删除的引用记录下来，在并发扫描结束之后，再将这些记录过的引用关系中的灰色对象为根，重新扫描一次。这也可以简化理解为，无论引用关系删除与否，都会按照刚刚开始扫描那一刻的对象图快照来进行搜索

以上无论是对引用关系记录的插入还是删除，虚拟机的记录操作都是通过写屏障实现的。在HotSpot虚拟机中，增量更新和原始快照这两种解决方案都有实际应用，譬如，CMS是基于增量更新来做并发标记的，G1、Shenandoah则是用原始快照来实现

#### 工作流程

![image-20210914195157956](imgs\53.png)

1. 初始标记：STW，一个GC线程独占一个CPU，仅标记GCroots能直接关联的对象

   > 只标记直接关联的，速度很快，STW也很短

2. 并发标记：可以和用户线程同时执行，标记所有可达对象（并发的可达性分析）

3. 重新标记：STW，GC线程独占CPU，多个GC线程并行，对并发标记阶段用户线程运行产生的垃圾对象进行标记修正

   > 并发标记阶段用户线程也在运行，有一部分对象的标记会发生变动，所以需要修正（增量更新）

4. 并发清理：可以和用户线程并行执行，清理垃圾

   > 为什么不用标记压缩？
   >
   > 因为整理对象的时候，需要移动对象，但此时用户线程仍然在执行，若对象地址改变，可能导致不可预计的问题

最耗时的并发标记和并发清除阶段是可以和用户线程并行的，所以整体是低停顿的

CMS等到一个占用内存到达一个阈值后就开始回收，如果CMS预留的内存不够，会出现“Concurrent Mode Failure”，这时虚拟机会采用后备的预案，即临时启用Serial Old进行老年代的收集

#### 优缺点

**优点**

- 并发收集
- 低延迟

**缺点**

- 产生内存碎片，导致并发清除后，用户线程可用的空间不足，导致无法分配大对象，提前触发FGC
- 对cpu资源敏感，并发阶段，GC线程会抢夺一部分属于用户线程的cpu资源，造成总吞吐量下降
- 无法处理浮动垃圾，并发标记阶段出现的新的垃圾无法标记，只能等到下一次GC

#### 参数设置

可以使用`-XX:+UseConcMarkSweepGC`，手动使用CMS，会自动打开`-XX:+UseParNewGC`，最终是ParNew+CMS+Serial Old的组合

使用`-XX:CMSInitiatingOccupanyFraction`设置堆内存老年代使用率的阈值，超过这个阈值开始CMS GC。JDK5及之前默认为68%，之后为92%。如果内存增长率低，可以将阈值设高一点，避免触发CMS，减少老年代GC可以改善性能；反之可以设低一点，避免频繁进入Serial Old GC。由此，根据这个参数可以有效降低FGC的次数

`-XX:+UseCMSCompactAtFullCollection`，FGC之后对内存空间进行压缩整理

`-XX:CMSFullGCsBeforeCompaction`，执行多少次FGC后进行压缩整理

`-XX:ParallelCMSThreads`，设置CMS的线程数量

> 默认为(处理器核心数+3)/4
>
> 当cpu资源紧张时，在并发清除阶段，应用程序的性能可能比较糟糕

JDK9中，CMS被标记为Deprecated，当使用`-XX:+UseConcMarkSweepGC`时，会收到一个warning，提醒将被废弃；14中删除了CMS，如果设置使用，会warning但不会退出，而是JVM回退，并改成默认的GC

### G1收集器

> 区域划、分代式、分区收集

> 随着内存的不断增大和处理器的不断增加，需要更好的GC来满足需求
>
> G1GC官方设定的目标，是在延迟可控的情况下获得尽可能高的吞吐量，即“全功能收集器”

G1是一个并行收集器，它把堆内存分割为很多不相关的区域（Region，物理上是不连续的），并用不同区域来表示Eden、Survivor0/1、Old等等

G1GC会有计划的避免在整个Java堆中进行全区域的垃圾收集。G1会跟踪各个Region里面的垃圾堆积的价值大小（这个价值与回收后所获得的空间大小以及回收所需的时间有关），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region

> 而不是像之前那样，局限于对新生代收集，对老年代收集，衡量标准不再是它属于哪个分代，而是哪块内存中存放的垃圾数量最多，回收收益最大

侧重点在于回收垃圾最大量的区间，因此称为Garbage First

是面向服务端应用的GC，能够在**极大概率**满足停顿时间的要求下，尽可能使吞吐量更高

JDK1.7正式启用，移除Experimental标志，是JDK9及之后的默认GC，CMS以及Parallel组合被废弃（deprecated），后续版本中被移除

#### Region

![image-20210915195521735](imgs\55.png)

- 内存的回收以region为基本单位，所有region大小相等

- 每个region的角色（Eden/Old/Survivor/Humongous）是可变的

- Humongous用来放置大对象，即超过0.5个region的对象

  > 对于那些超过了整个Region容量的超级大对象，将会被存放在N个连续的Humongous Region之中，G1的大多数行为都把Humongous Region作为老年代的一部分来进行看待，直接分配到了old gen，防止了反复拷贝移动
  >
  > H-obj在global concurrent marking阶段的 cleanup 和 full GC阶段回收
  >
  > 在分配H-obj之前先检查是否超过 initiating heap occupancy percent和the marking threshold, 如果超过的话，就启动global concurrent marking，为的是提早回收，防止 evacuation failures 和 full GC

- region之间是复制算法，整体上可看作是标记压缩的

**region的跨代引用问题**

每个Region都维护有自己的记忆集，这些记忆集会记录下别的Region指向自己的指针，并标记这些指针分别在哪些卡页的范围之内

G1的记忆集在存储结构的本质上是一种哈希表，Key是别的Region的起始地址，Value是一个集合，里面存储的元素是卡表的索引号

**CSet**

[美团技术博客](https://tech.meituan.com/2016/09/23/g1.html)

Collection Set（CSet），它记录GC要收集的Region集合，集合里的Region可以是任意年代的

#### 特点

- 并行与并发
- 分代收集，兼顾年轻代和老年代
- 可预测的停顿时间模型（即：软实时，soft real-time，大概率能在指定时间内完成收集），能够让用户明确指定在一个M毫秒的时间段内，GC的时间不超过N毫秒
  - 分区缩小的回收的范围，全局的停顿时间更易控制
  - 每次根据价值大小的优先队列选择性收集，能在有限时间内获得最大的收集效率
  - 相比于CMS，G1的停顿时间未必能达到CMS的最佳性能，但是最差情况下要好很多

相较于CMS，G1还不具备全方位、压倒性优势，比如在用户程序运行过程中，G1产生的内存占用（Footprint）还是额外执行负载（Overload）都比CMS要高。一般来说，小内存CMS表现好，大内存G1好，平衡点在6-8GB之间

> G1至少要耗费大约相当于Java堆容量10%至20%的额外内存来维持收集器工作

#### 参数设置

`-XX:+UseG1GC`：手动指定使用G1GC

`-XX:G1HeapRegionSize`：设置每个region的大小，为2的幂，范围1MB—32MB，目标是根据最小的java堆内存大小划分出2048个区域。默认是堆内存的1/2000

`-XX:MaxGCPauseMillis`：期望达到的最大停顿时间阈值，默认为200ms

`-XX:ParallelGCThreads`：STW时并行的GC线程数，默认为8

`-XX:ConcGCThreads`：并发标记时GC线程数，可以设置为ParallelGCThreads的1/4左右

`-XX:InitiatingHeapOccupanyPercent`：超过该堆内存占用率值就触发global concurrent marking，即Mixed GC流程，默认为45

G1的一大目的是简化JVM调优，主要分为三步：

1. 打开G1GC
2. 设置最大堆内存
3. 设置最大期望停顿时间

G1提供了YGC、Mixed GC和FGC（Serial Old），在不同条件下被触发

G1GC可以调用应用线程来加速GC，而其他GC收集器只能通过JVM内置的GC线程

#### 工作流程

**global concurrent marking**

[美团技术博客](https://tech.meituan.com/2016/09/23/g1.html)

[https://cloud.tencent.com/developer/article/1824886](https://cloud.tencent.com/developer/article/1824886)

[https://www.jianshu.com/p/aef0f4765098](https://www.jianshu.com/p/aef0f4765098)

G1提供了两种GC模式，Young GC和Mixed GC，两种都是完全Stop The World的

Young GC：选定所有年轻代里的Region。通过控制年轻代的region个数，即年轻代内存大小，来控制young GC的时间开销

Mixed GC：选定所有年轻代里的Region，外加根据global concurrent marking统计得出收集收益高的若干老年代Region。在用户指定的开销目标范围内尽可能选择收益高的老年代Region

由上面的描述可知，Mixed GC不是full GC，它只能回收**部分**老年代的Region，如果mixed GC实在无法跟上程序分配内存的速度，导致老年代填满无法继续进行Mixed GC，就会使用serial old GC（full GC）来收集整个GC heap。所以我们可以知道，G1是不提供full GC的

上文中提到了global concurrent marking，它的执行过程类似CMS，但是不同的是，在G1 GC中，它主要是为Mixed GC提供标记服务的，并不是一次GC过程的一个必须环节。global concurrent marking的执行过程分为四个步骤：

1. 初始标记（Initial Marking）

   仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS指针的值，让下一阶段用户线程并发运行时，能正确地在可用的Region中分配新对象

   这个阶段需要停顿线程，但耗时很短，而且是借用进行Minor GC的时候同步完成的，因为他们可以复用root scan操作，所以G1收集器在这个阶段实际并没有额外的停顿。所以可以说global concurrent marking是伴随Young GC而发生的

   > G1为每一个Region设计了两个名为TAMS（Top at Mark Start）的指针，把Region中的一部分空间划分出来用于并发回收过程中的新创建的对象的分配，并发回收时新分配的对象地址都必须要在这两个指针位置以上。G1收集器默认在这个地址以上的对象是被隐式标记过的，即默认它们是存活的，不纳入回收范围

   > 注意，是只复用了YGC的第一个步骤，后面步骤是另外的流程

2. 并发标记（Concurrent Marking）

   从GC Roots开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户程序并发执行

3. 最终标记（Final Marking）

   对用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留下来的最后那少量的SATB（原始快照）记录 

4. 清除垃圾（Clean up）

   清除空region（没有存活对象的），加入free list，空闲region列表。第四阶段Cleanup只是回收了没有存活对象的Region，所以它并不需要STW，并不会清理垃圾对象，也不会执行存活对象的拷贝

**整体流程**

1. global concurrent marking，不断循环进行

2. 回收阶段（Evacuation）

   1. global concurrent marking获取了老年代的垃圾占比数据，JVM决定是否进行Mixed GC

      > - G1HeapWastePercent：在global concurrent marking结束之后，我们可以知道old gen regions中有多少空间要被回收，在每次YGC之后和再次发生Mixed GC之前，会检查垃圾占比是否达到此参数，只有达到了，下次才会发生Mixed GC

   2. 筛选回收（Live Data Counting and Evacuation）：负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集（CSet），然后把决定回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间。这里的操作涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行完成的

      > - G1MixedGCLiveThresholdPercent：old generation region中的存活对象的占比，只有在此参数之下，才会被选入CSet
      > - G1MixedGCCountTarget：一次global concurrent marking之后，最多执行Mixed GC的次数
      > - G1OldCSetRegionThresholdPercent：一次Mixed GC中能被选入CSet的最多old generation region数量

![image-20210916201038076](imgs\56.png)

![image-20210915202340560](imgs\58.png)

**总结**

从上述阶段的描述可以看出，G1收集器除了并发标记和Clean up外，其余阶段也是要完全暂停用户线程的，换言之，它并非纯粹地追求低延迟，官方给它设定的目标是在延迟可控的情况下获得尽可能高的吞吐量，所以才能担当起“全功能收集器”的重任与期望。从Oracle官方透露出来的信息可获知，回收阶段（Evacuation）其实本也有想过设计成与用户程序一起并发执行，但这件事情做起来比较复杂，考虑到G1只是回收一部分Region，停顿时间是用户可控制的，所以并不迫切去实现，而选择把这个特性放到了G1之后出现的低延迟垃圾收集器（即ZGC）中。另外，还考虑到G1不是仅仅面向低延迟，停顿用户线程能够最大幅度提高垃圾收集效率，为了保证吞吐量所以才选择了完全暂停用户线程的实现方案

![image-20210916201038076](imgs\62.png) 

> 如果内存回收的速度赶不上内存分配的速度， 
>
> G1收集器也要被迫冻结用户线程执行，导致切换至Serial Old进行Full GC而产生长时间“Stop The World”

举个例子：一个web服务器，Java进程最大堆内存为4G，每分钟响应1500个请求，每45秒钟会新分配大约2G的内存。G1会每45秒进行一次年轻代回收，每31个小时整个堆的使用率会达到45%（InitiatingHeapOccupanyPercent），会开始老年代并发标记过程，也就是global concurrent marking过程，标记完成后开始四到五次的混合回收

#### 优化建议

避免使用`-Xmn`、`-NewRatio`来固定新生代内存大小，这样可能导致JVM无法动态调整内存，从而覆盖期待暂停时间参数的效果

期待暂停时间不要设置的太严苛

![image-20210922094007947](imgs\63.png)

> 注意，在后续版本中`-XX:PrintGC`和`-XX:PrintGCDetails`分别用`-Xlog:gc`和`-Xlog:gc*`来替代，变成了hotspot vm的稳定参数
>
> `-Xloggc:./path/gc.log`可用于线上日志的记录，每次gc会刷新一次gc.log，使用GCEasy进行分析查看

![image-20210922120919072](imgs\64.png)

![image-20210922121011806](imgs\65.png)

### 上述几种的区别

![img](imgs\52.png)

### 新发展

#### Epsilon

A No-Op Garbage Collector

只做内存分配，不回收

#### ZGC

JDK11

- 尽可能在不影响吞吐量的情况下降低延迟
- 并发的标记压缩算法

#### Shenandoah

JDK12

低停顿时间

# Class文件结构

- 字节码：一个字节长度，形式为`opcode operand`，如`bipush 30`
- Class本质：一组以8位字节为基础单位的二进制流，不一定以磁盘文件的形式存在

Class的结构不像XML等描述语言，由于它没有任何分隔符号。所以在其中的数据项，无论是字节顺序还是数量，都是被严格限定的，哪个字节代表什么含义，长度是多少，先后顺序如何，都不允许改变

**概览**

Class文件格式采用一种类似于C语言结构体的方式进行数据存储，这种结构中只有两种数据类型：无符号数和表

- 无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8 编码构成字符串值
- 表是由多个无符号数或者其他表作为数据项构成的复合数据类型，所有表都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据，整个Class文件本质上就是一张表。由于表没有固定长度，所以通常会在其前面加上个数说明

## 基本结构

- 魔数
- Class文件版本
- 常量池
- 访问标志
- 类索引，父类索引，接口索引集合
- 字段表集合
- 方法表集合
- 属性表集合

![image-20211018223907547](imgs\66.png)

```c++
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

### magic-魔数

class用magic魔数来作为class文件的标识，而不是扩展名，主要是基于安全方面的考虑。因为扩展名是可以随意更改的，且只是系统用来识别的；而magic固定是0xCAFEBABE





# 参考

[1] [尚硅谷JVM](https://www.bilibili.com/video/BV1PJ411n7xZ?p=56&spm_id_from=pageDriver)

[2] [https://www.zhihu.com/question/30300585/answer/51335493](https://www.zhihu.com/question/30300585/answer/51335493)

[3] [https://blog.csdn.net/lu322313/article/details/104706578——L.T.F.](https://blog.csdn.net/lu322313/article/details/104706578)

[4] [https://www.zhihu.com/question/49044988/answer/113961406——毛海山](https://www.zhihu.com/question/49044988/answer/113961406)

[5] [https://www.cnblogs.com/ding-dang/p/13085355.html——叮叮叮叮叮叮当](https://www.cnblogs.com/ding-dang/p/13085355.html)

[6] [https://www.zhihu.com/question/34341582/answer/58444959](https://www.zhihu.com/question/34341582/answer/58444959)

[7] [https://cloud.tencent.com/developer/article/1824886——落叶飞翔的蜗牛](https://cloud.tencent.com/developer/article/1824886)

[8] [美团技术博客之G1](https://tech.meituan.com/2016/09/23/g1.html)

[9] [https://www.jianshu.com/p/aef0f4765098——flycash](https://www.jianshu.com/p/aef0f4765098)
