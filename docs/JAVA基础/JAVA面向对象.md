# [返回](/)

![JAVA面向对象](imgs\JAVA面向对象.png)

# JAVA面向对象（一）

## “万事万物皆对象”

1. 在JAVA语言范畴中，都将结构、功能封装到类中，通过类的实例化，来调用具体的功能结构
2. 涉及JAVA语言与前端HTML，后端数据库交互时，前后端的结构在JAVA层面交互时，都体现为类、对象

## 三大特征、两大要素

**三大特征：**封装性、继承性、多态性、（抽象性）

**两大要素：**类、对象

## 内存解析

![image-20210420155828534](imgs\3-1.png)

> 对象中的属性（非static的）存放在堆中而不是栈中

![image-20210420163531660](imgs\3-2.png)

> 编译→字节码→JVM类加载器加载到内存→JVM解释器转译成机器码→运行
>
> - 虚拟机栈：局部变量
> - 堆：new出来的结构
> - 方法区：类的加载信息、运行时常量池、静态域
>
> *具体见JVM部分*

> **JAVA的“编译与解释共存”**
>
> 实际上，对于 .class->机器码 这一步，在这一步 JVM 类加载器首先加载字节码文件，然后通过解释器逐行解释执行，这种方式的执行速度会相对比较慢。而且，有些方法和代码块是经常需要被调用的(也就是所谓的热点代码)，所以后面引进了 JIT 编译器，而 JIT 属于运行时编译。当 JIT 编译器完成第一次编译后，其会将字节码对应的机器码保存下来，下次可以直接使用。而我们知道，机器码的运行效率肯定是高于 Java 解释器的。这也解释了我们为什么经常会说 Java 是编译与解释共存的语言

## 重载

同一类中允许同名的方法存在，**参数列表必须不同**，跟方法的权限修饰符、返回值类型等等都无关

## 可变个数形参

```java
// 与print(String[] strs)构成冲突，不构成重载
// 参数个数>=0
// 必须声明在末尾，且只能有一个
// 写sql时可能用到，比如where后面跟若干个条件
public static void print(String ... strs){
    for(String str: strs){
        System.out.println(str);
    }
}
// 构成重载时，先选择完全匹配（确定参数个数）的那个
// 如print("test")，会调用下面这个方法
public static void print(String str){
    System.out.println(str);
}
```

## 变量的赋值及值传递机制

- 如果变量给另一个变量赋值时（参数传递给形参时）是基本数据类型，则赋的值是变量真实的数据值（实参存储的真实数据值）
- 如果变量给另一个变量赋值时（参数传递给形参时）是引用数据类型，则赋的值是变量的地址值（实参的地址值）      *注意：地址值是被复制了一份的*

# JAVA面向对象（二）

## 封装性

### “高内聚，低耦合”

- 高内聚：类的内部数据操作细节自己完成，不允许外界干涉
- 低耦合：仅对外暴露少量方法用于使用

### 封装性的体现

- 类的属性私有化，同时提供公共的setter和getter
- 不对外暴露的私有方法
- 单例模式（将构造器私有化）
- ……

### 权限修饰符

> 封装性的体现，需要权限修饰符来配合

![image-20210421000443568](imgs\3-3.png)

**与基类不在同一个包中的子类，只能访问自身从基类继承而来的受保护成员，而不能访问基类实例本身的受保护成员**

比如：对于Object类的clone()方法

```java
public class C1{
	public static void main(String args){
		new C1().clone();  // 这是可以的
		new C2().clone();  // 这是不可以的
	}
}
class C2{}
```

## 继承性

- 子类获取父类的所有属性和方法

- 父类的私有属性/方法被子类继承，由于封装性的影响，不能够直接调用

  ```java
  public class Test {
      public static void main(String[] args) {
          System.out.println(new Son().getX());
      }
  }
  class Father{
      private int x = 10;
      public int getX(){
          System.out.println(this.getClass().getName());
          return this.x;
      }
  }
  class Son extends Father{
  }
  ```

  > Father中的x不能被son直接调用，但由于Son继承了Father的所有结构，在Son的堆空间中实际上是存在这个x和getX方法的，输出结果如下

  ![image-20210424032139850](imgs\3-15.png)

- 意义

  - 减少冗余，增加代码复用性
  - 便于功能扩展
  - 为多态性的实现提供前提

### this、super

- 理解为当前对象（父类对象）

- 如果一个类中有n个构造器，最多有n-1个构造器使用this(形参列表)来调用，且对象始终只有一个，因为this(形参列表)只是借用了这个方法的逻辑，没有新构造

- 至少有一个构造器直接或间接调用了父类的构造器（这也就是为什么最多有n-1个构造器使用this(形参列表)的原因），若父类没有无参构造器，当前类需要用super显式调用父类构造器

  ```java
  class Test{
      private int id;
      Test(int id){
          // 这里有一个隐式的super()，调用父类（这里是Object）的无参构造器
          // 如果父类没有无参构造器，则需要用super(形参)显式调用 
          (super();)
          this.id = id;
      }
      Test(){
          this(1);
      }
  }
  //-------------------------------------------------
  class Father{
      Father(int num){}
  }
  class Son extends Father{
      Son(){
          super(1); //没有这一句编译时会报错
      }
  }
  ```

- 每个构造器里只能有一个this(形参列表)或super(形参列表)，二选一，且必须在首行

### 重写

- 子类继承父类后，可以对父类同名同参数的方法进行重写（非static，要么都是非static，要么都是static，都是static时就不是重写了，*静态方法与类同时加载，不能被重写*）
- 子类重写方法的返回值类型不能大于父类的返回值类型，如果是基本数据类型必须一致
- 子类重写方法的权限修饰符不能小于父类的权限修饰符
- private修饰的方法可以被继承，但不能被重写
- 子类重写的方法抛出的异常不能大于父类的异常

### 子类对象实例化过程

- 子类继承父类以后，就获取了父类中声明的属性和方法，创建子类的对象，在堆空间中，就会加载所有父类中声明的属性

> **注意：只是调用了父类的构造器，并加载这些属性到堆空间，并没有new一个父类对象**
>
> ![image-20210421141823126](imgs\3-4.png)
>
> 对外只暴露Dog（黄色框）的地址，Animal等等结构的地址都不暴露，只是加载到了堆空间，且堆空间中只有Dog这一个对象

### 匿名子类

```java
public class Test {
    public static void main(String[] args) {
        // 匿名子类，同时也是Test类的匿名内部类，调用getClass()方法的结果是Test$1
        new Father(){
            public void print(){
                System.out.println("This is son");
            }
        }.print();
    }
}
class Father{
    public void fatherMethod(){}
}
```

### 继承链

**子类中没有某个属性或方法，首先去直接父类找，顺着继承链一个个往上（就近原则）**

## 多态性

### 定义

- 对象的多态性：父类的引用指向子类的对象
- 意义：实现代码的通用性，方法参数不止可以放指定对象，也可以放它的子类对象

> 典型例子：equals方法的各种重写；抽象类、接口的实现（一定有多态性）

### 多态的使用

#### 虚拟方法调用

> 此时父类的方法称为虚拟方法

- 编译期，只能调用父类中声明的方法

- 运行期，实际调用的是子类重写父类的方法**（动态绑定）**

**编译看左边，运行看右边**

>  多态是运行时行为，只有运行的时候才能知道具体调用了哪个对象中的方法

### 使用前提

- 类的继承
- 方法的重写（多态下子类除了重写的方法以外，特有的属性和方法是调用不了的，只有向下转型了之后才可以）

> 多态性不适用于属性（编译运行都看左边）

![image-20210421155023443](imgs\3-5.png)

### 例子

```java
public class Test {
    public static void main(String[] args) {
        Sub s = new Sub();  // Sub和Base的结构都被加载到堆空间，只不过此时Base的被屏蔽了
        System.out.println("s.count: " + s.count);  // 首先选择Sub中的count
        Base b = s;
        System.out.println(b == s);  // 地址值是一致的，值传递机制
        System.out.println("b的地址：" + b);
        System.out.println("s的地址：" + s);
        System.out.println("b.count：" + b.count);  // 编译运行都看左边
    }
}
class Base{
    int count = 10;
}
class Sub extends Base{
    int count = 20;
}
```

![image-20210421211055666](imgs\3-6.png)

> 简单的图示

![image-20210422153957467](imgs\3-7.png)

## 方法的重载和重写

> **重载**
>
> 对于编译器而言，重载的方法由于参数不同，是不同的方法，**调用地址在编译器就确定了**，所以，在方法调用之前就确定了要调用的方法，称为**“早绑定”**或**“静态绑定”**
>
> **重写**
>
> 只有等到方法调用那一刻，才知道调用的是哪个方法，是**运行期确定的**，称为**“晚绑定”**或**“动态绑定”**

# JAVA面向对象（三）

## ==和equals的区别

### ==

1. 如果比较的是基本数据类型，则比较两个变量保存的数据是否相等

   > 类型不一定要一致，比如double d = 10.0；d == 10结果为true，存在一个自动类型提升

2. 如果比较的是引用数据类型，则比较两个变量的地址值是否相等

   > 对于String str1 = "aa"，String str2 = "aa"，str1 == str2为true，原因是因为"aa"存放在了方法区的常量池中而不是堆中*（注意：JDK1.7后字符串常量池由方法区移到了堆中）*，所以自然地址相同

### equals

1. Object中是这样定义的，实现想要的功能，需要进行重写

   ```java
   public boolean equals(Object obj){
       return (this == obj);
   }
   ```

2. 重写的基本逻辑框架（仅供参考）

   ```java
   public boolean equals(Object obj){
       if (this == obj){
           return true;
       }
       if(obj instanceof MyClass){
           ......
       }
       return false;
   }
   ```

> 重写equals，要重写hashCode()
>
> [https://www.cnblogs.com/skywang12345/p/3324958.html——skywang12345](https://www.cnblogs.com/skywang12345/p/3324958.html)

## toString

1. 输出一个对象的引用时，实际上就是调用对象的toString

> 实际上，println是调用了String的valueof方法，valueof中再调用对象的toString
>
> ```java
> public static String valueOf(Object obj) {
>     return (obj == null) ? "null" : obj.toString();
> }
> ```

2. Object中：

```java
public String toString(){
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
    // hashCode():不同的类有不同的hashCode算法
    // 针对于Object来说，为虚拟的内存地址（逻辑地址），并不等同于真正的内存地址（物理地址）
}
```

## 包装类

### 基本类型、包装类间的转换

int i，Integer in，String str

#### JDK 5.0之前

- 基本类型→包装类：in = new Integer(i)
- 包装类→基本类型：i = in.intValue()

#### JDK 5.0新特性：自动装箱与拆箱

- 自动装箱：in = i
- 自动拆箱：i = in

> **探究一下自动装/拆箱的底层原理**
>
> ```java
> public class Test {
>     public static void main(String[] args) {
>         Integer in = 9;  // 自动装箱
>         int i = in;  // 自动拆箱
>     }
> }
> ```
>
> 在控制台通过javap命令反编译一下
>
> ![image-20210423213118912](imgs\3-12.png)
>
> ![image-20210423213221286](imgs\3-13.png)
>
> 可以看到，针对于Integer和int，自动装箱是用valueOf实现的，自动拆箱是用intValue实现的

### 基本类型、包装类和String的转换

- 基本数据类型、包装类→String：str = i + ""；str = String.valueof(i)
- String→基本数据类型、包装类：i = Integer.parseInt(str)      *可能出现NumberFormatException*

### 经典面试题

```java
Integer i = new Integer(1);
Integer j = new Integer(1);
System.out.println(i == j);  // false
/*
为什么是false？因为i和j指向堆中不同的地址
*/

Integer m = 1;
Integer n = 1;
System.out.println(m == n);  // true
/*
为什么是true？看下面的解析
*/

Integer x = 128;
Integer y = 128;
System.out.println(x == y);  // false
/*
为什么是false？看下面的解析
*/
```

解析：以Integer为例，查看源码

根据上面的探究知道：自动装箱是通过valueOf实现的

![image-20210423213456327](imgs\3-14.png)

可以看出如果处在IntegerCache的low和high之间，直接返回数组中的cache数组中的元素，否则新创建一个Integer对象，看一下IntegerCache

```java
/*
Integer包装类中，有IntegerCache这样一个内部类，随着Integer类的出现就加载到内存中
其中有一个Integer类型的数组cache存了low=-128到high=127的数（使用频率比较高），随着类的出现提前加载好
范围在[-128, 127]中的数就直接从静态区中的cache数组取出来，地址当然是一样的
而不在这个范围的，由于自动装箱机制，生成两个Integer类型的对象，所以地址不一致
*/
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer[] cache;
    static Integer[] archivedCache;

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                h = Math.max(parseInt(integerCacheHighPropValue), 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(h, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        // Load IntegerCache.archivedCache from archive, if possible
        VM.initializeFromArchive(IntegerCache.class);
        int size = (high - low) + 1;

        // Use the archived cache if it exists and is large enough
        if (archivedCache == null || size > archivedCache.length) {
            Integer[] c = new Integer[size];
            int j = low;
            for(int i = 0; i < c.length; i++) {
                c[i] = new Integer(j++);
            }
            archivedCache = c;
        }
        cache = archivedCache;
        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

注意代码中有这样一句

```java
String integerCacheHighPropValue =
            VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
```

> 这个high可以通过JVM参数来改变

又有这样一句

```java
int size = (high - low) + 1;
```

说明cache数组随着high的大小而改变，范围[-low, high]映射到数组的元素为[0, high+low]，size也就是high+low+1==high+129

所以代码中有这样一句，让high不能大于Integer.MAX_VALUE - 129

high如果等于Integer.MAX_VALUE - 129，数组大小为Integer.MAX_VALUE，会超过JVM的内存限制

```java
h = Math.min(h, Integer.MAX_VALUE - (-low) -1);
```

# JAVA面向对象（四）

## static关键字

> static可用来修饰属性、方法、代码块、内部类

### static的使用

- 类的多个对象共享
- 随着类的加载而加载，早于对象的创建
- 由于类只会在内存中加载一次，所以静态变量也只加载一次，在方法区的静态域中
- 静态方法只能调用静态的方法或属性，这是由于生命周期决定的
- 继承关系中，用static修饰的同名同参的方法不是重写

![image-20210422205217787](imgs\3-8.png)

- static型的变量也可以被子类继承，通过子类所调用

  ```java
  public class Test {
      public static void main(String[] args) {
          System.out.println(Son.x);  // 输出10
      }
  }
  class Father{
      static int x = 10;
  }
  class Son extends Father{
  }
  ```

- 通过对象调用的话，看左边，因为静态方法是随类加载的

  ```java
  public class Test {
      public static void main(String[] args) {
          Father s = new Son();
          s.method();  // 结果为“father”
      }
  }
  class Father{
      static void method(){
          System.out.println("father");
      }
  }
  class Son extends Father{
      static void method() {
          System.out.println("son");
      }
  }
  ```

## 单例设计模式（Singleton）

> **保证在整个系统中，某个类只能存在一个对象实例，并且只提供一个获得其对象实例的方法**

- 饿汉式实现

```java
class Singleton{
    // 1.私有化类的构造器
    private Singleton(){}
    // 2.内部创建类的对象
    private static Singleton instance = new Singleton();
    // 3.提供公共的方法，返回类的实例
    public static Singleton getInstance(){
        return instance;
    }
    // 4.对象和方法设为静态，与类的生命周期保持一致
}
```

- 懒汉式

```java
class Singleton{
    // 1.私有化类的构造器
    private Singleton(){}
    // 2.声明当前类的对象，没有初始化
    private static Singleton instance = null;
    // 3.提供公共的方法，返回类的实例
    public static Singleton getInstance(){
        if(instance == null){
            instance = new Singleton();
        }
        return instance;
    }
    // 4.对象和方法设为静态，与类的生命周期保持一致
}
```

- 饿汉式 vs 懒汉式

1. 懒汉式好处在于延迟对象创建，使用时才new，不用时不占用内存；饿汉式对象加载时间过长

2. 饿汉式是天然线程安全的，懒汉式不是（后面多线程部分时会改进  link：[单例线程安全的懒汉式](/JAVA进阶/JAVA多线程?id=单例模式懒汉式线程同步)）

   ```java
   // Singleton s1 = Singleton.getInstance();
   // Singleton s2 = Singleton.getInstance();
   // 在下述过程中
   if(instance == null){
       instance = new Singleton();
   }
   // s1判断instance为null，调用instance = new Singleton()之前，s2进行了判断，同样也是null
   // 这样就生成了两个对象
   ```

- 单例模式的优点：只有一个实例，减少了系统开销，当一个对象的产生需要比较多的资源，如读取配置、产生其他依赖对象时，这个对象可以采用单例模式，比如下面这个例子

![image-20210422235348959](imgs\3-9.png)

- 应用场景

  1. 网站的计数器

  2. 应用程序的日志应用

     > 一般共享的日志文件是一直打开的，因为只有一个实例能去操作，否则内容不好追加

  3. 数据库连接池

     > 一个连接池，若干个连接

  4. 读取配置文件的类

  5.  ......

## 代码块

> - 代码块/初始化块：用于初始化类或对象
> - 只能用static修饰
> - 继承时代码块同样被继承

- static

  1. 内部可以有输出语句
  2. 随着类的加载而**执行**
  3. 作用：初始化一些类的信息，如静态变量等

  > 数据库的连接池中较经常使用，用于一些参数的设置

- 非static

  1. 内部可以有输出语句
  2. 随着对象的创建而**执行**
  3. 作用：可以用于创建对象时属性的初始化，早于构造器

## 初始化顺序

### 对象的初始化顺序

​	**”由父及子，静态先行“**

### 类的属性初始化顺序

​	**默认初始化→显式初始化、在代码块中赋值（看位置的先后顺序）→构造器初始化→对象.方法**

## final关键字

- 可用来修饰类、对象、属性、变量、方法

- 用来修饰类，类无法被继承

  > 类中所有方法隐式final，因为是无法被重写的

- 用来修饰属性或变量，无法被修改

- 用来修饰方法，方法无法被重写

  > private方法隐式final，无法被重写

- 用来修饰对象，实质上是修饰对象的引用，该引用不能指向其他对象，实质上是地址值被固定了，但对象的内容可以变

### String的不可变问题

[String不可变](/JAVA进阶/JAVA常用类?id=String)

### final重排序规则

[https://blog.csdn.net/weixin_39762464/article/details/111129971?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control——weixin_39762464](https://blog.csdn.net/weixin_39762464/article/details/111129971?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control)

```java
class FinalDemo {
    private int a;  // 普通域
    private final int b;  // final域
    private static FinalDemo finalDemo;
    public FinalDemo(){
        a = 1;  // 1.写普通域
        b = 2;  // 2.写final域
    }
    public static void writer(){
        finalDemo = new FinalDemo();
    }
    public static void reader(){
        FinalDemo demo = finalDemo;  // 3.读对象引用
        int a = demo.a;  // 4.读普通域
        int b = demo.b;  // 5.读final域
    }
}
```

- 基本数据类型: 
  - final域写：禁止final域写与构造方法重排序，即禁止final域写重排序到构造方法之外，从而保证该对象对所有线程可见时，该对象的final域全部已经初始化过
  - final域读：禁止初次读对象的引用与读该对象包含的final域的重排序
- 引用数据类型： 
  - 额外增加约束：禁止在构造函数对一个final修饰的对象的成员域的写入与随后将这个被构造的对象的引用赋值给引用变量的重排序

## 抽象性

> abstract关键字可以修饰类、方法

### 抽象类

> 随着继承层次中一个个子类的定义，类变得越来越具体，有时候会将父类设计得非常抽象，以至于没有具体的实例，这样的类叫做抽象类

1. 抽象类不能实例化
2. 抽象类中一定有构造器，用于子类对象实例化

### 抽象方法

1. 抽象方法没有方法体

2. 包含抽象方法的类一定是抽象类；抽象类中可以没有抽象方法

3. 抽象方法必须被重写

4. 如果抽象类的子类没有重写全部的抽象方法，那么它也是抽象类

   ```java
   abstract class Father{
       abstract void method();
   }
   /*
   class Son extends Father{  // 编译时报错
   }
   */
   abstract class Son extends Father{
   }
   ```

5. 不能被private、static、final等修饰，抽象方法是必须被重写的，而private、final方法无法被重写，static修饰的同名同参方法不是重写

### 抽象类的匿名子类

```java
public class Test {
    public static void main(String[] args) {
        Son son = new Son();
        son.method();  // 非匿名子类的非匿名对象
        
        new Son().method();  // 非匿名子类的匿名对象
        
        // 匿名子类
        AbstractFather father = new AbstractFather(){  
            // 注意这里用到了多态
            @Override
            public void method() {
                System.out.println("here is a son too");
            }
        };
        father.method();
        // or
        new AbstractFather(){ 
            @Override
            public void method() {
                System.out.println("here is a son three");
            }
        }.method();
    }
}
abstract class AbstractFather{
    abstract public void method();
}
class Son extends AbstractFather{
    @Override
    public void method() {
        System.out.println("here is a son");
    }
}
```

执行结果

![image-20210423142729013](imgs\3-10.png)

### 模板方法设计模式（TemplateMethod）

- 解决的问题：当功能内部一部分实现是确定的，一部分实现是不确定的。这时可以把不确定的部分暴露出去，让子类去实现
- 抽象类体现的就是一种模板模式的设计，抽象类作为多个子类的通用模板，子类在抽象类的基础上进行扩展、改造，但子类总体上会保留抽象类的行为方式

例子

```java
public class Test {
    public static void main(String[] args) {
        TemplateTest templateTest = new Code1();
        templateTest.spendTime();
        templateTest = new Code2();
        templateTest.spendTime();
    }
}
abstract class TemplateTest{
    // 模板方法，子类一般不能重写
    public void spendTime(){
        Long start = System.currentTimeMillis();
        code();  // 像个钩子，具体执行时，挂哪个子类，就调哪个子类的方法，叫做钩子方法
        Long end = System.currentTimeMillis();
        System.out.println("time: " + (end - start));
    }
    abstract void code();
}
class Code1 extends TemplateTest{
    @Override
    void code() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("code1 processing...");
    }
}
class Code2 extends TemplateTest{
    @Override
    void code() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("code2 processing...");
    }
}
```

> 计算时间直接运行抽象父类的spendTime模板而不需要考虑code的具体实现，code的具体实现在子类中自己定义

![image-20210423154850198](imgs\3-11.png)

> 应用场景：有些父类的方法体不好确定，比如说计算几何图形的面积，每种图形面积公式不一样，这时候可以把父类”图形类“的面积计算方法抽像化等等

## 接口

### 基本特点

> 1. 有时候需要从几个类中派生出一个子类，继承他们所有的属性和方法，但JAVA不支持多继承，有了接口便可以实现多继承的效果
>
>    ```java
>    class Me implements Person, Man, Student, Son......
>    ```
>
> 2. 有些时候类之间并没有体现继承中is-a的关系，但又具有一些共同的特征，这时候便可以使用接口（has-a）
>
>    ```java
>    class TV implements USB{}
>    class PC implements USB{}
>    ......
>    ```

- 接口不能定义构造器（抽象类一定有构造器）
- 实现类必须覆盖所有的抽象方法，否则需声明为抽象类
- 接口之间也可以用extends来继承

> 接口实际上是一种规范，要求xxx接口，才能xxx

### 代理设计模式（Proxy）

> 为其他对象提供一种代理以控制对这个对象的访问

例子（简单的静态代理，专门针对某一种接口，如这里是专门针对NetWork这个接口，另一种就需要写另一套静态代理）

> 后面JDK的动态代理需要反射的知识

```java
public class Test {
    public static void main(String[] args) {
        // 被代理的对象生成之后，不用自己调用browse方法了，只要把对象扔给代理就可以了
        Server server = new Server();
        // 代理负责对方法的调用，同时还提供对方法的增强
        new ProxyServer(server).browse();
    }
}
interface NetWork{
    void browse();
}
// 被代理类
class Server implements NetWork{
    @Override
    public void browse() {
        System.out.println("真实的服务器访问网络");
    }
}
class ProxyServer implements NetWork{
    private NetWork work;
    public ProxyServer(NetWork work){
        this.work = work;
    }
    public void check(){
        System.out.println("联网之前的检查工作");
    }
    @Override
    public void browse() {
        // 方法增强
        check();
        // 调用被代理对象的方法
        work.browse();
    }
}
```

**应用场景**

- 安全代理：屏蔽对真实角色的直接访问
- 远程代理：通过代理类处理远程方法调用
- 延迟加载：先加载轻量级的代理对象，需要时才加载真正对象
- ……

### 不同JDK版本的接口定义

**JDK7及以前**

1. 接口中的成员
   - 全局常量：public static final修饰的（可以省略，默认已经加上了）
   - 抽象方法：public abstract修饰的（可以省略，默认已经加上了）

**JDK8**

1. 接口中的成员

   - 全局常量：public static final（可以省略，默认已经加上了）

   - 抽象方法：public abstract（可以省略，默认已经加上了）

   - 静态方法

     - 只能通过接口自己来调用，实现类无法调用

   - 默认方法

     - 可以通过实现类的对象来调用

     - 可以被重写

       > **类优先原则：**如果子类（实现类）继承的父类和继承的接口中声明了同名同参数的方法，那么子类在没有重写的情况下，调用父类中的方法
       >
       > ```java
       > public class Test {
       >     public static void main(String[] args) {
       >         new C().method();  // 输出结果是class
       >     }
       > }
       > interface A{
       >     default void method(){
       >         System.out.println("interface");
       >     }
       > }
       > class B{
       >     public void method(){
       >         System.out.println("class");
       >     }
       > }
       > class C extends B implements A{
       > }
       > ```

     - 多个接口中定义了同名同参数的默认方法，且子类中没有重写，则会报接口冲突错误

   > 注意：
   >
   > ![image-20210426161145667](imgs\3-16.png)
   >
   > 这时候会报错，为什么？*(勘误：这里的“C”全部改成“B”)*
   >
   > 我们来查看IDEA类图
   >
   > ![image-20210426163207628](D:\笔记\秋招\docs\JAVA基础\imgs\3-17.png)
   >
   > 可以发现，**默认方法的权限是public**（*打开的锁表示public*）
   >
   > 而D中继承了B的方法，也就是说D的堆空间中存在了一个权限为default的方法，而我们都知道方法重写时权限是不能缩小的，也就是说接口A中public的默认方法在它的子类中的实现也应当是public，个人认为，这就与继承的B的方法的default权限冲突了，因此有两种修改方法：①B中改成public，这时按照类优先原则，调用B中的方法；②D中显式地重写，且权限修饰符需要是public
   >
   > **此外还有一个值得注意的点**
   >
   > ```java
   > public class Test {
   >     public static void main(String[] args) {
   >         new D().method();
   >     }
   > }
   > interface A{
   >     default void method(){
   >         System.out.println("interfaceA");
   >     }
   > }
   > class B{
   >     void method(){
   >         System.out.println("class C");
   >     }
   > }
   > class D extends B implements A{
   >     public void method(){
   >         super.method();  // 调用B的方法
   >         A.super.method();  // 调用A的方法
   >     }
   > }
   > ```

**JDK9：添加私有方法**

## 抽象类与接口

- 同
  - 都不能被实例化
  - 都可以包含抽象方法
  - JDK8之后都可以包含静态方法，JDK9之后都可以包含私有方法
  - 子类要想实例化的话，必须实现所有的抽象方法
- 异
  - 接口可以实现多继承，抽象类不可以
  - 抽象类有构造器，接口没有
  - 抽象类可以有非静态的类内属性，接口只能有全局常量
  - 抽象类的静态方法可以被子类调用，接口的静态方法只能自己调用

## 继承性案例

1. ```java
   public class Test {
       public static void main(String[] args) {
           new C().printX();
       }
   }
   interface A{
       int x = 0;
   }
   class B{
       int x = 1;
   }
   class C extends B implements A{
       public void printX(){
           // 这里会编译报错，因为接口和类同级，不确定是哪一个x
           // System.out.println(x); 
           
           // 应当改成如下
           System.out.println(super.x);
           System.out.println(A.x);
       }
   }
   ```

2. ```java
   interface Playable{
       void play();
   }
   interface Bounceable{
       void play();
   }
   interface Rollable extends Playable, Bounceable{
       Ball ball = new Ball("Basketball");
   }
   class Ball implements Rollable{
       private String name;
       public Ball(String name) {
           this.name = name;
       }
       @Override
       public void play() {
           ball = new Ball("Football");  // ①
       }
   }
   // 排错？
   // 解答：我们知道ball应当是用public static final修饰的，因此不允许令他指向新的地址
   // 所以①处编译时报错
   ```

## 内部类

**内部类、匿名内部类可以访问外部类的对象的域，是因为内部类构造的时候，编译器会把外部类的对象this隐式地作为一个参数传递给内部类的构造方法**

### 实例化

```java
public class Test {
    public static void main(String[] args) {
        A.B b = new A.B();
        A.C c = new A().new C();
    }
}
class A{
    static class B{}  // 静态成员内部类
    class C{}  // 非静态成员内部类
}
```

### 成员内部类中区分调用外部类的结构

```java
public class Test {
    public static void main(String[] args) {
        new A().new B().display();
    }
}
class A {
    String name = "class A";
    class B {
        String name = "class B";
        public void display() {
            System.out.println(this.name);  // 输出结果为class B
            System.out.println(A.this.name);  // 输出结果为class A
        }
    }
}
```

### 开发中的局部内部类

```java
public class Test {
    public static void main(String[] args) {
        
    }
}
class A {
    public Comparable getComparable(){
        // 局部内部类
        class MyComparable implements Comparable{
            @Override
            public int compareTo(Object o) {
                return 0;
            }
        }
        return new MyComparable();
    }
	// 可用下面的匿名内部类的方式代替
    public Comparable getComparable2(){
        return new Comparable() {
            @Override
            public int compareTo(Object o) {
                return 0;
            }
        };
    }
}
```

局部内部类的方法中，如果调用该方法内的局部变量，局部变量必须是final的

> 为什么必须是final？
>
> 生命周期不同，内部类不会随方法结束而销毁，而局部变量会销毁，而内部类中又有对该局部变量的使用，这时就会产生错误
>
> 使用final的话，会将局部变量复制一份放在内部类，作为成员变量

```java
class A {
    public static void method(){
        int num = 10;  // 这里是final，JDK8后可以不写，默认就有
        class B{
            public void method(){
                System.out.println(num);
            }
        }
    }
}
```

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

[2] [https://www.cnblogs.com/skywang12345/p/3324958.html——skywang12345](https://www.cnblogs.com/skywang12345/p/3324958.html)
