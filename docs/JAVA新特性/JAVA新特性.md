

# [返回](/)

# JAVA新特性

> LTS：长期支持的版本，有8、11、15等

## JAVA 8 新特性(LTS)

### *Lambda表达式

> Lambda表达式是一个匿名函数，可以看作让代码像数据一样传递
>
> 针对接口或抽象类，且只有一个抽象方法，对其进行匿名实现

**本质：接口方法的实现**

#### 基本使用

—>：lambda操作符 

- 左边：lambda形参列表（接口中抽象方法的形参列表）
- 右边：lambda体（重写的抽象方法的方法体）

1. 无参，无返回值

```java
public class Test {
    public static void main(String[] args) {
        MyInterface myInterface = () -> System.out.println("method");
        myInterface.method();
    }
}
interface MyInterface{
    void method();
}
```

2. 有参，无返回值

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<String> myInterface = (String s) -> System.out.println(s);
        myInterface.method("hahaha");
    }
}
interface MyInterface<T>{
    void method(T t);
}
```

3. 数据类型可忽略，因为可由编译器推断得出，称为”类型推断“

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<String> myInterface = (s) -> System.out.println(s);
        myInterface.method("hahaha");
    }
}
interface MyInterface<T>{
    void method(T t);
}
```

4. 形参只有一个参数，可以省略小括号

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<String> myInterface = s -> System.out.println(s);
        myInterface.method("hahaha");
    }
}
interface MyInterface<T>{
    void method(T t);
}
```

5. 多个参数，多条语句，有返回值

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<Integer> myInterface = (i1, i2) -> {
            int i = i1.hashCode() + i2.hashCode();
            return i;
        };
        System.out.println(myInterface.method(1, 2));
    }
}
interface MyInterface<T>{
    T method(T t1, T t2);
}
```

6. 只有一条语句，return和大括号都可以省

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<Integer> myInterface = (i1, i2) -> i1.hashCode() + i2.hashCode();
        System.out.println(myInterface.method(1, 2));
    }
}
interface MyInterface<T>{
    T method(T t1, T t2);
}
```

### *函数式（Functional）接口

一个接口中只声明了一个抽象方法，就叫做函数式接口

和lambda配合

```java
@FunctionalInterface
interface MyInterface<T>{
    T method(T t1, T t2);
}
```

类似于Override注解的作用，校验是否是函数式接口

> 以后这种接口就可以不用写匿名实现类了，也不会生成相应字节码

![image-20210623172509690](imgs\1.png)

举例：

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<String> myInterface = (s, con) -> {
            con.accept(s);
            return s;
        };
        System.out.println(myInterface.method("test", s2 -> System.out.println("test consumer")));
    }
}
@FunctionalInterface
interface MyInterface<T>{
    T method(T t1, Consumer<T> consumer);
}
```

```java
public class Test {
    public static void main(String[] args) {
        MyInterface<String> myInterface = (s, func) -> func.apply(s);
        System.out.println(myInterface.method("test", s2 -> s2.toUpperCase()));
    }
}
@FunctionalInterface
interface MyInterface<T>{
    T method(T t1, Function<T, T> function);
}
```

### *方法引用与构造器引用

#### 方法引用

本质上也是lambda表达式，但lambda表达式要自己写方法体，而方法引用可以直接引用已有实现的方法，要求：接口的方法和引用的方法具有相同的形参列表和返回类型

- 对象 : : 非静态方法

```java
public class Test {
    public static void main(String[] args) {
        /**
         * consumer的accept和System.out.println方法体结构非常类似
         * void accept(T t);
         * void println(Object x)
         * accept便可以引用println方法
         */
        Consumer<String> con = System.out::println;
        con.accept("test");
        /**
         * function的apply方法和compareTo方法体结构非常类似
         * int compareTo(T o);
         * R apply(T t);
         * apply便可以引用compareTo方法
         */
        Function<Integer, Integer> func = Integer.valueOf(1)::compareTo;
        System.out.println(func.apply(2));
    }
}
```

- 类 : : 静态方法

```java
public class Test {
    public static void main(String[] args) {
        /**
         * function的apply方法和Math.round类似
         * long round(double a)
         * R apply(T t);
         * apply便可以引用round方法
         */
        Function<Double, Long> func = Math::round;
        System.out.println(func.apply(3.1415926));
    }
}
```

- 类 : : 非静态方法

```java
/**
 * Comparator的int compare(T o1, T o2)
 * 和String的int s1.compareTo(String s2)
 */
Comparator<String> comparator = String::compareTo;
```

#### 构造器引用

```java
public class Test {
    public static void main(String[] args) {
        /**
         * 空参构造器和T get();
         */
        Supplier supplier = String::new;
        System.out.println(supplier.get().getClass());  //  class Java.lang.S
    }
}
```

```java
public class Test {
    public static void main(String[] args) {
        /**
         * 有参构造器和R apply(T t);;
         */
        Function<Integer, ArrayList<String>> function = ArrayList::new;
        //  初始化capacity为15的list
        ArrayList<String> list = function.apply(15);
    }
}
```

- 数组引用

```java
public class Test {
    public static void main(String[] args) {
        Function<Integer, String[]> function = String[]::new;
        //  初始化capacity为15的list
        String[] s = function.apply(15);
    }
}
```

### *Stream API

1. Stream关注的是对数据的运算，与CPU打交道；集合关注对数据的存储，与内存打交道
2. Stream不存储元素，不改变源对象，操作是延迟执行的，需要结果时才执行

#### 执行流程

1. Stream的实例化
2. 一系列的中间操作（过滤、映射等）
3. 终止操作

**实例化**

1. 通过集合

```java
List list = new ArrayList();
Stream stream = list.stream();  //  返回一个顺序流
stream = list.parallelStream();  //  返回一个并行流
```

2. 通过数组

```java
IntStream stream1 = Arrays.stream(new int[0]);
Stream<Integer> stream2 = Arrays.stream(new Integer[0]);
```

3. 通过Stream的of

```java
Stream<Integer> stream = Stream.of(1, 2, 3);
```

4. 创建无限流

- 迭代

```java
public static<T> Stream<T> iterate(final T seed, final UnaryOperator<T> f)
```

UnaryOperator\<T>继承Function<T, T>

```java
Stream.iterate(0, t -> t + 2).limit(10).forEach(System.out::println);
```

- 生成

```java
public static<T> Stream<T> generate(Supplier<? extends T> s)
```

```java
Stream.generate(Math::random).limit(10).forEach(System.out::println);
```

> forEach相当于终止，终止的流不能再做操作了，否则会抛出异常
>
> ![image-20210623220627436](imgs\2.png)

**中间操作**

1. 筛选与切片

```java
List<Integer> list = new ArrayList<>(50){{
    add(65);
    add(66);
    add(66);
    add(67);
}};
Stream<Integer> stream = list.stream();
stream.filter(i -> i >= 66).forEach(System.out::println);
```

- filter：过滤数据

```java
stream.filter(i -> i >= 66).forEach(System.out::println);
```

- limit：截断流

```java
stream.limit(2).forEach(System.out::println);
```

- skip：跳过

```java
stream.skip(1).forEach(System.out::println);
```

- distinct：筛选，通过hashCode()、equals()去除重复元素

```java
stream.distinct().forEach(System.out::println);
```

2. 映射

- map(Function f)：将stream中的元素经f映射成其他形式

```java
stream.map(i -> i + 2).forEach(System.out::println);
```

```java
stream.map(i -> i.hashCode()).forEach(System.out::println);
```
- flatMap(Function f)：将流中每一个值都转换成一个流，然后合并到之前的流

  > 相当于list的addAll方法

```java
public class Test {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(50){{
            add(65);
            add(66);
            add(66);
            add(67);
        }};
        Stream<Integer> stream = list.stream();
        stream.flatMap(Test::int2Stream).forEach(System.out::println);

    }
    static Stream<Integer> int2Stream(Integer i){
        return Arrays.asList((int)(i / 10), i % 10).stream();
    }
}
```

3. 排序

- sorted()：产生一个新流，按自然顺序排序
- sorted()：产生一个新流，按比较器顺序排序

```java
stream.sorted(Comparator.reverseOrder()).forEach(System.out::println);
```

**终止操作**

1. 匹配与查找

- 匹配

```java
System.out.println(stream.allMatch(i -> i > 66));  //  是否都大于66
```

```java
System.out.println(stream.anyMatch(i -> i > 66));  //  是否有一个大于66
```

```java
System.out.println(stream.noneMatch(i -> i > 66));  //  是否都不大于66
```

- 查找

> 查找并返回Optional对象

```java
System.out.println(stream.findFirst());  //  找第一个
```

```java
Stream<Integer> stream = list.parallelStream();  //  顺序流的话总是返回第一个，换成并行流
System.out.println(stream.findAny());  //  找任一个
```

还有count()，max/min(Comparator c)等等

2. 归约

```java
System.out.println(stream.reduce((i1, i2) -> i1 + i2));
```

```java
System.out.println(stream.reduce(Integer::sum));
```

```java
System.out.println(stream.reduce(0, Integer::sum));  //  返回Integer对象
//  上面两个返回Optional对象，这个返回的是Integer对象，why？
//  因为这里有一个identity初始值了，所以不会是null
//  结果为元素里的值+初始值
```

> 练习场景：工资超过1000的员工的工资之和

```java
public class Test {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(){{
            add(new Employee("小一", 999.7));
            add(new Employee("小二", 1000.5));
            add(new Employee("小三", 999.5));
            add(new Employee("小四", 1202.0));
            add(new Employee("小五", 1000.6));
        }};
        Stream<Employee> stream = employees.parallelStream();
        Double sum = stream.map(Employee::getSalary).filter(v -> v > 1000).reduce(0.0, Double::sum);
        System.out.println(sum);
    }
}
class Employee{
    private String name;
    private Double salary;

    public Employee(String name, Double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Double getSalary() {
        return salary;
    }

    public void setSalary(Double salary) {
        this.salary = salary;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "name='" + name + '\'' +
                ", salary=" + salary +
                '}';
    }
}
```

3. 收集

> 将流转换为其他形式。接收一个Collector接口的实现，给流中的元素做汇总
>
> 收集到list、map、set等等

```java
Integer[] array = new Integer[]{1, 2, 3};
//  这种返回的并不是真正的ArrayList，而是Arrays的内部类
List<Integer> list1 = Arrays.asList(array);  
//  这种返回的是真正的ArrayList
List<Integer> list2 = Arrays.stream(array).collect(Collectors.toList());

//  对基本数据类型装箱
int[] array = new int[]{1, 2, 3};
Arrays.stream(array).boxed().collect(Collectors.toList());
```

> 练习场景：查找工资大于1000的员工并返回一个list

```java
public class Test {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(){{
            add(new Employee("小一", 999.7));
            add(new Employee("小二", 1000.5));
            add(new Employee("小三", 999.5));
            add(new Employee("小四", 1202.0));
            add(new Employee("小五", 1000.6));
        }};
        Stream<Employee> stream = employees.parallelStream();
        List<Employee> list = stream.filter(v -> v.getSalary() > 1000).collect(Collectors.toList());
        System.out.println(list);
    }
}
class Employee{
    private String name;
    private Double salary;

    public Employee(String name, Double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Double getSalary() {
        return salary;
    }

    public void setSalary(Double salary) {
        this.salary = salary;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "name='" + name + '\'' +
                ", salary=" + salary +
                '}';
    }
}
```

### Optional类

Optional\<T>可以保存T类型的值，表示它存在；或者保存为null，表示为空；避免出现空指针异常

- 创建

```java
public class Test {
    public static void main(String[] args) {
        B b = new B();
        B nullB = null;
        System.out.println(Optional.of(b));  //  必须非空
        Optional.ofNullable(nullB);  //  可以空
    }
}
class A{
    private B b;

    public B getB() {
        return b;
    }

    public void setB(B b) {
        this.b = b;
    }

    @Override
    public String toString() {
        return "A{" +
                "b=" + b +
                '}';
    }
}
class B{}
```

- 基本操作

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(getBS(null));
        System.out.println(getBS(new A()));
        System.out.println(getBS(new A(new B("notNull"))));
    }
    public static String getBS(A a){
        Optional<A> aOptional = Optional.ofNullable(a);
        A a1 = aOptional.orElse(new A(new B("A is null")));  //  如果Optional不为空则返回，为空则返回orElse中替代值
        Optional<B> bOptional = Optional.ofNullable(a1.getB());
        B b = bOptional.orElse(new B("B is null"));
        return b.getS();
    }
}
class A{
    private B b;

    public A() {
    }

    public A(B b) {
        this.b = b;
    }

    public B getB() {
        return b;
    }

    public void setB(B b) {
        this.b = b;
    }

    @Override
    public String toString() {
        return "A{" +
                "b=" + b +
                '}';
    }
}
class B{
    private String s;

    public B(String s) {
        this.s = s;
    }

    public String getS() {
        return s;
    }

    public void setS(String s) {
        this.s = s;
    }
}
```

isPresent：判断非空

get：返回t

### 接口

接口可以用静态方法和默认方法

## JAVA 9 新特性

### *模块化系统

![image-20210624210639576](imgs\3.png)

对于这样两个模块module1和module2

类Module1：![image-20210624210745095](imgs\4.png)

类Module2：![image-20210624210812979](imgs\5.png)

可以看到，是没法引用另一个模块的内容的

这时就可以在module2中引入module1，首先在module1中创建module-info

![image-20210624211014518](imgs\6.png)

```java
module module1 {
    exports me.test.module1;  //  把module1包暴露出去
}
```

再在module2中创建module-info

```java
module module2 {
    requires module1;  //  引入module1包
}
```

![image-20210624211251143](D:\笔记\秋招\docs\JAVA新特性\imgs\7.png)

module2中就可以正常使用module1的内容了

### *jshell

直接执行java语句

![image-20210624211957052](imgs\8.png)

### 接口

增添私有方法

```java
interface A{
    /**
     * java 8 特性
     * 下面三个方法的权限修饰符都是public
     */
    void method();
    static void staticMethod(){
        System.out.println("static");
    }
    default void defaultMethod(){
        System.out.println("default");
        privateMethod();
    }
    /**
     * java 9 特性
     */
    private void privateMethod(){
        System.out.println("private");
    }
}
class AImpl implements A{
    @Override
    public void method() {
        System.out.println("normal");
    }

    public static void main(String[] args) {
        //  接口中的静态方法只能由自己调用，不能由实现类调用
        A.staticMethod();
        new AImpl().defaultMethod();
    }
}
```

### try-with-resource

java 8中提供了try-with-resource，但必须在try中初始化资源

java 9中可以在外部初始化，try中的资源变成了final的

```java
InputStream in = new FileInputStream("");
OutputStream out = new FileOutputStream("");
try(in; out){
    in.read();
}catch (IOException e){
    e.printStackTrace();
}
```

### String底层存储

`char[]——> byte[]`

### 快速创建只读集合

```java
List.of(1, 2, 3);
```

### transferTo

直接将输入流的数据复制到输出流

```java
InputStream in = new FileInputStream("1.txt");
OutputStream out = new FileOutputStream("2.txt");
try(in; out){
    in.transferTo(out);
}catch (IOException e){
    e.printStackTrace();
}
```

### Stream新增方法

- takeWhile：根据规则从头取，直到遇到不满足条件的
- dropwhile：根据规则从头抛，直到遇到不满足的

```java
Stream<Integer> stream = List.of(1, 2, 3, 4, 5, 6).stream();
stream.takeWhile(i -> i <= 4).forEach(System.out::println);  //  1 2 3 4
stream = List.of(1, 2, 3, 4, 5, 6).stream();
stream.dropWhile(i -> i <= 4).forEach(System.out::println);  //  5 6
```

> 对排好序的集合，比filter要快

- Stream.of

java 8 中不可以只创建null的流`Stream.of(null);`

java 9 中新增ofNullable`Stream.ofNullable(null);`

- iterator添加判断逻辑

```java
Stream.iterate(0, i -> i < 10, i -> i + 2).forEach(System.out::println);
```

- Optional的Stream

```java
List<String> list = List.of("str1", "str2", "str3");
Optional<List<String>> optional = Optional.of(list);
//  1个list元素，[str1, str2, str3]
optional.stream().forEach(System.out::println);
//  将list元素转换成流，再遍历，有3个String元素
optional.stream().flatMap(l -> l.stream()).forEach(System.out::println);
```

### Nashorn

JVM运行js

## JAVA 10 新特性

### *局部变量类型推断

根据值推断类型

```java
var set = new LinkedHashSet<Integer>();
var i = 10;
```

> var不是关键字，相当于类型的占位符，编译器会对其进行推断；反编译的时候可以看到还原成了推断出的原类型

### 集合拷贝

```java
List.copyOf(new ArrayList<>());  //  如果参数是只读集合则直接返回；如果不是，则返回只读集合
```

## JAVA 11 新特性(LTS)

### *Epsilon

### *ZGC

### String新增方法

![image-20210625004559630](imgs\9.png)

### Optional新增方法

![image-20210625005959582](imgs\10.png)

### 局部变量类型推断增强

可以利用var在lambda表达式时给参数加注解

```java
Consumer<String> consumer = (@Deprecated var s) -> System.out.println(s.toUpperCase());
```

### 全新的HTTP客户端API

用HttpClient替代blocking模式的HttpURLConnection

### 简便的编译运行程序

直接用java命令，而不是javac之后再java

> 只执行第一个类，且类中必须包含main方法
>
> 不可以使用其他源文件中的自定义类

### 废弃Nahorn

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

