# [返回](/)

# JAVA泛型与File

## JAVA泛型

**为什么使用泛型而不直接使用Object**

- 往容器添加时：类型不安全
- 从容器取出时：强转时发生ClassCastException

### 类型擦除

泛型是一种编译期时的安全检测策略，运行时泛型信息会被擦除，擦除到父类，运用了反射

> 比如T extends A，运行时会擦除到A，若是只有T，就擦除到Object
>
> - List\<String>、List\<T> 擦除后的类型为 List。
> - List\<String>[]、List\<T>[] 擦除后的类型为 List[]。
> - List<? extends E>、List<? super E> 擦除后的类型为 List\<E>。
> - List<T extends Serialzable & Cloneable> 擦除后类型为 List\<Serializable>

### 泛型方法

```java
class A<T>{
    //  不是泛型方法
    T method1(){return null;}
    //  不是泛型方法
    void method2(T t){}
    //  是泛型方法
    <E> E method3(E e){return null;}
}
```

泛型方法特点：

1. 用\<E>作为泛型方法的标识

   > 不作标识的话，编译器会把E当作是类似于String、Object类似的确定类型，所以需要用\<E>告诉编译器：这是个泛型方法

2. 泛型方法与类是不是泛型没有关系

### 通配符

？与 T、E、K、V 等的不同在于，这是一个限定好的范围，而其他的是一个确定的类型

#### 为什么要使用通配符

考虑这样一个场景，一个苹果类，一个水果类

```java
class Fruit{}
class Apple extends Fruit{}
```

有一个用来装东西的盘子类

```java
class Plate<T>{
    private T thing;
    public Plate(T t){
        this.thing = t;
    }
    public T take(){
        return thing;
    }
    public void put(T t){
        this.thing = t;
    }
}
```

现在我们定义一个”水果盘子“，并用它来装一个苹果

```java
Plate<Fruit> plate = new Plate<Apple>(new Apple());
//  编译器报错，java: 不兼容的类型: Plate<Apple>无法转换为Plate<Fruit>
```

编译器会报错，由此可见T是确定的类型，且容器A\<Son>和A\<Father>之间是没有继承关系的，两个容器是并列的

> 为什么设计成这样？举一个简单的例子，对于
>
> ```java
> List<Object> objectList = new ArrayList<>();
> List<String> stringList = new ArrayList<>();
> ```
>
> 如果是继承关系的话，则可以objectList = stringList，令objectList指向stringList，并给string的list里面添加object，这样类型安全性就被破坏了
>
> **因此，泛型是无法协变的，是抗变的**

> 协变、逆变、抗变
>
> - 协变(Covariance)：`List<Son>` 是`List<Father>`的子类型
> - 逆变(Contravariance): `List<Father>` 是`List<Son>`的子类型
> - 抗变(Invariant): `List<Father>` 与`List<Son>`没有任何继承关系

因此，设计出通配符，就可以指定一个范围的类型，而不是某个确定的类型

```java
Plate<? extends Fruit> plate = new Plate<Apple>(new Apple());
```

#### PECS思想

- 对于上界通配符<? extends A>

```java
Plate<? extends Fruit> plate = new Plate<Apple>(new Apple());
```

存在一个问题，如果要往盘子中放，只知道是一种水果，但不知道具体是哪一种水果，如果指明了泛型是Plate\<Apple>，往里存入Fruit或者Banana等，显然是不合理的，因此put方法会报错，就算存入Apple也会报错

```java
plate.put(new Apple());
//  java: 不兼容的类型: Apple无法转换为capture#1, 共 ? extends Fruit
```

get的话就可以，用一个基类Fruit来承载，利用多态

```java
Fruit fruit = plate.take();
//  不报错
```

- 对于下界通配符<? super A>

如果要往盘子中放一种水果，由于它的基类是Fruit的超类，根据多态，放入Fruit以及它的子类，一定是没问题的，所以put不会报错

```java
Plate<? super Fruit> plate = new Plate<Food>(new Food());
plate.put(new Fruit());
plate.put(new Apple());
```

取出来时，由于不知道它具体是什么类，可能是Fruit也可能不是，只能用Object承载，类型信息都被破坏了

```java
Plate<? super Fruit> plate = new Plate<Food>(new Food());
Fruit fruit = plate.take();  //  报错
Object obj = plate.take();  //  不报错，但类型信息被破坏
```

**由此，引出PECS思想，即”Producer-Extends Consumer-Super“**

- 容器作为Producer，不断从中取数据时，采用<? extends A>
- 容器作为Consumer，不断向其中插数据时，采用<? super A>

## File

### 路径分隔符

- Windows、DOS：\
- UNIX、URL：/

由于java的跨平台特性，路径分隔符不一致会产生问题，于是File类提供了一个separator常量

```java
String path;
new File(path = ("D:" + File.separator + "xxx"));
System.out.println(path);
//  Windows系统下是D:\xxx
```

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

[2] [https://www.zhihu.com/question/20400700/answer/117464182](https://www.zhihu.com/question/20400700/answer/117464182)

[3] [https://zhuanlan.zhihu.com/p/268523581——shusheng007](https://zhuanlan.zhihu.com/p/268523581)
