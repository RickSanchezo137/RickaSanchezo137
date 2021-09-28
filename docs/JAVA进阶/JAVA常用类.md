# [返回](/)

# JAVA常用类

## String

- 字面量的定义方式

  ```java
  String str = "hello";  // 字面量
  ```

### 字符串源码

![image-20210508162520259](imgs\JAVA常用类\7.png)

可以看到，字符串底层使用一个byte数组存储数据，并用coder标识所存数据的编码，0是LATIN1，1是UTF16

![image-20210508162450711](imgs\JAVA常用类\8.png)

> jdk9之前是char数组，需要两个字节，改成byte后，可以根据实际需求来存储，并不一定非要两个字节，这样就节省了内存空间

### 字符串常量池

```java
String str1 = "ABC";
String str2 = "ABC";
System.out.println(str1 == str2);  // true
// 都指向字符串常量池中的相同地址
```

> 字符串常量池中相同内容的字符串的引用只会存在一个

```java
String str1 = "ABC";
str1 = "hello";
str1 += " world";
str2 = str1.replace("h", "H");
```

> 对字符串重新赋值或连接或替换操作时，不能使用原有的value进行赋值，而是需要重新指定内存区域赋值
>
> 这些操作都重新生成了字符串（不可变性）

### String的初始化

**前置知识：字符串常量池中存在一个StringTable，是一个HashTable，key是根据String实例的值和长度计算生成的hash，value存放了对String实例的引用，注意，是引用，而不是实例，StringTable中只有引用**

> 抛出问题：String str = "hello"和String str = new String("hello")有什么区别？

- String str = "hello"，以首次初始化为例

  1. 在堆中创建一个值为“hello”的byte[]数组和String对象，并令String对象的成员变量value指向byte[]数组

  2. 发现字符串常量池中没有指向该String对象的引用，所以在字符串常量池中驻留一个指向该String对象的引用。由于线程运行期间这个引用一直在字符串常量池驻留，所以该String对象在线程死亡前不会被gc回收

     > 注意：出现字面量了，它的引用就会被放到常量池中，具体过程涉及JVM部分

  3. 将堆中String对象的引用复制给str变量

     > 看图，此时str == value2，都指向0x1234

  4. 如果再次通过字面量初始化，jvm计算key发现StringTable中已经存在对于某个String对象的引用了，直接把这个引用复制给初始化变量

  ![1](imgs\JAVA常用类\1.png)

- String str = new String("hello")

我们来看一看String构造器的源码

![image-20210507104859764](imgs\JAVA常用类\2.png)

可以看到，是直接将字符串常量池中的引用所指向的String对象的value，复制给了新构造的String对象的value

因此，以首次构造为例

1. 先进行“hello”字面量的初始化，在堆中创建一个String对象1，并令它的value指向byte[]数组
2. 发现字符串常量池中没有指向该String对象的引用，所以在字符串常量池中驻留一个指向该String对象的引用
3. new一个String对象2，并把String对象1的value赋值给String对象2的value
4. 把String对象2的引用赋值给str

![image-20210507110048604](imgs\JAVA常用类\3.png)

> 案例：
>
> ```java
> String str1 = "hello";
> String str2 = new String("hello");
> System.out.println(str1 == str2);  // false
> // 按上图，str1是value2==0x1234，str2是0x9999，当然不一样
> 
> str2 = str2.intern(); 
> // intern可以把str2放入字符串常量池并返回引用，如果常量池中有，直接返回已有的引用（这里就是str1）
> 
> System.out.println(str1 == str2);  // true
> // intern后，由于str1的初始化，常量池中已经有对“hello”的引用，所以str1和str2都是常量池中的引用了
> ```

### String的不可变性

很多人只答String的存值是用private final byte[] value，这样是不够的，因为只是引用型变量value指向的数组首地址被固定，不允许它指向其他地址，数组内的内容可以变呀。所以应当加上下述：

1. 首先，byte数组是private的，并且 String类没有对外提供修改这个数组的方法，所以它初始化之后外界没有有效的手段去改变它
2. 其次，String类被final修饰的，也就是不可继承，避免被他人继承后破坏
3. 最重要的是因为Java作者在String的所有方法里面，都很小心地避免去修改了byte数组中的数据，涉及到对byte数组中数据进行修改的操作全部都会重新创建一个String对象

> **为什么String设计成不可变的**
>
> - 从内存角度，对于某个常量池中已有的字符串，多个引用变量指向它时，如果是可变的，改变一个就会导致其他引用值的错误，这是很危险的
> - String不可变保证了hashcode的相同，因此对于某些集合比如HashSet、HashMap等就可以直接比较hashcode而不是调用equals方法
> - 方便其他类的使用，如Set等
> - 不可变保证了线程安全，令String可以在多个线程间自由共享

**所以不可变的关键在于SUN公司的工程师，在所有String的方法里很小心的没有去动value数组里的元素， 没有暴露内部成员字段**

### String拼接

#### 案例一

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

- s4：JVM对于这种拼接方式，会自动编译成”s1s2“，反编译一下可以证明

  ![image-20210508153434567](imgs\JAVA常用类\4.png)

- s5：JDK9之前是这样实现的

  ![image-20210508154642340](imgs\JAVA常用类\6.png)

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
  ```

  JDK9之后，源码不再将拼接过程编译成字节码，而是通过动态编译，调用的是StringConcatFactory的makeConcatWithConstants方法

  ![image-20210508154339899](imgs\JAVA常用类\5.png)

  有六种拼接策略，其中前五种也是通过StringBuffer实现的

  参考[https://zhuanlan.zhihu.com/p/259254321——君慕贤](https://zhuanlan.zhihu.com/p/259254321)

- s6：基本同s5，也在堆中产生一个新的String对象

这样的话，结果就不言而喻了，s3、s4都等同于常量池中的引用，而s5、s6各自在堆中new了一个新的String

> 此时堆里有三个”s1s2“的对像，一个是s5指向的，一个是s6指向的，一个是常量池中的引用/s3/s4指向的

**注意**

> 上题中，将s1改为**final**，对于String s5 = s1 + "s2"，JVM会优化成等同于"s1"+"s2"，那么s3==s5就为true了

> 用+拼接会不停new StringBuffer，所以拼接时建议自己new一个StringBuffer

#### 案例二

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

   a. 这里可以不管输出语句后面的那句，考虑s2.intern()，s2.intern()返回的是字符串常量池中指向“s1s2”的对象，发现没有，于是将堆中指向"s1s2"的引用，也就是s2，放一份在字符串常量池中。也就是说，此时有两个引用指向堆中的"s1s2"，一个是s2，一个是字符串常量池中的引用，它们的值是相等的

   b. 所以 s2 == s2.intern()

## StringBuffer

#### 初始化

**源码跟踪**

![image-20210510222551020](imgs\JAVA常用类\9.png)

可以看到，这三个构造方法，都调用了父类中的方法，点进去查看

![image-20210510222757841](imgs\JAVA常用类\10.png)

进入了AbstractStringBuilder类，其中value就是指向存放具体数据的byte数组，和String不同的是，这个value并不是final的，说明可以指向其他新的数组（扩容时会指向新的数组）；不是private的，说明暴露给其他类，可以其他类中改变value中的值（字符替换等等）

- 对于StringBuffer的无参构造方法

![image-20210510223111081](imgs\JAVA常用类\11.png)

StringBuffer中的无参构造方法调用了这个方法，生成一个长度为16个**字符**的byte数据；带capacity参数的则依capacity参数而定。coder用来标识编码，如果是UTF16，在newBytesFor方法里，返回了一个长度乘2的数组，因为每个UTF16字符占两个字节

![image-20210510223439952](imgs\JAVA常用类\12.png)

- 对于带str参数的构造方法

![image-20210510223756728](imgs\JAVA常用类\13.png)

创建一个容量为str+16的byte数组，注意，这时还是空数组，需要调用append方法把str放进去

![image-20210510224056722](imgs\JAVA常用类\14.png)

这里返回的是this，也就是说，可以这样调用，sb.append().append()...，构成**方法链**

1. **ensureCapacityInternal：扩容策略**

![image-20210510224154330](imgs\JAVA常用类\15.png)

>  LATIN1时，coder = 0；UTF16时，coder等于1，也就是说coder为UTF16时，字符长度为数组长度 / 2

oldCapacity为调用append之前的**字符容量**，minimumCapacity==旧的字符长度+append的str的长度，也就是将要生成的新字符串的长度，如果大于容量了，则调用Arrays工具类的copyOf方法新生成一个扩容的数组，并令value指向它，新生成的数组容量为newCapacity(minimumCapacity) ，**newCapacity(minimumCapacity)是多少？**

![image-20210510225038589](imgs\JAVA常用类\16.png)

新的容量为oldCapacity*2再+2；如果这样操作之后容量还是小于上面提到的将要生成的新字符串的长度，则直接令新容量等于将要生成的新字符串的长度；如果新的容量>SAFE_BOUND，则调用hugeCapacity，直接令新容量==将要生成的新字符串的长度，或者抛出OOM异常（将要生成的新字符串的长度>Integer.MAX_VALUE）

1. putStringAt

![image-20210510230223250](imgs\JAVA常用类\17.png)

inflate是处理str和原字符串编码不一致的情况，暂且不表

调用了str的getBytes方法

![image-20210510230517721](imgs\JAVA常用类\18.png)

调用arraycopy方法拼接字符串数组

> 以sb.append(str)为例，抽象意义上来说，这里的value为sb.append(str)中的str，dst为StringBuffer里面的value数组，也就是这里的sb中的value数组，desBegin为sb的末尾，意思就是，将str复制到sb的末尾
>
> *这块儿两个value可能混淆，要仔细分辨一下*

#### 为什么是可变的

#### String、StringBuffer和StringBuilder的区别

- String：不可变字符序列
- StringBuffer：线程安全的可变字符序列，内部使用同步方法，效率不如StringBuilder
- StringBuilder：线程不安全的可变字符序列

## 常用时间API

### JDK8之前

#### SimpleDateFormat

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
System.out.println(sdf.format(new Date()));  // 格式化Date为字符串
System.out.println(sdf.parse("2020-05-20 13:14:00"));  // 解析字符串为Date
```

#### Calendar

```java
Calendar calendar = Calendar.getInstance();
// 或者
calendar = new GregorianCalendar();
// 
calendar.setTime(new Date());  // 根据Date设置calendar
calendar.getTime();  // 获取Date

System.out.println(calendar.get(Calendar.DAY_OF_WEEK));  
// 注意，周日显示是1，周一是2，...，周六是7

System.out.println(calendar.get(Calendar.MONTH));
// 注意，从0开始，一月显示是0

calendar.add(Calendar.MINUTE, 1);  // 日期运算
```

### JDK8之后

> Calendar不太好用，因此有了新的
>
> 比如：Calendar会有一个偏移量，容易弄乱
>
> 也无法保证线程安全
>
> ```java
> SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
> Calendar calendar = Calendar.getInstance();
> calendar = new GregorianCalendar();
> calendar.set(Calendar.YEAR, 2020);
> calendar.set(Calendar.MONTH, 5);
> calendar.set(Calendar.DAY_OF_WEEK, 20);
> System.out.println(sdf.format(calendar.getTime()));  // 结果是2020-06-19
> ```

#### LocalDate、LocalTime、LocalDateTime

- 初始化

  ```java
  LocalDate date = LocalDate.now();  // 2021-05-11
  LocalTime time = LocalTime.now();  // 01:03:01.135762600
  LocalDateTime dateTime = LocalDateTime.now();  // 2021-05-11T01:03:01.135762600
  ```

  

  ```java
  LocalDateTime dateTime = LocalDateTime.of(2020, 05, 20, 13, 14);  // 2020-05-20T13:14
  ```

- 获取时间

  ```java
  LocalDateTime dateTime = LocalDateTime.of(2020, 05, 20, 13, 14);
  System.out.println(dateTime.getDayOfWeek());  // WEDNESDAY
  System.out.println(dateTime.getDayOfWeek().getValue());  // 3
  System.out.println(dateTime.getDayOfMonth());  // 20
  System.out.println(dateTime.getDayOfYear());  // 141
  System.out.println(dateTime.getMonth());  // May
  System.out.println(dateTime.getMonthValue());  // 5
  ```

- 设置时间

  ```java
  LocalDateTime dateTime = LocalDateTime.of(2020, 05, 20, 13, 14);
  System.out.println(dateTime);  // 2020-05-20T13:14
  System.out.println(dateTime.withDayOfMonth(21));  // 2020-05-21T13:14
  // 与Calendar不同，这里是直接返回一个新对象，体现了不可变性
  ```

- 时间运算

  ```java
  LocalDateTime dateTime = LocalDateTime.of(2020, 05, 20, 13, 14);
  System.out.println(dateTime);  // 2020-05-20T13:14
  System.out.println(dateTime.plusDays(1));  // 2020-05-21T13:14
  ```

#### Instant：瞬时（时间戳）

> 起始于1970年1月1日0时0分0秒，纳秒级

- 初始化

  ```java
  Instant instant = Instant.now();  // UTC时间
  System.out.println(instant);  // 2021-05-10T17:25:13.423739400Z
  // 注意，我们是东八区，所以这里的值是我们的北京时间减8小时，是伦敦的本初子午线时间
  ```

  可以按实际情况添加偏移量

  ```java
  Instant instant = Instant.now();
  System.out.println(instant.atOffset(ZoneOffset.ofHours(8)));
  // 2021-05-11T01:32:21.631253700+08:00
  ```

  起始于1970年1月1日0时0分0秒的毫秒数及转换

  ```java
  Instant instant = Instant.now();
  long millis = instant.toEpochMilli();
  Instant instant1 = Instant.ofEpochMilli(millis);
  ```

#### DateTimeFormatter

- 转换

  ```java
  // 自带的枚举
  LocalDateTime localDateTime = LocalDateTime.now();
  DateTimeFormatter formatter = DateTimeFormatter.ISO_LOCAL_DATE_TIME;
  System.out.println(formatter.format(localDateTime));
  // 2021-05-11T01:48:11.8582478
  
  //----------------------------------------------------------------------------
  // ofLocalizedDateTime
  formatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT);
  System.out.println(formatter.format(localDateTime));
  // 2021/5/11 上午1:48
  
  formatter = DateTimeFormatter
      .ofLocalizedDateTime(FormatStyle.LONG)
      .withZone(ZoneOffset.ofHours(8));
  System.out.println(formatter.format(localDateTime));
  // 2021年5月11日 +08:00 上午1:49:47
  
  //----------------------------------------------------------------------------
  // 自定义
  formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd hh:mm:ss");
  System.out.println(formatter.format(localDateTime));
  // 2021-05-11 01:55:19
  
  // 解析
  TemporalAccessor accessor = formatter.parse("2021-05-11 01:59:59");
  System.out.println(accessor);
  /* {HourOfAmPm=1, MinuteOfHour=59, SecondOfMinute=59, NanoOfSecond=0, MilliOfSecond=0, MicroOfSecond=0},ISO resolved to 2021-05-11
  */
  ```

## 比较器

### Comparable

> 重写compareTo方法，按自己的规则排序

场景：按学生年龄从小到大排序

```java
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.util.Arrays;

public class Test {
    public static void main(String[] args) {
        Student[] students = new Student[]{
                new Student(19),
                new Student(18),
                new Student(20)
        };
        System.out.println(Arrays.toString(students));  // [19, 18, 20]
        Arrays.sort(students);
        System.out.println(Arrays.toString(students));  // [18, 19, 20]
    }
}
class Student implements Comparable{
    int age;
    public Student(int age) {
        this.age = age;
    }
    @Override
    public int compareTo(Object o) {
        if(o instanceof Student){
            if(this.age > ((Student) o).age) return 1;
            else if(this.age < ((Student) o).age) return -1;
            else return 0;
        }
        throw new RuntimeException();
    }
    @Override
    public String toString() {
        return "" + age;
    }
}
```

### Comparator

> 如果想要临时按学生年龄从大到小排序，又不想改变已经定义好的compareTo方法，就可使用临时的比较器

```java
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.util.Arrays;
import java.util.Comparator;

public class Test {
    public static void main(String[] args) {
        Student[] students = new Student[]{
                new Student(19),
                new Student(18),
                new Student(20)
        };
        System.out.println(Arrays.toString(students));  // [19, 18, 20]
        Arrays.sort(students, new Comparator<Student>() {
            @Override
            public int compare(Student o1, Student o2) {
                return -o1.compareTo(o2);
            }
        });
        System.out.println(Arrays.toString(students));  // [20, 19, 18]
    }
}
class Student implements Comparable{
    int age;
    public Student(int age) {
        this.age = age;
    }
    @Override
    public int compareTo(Object o) {
        if(o instanceof Student){
            if(this.age > ((Student) o).age) return 1;
            else if(this.age < ((Student) o).age) return -1;
            else return 0;
        }
        throw new RuntimeException();
    }
    @Override
    public String toString() {
        return "" + age;
    }
}
```



## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

[2] [https://blog.csdn.net/qq_39404258/article/details/112271882——干了这杯柠檬多](https://blog.csdn.net/qq_39404258/article/details/112271882)

[3] [https://www.zhihu.com/question/29884421](https://www.zhihu.com/question/29884421)

[4] [https://segmentfault.com/a/1190000017952075?utm_source=tag-newest](https://segmentfault.com/a/1190000017952075?utm_source=tag-newest)

