# [返回](/)

# Springboot

## 两大特性

### 依赖管理

```java
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.4.RELEASE</version>
</parent>
    
<!--父项目-->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.3.4.RELEASE</version>
</parent>
```

- 父项目做依赖管理
- 几乎声明了所有开发中常用依赖的版本号，即：自动版本仲裁机制

`spring-boot-starter-*`：官方starter；`*-spring-boot-starter`：第三方自定义starter

所有场景启动器最底层的依赖：

```java
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
  <version>2.5.2</version>
  <scope>compile</scope>
</dependency>
```

main：

```java
//  1. 返回IOC容器
ConfigurableApplicationContext run = SpringApplication.run(Demo1Application.class, args);
//  2. 显示所有组件
Arrays.stream(run.getBeanDefinitionNames()).forEach(System.out::println);
//  可以查看各种组件，例如mvc的DispatcherServlet、ViewResolver等等，以及主程序所在包下的所有组件
//  ps：可通过设置@SpringBootApplication(scanBasePackages = "xxx")改变扫描的包，扫描主程序包外的组件
```

### 自动配置

`spring-boot-autoconfigure`

#### 组件添加

针对User类

```java
@Data
@NoArgsConstructor
public class User {
    private Integer age;
    private String name;
}
```

- 按以前Spring的方式：

创建xml文件→编写xml文件

```xml
<bean id="user" class="com.example.demo.pojo.User">
    <property name="age" value="20"/>
    <property name="name" value="zhangsan"/>
</bean>
```

- springboot注解配置：

1. Full模式

```java
@Configuration(proxyBeanMethods = true)  // 默认为true
public class BeanConfiguration {
    @Bean("user")
    public User getUser(){
        return new User(20, "zhangsan");
    }
}
```

```java
ConfigurableApplicationContext run = SpringApplication.run(Demo1Application.class, args);
BeanConfiguration beanConfiguration = (BeanConfiguration) run.getBean("beanConfiguration");  
// beanConfiguration为默认名字，第一个字母小写
// OR
// run.getBean(BeanConfiguration.class);
System.out.println(beanConfiguration);
User u1 = beanConfiguration.getUser();
User u2 = beanConfiguration.getUser();
System.out.println(u1 == u2); 
```

![image-20210702224931286](imgs\1.png)

**解释：**proxyBeanMethods = true说明通过cglib生成了BeanConfiguration的代理对象，并代理了相应方法，对方法进行了增强，使每次添加组件时都会检测ioc容器中是否已有该组件，有就直接返回，没有再创建

因此，此时的组件是单例的

1. Lite模式

令proxyBeanMethods = false

![image-20210702225247044](imgs\2.png)

生成的是普通的BeanConfiguration，每次直接调用getUser，自然user组件不是单例了

**组件依赖**

添加一个宠物类

```java
@Data
@NoArgsConstructor
public class Pet {
}
```

在User中

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Integer age;
    private String name;
    private Pet pet;
}
```

现在需要两个不同用户拥有同一只宠物

```java
@Configuration(proxyBeanMethods = true)
public class BeanConfiguration {
    @Bean("user")
    public User getUser(){
        User zhangsan = new User(20, "zhangsan", null);
        zhangsan.setPet(getPet());
        return zhangsan;
    }

    @Bean("cat")
    public Pet getPet(){
        return new Pet();
    }
}
```

```java
ConfigurableApplicationContext run = SpringApplication.run(Demo1Application.class, args);
BeanConfiguration beanConfiguration = (BeanConfiguration) run.getBean("beanConfiguration");
User u1 = new BeanConfiguration().getUser();
User u2 = new BeanConfiguration().getUser();
u1.setPet(beanConfiguration.getPet());
u2.setPet(beanConfiguration.getPet());
System.out.println(u1 == u2);  // false
System.out.println(u1.getPet() == u2.getPet());  // 结果已验证为true
```

采用Full模式，可以实现

改为Lite模式则无法实现

![image-20210702230903878](imgs\3.png)

**总结**

> 在不要求组件依赖或者其他什么单例需求的情况下，由于Full模式生成代理类，且每次获取组件都需要检测组件是否在ioc容器存在，因此推荐使用Lite模式，加载速度更快

**组件导入**

利用Import关键字，调用组建的无参构造器在ioc容器中创建组件，名字为全限定类名

```java
@Configuration(proxyBeanMethods = false)
@Import({User.class})
public class BeanConfiguration {
    @Bean("cat")
    public Pet getPet(){
        return new Pet();
    }
}
```

#### 条件判断注解

> ConditionalOnXXX：在XXX条件下才执行下面的操作

![image-20210703015629159](imgs\4.png)

```java
@Configuration
public class BeanConfiguration {
    @ConditionalOnBean({Pet.class})  // 有Pet组件的情况下才注册User组件
    @Bean("user")
    public User getUser(){
        User zhangsan = new User(20, "zhangsan", null);
        zhangsan.setPet(getPet());
        return zhangsan;
    }

    // @Bean("cat")
    public Pet getPet(){
        return new Pet();
    }
}
```

> ImportResource：根据xml文件注入组件
>
> ```java
> @Configuration
> @ImportResource("/beans.xml")
> public class BeanConfiguration {
>     ......
> }
> ```

#### 配置绑定

> 场景：实现User组件按yml文件中生成

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Component  // 只针对存在于容器中的组件，能进行配置绑定
@ConfigurationProperties(prefix = "user")
public class User {
    private Integer age;
    private String username;  // not name
    private Integer sex;
}
```

```java
@Configuration
public class BeanConfiguration {

    final private User user;

    @Autowired
    public BeanConfiguration(User user) {
        this.user = user;
    }
}
```

```yaml
user:
  username: zhangsan  
  age: 24
  sex: 1
```

**配置绑定是通过getter和setter来实现的，必须有getter和setter**

> 这里有个深坑...命名user.name的时候，由于systemProperties里面也有个user.name，会优先加载系统变量，自己的就被覆盖了......尽量避免这种命名，[相关链接](https://blog.csdn.net/kingwinstar/article/details/107563823)
>
> ```xml
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-configuration-processor</artifactId>
> </dependency>
> ```
>
> processor的依赖用于在yaml文件中显示相应属性的提示，自动补全
>
> ![image-20210924094609923](imgs\5.png)

还有一种方法也可以，去掉实体类中Component注解并在主方法添加EnableConfigurationProperties注解

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@ConfigurationProperties(prefix = "user")
public class User {
    private Integer age;
    private String username;
    private Integer sex;
}
```

```java
@SpringBootApplication
@EnableConfigurationProperties({User.class})
public class Demo1Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(Demo1Application.class, args);
        run.getBeansOfType(User.class).forEach((s, user) -> System.out.println(user));
    }
}
```

> ps：仍存在其他方法，比如@Value等
>
> ```java
> @Data
> @NoArgsConstructor
> @AllArgsConstructor
> @Component
> public class User {
>     @Value("${user.age}")
>     private Integer age;
>     @Value("${user.username}")
>     private String username;
>     @Value("${user.sex}")
>     private Integer sex;
> }
> ```



## 参考

[1] [尚硅谷Springboot2——雷神](https://www.bilibili.com/video/BV19K4y1L7MT)

