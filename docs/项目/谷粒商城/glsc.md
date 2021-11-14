# [返回](/)

# 准备工作

## 分布式基本概念

### 分布式和集群

分布式：不同业务分布在不同地方

集群：将几台服务器集中在一起，实现同一个业务

### 远程调用

分布式系统中，各个服务可能处于不同主机，但服务之间需要互相调用，便称为远程调用

SpringCloud使用HTTP+JSON；Dubbo使用RPC

### 负载均衡

A服务需要调用B服务，B服务在多台机器中都存在，A调用任意一个都能完成功能

为了使机器不会太忙或太闲，采用负载均衡的调用，提高网站健壮性

**常见算法**：

- 轮询：按顺序依此循环选择
- 最小连接：优先选择连接数最少的，也就是压力最小的服务器
- 散列：同一请求散列到同一服务器

### 服务发现/注册&注册中心

A服务想要调用B服务，但它并不知道B服务在哪些服务器有，哪些正常运行，哪些已下线，所以引入注册中心

**服务注册**：B将自己的服务状态等信息注册到注册中心；**服务发现**：A通过注册中心感知B服务，并采用负载均衡算法等来选择服务器

### 配置中心

每一个服务都可能有大量配置，如果改每个服务器上的配置就很麻烦，可以让每个服务在配置中心获取自己的配置

### 服务熔断/降级

微服务架构中，服务间通过网络通信，存在相互依赖，如果由于网络不稳定或其他因素，比如一个服务响应时间太久，造成其他服务停等，最终一环套一环，大量请求积压，造成所有服务不可用，造成雪崩效应。因此，必须要有容错机制来防止这一情况

**服务熔断**（单个断路）：设置服务的超时，当被调用的服务经常超时失败，失败次数达到某个阈值，开启断路保护机制，后来的请求不再调用这个服务，本地直接返回默认的数据

**服务降级**（整体把控）：当系统处于高峰期，系统资源紧张，可以让非核心业务降级运行，即某些服务不处理，或简单处理

### API网关

所有请求的入口，相当于一个”安检口“

抽象了微服务中各服务都需要的通用功能，同时提供客户端负载均衡、熔断、灰度发布、统一认证、限流监控、日志统计等功能

![image-20211014003544579](imgs\1.png)

## 微服务架构

![image-20211014003809595](imgs\2.png)

## 仓库配置

![image-20211014010857950](imgs\3.png)

> 小技巧：可以在idea里直接拉取
>
> ![image-20211014011054135](imgs\4.png)

## 项目初始化

### 创建各个微服务模块

> 注意提前导入spring-web和openFeign

商品服务、仓储服务、订单服务、优惠券服务、用户服务

![image-20211014012230130](imgs\5.png)

给总项目gulimall添加pom文件并修改成如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>person.hf</groupId>
    <artifactId>gulimall</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>gulimall</name>
    <description>聚合服务</description>
    <packaging>pom</packaging>

    <modules>
        <module>gulimall-product</module>
        <module>gulimall-ware</module>
        <module>gulimall-order</module>
        <module>gulimall-coupon</module>
        <module>gulimall-member</module>
    </modules>
</project>
```

![image-20211014012518351](imgs\6.png)

添加git版本控制忽略项，其他项Add to VCS，纳入版本控制

![image-20211014013311847](imgs\7.png)

### 创建数据库和表

PowerDesigner逆向

> 实际项目一般都不用外键，太消耗性能

### 使用人人开源快速开发平台

![image-20211014232427818](imgs\8.png)

#### 下载并导入

将renren-fast和renren-fast-vue克隆下来

- renren-fast导入项目，作为项目后台管理系统，并加入总pom文件的module中

> pom文件的build内的内容有时会飘红，且无法连接远程仓库下载。一个简单的解决办法是，先移到dependency里面，远程下载好后，再移回去

- 下载renren-fast-vue并运行

> nass错误：Node Sass does not yet support your current environment: Windows 64-bit...。移除modules的nass模块，并用npm install nass重新下载*（可以配置淘宝源，然后用cnpm）*

同时运行人人前后端模块，接通项目

#### 逆向工程

- 下载renren-generator并运行，逆向生成对应的mapper.xml和其他各种代码

需要生成哪个数据库的，就在配置文件yml里修改成相应的数据库；同时也要修改properties文件中的配置

登入localhost:80

![image-20211020005710827](imgs\9.png)

将生成的main文件夹导入到相应的module，如这里对应的是gulimall-product表和product模块

> 可以在template文件夹中定制生成模板

#### 公共模块

可以看见很多依赖都是爆红的，新创建一个module，**将所有的公共依赖、工具类、bean等都放入这个module中**

![image-20211020010153180](imgs\10.png)

添加公共依赖，并将gulimall-common作为其他子模块的dependency的一员

![image-20211020010654073](imgs\11.png)

于是，相应模块就不爆红了

![image-20211020010857426](imgs\12.png)

同理，将一些公共类和bean放到common中

### 测试基本CRUD

针对product模块

#### 整合mybatis-plus

1. 配置数据源

   1. 导入数据库驱动

   2. 配置

      ```yaml
      spring:
        datasource:
          username: root
          password: xsfhHF981123
          url: jdbc:mysql://118.31.246.207:3306/gulimall_pms?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
          driver-class-name: com.mysql.cj.jdbc.Driver
      ```

2. 配置mybatis-plus

   1. 添加注解，告知注解扫描位置

      ```java
      @MapperScan("person.hf.product.dao")
      @SpringBootApplication
      public class GulimallProductApplication {
      
          public static void main(String[] args) {
              SpringApplication.run(GulimallProductApplication.class, args);
          }
      
      }
      ```

   2. 添加配置，告知mapper.xml映射文件位置

      ```yaml
      mybatis-plus:
        mapper-locations: classpath*:/mapper/**/*.xml
      ```

      > classpath*：除自己以外，也扫描依赖的jar包的classpath

   3. 添加配置，设置主键自增

      ```yaml
      mybatis-plus:
        mapper-locations: classpath*:/mapper/**/*.xml
        global-config:
          db-config:
            id-type: auto
      ```

3. 测试

   ```java
   @SpringBootTest
   class GulimallProductApplicationTests {
   
       @Autowired
       private BrandService brandService;
   
       @Test
       void contextLoads() {
           BrandEntity brandEntity = new BrandEntity(){{
               setName("HuaWei");
           }};
           brandService.save(brandEntity);
       }
   
   }
   ```

   ![image-20211020021244851](imgs\13.png)

   成功插入

> 用同样的步骤，逆向生成各子模块的工程，分别采用端口7000、8000、9000、11000、12000（6000、10000被占用）

## 微服务环境搭建

> 初代spring cloud技术搭配方案：Eureka注册中心 + Spring Cloud Config配置中心 + Hystrix断路器 + Zuul网关等等
>
> 缺点：几大组件停止维护和更新、配置复杂等等......
>
> 因此采用Spring Cloud Alibaba的方案：
>
> - Spring Cloud Alibaba - Nacos：注册中心（服务注册/发现）
> - Spring Cloud Alibaba - Nacos：配置中心（动态配置管理）
> - Spring Cloud Ribbon：负载均衡
> - Spring Cloud Feign：声明式HTTP客户端（远程服务调用）
> - Spring Cloud Alibaba - Sentinel：服务容错（熔断、限流、降级）
> - Spring Cloud Gateway：API网关（Webflux编程模式）
> - Spring Cloud Sleuth：调用链监控
> - Spring Cloud Alibaba - Seata：原Fescar，分布式事务解决方案

在common模块中引入：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2021.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 注册中心-Nacos

#### 启动注册中心

1. 在common模块引入nacos

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    ...
</dependencies>
<!-- 版本号在dependencyManagement统一管理 -->
```

2. 启动Nacos服务器，令注册中心运行起来

> 具体见：
>
> - 官方：[https://nacos.io/zh-cn/docs/quick-start-docker.html](https://nacos.io/zh-cn/docs/quick-start-docker.html)
> - 其他：[https://www.jianshu.com/p/3d3e17bc629f](https://www.jianshu.com/p/3d3e17bc629f)
>
> 这里选择官方docker版

单机方式启动

```shell
git clone https://github.com/nacos-group/nacos-docker.git
cd nacos-docker
docker-compose -f example/standalone-mysql-5.7.yaml up # or start，后台运行
# 8版本各种报错，cpu也飙贼高，还卡住。。。这里先贴几个issue，留着以后用：
# https://www.jianshu.com/p/36926d3cd700
# https://www.cnblogs.com/gyli20170901/p/11245270.html
# https://www.freesion.com/article/8293991129/
```

> docker-compose not found：
>
> ```shell
> yum -y install epel-release
> yum -y install python3-pip
> pip3 install --upgrade pip
> # pip -V 查看版本
> pip install docker-compose
> ```
>
> 或：[https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

**问题**

> 1. **记得停用自己的3306端口和33060端口并删除名字为“mysql”的容器，否则会产生冲突**
>
> 2. 容器部署nacos之后，再开启另外一个mysql容器，疯狂重启，机器卡死
>
>    - 原因见：[https://blog.csdn.net/qq_42915936/article/details/105184212?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_aggregation-12-105184212.pc_agg_rank_aggregation&utm_term=docker%E5%90%AF%E5%8A%A8%E4%B8%80%E4%B8%AAnacos%E5%AE%B9%E5%99%A8%E5%90%8E%E6%97%A0%E6%B3%95%E8%AE%BF%E9%97%AE&spm=1000.2123.3001.4430](https://blog.csdn.net/qq_42915936/article/details/105184212?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_aggregation-12-105184212.pc_agg_rank_aggregation&utm_term=docker%E5%90%AF%E5%8A%A8%E4%B8%80%E4%B8%AAnacos%E5%AE%B9%E5%99%A8%E5%90%8E%E6%97%A0%E6%B3%95%E8%AE%BF%E9%97%AE&spm=1000.2123.3001.4430)
>
>      nacos默认启动内存为1g，docker内存不够，导致疯狂重启，解决方法：① 增大docker运行内存；② 修改默认启动内存
>
>    - **如何修改nacos-docker启动参数**
>
>      因为我们是通过docker-compose -f example/standalone-mysql-5.7.yaml up打开的，点开该文件
>
>      ![image-20211026102150092](imgs\14.png)
>
>      可以看到../env/nacos-standlone-mysql.env为启动环境相关的文件，打开看看
>
>      ![image-20211026102442005](imgs\15.png)
>
>      修改启动内存，不再报错，大功告成~
>
>      PS：[附上env的各种参数](https://nacos.io/zh-cn/docs/quick-start-docker.html)

![image-20211026103437971](imgs\16.png)

启动成功！

#### 项目接入

> 以coupon服务为例

1. **启动注册中心后**，在每个模块的配置文件中指定nacos服务器的地址**和服务名**

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  application:
    name: coupon
```

2. 主程序添加`@EnableDiscovery`注解开启服务注册与发现功能

```java
@MapperScan("person.hf.coupon.dao")
@SpringBootApplication
@EnableDiscoveryClient
public class GulimallCouponApplication {

    public static void main(String[] args) {
        SpringApplication.run(GulimallCouponApplication.class, args);
    }

}
```

3. 运行该服务

> 1. 运行错误：`java.lang.IllegalArgumentException: Param 'serviceName' is illegal, serviceName is blank`
>
>    配置中没有添加服务名导致的
>
> 2. 异常INFO提示但运行正常：`Cannot determine local hostname`
>
>    貌似跟读取网卡的超时时间有关，添加`spring.cloud.inetutils.timeout=10`解决

可以看到成功注册该服务

![image-20211028011920858](imgs\17.png)

同样的方法注册其他几个服务

### 远程调用-OpenFeign

一个声明式的HTTP客户端，通过HTTP请求的方式调用其他服务

#### 项目接入

以member服务调用coupon服务某功能为例

```java
@RestController
@RequestMapping("coupon/coupon")
public class CouponController {
    @Autowired
    private CouponService couponService;

    @Deprecated
    @RequestMapping("/test/list")
    public R list(){
        CouponEntity couponEntity = new CouponEntity();
        couponEntity.setCouponName("满100减10");
        return R.ok().put("coupons", couponEntity);
    }
    // ......
}
```

1. 在调用方引入openfeign的依赖；并引入loadbalancer依赖*（可放在common中）*

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2. 在调用方编写一个专门用于调用远程服务的接口*(feign包也可放在common中，便于统一管理)*

```java
public interface CouponFeignService {
    // 以被调用的服务命名，易定位
}
```

3. 调用方的接口添加相关注解

```java
/**
 * 注册中心会先根据名字去找coupon服务，再调用里面对应的方法，即服务发现
 */
// 服务名
@FeignClient("coupon")
@Service
public interface CouponFeignService {
    // 被调用方的方法的完整签名和完整路径
    @RequestMapping("/coupon/coupon/test/list")
    R list();
}
```

4. 在调用方的主程序添加`@EnableFeignClients`注解

```java
@EnableFeignClients(basePackages = "person.hf.member.feign")
```

5. 编写一个方法来进行openfeign调用

```java
@RestController
@RequestMapping("member/member")
public class MemberController {
    @Autowired
    private MemberService memberService;

    @Deprecated
    @Autowired
    private CouponFeignService couponFeignService;
    
    @Deprecated
    @RequestMapping("/test/coupons")
    public R testMemberCoupons(){
        R list = couponFeignService.list();
        MemberEntity memberEntity = new MemberEntity();
        memberEntity.setNickname("hufei");
        return R.ok().put("member", memberEntity).put("coupon", list.get("coupons"));
    }
    // ......
}
```

> 都是在主动调用方的操作

运行结果如图：

![image-20211028015918812](imgs\18.png)

### 配置中心-Nacos

#### 使用场景

当我们需要使用配置项里面的属性时，通常的写法如下：

```yaml
coupon:
  username: zhangsan
  age: 22
```

并添加`@Value`注解

```java
@RestController
@RequestMapping("coupon/coupon")
public class CouponController {

    @Value("${coupon.username}")
    private String username;
    @Value("${coupon.age}")
    private int age;

    @Deprecated
    @RequestMapping("/test")
    public R test(){
        return R.ok().put("username", username).put("age", age);
    }
    // ......
}
```

但如果想要修改这个属性，就必须手动修改配置文件，然后重新启动项目

而配置中心，可以实现动态配置

#### 项目接入

1. 引入相关依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

2. 创建bootstrap.properties并添加字段

优先级大于application.properties

连接配置中心，即nacos节点

```xml
spring.application.name=coupon
spring.cloud.nacos.config.server-addr=118.31.246.207:8848
```

2.0版本之后，还需要引入依赖使bootstrap生效

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
    <version>3.0.4</version>
</dependency>
```

3. 在nacos中新建一个配置

Data ID为服务名.properties

![image-20211109010518303](imgs\19.png)

4. 在配置对应的类添加`@RefreshScope`，刷新配置的注解，并配合`@Value`注解

```java
@RestController
@RefreshScope
@RequestMapping("coupon/coupon")
public class CouponController {

    @Value("${coupon.username}")
    private String username;
    @Value("${coupon.age}")
    private int age;
    // ......
}
```

4. 重启应用

会看到启动日志多了很多配置相关信息

5. 修改配置，应用的属性也相应改变

![image-20211109011846314](imgs\20.png)

![image-20211109011906504](imgs\21.png)

- 并不局限于属性，也可以是其他配置
- 配置中心的配置优先级大于application.properties

> - 可以使用bootstrap.yml
>
> - 可以使用服务名.yml作为Data ID，只不过要在bootstrap.properties/.yml中设置
>
>   ```properties
>   spring.cloud.nacos.config.extension=yml
>   ```
>
> - 可以不一定使用服务名作为Data ID，需要在bootstrap中指定`prefix`作为名字

#### 命名空间与配置分组

当我们有很多微服务时和多个运行环境时，不可能都把配置放在一个地方，需要将他们分散开来，并实现配置隔离

采用的策略是：每个微服务对应一个命名空间，每个运行环境（dev/test/prod）对应一个配置组

- 命名空间

  命名空间默认为public，可以建造一个以微服务名来定义的命名空间

  1. 新建命名空间，拷贝其ID

     ![image-20211114164239068](imgs\22.png)

     ![image-20211114164306716](imgs\23.png)

  2. 修改coupon服务的bootstrap.properties，写入命名空间

     ![image-20211114164524889](imgs\24.png)

  3. 启动应用，新增一个配置，或将之前的配置拷贝到新的命名空间

     ![image-20211114164848400](imgs\25.png)

     ![image-20211114164929192](imgs\26.png)

     点击切换命名空间

> 注意：注册在不同命名空间下的服务之间无法相互调用，所以在注册服务时可将dev/test/prod作为命名空间

- 配置分组

  1. 给配置添加一个分组

     ![image-20211114172441791](imgs\27.png)

  2. bootstrap.properties/.yml里添加group信息

     ```yml
     spring:
       application:
         name: coupon
       cloud:
         nacos:
           config:
             server-addr: 118.31.246.207:8848
             namespace: 007fd91a-cf3f-4c1d-943f-9369975e7062
             group: dev
             file-extension: yml
     ```

#### 总结

1. 实际上，任何配置都可以交给配置中心管理，包括任务启动的初始配置

   需要在bootstrap.yml里添加：

   ```yaml
   spring:
     application:
       name: coupon
     cloud:
       nacos:
         config:
           server-addr: 118.31.246.207:8848
           namespace: 007fd91a-cf3f-4c1d-943f-9369975e7062
           group: dev
           file-extension: yml
           extension-configs:
             - data-id: datasource.yml
               group: dev
   
             - data-id: mybatis.yml
               group: dev
   
             - data-id: others.yml
               group: dev
   ```

   ![image-20211115001015705](imgs\28.png)

   便可以通过配置中心管理启动配置，只不过每次修改后，需要重启应用，而不能够动态更改

2. 任何springboot常用的读取配置的@Value、@ConfigurationProperties等等，都可以使用配置中心来管理并动态修改

3. 配置中心的优先级大于其他配置

### 网关-Gateway

#### 使用场景

- 鉴权

- 路由转发、反向代理

- 过滤机制、断言机制等等

  [官方文档]([Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/docs/2.2.7.RELEASE/reference/html/#the-path-route-predicate-factory))

#### 项目接入

1. 构建gateway模块

   引入common模块；添加到注册中心和配置中心；去除冗余的组件

   ```yaml
   spring:
     cloud:
       nacos:
         discovery:
           server-addr: 118.31.246.207:8848
       inetutils:
         timeout-seconds: 10
     application:
       name: gateway
   
   server:
     port: 88
   ```

   ```yaml
   spring:
     application:
       name: gateway
     cloud:
       nacos:
         config:
           server-addr: 118.31.246.207:8848
           namespace: 007fd91a-cf3f-4c1d-943f-9369975e7062
           group: dev
           file-extension: yml
   ```

   ```java
   @EnableDiscoveryClient
   @SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
   public class GulimallGatewayApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(GulimallGatewayApplication.class, args);
       }
   
   }
   ```

2. 测试一个简单的例子

   ```yaml
   spring:
     cloud:
       nacos:
         discovery:
           server-addr: 118.31.246.207:8848
       inetutils:
         timeout-seconds: 10
       gateway:
         routes:
           - id: test
             # 转发到coupon服务（只能填服务名，代表从注册中心获取服务，且以lb(load-balance)负载均衡方式转发）
             uri: lb://coupon
             predicates:
               # 将88/myTest/coupon/coupon/test/**转发到coupon/coupon/coupon/test/**（去掉了1个前缀（myTest），在filter中指定）
               - Path=/myTest/coupon/coupon/test/**
             filters:
               - StripPrefix=1
   
           # 如果请求参数有url=baidu，则转发到baidu.com
           - id: test2
             # 一定要加上协议，如lb/http/https
             uri: https://www.baidu.com
             predicates:
               - Query=url,baidu
   
     application:
       name: gateway
   server:
     port: 88
   ```

![image-20211115020326220](imgs\29.png)

![image-20211115024241564](imgs\30.png)



更多规则可参考：[官方文档]([Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/docs/2.2.7.RELEASE/reference/html/#the-path-route-predicate-factory))

# 问题总结

## 前端

- nass错误：Node Sass does not yet support your current environment: Windows 64-bit...

  > 移除modules的nass模块，并用npm install nass重新下载*（可以配置淘宝源，然后用cnpm）*

## 后端项目构建相关

### POM

- pom文件的build内的内容有时会飘红，且无法连接远程仓库下载

  > 先查看本地仓库，加入本地仓库有的版本号
  >
  > 如果本地仓库没有，一个简单的解决办法是，先移到dependency里面，远程下载好后，再移回去

### 编译

- java: JPS incremental annotation processing is disable...

  > 这种情况，将lombok插件版本更新到1.18.20即可

## 微服务

### 注册中心

- 容器部署nacos之后，再开启另外一个mysql容器，疯狂重启，机器卡死

  > - 原因见：[https://blog.csdn.net/qq_42915936/article/details/105184212?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_aggregation-12-105184212.pc_agg_rank_aggregation&utm_term=docker%E5%90%AF%E5%8A%A8%E4%B8%80%E4%B8%AAnacos%E5%AE%B9%E5%99%A8%E5%90%8E%E6%97%A0%E6%B3%95%E8%AE%BF%E9%97%AE&spm=1000.2123.3001.4430](https://blog.csdn.net/qq_42915936/article/details/105184212?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_aggregation-12-105184212.pc_agg_rank_aggregation&utm_term=docker%E5%90%AF%E5%8A%A8%E4%B8%80%E4%B8%AAnacos%E5%AE%B9%E5%99%A8%E5%90%8E%E6%97%A0%E6%B3%95%E8%AE%BF%E9%97%AE&spm=1000.2123.3001.4430)
  >
  >   *采用官方方式：[https://nacos.io/zh-cn/docs/quick-start-docker.html](https://nacos.io/zh-cn/docs/quick-start-docker.html)*
  >
  >   nacos默认启动内存为1g，docker内存不够，导致疯狂重启
  >
  > - 解决方法：① 增大docker运行内存；② 修改默认启动内存
  >
  >   **如何修改nacos-docker启动参数**
  >
  >   因为我们是通过docker-compose -f example/standalone-mysql-5.7.yaml up打开的，点开该文件
  >
  >   ![image-20211026102150092](imgs\14.png)
  >
  >   可以看到../env/nacos-standlone-mysql.env为启动环境相关的文件，打开看看
  >
  >   ![image-20211026102442005](imgs\15.png)
  >
  >   修改启动内存，不再报错，大功告成~
  >
  >   PS：[附上env的各种参数](https://nacos.io/zh-cn/docs/quick-start-docker.html)

- 注册中心运行时报错

  1. 运行错误：`java.lang.IllegalArgumentException: Param 'serviceName' is illegal, serviceName is blank`

     > 配置中没有添加服务名导致的

  2. 异常INFO提示但运行正常：`Cannot determine local hostname`

     > 貌似跟读取网卡的超时时间有关，添加`spring.cloud.inetutils.timeout=10`解决

# 参考



