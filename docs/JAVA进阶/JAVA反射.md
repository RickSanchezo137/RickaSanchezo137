# [返回](/)

# JAVA反射与动态代理

## JAVA反射

### 基本使用

```java
public class Test {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, NoSuchFieldException {
        Class<Person> clz = Person.class;

        //  Constructor
        Constructor<Person> constructor = clz.getDeclaredConstructor(new Class[]{int.class, String.class});
        constructor.setAccessible(true);
        Person p = constructor.newInstance(20, "张三");
        System.out.println(p);

        //  Method
        Method method = clz.getDeclaredMethod("method");
        method.setAccessible(true);
        method.invoke(p);

        //  Field
        Field field = clz.getDeclaredField("age");
        field.setAccessible(true);
        System.out.println(field.get(p));
        field = clz.getDeclaredField("name");
        field.setAccessible(true);
        System.out.println(field.get(p));
    }
}
class Person{
    private int age;
    private String name;

    private Person(int age, String name) {
        this.age = age;
        this.name = name;
    }

    private void method(){
        System.out.println("age: " +age + "  name: " + name);
    }

    @Override
    public String toString() {
        return "Person{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }
}
```

> 使用场景：编译期时无法确定要new哪一个对象，可以通过反射来动态生成对象，体现动态性；实现程序的解耦

> getXXX：自己及父类的public的数据
>
> getDeclaredXXX：自己的所有数据，不包括父类

### 获取Class对象的方式

#### 会进行Class对象初始化

```java
Class clz = new Person(20, "张三").getClass();
```

```java
Class clz = Class.forName("Person");
//  默认的initialize参数为true，设为false的话，也不会初始化
```

#### 不会进行Class对象初始化

```java
Class<Person> clz = Person.class;
```

```java
Class clz = Test.class.getClassLoader().loadClass("Person");
```

### 类加载

#### 对Class对象的理解

java编译解释并运行的过程是：javac.exe编译生成字节码→java.exe解释运行字节码，相当于将字节码加载到内存（方法区），这个过程就叫做类的加载。初始化时会在堆中生成一个Class对象，相当于Class的实例，也叫做运行时类

#### 类加载的过程

> 三大块五小步

- 加载
  1. 加载
- 链接
  2. 校验
  3. 准备
  4. 解析
- 初始化
  5. 初始化

**加载**

主要是获取class文件，加载阶段并不会检查class的语法和格式；由类加载器完成

**链接**

将类的二进制数据合并到JRE中

1. 校验

校验是链接的第一个阶段，确保class文件里的字节流信息符合当前虚拟机的要求，不会危害虚拟机的安全。校验过程检查 classfile 的语义，判断常量池中的符号，并执行类型检查

2. 准备

准备阶段将会创建静态字段，并将其初始化为标准默认值(比如null或者0值)，并分配方法表，即在方法区中分配这些变量所使用的内存空间。请注意，准备阶段并未执行任何Java代码

3. 解析

虚拟机常量池内的符号引用（常量名）替换为直接引用（地址）

**初始化**

执行类构造器\<clinit>()方法的过程，进行类变量的赋值，以及静态代码块的执行

> JVM会保证正确的加锁和同步

```java
class Test{
    static{
        m = 10;
    }
    int m = 20;
}
//  链接阶段，m = 0
//  初始化阶段，执行变量赋值和静态代码块
//  按顺序，相当于先执行m = 10，再执行m = 20
```

#### 类加载器

- 引导类加载器（Bootstrap ClassLoader）：加载核心类库，在代码层面无法直接获取到启动类加载器的引用，所以不允许直接操作它

- 扩展类加载器（Extension ClassLoader）：加载JRE的扩展目录，加载jre/lib/ext/或者由java.ext.dirs系统属性指定的目录中的JAR包的类

- 系统类加载器（System ClassLoader）：装入java -classpath或-Djava.class.path目录下的类，最常用

  > classpath指的是src.main.java和src.main.resources以及第三方jar包的根路径。我们自己编写的代码都位于classpath中

彼此之间是继承关系

```java
System.out.println(String.class.getClassLoader());
System.out.println(Person.class.getClassLoader().getParent());
System.out.println(Person.class.getClassLoader());
```

结果：

```java
null
jdk.internal.loader.ClassLoaders$PlatformClassLoader@7cc355be
jdk.internal.loader.ClassLoaders$AppClassLoader@3fee733d
```

> 对于PlatformClassLoader，在JDK1.8以及以前的版本里面提供的加载器为ExtClassLoader。因为在JDK的安装目录里面提供一个ext的目录，开发者可以将*.jar文件拷贝到此目录里面，这样就可以直接执行了，但是这样的处理并不安全，最初的时候也是不提倡使用的，所以从JDK9开始将其彻底废除，同时为了和系统加载器与应用类加载器之间保持设计的平衡，提供有PlatformClassLoader

- 加载配置文件

```java
public class Test {
    public static void main(String[] args) throws IOException {
        ClassLoader classLoader = Test.class.getClassLoader();
        InputStream is = classLoader.getResourceAsStream("test.properties");
        InputStreamReader reader = new InputStreamReader(is);
        Properties properties = new Properties();
        properties.load(reader);
        for(Map.Entry entry: properties.entrySet()){
            System.out.println(entry);
        }
    }
}
```

#### 双亲委派机制

- 当类加载器需要加载一个类，会先委托自己的父加载器去加载，父加载器如果发现自己还有父加载器，会一直继续往上委托，直至委托到引导/启动类加载器。启动类加载器尝试去加载这个类，如果启动类加载器能加载这个类，就会告诉它的子加载器，然后子类加载继续告诉它的子类加载器
- 如果启动类加载器无法加载，那么启动类加载的子加载器就会尝试去加载，如果启动类加载器的子加载器也无法进行加载，则启动类加载器的子加载器的子加载器则尝试去加载。如果所有类加载器都没有加载到指定名称的类，那么会抛出ClassNotFountException异常

### 反射调用方法的效率问题

反射调用方法的步骤：

1. 加载类并获取类对象的引用
2. 通过类对象的引用调用getMethod或getDeclaredMethod
3. 调用invoke方法来调用原方法

想要了解为什么说反射效率低，就需要深入源码去了解它的原理

#### getMethod/getDeclaredMethod

- getMethod源码跟踪

```java
public Method getMethod(String name, Class<?>... parameterTypes)
    throws NoSuchMethodException, SecurityException {
    Objects.requireNonNull(name);
    SecurityManager sm = System.getSecurityManager();
    //  1. 检查方法权限，注意，在getMethod中，是检查是否为PUBLIC
    if (sm != null) {
        checkMemberAccess(sm, Member.PUBLIC, Reflection.getCallerClass(), true);
    }
    //  2. 调用getMethod0获取方法
    Method method = getMethod0(name, parameterTypes);
    if (method == null) {
        throw new NoSuchMethodException(methodToString(name, parameterTypes));
    }
    //  3. 返回方法的拷贝
    return getReflectionFactory().copyMethod(method);
}
```

也就是说，获取方法分为下面三个主要步骤

1. 检查方法权限
2. 调用getMethod0获取方法
3. 返回方法的拷贝

**1. 检查方法权限**

其中有用到Member.PUBLIC，点进来看

![image-20210623092858069](imgs\JAVA反射\1.png)

包含所有的public成员，以及继承的public成员

**2. 调用getMethod0获取方法**

```java
private Method getMethod0(String name, Class<?>[] parameterTypes) {
    PublicMethods.MethodList res = getMethodsRecursive(
        name,
        parameterTypes == null ? EMPTY_CLASS_ARRAY : parameterTypes,
        /* includeStatic */ true);
    return res == null ? null : res.getMostSpecific();
}
```

传入方法名和参数类型，并调用getMethodRecursive，这里分为两步

1. 调用getMethodsRecursive获取MethodList

```java
private PublicMethods.MethodList getMethodsRecursive(String name,
                                                     Class<?>[] parameterTypes,
                                                     boolean includeStatic) {
    // 1st check declared public methods
    Method[] methods = privateGetDeclaredMethods(/* publicOnly */ true);
    PublicMethods.MethodList res = PublicMethods.MethodList
        .filter(methods, name, parameterTypes, includeStatic);
    // if there is at least one match among declared methods, we need not
    // search any further as such match surely overrides matching methods
    // declared in superclass(es) or interface(s).
    if (res != null) {
        return res;
    }
        
    // if there was no match among declared methods,
    // we must consult the superclass (if any) recursively...
    Class<?> sc = getSuperclass();
    if (sc != null) {
        res = sc.getMethodsRecursive(name, parameterTypes, includeStatic);
    }

    // ...and coalesce the superclass methods with methods obtained
    // from directly implemented interfaces excluding static methods...
    for (Class<?> intf : getInterfaces(/* cloneArray */ false)) {
        res = PublicMethods.MethodList.merge(
            res, intf.getMethodsRecursive(name, parameterTypes,
                                          /* includeStatic */ false));
    }

    return res;
}
```

首先调用privaeGetDeclaredMethods找到一系列methods，注意publicOnly为true，也就是说只找public的方法

```java
private Method[] privateGetDeclaredMethods(boolean publicOnly) {
    Method[] res;
    ReflectionData<T> rd = reflectionData();
    if (rd != null) {
        res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
        if (res != null) return res;
    }
    // No cached value available; request value from VM
    res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
    if (rd != null) {
        if (publicOnly) {
            rd.declaredPublicMethods = res;
        } else {
            rd.declaredMethods = res;
        }
    }
    return res;
}
```

ReflectionData rd是反射数据的缓存，如果缓存未命中，则调用native的getDeclaredMethods0方法寻找方法并存入缓存

接着回到getMethodsRecursive，有这样一个注释

`if there was no match among declared methods,
    // we must consult the superclass (if any) recursively...`

也就是说，如果在本类中没有找到，会递归调用getMethodsRecursive去父类和接口中寻找，并放入MethodList

2. 从MethodList中选出最明确的方法

这里怎么理解？点进源码

```java
/**
 * @return 1st method in list with most specific return type
 */
Method getMostSpecific() {
    Method m = method;
    Class<?> rt = m.getReturnType();
    for (MethodList ml = next; ml != null; ml = ml.next) {
        Method m2 = ml.method;
        Class<?> rt2 = m2.getReturnType();
        if (rt2 != rt && rt.isAssignableFrom(rt2)) {
            // found more specific return type
            m = m2;
            rt = rt2;
        }
    }
    return m;
}
```

源码有这样一句，`return 1st method in list with most specific return type`

即选出返回值最为明确的方法，对于MethodLIst中的方法的返回值，如果其中一个是父类，另一个是子类，则返回子类的这个。也就是说，选取返回值类型中”最子类“的那一个

**因此，getMethod0整体调用思路如下**

1. 通过getMethodsRecursive获取方法链

   1. 通过privateGetDeclaredMethod找方法，且publicOnly为true，只找public方法

      - ReflectionData缓存找

      - native的getDeclaredMethod0找

   2. 没找到，递归调用getMethodsRecursive在父类和接口中找并放入方法链

2. 筛选返回值类型最具体的方法

**3. 返回方法的拷贝**

调用的是Method类的copy()方法

```java
Method copy() {
    // This routine enables sharing of MethodAccessor objects
    // among Method objects which refer to the same underlying
    // method in the VM. (All of this contortion is only necessary
    // because of the "accessibility" bit in AccessibleObject,
    // which implicitly requires that new java.lang.reflect
    // objects be fabricated for each reflective call on Class
    // objects.)
    if (this.root != null)
        throw new IllegalArgumentException("Can not copy a non-root Method");

    Method res = new Method(clazz, name, parameterTypes, returnType,
                            exceptionTypes, modifiers, slot, signature,
                            annotations, parameterAnnotations, annotationDefault);
    res.root = this;
    // Might as well eagerly propagate this if already present
    res.methodAccessor = methodAccessor;
    return res;
}
```

- 原方法作为拷贝方法的root成员变量
- 将原方法的methodAccessor赋值给拷贝方法

> getDeclaredMethod基本流程差不多，就是将getMethod0替换成searchMethods，根据方法名、参数、返回类型来寻找方法

#### invoke

```java
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    //  1. 检查override变量检查权限
    if (!override) {
        Class<?> caller = Reflection.getCallerClass();
        checkAccess(caller, clazz,
                    Modifier.isStatic(modifiers) ? null : obj.getClass(),
                    modifiers);
    }
    //  2. 获取MethodAccessor
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    //  3. 通过MethodAccessor调用方法
    return ma.invoke(obj, args);
}
```

**1. 检查权限**

检查override变量，查看是否有权限调用方法，setAccessible就是修改的这个变量

**2. 获取MethodAccessor**

回顾上面在拷贝方法的时候，有这样一句`res.methodAccessor = methodAccessor;`令MethodAccessor为原方法的MethodAccessor，如果这个MethodAccessor为null，则创建一个新的MethodAccessor

**3. 通过MethodAccessor调用方法**

```java
public interface MethodAccessor {
    /** Matches specification in {@link java.lang.reflect.Method} */
    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException;
}
```

可以看到有这样两个实例

![image-20210623105152231](imgs\JAVA反射\2.png)

```java
class DelegatingMethodAccessorImpl extends MethodAccessorImpl {
    private MethodAccessorImpl delegate;

    DelegatingMethodAccessorImpl(MethodAccessorImpl delegate) {
        setDelegate(delegate);
    }

    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        return delegate.invoke(obj, args);
    }

    void setDelegate(MethodAccessorImpl delegate) {
        this.delegate = delegate;
    }
}
```

可以看出，DelegatingMethodAccessorImpl只是一个代理，真正实现还是NativeMethodAccessorImpl

```java
public Object invoke(Object obj, Object[] args)
    throws IllegalArgumentException, InvocationTargetException
{
    // We can't inflate methods belonging to vm-anonymous classes because
    // that kind of class can't be referred to by name, hence can't be
    // found from the generated bytecode.
    if (++numInvocations > ReflectionFactory.inflationThreshold()
            && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
        MethodAccessorImpl acc = (MethodAccessorImpl)
            new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        parent.setDelegate(acc);
    }

    return invoke0(method, obj, args);
}
```

numInvocations记录调用次数，如果超过inflationThreshold()返回的阈值，则通过MethodAccessorGenerator().generateMethod生成一个MethodAccessor实例并用这个实例来调用方法；没超过阈值，调用native的invoke0方法

```java
private static int     inflationThreshold = 15;
```

阈值为15次

调用的是`private MagicAccessorImpl generate`这个方法，返回MagicAccessorImpl

**也就是说，超过15次调用的阈值，jvm会动态生成MethodAccessor的字节码并调用其中的invoke方法，没超过则使用NativeMethodAccessorImpl，DelegatingMethodAccessorImpl作为一个代理对象，它的作用是通过setDelegate控制具体使用哪一个MethodAccessor实例**

> 实际上，java的MagicAccessorImpl调用效率要高于Native版本，但加载时需要消耗多的资源和时间，所以默认采用native
>
> native的是不好优化，它相当于一个黑箱，虚拟机难以对其分析也难以内联；虚拟机有这样一个共同点：跨越native边界会对优化有阻碍作用

**认为反射效率低的原因：**

1. invoke每次调用都需要检查方法调用等权限
2. invoke的参数是Object... args，需要对传入的参数进行封装；通过emitInvoke()方法动态生成的invoke方法又需要对参数解封（unbox）恢复
3. 需要检查参数，是否匹配等等
4. 难以内联、JIT难以优化

> 实际上，动态生成的MethodAccesor会针对方法特征，进行代码层面上的优化；生成的MethodAccessor，JIT也会对其编译优化，这样反射也能达到较高效率

> 注意invoke方法的参数，Object... args是一个变长参数，实际上是一种语法糖，最终jvm会为其生成一个数组，在调用空参方法时，我们可以使用method.invoke(obj, new Object[0])自己手动创建一个，这样就不会频繁触发gc了

## 动态代理

> 运行时动态创建代理对象

### JDK动态代理

```java
public class Test {
    public static void main(String[] args) {
        Person p = new Person(20, "张三");
        Behavior proxy = (Behavior) Proxy.newProxyInstance(Person.class.getClassLoader(), Person.class.getInterfaces(), new MyHandler(p));
        proxy.changeName("李四");
        System.out.println(proxy.getAge());
        proxy.walk();
    }
}
interface Behavior{
    void walk();
    void changeName(String name);
    Integer getAge();
}
class Person implements Behavior{
    private Integer age;
    private String name;

    public Person() {
    }

    public Person(Integer age, String name) {
        this.age = age;
        this.name = name;
    }
    public void walk(){
        System.out.println("I'm " + name + ", I'm working here");
    }
    public void changeName(String name){
        this.name = name;
    }
    public Integer getAge(){
        return age;
    }
}
class MyHandler implements InvocationHandler{
    public Object target;

    public MyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before " + method.getName() + " ...");
        Object returnVal = method.invoke(target, args);
        System.out.println("after " + method.getName() + " ...");
        return returnVal;
    }
}
```

![image-20210623143112555](imgs\JAVA反射\3.png)

要求被代理类实现一个接口

### cglib动态代理

```java
public class Test {
    public static void main(String[] args) {
        Person p = new Person(20, "张三");
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Person.class);
        enhancer.setCallback(new MyInterceptor());
        enhancer.setClassLoader(Test.class.getClassLoader());
        Person proxy = (Person) enhancer.create();
        proxy.walk();
    }
}
class MyInterceptor implements MethodInterceptor{
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before " + method.getName());
        Object returnVal = methodProxy.invokeSuper(o, objects);
        System.out.println("after " + method.getName());
        System.out.println("被代理对象: " + o.getClass());
        return returnVal;
    }
}
class Person {
    private Integer age;
    private String name;

    public Person() {
    }

    public Person(Integer age, String name) {
        this.age = age;
        this.name = name;
    }
    public void walk(){
        System.out.println("I'm " + name + ", I'm working here");
    }
    public void changeName(String name){
        this.name = name;
    }
    public Integer getAge(){
        return age;
    }
}
```

![image-20210623145515374](imgs\JAVA反射\4.png)

不需要实现接口，相当于是生成了一个对象的子类对象，并代理该子类对象

需要一个空参构造器，因为生成的子类会调用父类的空参构造器

### 动态性

```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.*;

public class Test {
    public static void main(String[] args) {
        Sing singer = new Singer();
        Sing proxy1 = ProxyFactory.newInstance(singer, ProxyFactory.Mode.JDK);
        proxy1.sing();
        Sing proxy2 = ProxyFactory.newInstance(singer, ProxyFactory.Mode.CGLIB);
        proxy2.sing();
    }
}
interface Sing{
    void sing();
}
class Singer implements Sing{
    @Override
    public void sing() {
        System.out.println("lalala");
    }
}
class ProxyFactory{
    enum Mode{
        JDK, CGLIB
    }
    final static <T> T newInstance(Object target, Mode mode){
        switch (mode){
            case JDK:{
                return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new MyHandler(target));
            }
            case CGLIB:{
                Enhancer enhancer = new Enhancer();
                enhancer.setClassLoader(target.getClass().getClassLoader());
                enhancer.setCallback(new MyInterceptor());
                enhancer.setSuperclass(target.getClass());
                return (T) enhancer.create();
            }
            default:throw new RuntimeException("出现异常");
        }
    }
}
class MyHandler implements InvocationHandler{
    public final Object target;

    public MyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before " + method.getName());
        Object returnVal = method.invoke(target, args);
        System.out.println("after " + method.getName());
        return returnVal;
    }
}
class MyInterceptor implements MethodInterceptor{
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before " + method.getName());
        Object returnVal = methodProxy.invokeSuper(o, objects);
        System.out.println("after " + method.getName());
        System.out.println("被代理对象: " + o.getClass());
        return returnVal;
    }
}
```

任意传入对象，都可以生成对应代理对象，不需要对每个目标类都单独写一个代理类，也不怕目标类方法改变

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

[2] [https://blog.csdn.net/qq_42799615/article/details/112549714——ZhaoSimonone](https://blog.csdn.net/qq_42799615/article/details/112549714)

[3] [https://zhuanlan.zhihu.com/p/86993361——Java程序猿阿谷](https://zhuanlan.zhihu.com/p/86993361)
