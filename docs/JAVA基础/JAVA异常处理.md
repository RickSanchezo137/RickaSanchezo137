# [返回](/)

# JAVA异常处理

## 基本概念

- Error：JVM无法解决的严重错误，需要改代码
  - StackOverflowError：栈溢出
  - OOM（OutOfMemoryError）：堆溢出
- Exception：可写代码来进行处理

## 异常分类

![异常](imgs\异常.png)

> checked：需要显式的处理或抛出

## 处理方式

1. try-catch-finally：一般开发中只在编译时异常处来处理

   - finally中的代码一定会被执行，就算在try/catch中有return或其他异常，finally中的语句也会在return或其他异常抛出之前执行

   - 对类似于数据库连接、输入输出流、Socket连接等物理连接，垃圾回收机制无能为力，常在finally中手动关闭

     > 垃圾回收机制只回收JVM堆内存里的对象空间

2. throws

   - 子类重写方法时抛出的异常类型不能大于父类被重写的方法

     > 为什么？
     >
     > ```java
     > public class Test {
     >     public static void main(String[] args) {
     >         Father n = new Son();
     >         try{
     >             n.method();
     >         }catch (IOException e){
     >             e.printStackTrace();
     >         }
     >     }
     > }
     > class Son extends Father{
     >     @Override
     >     public void method() throws Exception {
     >     }
     > }
     > class Father{
     >     public void method() throws IOException{}
     > }
     > ```
     >
     > 在java多态机制中，对象的引用n在编译时期是属于父类类型，但是在运行时属于子类类型。也就是说在编译的时候，编译器发现catch完全能将父类方法中抛出的IOException异常捕获，因此编译通过，但是在运行时期，由于对象变成了子类类型，子类重写的方法抛出的异常是Exception，显然IOException不能捕获这个比它更大的异常，因此在运行时期也就出现失败

   - 如果父类没有抛异常，子类也不能抛异常

   - 某个方法a中调用了其他方法，建议开发中其他方法用throws抛出，a中用try-catch统一处理

## 自定义异常

**模板**

```java
public MyException extends XXException{
    static final long serialVersionUID = -xxxxxxxxxxl;
    public MyException(){}
    public MyException(String msg){
        super(msg);
    }
}
```

## throw和throws

- throws属于异常处理的方式，声明在方法的声明处
- throw表示抛出一个异常，是生成一个异常对象，声明在方法体内

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)