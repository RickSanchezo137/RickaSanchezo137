# [返回](/)

# JAVA枚举类

**枚举类的使用**

- 类的对象只有确定的有限个
- 需要定义一组常量时
- 如果枚举类中只有一个对象，可以作为单例模式的实现方式

## 自定义枚举类

```java
public class Test {
    public static void main(String[] args) {
        Season season = Season.SPRING;
        System.out.println(season);
    }
}
class Season{
    // 1. 创建不可变的私有属性
    private final String seasonName;
    // 2. 私有化构造器
    private Season(String seasonName){
        this.seasonName = seasonName;
    }
    // 3. 需要的若干个枚举对象
    static final Season SPRING = new Season("spring");
    static final Season SUMMER = new Season("summer");
    static final Season AUTUMN = new Season("autumn");
    static final Season WINTER = new Season("winter");

    // 4. 提供一个外部访问属性的方法
    public String getSeasonName() {
        return seasonName;
    }
    @Override
    public String toString() {
        return seasonName;
    }
}
```

## enum关键字

```java
public class Test {
    public static void main(String[] args) {
        Season season = Season.SPRING;
        System.out.println(season);  // 这里会打印“SPRING”
    }
}
enum Season{
    // 1. 一开始就要写上需要的若干个枚举对象，强制省略掉类型声明、权限修饰符及new语句
    SPRING("spring"),
    SUMMER("summer"),
    AUTUMN("autumn"),
    WINTER("winter");
    // 2. 创建不可变的私有属性
    private final String seasonName;
    // 3. 私有化构造器
    private Season(String seasonName){
        this.seasonName = seasonName;
    }
    // 4. 提供一个外部访问属性的方法
    public String getSeasonName() {
        return seasonName;
    }
}
```

注意，这里的Season为java.lang.Enum的子类

- toString

看父类Enum中：

![image-20210511203247815](imgs\JAVA枚举类与注解\1.png)

会返回声明的枚举对象变量名，当然这是可以被重写的

- values和valueOf

values返回枚举的对象数组，用于遍历；valueOf根据字符串名称返回对应的枚举类对象

```java
public class Test {
    public static void main(String[] args) {
        Season[] seasons = Season.values();
        for(Season season: seasons){
            System.out.println(season);
            /* SPRING SUMMER AUTUMN WINTER */
        }
        Season season = Season.valueOf("SPRING");
        System.out.println(season);  // SPRING

    }
}
enum Season{
    // 1. 一开始就要写上需要的若干个枚举对象，强制省略掉类型声明、权限修饰符及new语句
    SPRING("spring"),
    SUMMER("summer"),
    AUTUMN("autumn"),
    WINTER("winter");
    // 2. 创建不可变的私有属性
    private final String seasonName;
    // 3. 私有化构造器
    private Season(String seasonName){
        this.seasonName = seasonName;
    }
    // 4. 提供一个外部访问属性的方法
    public String getSeasonName() {
        return seasonName;
    }
}
```

- 实现接口

```java
public class Test {
    public static void main(String[] args) {
        Show show = Season.SPRING;  // 这时，SPRING是Season的子类对象
        show.show();
    }
}
interface Show{
    void show();
}
enum Season implements Show{
    // 1. 一开始就要写上需要的若干个枚举对象，强制省略掉类型声明、权限修饰符及new语句
    SPRING("spring"){
        @Override
        public void show() {
            System.out.println("春天");
        }
    },
    SUMMER("summer"){
        @Override
        public void show() {
            System.out.println("夏天");
        }
    },
    AUTUMN("autumn"){
        @Override
        public void show() {
            System.out.println("秋天");
        }
    },
    WINTER("winter"){
        @Override
        public void show() {
            System.out.println("冬天");
        }
    };
    // 2. 创建不可变的私有属性
    private final String seasonName;
    // 3. 私有化构造器
    private Season(String seasonName){
        this.seasonName = seasonName;
    }
    // 4. 提供一个外部访问属性的方法
    public String getSeasonName() {
        return seasonName;
    }

    @Override
    public void show() {
    }
}
```

以SPRING为例，写成上述格式时，便成为了Season的匿名子类，Season继承了Show接口

SPRING.getClass().getSuperclass()==class Season

# 注解

## 三个基本注解

- @Override：编译期检查是否是重写
- @Deprecated：不建议使用的过时方案
- @SuppressWarnings：抑制编译器警告

## 自定义注解

```java
@MyAnnotation(value = "hello")  // 对于没有默认值的成员变量，需要赋值
public class Test {
    public static void main(String[] args) {
    }
}
// 用@interface声明注解
@interface MyAnnotation{
    String value();  // 以无参方法的格式声明成员变量
    String name() default "annotation";  // 成员变量默认值
}
```

> 注解具体功能需要通过后面提到的反射来实现

## 元注解

> 概念：修饰注解的注解

### Retention

> 注解的生命周期

![image-20210512004913824](imgs\JAVA枚举类与注解\2.png)

成员变量value的类型为RetentionPolicy

![image-20210512004957785](imgs\JAVA枚举类与注解\3.png)

- SOURCE：不被编译
- CLASS：编译但不包含在运行过程中（不指明的话，默认是这个）
- RUNTIME：运行过程中，加载进内存，可通过反射来实现功能

### Target

> 指定注解可以声明的位置

以SuppressWarnings为例

![image-20210512005136706](imgs\JAVA枚举类与注解\4.png)

有构造器、类、变量......等等

### Documented

> 注解可以在javadoc解析时保留下来

### Inherited

> 赋予注解继承性，子类自动继承该注解

举例

```java
import java.lang.annotation.Annotation;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

public class Test {
    public static void main(String[] args) {
        Class clazz = Son.class;
        for (Annotation annotation : clazz.getAnnotations()) {
            System.out.println(annotation);  // 显示@MyAnnotation(value="test")
        }
    }
}
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface MyAnnotation{
    String value() default "test";
}

@MyAnnotation
class Father{}

class Son extends Father{}
```

可以看到，子类并没有用注解修饰，然而用反射来看一下结果，子类上有MyAnnotation注解

## JDK8新特性

### 可重复注解

- 1.8之前写法

```java
import java.lang.annotation.*;

@MyAnnotations(values = {@MyAnnotation("1"), @MyAnnotation("2")})
public class Test {
    public static void main(String[] args) {
    }
}

@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface MyAnnotation{
    String value();
}

@interface MyAnnotations{
    MyAnnotation[] values();
}
```

- 1.8之后写法

```java
import java.lang.annotation.*;

// 3. MyAnnotations里面各种元注解改为与MyAnnotation一致
@MyAnnotation("1")
@MyAnnotation("2")
public class Test {
    public static void main(String[] args) {
    }
}

@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(MyAnnotations.class)  // 添加Repeatable
@interface MyAnnotation{
    String value();
}

@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface MyAnnotations{
    MyAnnotation[] value();  // 2. 成员变量名称改成一致
}
```

### 类型注解

1. TYPE_PARAMETER

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;

public class Test<@MyAnnotation T> {
    public static void main(String[] args) {
    }
}

@Target(ElementType.TYPE_PARAMETER)
@interface MyAnnotation{
}
```

2. TYPE_USE

```JAVA
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;
import java.util.ArrayList;

public class Test<@MyAnnotation T> {
    public static void main(String[] args) throws @MyAnnotation2 RuntimeException{
        ArrayList<@MyAnnotation2 String> list;
        int i = (@MyAnnotation2 int)8l;
    }
}

@Target(ElementType.TYPE_PARAMETER)
@interface MyAnnotation{
}

@Target(ElementType.TYPE_USE)
@interface MyAnnotation2{
}
```



## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

