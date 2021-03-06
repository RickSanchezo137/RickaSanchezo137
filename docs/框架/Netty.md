# [返回](/)

# Netty

异步、基于事件驱动的网络应用框架

应用场景：RPC框架等

# TCP通信基础

## 引言

以一个例子入手：

```sh
exec 8<> /dev/tcp/www.baidu.com/80
echo -e "GET / HTTP/1.0\n" 1>& 8
cat 0<& 8
```

逐行解释：

- `exec 8<> /dev/tcp/www.baidu.com/80`

  - linux一切皆文件，/dev/tcp/www.baidu.com/80看似是路径实则是一个socket，用8指向了一个socket，8就相当于一个通信的channel，<>代表输入输出流

  - 8 → fd：文件描述符，可以理解成类似Java中的变量引用

    > [https://www.zhihu.com/search?type=content&q=%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6](https://www.zhihu.com/search?type=content&q=%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6)

  - exec：首先理解一下shell是什么？

    > shell的英文含义是“壳”；
    >
    > 它是相对于内核来说的，因为它是建立在内核的基础上，面向于用户的一种表现形式，比如我们看到一个球，见到的是它的壳，而非核
    >
    > Linux中的shell，是指一个面向用户的命令接口，表现形式就是一个可以由用户录入的界面，这个界面也可以反馈运行信息；相当于一个死循环，不断read用户的指令录入，并返回结果

    - exec后面接上指令，该指令会替换掉shell，比如exec ls命令，ls会显示目录后退出，如果用ls替换shell，执行完毕后，用户与系统的链接也就断开了

      ![image-20211209165351917](imgs\netty\16.png)

    - 任何程序都有输入输出流，比如ls 1>xxx.txt，1是一个fd，代表标准输出流，通过>重定向到xxx.txt

      ![image-20211209165756780](imgs\netty\17.png)

    - 可以通过exec给出重定向的绑定，解释起来就是：建立文件描述符8，并将它的输入输出流与socket绑定到一起

- `echo -e "GET / HTTP/1.0\n" 1>& 8`

  - 1是echo命令的绑定标准输出流的fd
  - 发送一个http协议头到8指向的socket
  - \>：到文件；>&：到另一个fd

- `cat 0<& 8`

  - 输入到cat的fd 0
  - cat的作用是打印

执行结果：

![image-20211209170735795](imgs\netty\18.png)

## 什么是TCP协议？

传输控制层的**，面向连接的，可靠的传输协议**（两个重点）

### 什么是面向连接的？

**三次握手概述：**

![image-20211209172534759](imgs\netty\19.png)

双方确认对方都能收到后，会分别自己开启服务资源，用于处理自己包的发送和接收对方的包

**只有在三次握手之后，双方完成了服务资源的开辟，才可以说连接被建立了**

这里的连接，不是指物理的，而是双方有相互的通信信道及对应的服务资源

- **三次握手带来了连接，确认机制保证了可靠的传输**
- 三次握手是由四层（TCP/IP五层模型/OSI七层模型下面四层）内核完成的，数据传输是由七层用户完成的

---

**四次挥手概述：**

![image-20211209200639113](imgs\netty\22.png)

双方都确认关闭资源了，才断开连接

### 什么是socket？

**socket是对TCP/IP协议的封装，它的出现只是使得程序员更方便地使用TCP/IP协议栈而已。socket本身并不是协议，它是应用层与TCP/IP协议族通信的中间软件抽象层，是一组调用接口（TCP/IP网络的API函数）**

socket一定是成对的：ip+port对应另一个ip+port，能够代表一个独立的连接

![image-20211209173604365](imgs\netty\20.png)

> 面试题：
>
> 1. ipC为客户端，向ipA:80最多能建立多少个连接？
>
>    65535个，客户端端口随机分配，范围0~65535，0是被保留的
>
> 2. 与ipA:80建立了65535个连接后，可以继续和ipB:80建立链接吗？
>
>    可以，因为一个套接字是由4个维度来确定的

对于服务端的22端口已经建立了若干个连接，有另外的客户端带着数据包过来，经过三次握手后，服务端开辟资源并fork进程/线程，令一个单独的进程/线程持有该socket（通过文件描述符来唯一标识，比如上面那个例子中的“8”）

> 多个socket对应一个进程/线程：多路复用（select/epoll）

### 抓包验证

开一个终端，运行`tcpdump -nn (加-X可以打印出详细的抓到的数据) -i eth0 port 80`抓包

新开一个终端，运行`curl www.baidu.com`

![image-20211209195727445](imgs\netty\21.png)

- 第一段为三次握手，数据包大小为0
- 第二段为数据传输，随机分配的57300端口和百度80端口之间互相通信，注意发送和确认机制
- 第三段为四次挥手，数据包大小为0

## 网络通信流程简述

> [https://blog.csdn.net/Stephen___Qin/article/details/120466415](https://blog.csdn.net/Stephen___Qin/article/details/120466415)
>
> [https://blog.csdn.net/weixin_44786530/article/details/89382641](https://blog.csdn.net/weixin_44786530/article/details/89382641)

![img](imgs\netty\23.png)

以一个例子来说，一个HTTP数据包想要发送给百度

> 查看本机网卡：`vi /etc/sysconfig/network-scripts/ifcfg-enp0s3 `
>
> ![image-20211209212605473](imgs\netty\24.png)
>
> dhcp随机分配ip

查看本机IP：`ifconfig`

![image-20211209212828680](imgs\netty\25.png)

查看百度IP：`ping`

![image-20211209213314405](imgs\netty\26.png)

也就是说：从192.168.199.66某个端口向110.242.68.3某个端口进行通信

> 一个ip地址，掩码覆盖的范围为网络部分，剩下为主机部分

- 首先，将HTTP数据包封装成TCP数据包，准备向目标主机端口发送

- 传输TCP数据包需要经过网络层，封装成IP数据包并转发

  在路由表中，将目标地址与每一条的掩码相与求出网络部分，看看是否能与Destination匹配

  ![image-20211209214006461](D:\笔记\秋招\docs\框架\imgs\netty\27.png)

  ​	目标地址为110.242.68.3，相与结果为：

  ​	① 0.0.0.0；② 110.242.0.0；③ 110.242.68.0；④ 110.242.68.0

  只有第一个能够匹配，于是发送给Gateway 192.168.199.1转发，192.168.199.1再经历相同过程向下一个路由节点转发

- 设想一下：路由节点转发过程中，目标IP（110.242.68.3）与本机匹配不上，为什么不会选择丢包而是继续转发呢？这就涉及到链路层ARP协议了

  - 在路由转发过程中，首先会根据下一跳节点的IP，请求对应主机返回自己的MAC地址，存在本机ARP列表

    ![image-20211209215456612](imgs\netty\29.png)

  - 根据ARP列表，发现对应的MAC为34:96:72:1c:fa:3e，将IP数据包封装成MAC数据包，发送到主机192.168.199.1

  - 192.168.199.1发现MAC包就是自己的地址，但IP不匹配，于是需要继续转发，将MAC头清除，并根据ARP列表封装下一跳的MAC地址

  - 发送给物理层进行传输

- 若干跳之后，终于到达百度，发现IP匹配，拆封，根据目标端口号在对应端口号上开辟服务资源

> 同一网段下的主机，路由表对应的掩码为255.255.255.0，gateway为0.0.0.0，表示不用利用网关进行转发，只需要通过链路层交换机即可

> 内网数据传输：主机 → 交换机（LAN口） → 主机
>
> 外网数据传输：主机 → 交换机 → 路由器（WAN口） → 若干个路由器 → 交换机 → 主机
>
> 传输过程中，目标IP不变，MAC地址不断变化

> VPN传输：IP数据包里面再包一个加密的IP数据包，外面的到香港，里面的到美国（举个简单的例子）

# IO模型

## 一些概念

在编写服务器端网络程序时，我们最常见到阻塞、非阻塞、同步和异步这四个词。它们的解释分别如下：

- 阻塞： 阻塞调用是指调用返回之前，当前线程会被挂起，只有当调用得到结果后才返回
- 非阻塞：与阻塞相反，非阻塞调用是指在不能立即得到结果之前，该函数不会将当前线程阻塞，而是立即返回
- 同步：所谓同步，就是在发出一个功能调用时，在没有得到结果之前，该调用就不返回。等前一件做完了才能做下一件事
- 异步：异步的概念和同步相对。当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的部件在完成后，通过状态、通知和回调来通知调用者

> 阻塞可以是实现同步的一种手段；例如两个东西需要同步，一旦出现不同步情况，我就阻塞快的一方，使双方达到同步
>
> 同步是两个对象之间的关系，而阻塞是一个对象的状态
>
> **阻塞非阻塞的划分标准是“调用者是否需要一直阻塞等待IO操作就绪”**
>
> **同步异步的划分标准是“调用者是否需要等待I/O操作完成”**

## 五种 IO 模型

服务器端 IO 主要分为两种：磁盘 IO 和网络 IO，在讲服务器端高性能网络编程时更多时候我们讲的是网络 IO 模型。一次完整的服务器端处理网络请求流程图如下（简化版，以 Web 服务器为例）：

![img](imgs\netty\1.png)



客户端请求数据 → 网卡 → 内核空间 → 用户空间

用户处理后的数据 → 内核空间 → 网卡 → 返回给客户端

### 1、阻塞 IO 模型（blocking IO）

![img](imgs\netty\2.png)

应用程序进行 recvfrom 系统调用时将阻塞在此调用，直到该套接字上有数据并且复制到用户空间缓冲区。该模式一般配合多线程使用，应用进程每接收一个连接，为此连接创建一个线程来处理该连接上的读写以及业务处理

- 优点：实现简单
- 缺点：如果套接字上没有数据，进程将一直阻塞。这时其他套接字上有数据也不能进行及时处理。如果是多线程方式，除非连接关闭否则线程会一直存在且阻塞，而线程的创建、维护和销毁非常消耗资源，所以能建立的连接数量非常有限

#### 编程实现

1. 服务端建立一个ServerSocket*（Socket，应用层与传输层之间的抽象层，向应用程序提供一套使用TCP等通信协议通信的简单接口）*
2. 客户端建立Sokcet对服务器进行通信，默认情况下服务端要为每个客户端请求建立一个通信线程
3. 客户端发出请求后，先咨询服务端是否有线程响应
   - 没有，则等待，或拒绝
   - 有，则客户端阻塞等待到请求完成后，再执行接下来的操作

**服务端：**

- 旧IO实现：

```java
/**
 *  @ClassName: BIOServer
 *  @Description:
 *  @Author: 747591245@qq.com
 *  @Date: 2021/12/6
 */
public class BIOServer {
    public static void main(String[] args) throws IOException {
        ExecutorService service = Executors.newFixedThreadPool(10);
        ServerSocket serverSocket = new ServerSocket(6666);
        while (true){
            System.out.println("当前线程数: " + ((ThreadPoolExecutor) service).getActiveCount());
            final Socket accept = serverSocket.accept();
            service.execute(new Runnable() {
                @Override
                public void run() {
                    handler(accept);
                }
            });
        }
    }
    public static void handler(Socket socket){
        try (InputStream inputStream = socket.getInputStream()){
            byte[] bytes = new byte[1024];
            while (true){
                int read = inputStream.read(bytes);
                if (read != -1){
                    System.out.println(new String(bytes, 0, read));
                }else {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

- Java NIO实现版：

```java
/**
 *  @ClassName: BIOServer
 *  @Description:
 *  @Author: 747591245@qq.com
 *  @Date: 2021/12/6
 */
public class BIOServer {
    public static void main(String[] args) throws IOException {
        ExecutorService service = Executors.newFixedThreadPool(10);
        ServerSocketChannel ss = ServerSocketChannel.open();
        ss.bind(new InetSocketAddress(6666));
        while (true){
            System.out.println("当前线程数: " + ((ThreadPoolExecutor) service).getActiveCount());
            final SocketChannel accept = ss.accept();
            service.execute(new Runnable() {
                @Override
                public void run() {
                    handler(accept);
                }
            });
        }
    }
    public static void handler(SocketChannel socket){
        try (socket){
            ByteBuffer bytes = ByteBuffer.allocate(1024);
            while (true){
                int read = socket.read(bytes);
                if (read != -1){
                    System.out.println(new String(bytes.array(), 0, read));
                }else {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**客户端利用telnet建立连接：**

```sh
telnet 127.0.0.1 6666
```

![image-20211206205206839](imgs\netty\8.png)

每新建一个连接，创建一个线程

大并发需要创建大量线程，消耗资源；read读不到数据会阻塞，造成系统资源的浪费

### 2、非阻塞 IO 模型（nonblocking IO）

![img](imgs\netty\3.png)

应用进程每次调用 recvfrom 即使没有数据准备好也不会阻塞，会继续往下执行，避免了进程阻塞在某个连接上的弊端

- 优点：代码编写相对简单，进程不会阻塞，可以在同一线程中处理所有连接
- 缺点：需要频繁的轮询，比较耗CPU，在并发量很大的时候将花费大量时间在没有任何数据的连接上轮询，且每次轮询所有channel，调用recv，会产生相应次数的软中断，发生系统调用，开销很大。所以该模型只在专门提供某种功能的系统中才会出现

#### 编程实现

```java
ServerSocketChannel ss = ServerSocketChannel.open();
ss.bind(new InetSocketAddress(6666));
ss.configureBlocking(false);
List<SocketChannel> channels = new LinkedList<>();
while (true){
    final SocketChannel accept = ss.accept();
    if (accept != null){
        accept.configureBlocking(false);
        channels.add(accept);
    }
    for (SocketChannel channel: channels){
        ByteBuffer bytes = ByteBuffer.allocate(1024);
        bytes.clear();
        int read = channel.read(bytes);
        if (read > 0){
            System.out.println(new String(bytes.array(), 0, read));
        }
    }
}
```

改成非阻塞模式，一个主线程就可以处理多个IO

![image-20211210014721230](imgs\netty\31.png)

设想一下，每次需要轮询所有的channel，假如有10000个连接，就需要轮询10000次，进行10000次系统调用，开销非常大，如果只有一个channel有效，则其他9999次都是浪费的。有一种方法，能够针对有效的channel进行一次recv系统调用，然后针对其他所有的9999个channel，只发起一次系统调用，总共只有两次系统调用，这，就是**多路复用**

### 3、IO 复用模型（IO multiplexing）

![img](imgs\netty\4.png)

应用进程阻塞于 select/poll/epoll 等系统函数等待某个连接变成可读（有数据过来），再调用 recvfrom 从连接上读取数据。虽然此模式也会阻塞在 select/poll/epoll 上，但与阻塞IO 模型不同它阻塞在等待多个连接上有读（写）事件的发生，明显提高了效率且增加了单线程/单进程中并行处理多连接的可能

- 优点：统一管理连接，不一定采用多线程的方式，同时也不需要轮询。只需要阻塞于 select 即可，可以同时管理多个连接
- 缺点：当 select/poll/epoll 管理的连接数过少时，这种模型将退化成阻塞 IO 模型。并且还多了一次系统调用：一次 select/poll/epoll 一次 recvfrom

### 4、信号驱动 IO 模型（signal-driven IO）

![img](imgs\netty\5.png)

应用进程创建 SIGIO 信号处理程序，此程序可处理连接上数据的读写和业务处理。并向操作系统安装此信号，进程可以往下执行。当内核数据准备好会向应用进程发送信号，触发信号处理程序的执行。再在信号处理程序中进行 recvfrom 和业务处理

- 优点：非阻塞
- 缺点：在前一个通知信号没被处理的情况下，后一个信号来了也不能被处理。所以在信号量大的时候会导致后面的信号不能被及时感知

### 5、异步 IO 模型（asynchronous IO）

![img](imgs\netty\6.png)

应用进程通过 aio_read 告知内核启动某个操作，并且在整个操作完成之后再通知应用进程，包括把数据从内核空间拷贝到用户空间。信号驱动 IO 是内核通知我们何时可以启动一个 IO 操作，而异步 IO 模型是由内核通知我们 IO 操作何时完成

> 注：前 4 种模型都是带有阻塞部分的，有的阻塞在等待数据准备好，有的阻塞在从内核空间拷贝数据到用户空间。而这种模型应用进程从调用 aio_read 到数据被拷贝到用户空间，不用任何阻塞，所以该种模式叫异步 IO 模型

- 优点：没有任何阻塞，充分利用系统内核将 IO 操作与计算逻辑并行
- 缺点：编程复杂、操作系统支持不好。目前只有 windows 下的 iocp 实现了真正的 AIO。linux 下在 2.6 版本中才引入，目前并不完善，所以 Linux 下一般采用多路复用模型

## 各 IO 模型对比

前四种模型的主要区别于第一阶段，因为他们的第二阶段都是一样的：在数据从内核拷贝到应用进程的缓冲区期间，进程阻塞于 recvfrom 调用。相反，异步 IO 模型在这两个阶段都需要处理，从而不同于其他四种模型

![img](imgs\netty\7.png)

# 操作系统中的SELECT/POLL/EPOLL

> :star2:这里有一个写得很清晰易懂的博客：[https://blog.csdn.net/wangwei19871103/article/details/104080859](https://blog.csdn.net/wangwei19871103/article/details/104080859)
>
> :star:源码级解析：[https://www.cnblogs.com/Anker/p/3265058.html](https://www.cnblogs.com/Anker/p/3265058.html)
>
> :star::star:源码级解析：[https://www.cnblogs.com/200911/p/7016843.html](https://www.cnblogs.com/200911/p/7016843.html)

设想一下，在NIO模式下，每次需要轮询所有的channel，假如有10000个连接，就需要轮询10000次，进行10000次系统调用，开销非常大，如果只有一个channel有效，其他9999次都是浪费的。能不能有一种方法，针对有效的channel进行一次recv系统调用，然后针对其他所有的9999个channel，只发起一次内核系统调用，总共只有两次系统调用，这，就是**多路复用**

内核提供select、poll、epoll实现多路复用

## select

**同步的IO多路复用器**

### 细节

`man (2) select`：（2表示系统调用，可用`man man`查看）

![image-20211210030513430](imgs\netty\32.png)

> - nfds：所有已注册的最大文件描述符 + 1
>
>   遍历小于该值的值，就能遍历到所有文件描述符。会一次性将文件描述符从用户态拷贝到内核态
>
> - 可读集合
>
>   fd_set可以看作是一个位图，内核遍历过程中发现某一socket发生了事件，假定为读事件，就根据它的文件描述符映射到readfds的某一位，将该位 置为1
>
> - 可写集合
>
> - 异常集合
>
> - 超时时间

- 允许一个程序监控多个文件描述符，select**返回文件描述符状态**，真正IO调用还需要通过程序使用recv等指令

- **如果程序自己读取IO，那么它就是同步的**

- select限制1024个文件描述符

- 用户一次性将所有文件描述符交给内核，由内核去遍历并返回状态，**只需一次系统调用**，而非NIO中，需进行nfds次系统调用（用户空间遍历若干次，若干次陷入内核 → 用户陷入一次内核，在内核内部遍历）

  > 状态的返回是通过将fd_set拷贝到用户空间

弊端：在nfds大的情况下：① 将nfds拷贝到内核态的大开销；② 内核遍历的大开销；③ 只支持1024个，太小了，若要增加，除非修改源码并重新编译内核；④ 调用select后，根据返回值 > 0能够判断发生了事件，但具体不知道是哪里发生了什么事件，必须遍历每个fd_set的每一位，确定哪一位为1，这一位的索引位置就是文件描述符，再令程序进行IO操作

### 流程简析

> 参考：
>
> - [https://www.cnblogs.com/zengzy/p/5113910.html](https://www.cnblogs.com/zengzy/p/5113910.html)
> - [https://baijiahao.baidu.com/s?id=1710440642419184726&wfr=spider&for=pc](https://baijiahao.baidu.com/s?id=1710440642419184726&wfr=spider&for=pc)
> - [https://www.cnblogs.com/tomato0906/articles/7590746.html](https://www.cnblogs.com/tomato0906/articles/7590746.html)

1. 调用select，os将select()的参数拷贝进内核，即所有的文件描述符拷贝进内核
2. 底层执行do_select，遍历所有0~nfds-1的文件描述符；对每个文件描述符，调用所属的驱动程序的poll函数，做了两件事：
   - 将当前线程挂到文件描述符的等待队列上（Linux中等待队列是和文件一一对应的）
   - 判断对应socket上是否有事件发生，并将结果标记在返回值上
3. 遍历一遍后
   - 如果某个驱动上面有事件发生，则返回，并将fd_set从内核态拷贝到用户态，供用户处理
   - 如果都没有事件，则线程进入阻塞
     - 当某个驱动上有数据到来，发生中断，中断程序会唤醒对应等待队列中的线程，重新开始do_select中的循环，如果有数据，则在fd_set中记录下结果，最后，将该线程从所有等待队列中移除，返回结果
     - 当阻塞到达超时时间后，也会重启循环
4. 用户遍历三个fd_set，根据对应位置找到文件描述符，并作相应处理

## poll

![image-20211210161825305](imgs\netty\35.png)

```c
struct pollfd {
    int fd;        /* 文件描述符 */
    short events; /* 等待的事件 */
    short revents; /* 实际发生了的事件 */
};
```

poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，且不限制1024，其他的都差不多；此外，调用select时需要自定义读、写、异常的fd_set，而poll提供了pollfd数组，每个pollfd元素包含文件描述符和要监听的事件，以及对应产生了的事件，加入时直接往pollfd数组中放就行，返回时去查看revents

> 即想要注册一个监听读事件的fd，假设为7
>
> 在select中：
>
> ```c++
> fd_set rfds;//定义一个读集合
> struct timeval tv;//定义超时结构体
> int retval;//返回值
> 
> /* 监听标准输入*/
> FD_ZERO(&rfds);//将集合清0
> FD_SET(7, &rfds);//将描述符7添加进去
> 
> /* 设置超时 */
> tv.tv_sec = 5;
> tv.tv_usec = 0;
> /* 开始监听，写事件和异常不监听 */
> retval = select(8, &rfds, NULL, NULL, &tv);
> ```
>
> 在poll中：
>
> ```c++
> struct pollfd pfds[1];
> pfds[0].fd=7;
> pfds[0].events=POLLIN;//读事件
> 
> retval = poll(pfds, 8,-1);
> ```
>
> 方便了很多

## epoll

### 细节

![image-20211210162320729](imgs\netty\36.png)

> 返回epoll句柄，创建红黑树，用于存放fd（用epitem包装）

![image-20211210112417854](imgs\netty\34.png)

> 参数分别是：epoll句柄（描述符）、操作、socket、要监听的事件

![image-20211210162418418](imgs\netty\37.png)

> 返回事件的个数
>
> epoll_wait中的events是个传出参数（是一个指针，对它的修改当然是调用者可见的，相当于Java中的引用类型的参数），事件和对应的fd等等都在这里，就很方便
>
> ![image-20211210162705232](imgs\netty\38.png)

改进：

select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生

1. 重复拷贝fd，解决方案：内核开辟空间存放fd

   > 每次调用epoll_ctl时注册fd时就拷贝进内核的某个开辟好的空间，避免了重复拷贝

2. 每调用一次select/poll，就会全部重新遍历一次，解决方案：利用回调机制，将就绪的IO操作对应的socket放入ready队列

   > 在epoll_ctl时为fd指定一个回调函数，当设备就绪（注册时的类型事件发生），就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd

![image-20211210104832226](imgs\netty\33.png)

> epoll_wait在没有事件发生，即就绪链表为空时会阻塞，不过可以设置超时时间
>
> select/poll在内核遍历时会阻塞直到事件发生，这期间内核会不断循环遍历，同样可以设置超时时间，下次调用依然要重新遍历，而epoll只需要调用epoll_wait取，是接近O(1)的

### 流程简析

> 参考：
>
> - [https://www.coonote.com/network-note/epoll-principle.html](https://www.coonote.com/network-note/epoll-principle.html)
> - :star::star: [https://zhuanlan.zhihu.com/p/361750240](https://zhuanlan.zhihu.com/p/361750240)
>
> ![img](imgs\netty\41.png)

1. 调用epoll_create，在内核中创建一个eventpoll的实例，包括rdr（红黑树头节点），wait_queue（存放调用epoll_wait的线程），rdllist（存放已就绪的epitem）
2. 每次注册一个连接，epoll_ctl将socket的ffd（fd→file实例（struct file）→socket，ffd→file实例→socket，但两者是不同的指针）和wq指针、event_poll指针等封装成epitem
   - 首先将epitem添加到对应socket的等待队列中，令socket等待队列项中的private项指向null（不指向线程，表示唤醒工作不再与socket绑定，而是全权交给epoll管理）
   - 接着将回调函数ep_poll_callback注册到内核中断程序中，并在socket对应等待队列中令func项指向它
   - 通过ep指针找到event_poll实例，再找到红黑树，判断是否已存在该socket，不存在则添加
3. 调用epoll_wait时，通过event_poll找到rdllist
   - 如果不为空，则取出epitem并处理
   - 如果为空，则加入wq中，并将自己阻塞起来
4. 当某socket有事件到来时，调用对应等待队列中的func也就是ep_poll_callback函数
   - 将等待队列中的epitem从红黑树rdr移到就绪链表rdllist
   - 根据epitem找到wq，查看是否有阻塞的线程，唤醒它

## 水平触发和边缘触发

> 参考：[https://blog.csdn.net/daaikuaichuan/article/details/83862311?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164000455316780264098548%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=164000455316780264098548&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-2-83862311.pc_search_result_cache&utm_term=epoll%E5%8E%9F%E7%90%86&spm=1018.2226.3001.4187](https://blog.csdn.net/daaikuaichuan/article/details/83862311?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164000455316780264098548%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=164000455316780264098548&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-2-83862311.pc_search_result_cache&utm_term=epoll%E5%8E%9F%E7%90%86&spm=1018.2226.3001.4187)

epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。

- LT（水平触发）模式下，只要这个文件描述符还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作；
- ET（边缘触发）模式下，在它检测到有 I/O 事件时，通过 epoll_wait 调用会得到有事件通知的文件描述符，对于每一个被通知的文件描述符，如可读，则必须将该文件描述符一直读到空，让 errno 返回 EAGAIN 为止，否则下次的 epoll_wait 不会返回余下的数据，会丢掉事件。如果ET模式不是非阻塞的，那这个一直读或一直写势必会在最后一次阻塞。

【epoll为什么要有EPOLLET触发模式？】：

  如果采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.。而采用EPOLLET这种边缘触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你！！！这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符

# Java NIO

> 注意：是Java New IO，操作系统中NIO是指Non-Blocking IO，是内核提供的SOCK_NONBLOC功能（指令`man accept`查看）
>
> ![image-20211209225356014](imgs\netty\30.png)
>
> Java中可以设置阻塞和非阻塞模式，证明不是单指Non-Blocking IO；实际上，Java NIO是多路复用，select、epoll等都是需要阻塞的
>
> ```java
> ServerSocketChannel socketChannel = ServerSocketChannel.open();
> socketChannel.bind(new InetSocketAddress(8080));
> // true - blocking;  false - non-blocking
> socketChannel.configureBlocking(true); 
> ```

## Java IO流程

> [https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd](https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd)

> 下面主要讨论的是IO就绪后的流程，BIO/NIO/多路复用主要讨论的是等待IO就绪的阶段

### 传统IO

![image-20211209004227281](imgs\netty\10.png)

**读写流程**

- 写（棕色线条）：

  用户向Java heap的Buffer对象写数据并调用相关api 

  → cpu将数据拷贝到堆外内存的DirectBuffer 

  → cpu调用JNI的pwrite0/write0方法向内核空间写（用户态切换到内核态） 

  → DMA控制器将内核缓冲区的数据拷贝到硬盘/显卡（切换回用户态）

- 读（绿色线条）：

  用户调用read相关api 

  → cpu调用到底层JNI的pread0/read方法（用户态切换到内核态） 

  → DMA将数据从硬盘/显卡拷贝到内核缓冲区，并读到DirectBuffer（内核态切换到用户态） 

  → 读进Java heap的Buffer对象，返回数据

**源码验证**

以`FileChannel.write(ByteBuffer)`的关键语句为例：

👉首先进入实例方法，`FileChannelImpl.write`

```java
public int write(ByteBuffer src) throws IOException {
    // ......（省略了其他部分）
            do {
                // 🔥关键语句
                n = IOUtil.write(fd, src, -1, direct, alignment, nd);
            } while ((n == IOStatus.INTERRUPTED) && isOpen());
            // ......
}
```

👉进入`IOUtil.write`

```java
static int write(FileDescriptor fd, ByteBuffer src, long position,
                 boolean directIO, int alignment, NativeDispatcher nd)
    throws IOException
{
    // 如果是DirctBuffer，直接调用writeFromNativeBuffer并返回
    if (src instanceof DirectBuffer) {
        return writeFromNativeBuffer(fd, src, position, directIO, alignment, nd);
    }
    // ......
    ByteBuffer bb;
    if (directIO) {
        Util.checkRemainingBufferSizeAligned(rem, alignment);
        // 🔥关键语句
        bb = Util.getTemporaryAlignedDirectBuffer(rem, alignment);
    } else {
        // 🔥关键语句
        bb = Util.getTemporaryDirectBuffer(rem);
    }
    // ......
}
```

👉进入`Util.getTemporaryDirectBuffer`或`Util.getTemporaryAlignedDirectBuffer`

```java
public static ByteBuffer getTemporaryDirectBuffer(int size) {
    if (isBufferTooLarge(size)) {
        return ByteBuffer.allocateDirect(size);
    }

    BufferCache cache = bufferCache.get();
    ByteBuffer buf = cache.get(size);
    if (buf != null) {
        return buf;
    } else {
// ......
        return ByteBuffer.allocateDirect(size);
    }
}
```

最终，都返回了一个`ByteBuffer.allocateDirect(size);`，也就是一个DirectBuffer的实例对象

👉重新进入`IOUtil.write`

我们可以知道，bb就是一个DirectBuffer的实例对象

```java
static int write(FileDescriptor fd, ByteBuffer src, long position,
                 boolean directIO, int alignment, NativeDispatcher nd)
    throws IOException
{
    // ......
    try {
        // 🔥关键语句
        bb.put(src);
        bb.flip();
        // Do not update src until we see how many bytes were written
        src.position(pos);
		// 🔥关键语句
        int n = writeFromNativeBuffer(fd, bb, position, directIO, alignment, nd);
        if (n > 0) {
            // now update src
            src.position(pos + n);
        }
        return n;
    } finally {
        Util.offerFirstTemporaryDirectBuffer(bb);
    }
}
```

调用`bb.put(src);`将原ByteBuffer里的数据写到DirectBuffer bb，验证了cpu复制那一步

👉进入`writeFromNativeBuffer`

```java
private static int writeFromNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                         long position, boolean directIO,
                                         int alignment, NativeDispatcher nd)
    throws IOException
{
    // ......
    if (position != -1) {
        // 🔥关键语句
        written = nd.pwrite(fd,
                            ((DirectBuffer)bb).address() + pos,
                            rem, position);
    } else {
        // 🔥关键语句
        written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);
    }
    if (written > 0)
        bb.position(pos + written);
    return written;
}
```

调用了`write`和`pwrite`

👉进入`write`和`pwrite`

发现调用的是`FileDispatcherImpl`的方法：

```java
int write(FileDescriptor fd, long address, int len) throws IOException {
    return write0(fd, address, len, fdAccess.getAppend(fd));
}

int pwrite(FileDescriptor fd, long address, int len, long position)
    throws IOException
{
    return pwrite0(fd, address, len, position);
}
```

点进去一看

```java
static native int write0(FileDescriptor fd, long address, int len, boolean append)
    throws IOException;

static native int pwrite0(FileDescriptor fd, long address, int len,
                         long position) throws IOException;
```

是JNI调用，发起了用户态向内核态的上下文切换，验证了流程

> PS：DMA
>
> 对于一个IO操作而言，都是通过CPU发出对应的指令来完成，但是相比CPU来说，IO的速度太慢了，CPU有大量的时间处于等待IO的状态。因此就产生了DMA（Direct Memory Access）直接内存访问技术，本质上来说他就是一块主板上独立的芯片，通过它来进行内存和IO设备的数据传输，从而减少CPU的等待时间

> PS：很多人以为DirectBuffer是内核态的缓冲区，这是错误的，DirectBuffer是由malloc()方法分配的Java堆外空间，但仍是用户空间
>
> 本文图片为了简明易懂，将DirectBuffer画到了堆外，实际上在Java中DirectBuffer对象肯定是在堆内的，是他的address属性为堆外的某个地址，一块调用 malloc() 申请到的native memory，类似于下图：
>
> ![image-20211209093203742](imgs\netty\15.png)

> 🔥为什么一定要先拷贝到DirectBuffer？直接从堆中的Buffer到内核空间不可以吗？
>
> **因为HotSpot VM里的GC除了CMS之外都是要移动对象的**，当一个Java里的 byte[] 对象的引用传给native代码，让native代码直接访问数组的内容，就必须要保证native代码在访问的时候这个 byte[] 对象不能被移动，即**这个地址上的内容不能失效**，这就与上面相悖了，内存可能因为GC整理内存而失效
>
> 有两种解决方法：
>
> 1. 暂时禁用GC
> 2. 先把 HeapByteBuffer 背后的 byte[] 的内容拷贝到一个 DirectByteBuffer 背后的native memory去，GC管不着了
>
> 于是采用了方法2，数据被拷贝到native memory之后，就将 DirectByteBuffer 背后的native memory地址传给真正做I/O的函数，保证地址不会失效了

### 直接内存

> [https://www.zhihu.com/question/57374068](https://www.zhihu.com/question/57374068)

如果是直接使用堆外内存呢？`ByteBuffer buffer = ByteBuffer.allocateDirect(x)`

![image-20211209013530875](imgs\netty\11.png)

就少了一次在Java堆内和堆外之间拷贝的过程，源码中表现为：

👉进入`IOUtil.write`

```java
static int write(FileDescriptor fd, ByteBuffer src, long position,
                 boolean directIO, int alignment, NativeDispatcher nd)
    throws IOException
{
    // 如果是DirctBuffer，直接调用writeFromNativeBuffer并返回
    if (src instanceof DirectBuffer) {
        return writeFromNativeBuffer(fd, src, position, directIO, alignment, nd);
    }
    // ......
}
```

### 零拷贝之—MMAP

> [https://www.zhihu.com/question/48161206](https://www.zhihu.com/question/48161206)

**零拷贝是什么？**

零拷贝技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域（以内核的角度看待），这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽

一般在Java中，可用MMAP实现零拷贝

**MMAP是什么？**

是一种内存映射方式，将虚拟地址的某一段与磁盘文件的某一段进行映射，造成直接操作磁盘文件的假象

```java
FileChannel fc = file.getChannel();
// 返回DirectByteBuffer对象，建立DirectByteBuffer与磁盘文件之间的映射
MappedByteBuffer map = fc.map(FileChannel.MapMode.READ_WRITE, 0, 5);
```

以读操作为例，以往的IO：

![image-20211209021458752](imgs\netty\12.png)

采用了MMAP：

![image-20211209021921471](imgs\netty\13.png)

少了一次内核copy到用户空间的过程

但实际上，还是会进入内核态的，因为一开始用户空间的虚拟内存是空的，mmap只是做了映射，没有把数据加载到内存中。在后面访问的时候，如果没有加载到内存就会产生缺页异常，陷入内核，内核会分配出对应的物理页，并把文件数据从磁盘读到物理内存中，然后把物理页与虚拟地址建立映射，这样间接映射了虚拟地址与文件，用户就可以读写操作了。流程如下：

![image-20211209022911884](imgs\netty\14.png)

1. 建立mmap映射
2. 用户进行读写操作，发现对应的虚拟地址页框是空的，产生缺页中断，陷入内核态
3. os根据mmap映射关系找到磁盘上对应文件段，读到物理内存中，并在mmu（内存管理单元）下建立虚拟内存到对应物理内存的映射
4. 读操作，直接返回结果；写操作，写物理页对应内容，然后系统调用fsync刷脏，刷回磁盘

## Java NIO原理

> 参考：[https://tech.meituan.com/2016/11/04/nio.html](https://tech.meituan.com/2016/11/04/nio.html)



## 三大核心组件

### 概述



### Buffer

#### 概述

本质上是一个可以读写数据的内存块，可以理解成一个容器对象（底层由数组存储）。Buffer是一个顶层父类，子类有ByteBuffer*（最常用）*、IntBuffer等

在旧I/O类库中（相对java.nio包）中的BufferedInputStream、BufferedOutputStream、BufferedReader和BufferedWriter在其实现中都运用了缓冲区。java.nio包公开了Buffer API，使得Java程序可以直接控制和运用缓冲区

#### 源码分析

以IntBuffer为例

- 初始化

```java
IntBuffer intBuffer = IntBuffer.allocate(5);
```

```java
public static IntBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapIntBuffer(capacity, capacity);
}
```

返回一个HeapIntBuffer，capacity为参数

```java
HeapIntBuffer(int cap, int lim) {            // package-private
    super(-1, 0, lim, cap, new int[cap], 0);
    /*
    hb = new int[cap];
    offset = 0;
    */
}
```

调用了父类的构造器

```java
IntBuffer(int mark, int pos, int lim, int cap,   // package-private
             int[] hb, int offset)
{
    super(mark, pos, lim, cap);
    this.hb = hb;
    this.offset = offset;
}
```

结合上面可知，`hb = new int[cap]`，`offset = 0`，调用了`super(mark, pos, lim, cap)`，其中：`mark=-1`，`pos=0`，`lim=最初的capacity=5`，`cap=最初的capacity=5`

```java
Buffer(int mark, int pos, int lim, int cap) {       // package-private
    if (cap < 0)
        throw new IllegalArgumentException("Negative capacity: " + cap);
    this.capacity = cap;
    limit(lim);
    position(pos);
    if (mark >= 0) {
        if (mark > pos)
            throw new IllegalArgumentException("mark > position: ("
                                               + mark + " > " + pos + ")");
        this.mark = mark;
    }
}
```

到最顶层的构造方法，分别给Buffer的四大属性：`capacity、limit、position、mark`赋值为5，5，0，-1

初始化顺序：Buffer → IntBuffer → HeapIntBuffer

- 添加

```java
public abstract IntBuffer put(int i);
```

返回一个IntBuffer，说明是链式的，一条语句可以多次put

点进实现类的方法

```java
public IntBuffer put(int x) {
    hb[ix(nextPutIndex())] = x;
    return this;
}
```

点进nextPutIndex

```java
final int nextPutIndex() {                          // package-private
    if (position >= limit)
        throw new BufferOverflowException();
    return position++;
}
```

返回的是position，并令position++，由此可见，position相当于一个游标的作用

点进ix方法

```java
protected int ix(int i) {
    return i + offset;
}
```

offset初始化时设置为0，拉下来整体看一下，大概就是这样的流程：`hb[position++]=x`

position>=limit，会报错

- 读取

以一个现象开头：

```java
IntBuffer intBuffer = IntBuffer.allocate(5);
intBuffer.put(1).put(2).put(3);
System.out.println(intBuffer.get());
```

打印0，为何？

点进get

```java
public int get() {
    return hb[ix(nextGetIndex())];
}

public int get(int i) {
    return hb[ix(checkIndex(i))];
}
```

```java
final int nextGetIndex() {                          // package-private
    if (position >= limit)
        throw new BufferUnderflowException();
    return position++;
}
```

发现从position开始读，也就是将要写的新位置，自然会读到0。读完后position++

- 读写切换

```java
intBuffer.flip();
```

```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

position写到哪儿，limit就限制到哪儿；position置位，mark置位

```java
IntBuffer intBuffer = IntBuffer.allocate(5);
intBuffer.put(1).put(2).put(3);
intBuffer.flip();
System.out.println(intBuffer.get());
```

这样便可正常从开头读

**每次flip，limit都会变成上次读/写的位置，然后从开头或设置的位置往后操作**

#### 类型化与只读

- 类型化

```java
ByteBuffer buffer = ByteBuffer.allocate(3);
// 会异常，只给buffer分配了3字节，溢出了
buffer.putInt(1);

ByteBuffer buffer = ByteBuffer.allocate(14);
buffer.putInt(1).putChar('A').putLong(8);
buffer.flip();
// 必须按顺序读取
System.out.println(buffer.getInt());
System.out.println(buffer.getChar());
System.out.println(buffer.getLong());
```

- 只读

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.putInt(1).putChar('A').putLong(8);
buffer.flip();
// 只读
// 返回HeapByteBufferR的实例，做写操作会抛出异常
buffer = buffer.asReadOnlyBuffer();
buffer.putInt(1);
// Exception in thread "main" java.nio.ReadOnlyBufferException
// at java.base/java.nio.HeapByteBufferR.putInt(HeapByteBufferR.java:448)
// at Demo.main(Demo.java:20)
```

#### 堆外内存

可直接对堆外内存操作，减少了一次堆外内存和堆内内存之间拷贝的过程



```java
try(RandomAccessFile file = new RandomAccessFile("1.txt", "rw");
    FileChannel fc = file.getChannel();){
    // 以读写模式
    // 从0位置开始将文件后面5个字节大小映射到堆外内存(不是索引位置为5)
    // 返回DirectByteBuffer对象
    MappedByteBuffer map = fc.map(FileChannel.MapMode.READ_WRITE, 0, 5);
    map.put(3, (byte) 'X');
} catch (IOException e) {
    e.printStackTrace();
}
```

#### 分散和聚集

- 分散（Scattering）



- 聚集（Gathering）



### Channel

#### 概述

![img](imgs\netty\9.png)

- 双向的（不同于流），可以读也可以写
- 是一个接口

常用：

- FileChannel：文件读写
- DatagramChannel：UDP读写
- ServerSocketChannel、SocketChannel：TCP读写

客户端连接服务器时，服务器端给客户端分配一个ServerSocketChannel，后续与该客户端的通信都基于这个Channel来进行

#### FileChannel

文件读写操作

常用方法：

```java
public int read(ByteBuffer dst);
public int write(ByteBuffer src);
public long transferFrom(ReadableByteChannel src, long position, long count);
public long transferTo(long position, long count, WritableByteChannel dst);
```

> 不同于InputStream/Reader只能读，OutputStream/Writer只能写

- 读写

实现1.txt的内容拷贝到2.txt

```java
public static void main(String[] args) throws IOException {
    File file = new File("1.txt");
    File file2 = new File("2.txt");
    ByteBuffer buffer = ByteBuffer.allocate(5);
    try (FileChannel readChannel = FileChannel.open(file.toPath(), StandardOpenOption.READ);
         FileChannel writeChannel = FileChannel.open(file2.toPath(), StandardOpenOption.WRITE)){
        while (true){
            // 缓存读完后需要clear，否则会一直read 0，position位置不变
            buffer.clear();
            int read = readChannel.read(buffer);
            if (read == -1){
                break;
            }
            // 注意Java内码和外码的区别，内码UTF-16，字符占2字节；外码比如UTF-8存储中英文是不等长的
            System.out.print(new String(buffer.array(), "UTF-8"));

            //读写切换
            buffer.flip();
            writeChannel.write(buffer);
        }
    }
}
```

- 通道拷贝

```java
public static void main(String[] args) throws IOException {
    File file = new File("1.txt");
    File file2 = new File("2.txt");
    try (FileChannel readChannel = FileChannel.open(file.toPath(), StandardOpenOption.READ);
         FileChannel writeChannel = FileChannel.open(file2.toPath(), StandardOpenOption.WRITE)){
        writeChannel.transferFrom(readChannel, 0, readChannel.size());
    }
}
```

#### SocketChannel

服务端：ServerSocketChannel；客户端：SocketChannel

可以看作是Socket的再一层抽象，在Socket的基础上封装了一些其他的内容

### Selector

**多路复用器**

Linux下默认为epoll，不过会判断内核版本，低版本为select或epoll；基本所有操作系统都支持select

> windows下的AIO，即IOCP没有支持，因为考虑到Java程序大多跑在Linux服务器上

#### 概述

- 一个线程，处理多个连接
- 多个Channel以事件的方式可以注册到同一个Selector
- 能够检测多个注册的通道上是否有事件发生，如果有，便获取对应事件并处理
- 不必为每个连接都创建一个线程，减少了多线程上下文切换导致的开销

#### 关键代码

```java
select(); // 阻塞调用select/poll/epoll
select(long timeout); // 阻塞直到超时
selectNow(); // 有无事件产生都直接返回
```

```java
// 返回所有向多路复用器注册的SelectionKey
keys(); 
// 返回所有产生事件对应的SelectionKey
selectedKey(); 
```

> SelectionKey：
>
> ```java
> public abstract class SelectionKey {
>     protected SelectionKey() { }
>     // 返回channel，理解成socket，或fd
>     public abstract SelectableChannel channel();
>     // 返回多路复用器
>     public abstract Selector selector();
>     // 取消一个selectionKey并添加到cancelled-key set 中
>     // 所关联的channel并没有立即被撤销注册
>     // 直到发生下次 select, 这些channel才被从selector中撤销登记
>     public abstract void cancel();
>     // 返回要监听的事件类型
>     public abstract int interestOps();
>     // 设置要监听的事件类型
>     public abstract SelectionKey interestOps(int ops);
>     // 或操作
>     public int interestOpsOr(int ops) {
>         synchronized (this) {
>             int oldVal = interestOps();
>             interestOps(oldVal | ops);
>             return oldVal;
>         }
>     }
>     // 与操作
>     public int interestOpsAnd(int ops) {
>         synchronized (this) {
>             int oldVal = interestOps();
>             interestOps(oldVal & ops);
>             return oldVal;
>         }
>     }
> 	// 就绪的事件的类型
>     public abstract int readyOps();
> 
>     public static final int OP_READ = 1 << 0;
>     public static final int OP_WRITE = 1 << 2;
>     public static final int OP_CONNECT = 1 << 3;
>     public static final int OP_ACCEPT = 1 << 4;
>     // 读事件是否发生
>     public final boolean isReadable() {
>         return (readyOps() & OP_READ) != 0;
>     }
> 	// 写事件是否发生
>     public final boolean isWritable() {
>         return (readyOps() & OP_WRITE) != 0;
>     }
> 	// 连接事件是否发生
>     public final boolean isConnectable() {
>         return (readyOps() & OP_CONNECT) != 0;
>     }
> 	// accept是否发生
>     public final boolean isAcceptable() {
>         return (readyOps() & OP_ACCEPT) != 0;
>     }
> 
>     // -- Attachments --
> 
>     private volatile Object attachment;
>     private static final AtomicReferenceFieldUpdater<SelectionKey,Object>
>         attachmentUpdater = AtomicReferenceFieldUpdater.newUpdater(
>             SelectionKey.class, Object.class, "attachment"
>         );
>     public final Object attach(Object ob) {
>         return attachmentUpdater.getAndSet(this, ob);
>     }
>     public final Object attachment() {
>         return attachment;
>     }
> 
> }
> ```
>
> 实例SelectionKeyImpl：
>
> ```java
> public final class SelectionKeyImpl
>     extends AbstractSelectionKey
> {
>     private static final VarHandle INTERESTOPS =
>             ConstantBootstraps.fieldVarHandle(
>                     MethodHandles.lookup(),
>                     "interestOps",
>                     VarHandle.class,
>                     SelectionKeyImpl.class, int.class);
> 
>     // 连接句柄
>     private final SelChImpl channel;
>     private final SelectorImpl selector;
> 
>     // 监听的事件
>     private volatile int interestOps;
>     // 返回的事件，看到这里，感觉跟pollfd或者epoll_event的结构挺像的
>     private volatile int readyOps;
> 
>     // registered events in kernel, used by some Selector implementations
>     private int registeredEvents;
> 
>     // index of key in pollfd array, used by some Selector implementations
>     private int index;
>     //......
> }
> ```
>
> 可以简单在抽象层面对标一下：
>
> - 对标select中的fd_set和对应位置置为1的文件描述符（已产生的事件+fd）
> - poll中的revents不为空的pollfd
> - 以及epoll就绪链表中的epoll_event（感觉这个最贴切）

## Netty思想

### Java多路复用简单实现

```java
public class Demo {
    public static void main(String[] args) throws IOException {
        // 创建多路复用器
        Selector selector = Selector.open();
        // 将服务端的ServerSocket注册到多路复用器，并监听accept事件
        ServerSocketChannel ss = ServerSocketChannel.open();
        ss.bind(new InetSocketAddress(9090));
        // 设置为非阻塞
        ss.configureBlocking(false);
        // 注册
        ss.register(selector, SelectionKey.OP_ACCEPT);
        while (true){
            // 阻塞，如果监听到有事件发生，进入if循环
            if (selector.select() > 0){
                // 遍历就绪链表
                for (SelectionKey k: selector.selectedKeys()){
                    // 发现accept事件状态显示就绪
                    if (k.isAcceptable()){
                        // 拿到注册为accept的ServerSocket句柄
                        ServerSocketChannel readySs = (ServerSocketChannel) k.channel();
                        // 开始监听，这里是一开始就设置了非阻塞的
                        SocketChannel accept = readySs.accept();
                        // 监听到客户端连接
                        if (accept != null){
                            System.out.println("------------------------------");
                            System.out.println("TYPE: ACCEPT");
                            System.out.println("客户端地址为: " + accept.getRemoteAddress());
                            // 客户端channel设置为非阻塞
                            accept.configureBlocking(false);
                            // 将客户端channel注册到多路复用器，并监听读状态
                            accept.register(selector, SelectionKey.OP_READ);
                        }
                    }else if (k.isReadable()){
                        // 客户端channel读事件产生
                        SocketChannel readyCs = (SocketChannel) k.channel();
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        int read = readyCs.read(buffer);
                        if (read > 0){
                            System.out.println("------------------------------");
                            System.out.println("TYPE: READ, 输入来源为: " + readyCs.getRemoteAddress());
                            // 注意直接内存不支持array()方法
                            System.out.println(new String(buffer.array(), 0, read));
                        }
                    }
                    selector.selectedKeys().clear();
                }
            }
        }
    }
}
```

![image-20211211031528967](imgs\netty\39.png)

注意的点：

- 需要写成非阻塞的，否则会抛出异常

  > why？
  >
  > 每次通过 `read` 系统调用读取数据时，最多只能读取缓冲区大小的字节数；如果某个文件描述符一次性收到的数据超过了缓冲区的大小，那么需要对其 `read` 多次才能全部读取完毕
  >
  > - **`select`** **可以使用阻塞 I/O**。通过 `select` 获取到所有可读的文件描述符后，遍历每个文件描述符，`read` **一次**数据
  >   - 这些文件描述符都是可读的，因此即使 `read` 是阻塞 I/O，也一定可以读到数据，不会一直阻塞下去
  >   - `select` 采用水平触发模式，因此如果第一次 `read` 没有读取完全部数据，那么下次调用 `select` 时依然会返回这个文件描述符，可以再次 `read`
  >   - **`select`** **也可以使用非阻塞 I/O**。当遍历某个可读文件描述符时，使用 `for` 循环调用 `read` **多次**，直到读取完所有数据为止（返回 `EWOULDBLOCK`）。这样做会多一次 `read` 调用，但可以减少调用 `select` 的次数
  >
  > - 在 `epoll` 的边缘触发模式下，只会在文件描述符的可读/可写状态发生切换时，才会收到操作系统的通知
  >   - 因此，如果使用 `epoll` 的**边缘触发模式**，在收到通知时，**必须使用非阻塞 I/O，并且必须循环调用** `read` **或** `write` **多次，直到返回** `EWOULDBLOCK` **为止**，然后再调用 `epoll_wait` 等待操作系统的下一次通知
  >   - 如果没有一次性读/写完所有数据，那么在操作系统看来这个文件描述符的状态没有发生改变，将不会再发起通知，调用 `epoll_wait` 会使得该文件描述符一直等待下去，服务端也会一直等待客户端的响应，业务流程无法走完
  >   - 这样做的好处是每次调用 `epoll_wait` 都是**有效**的——保证数据全部读写完毕了，等待下次通知。在水平触发模式下，如果调用 `epoll_wait` 时数据没有读/写完毕，会直接返回，再次通知。因此边缘触发能显著减少事件被触发的次数
  >
  >   为什么 `epoll` 的**边缘触发模式不能使用阻塞 I/O**？很显然，边缘触发模式需要循环读/写一个文件描述符的所有数据。如果使用阻塞 I/O，那么一定会在最后一次调用（没有数据可读/写）时阻塞，导致无法正常结束

- 每次读取selectedKeys并IO操作后，需要clear

  > why？
  >
  > selector无法自己移除selectedKey，若没有移除的话，下次遍历还会遍历到对应的selectedKey，取出channel，进行IO操作，而此时channel是没有就绪的，则程序就会出现异常
  >
  > 以上面的例子为例，如果第一次accept之后没有清除，首先轮询到刚刚添加的read，如果可read，则read出来；如果还不可，进入下一次循环，阻塞在select。接着，另一个连接开启，进入循环，然而由于上一个accept的key没有清除，于是遍历到上一个accept，监听已经建立的连接的客户端channel，所以导致后面一系列混乱

考虑出现这样的情况：某次发生一个读事件，如果内容很多，又是在while循环里面读取的话，就会在卡在循环里面直到读取完成，如果读取过程中有accept事件发生，则这个客户端也得等，更别说客户端传来的数据读取了

思考这样的解决：开启多个selector，一个selector卡在读取那儿时，另一个可以工作。进一步地，令一个selector完全用于监听accept事件，另外的selector，处理其他类型的事件，下面来简单实现一下

### Java多selector实现

> 设计成主线程的selector完全用于监听accpet，其他用于处理其他事件

主线程做accpect工作，每次监听到一个客户端，将客户端的读取事件注册到selector1和selector2其中随机一个

```java
public class Demo {
    static volatile Selector selectorAccept = null;
    static volatile Selector selectorRead1 = null;
    static volatile Selector selectorRead2 = null;
    static volatile Selector[] selectorReads = null;
    public static void main(String[] args) throws IOException {
        selectorAccept = Selector.open();
        selectorRead1 = Selector.open();
        selectorRead2 = Selector.open();
        selectorReads = new Selector[]{selectorRead1, selectorRead2};
        ServerSocketChannel ss = ServerSocketChannel.open();
        ss.bind(new InetSocketAddress(9090));
        ss.configureBlocking(false);
        ss.register(selectorAccept, SelectionKey.OP_ACCEPT);
        new Thread(new Task(selectorRead1), "thread-A").start();
        new Thread(new Task(selectorRead2), "thread-B").start();
        // 开始监听accept
        while (true) {
            synchronized (selectorAccept) {
                if (selectorAccept.select() > 0) {
                    for (SelectionKey sk : selectorAccept.selectedKeys()) {
                        if (sk.isAcceptable()) {
                            ServerSocketChannel ssc = (ServerSocketChannel) sk.channel();
                            SocketChannel accept = ssc.accept();
                            if (accept != null) {
                                System.out.println("----------ACCEPT----------");
                                System.out.println("线程: " + Thread.currentThread().getName());
                                System.out.println("新的客户端来了: " + accept.getRemoteAddress());
                                accept.configureBlocking(false);
                                int random = new Random().nextInt(2);
                                accept.register(selectorReads[random], SelectionKey.OP_READ);
                            }
                        }
                    }
                    selectorAccept.selectedKeys().clear();
                }
            }
        }
    }
}
class Task implements Runnable{
    private final Selector selector;

    public Task(Selector selector) {
        this.selector = selector;
    }

    @Override
    public void run() {
        synchronized (selector){
            while (true){
                try {
                    if (selector.select(200) > 0) {
                        handler(selector);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    static void handler(Selector selectorRead) throws IOException {
        for (SelectionKey sk: selectorRead.selectedKeys()){
            SocketChannel readChannel = (SocketChannel) sk.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1);
            int read;
            System.out.println("----------READ----------");
            System.out.println("源: " + readChannel.getRemoteAddress());
            System.out.println("处理线程: " + Thread.currentThread().getName());
            while (true){
                buffer.clear();
                if ((read = readChannel.read(buffer)) == 0){
                    break;
                }
                if (read < 0){
                    // 连接断开，取消注册
                    sk.cancel();
                    System.out.print("连接断开: " + readChannel.getRemoteAddress());
                    break;
                }
                System.out.print(new String(buffer.array(), 0, read, "UTF-8"));
            }
            System.out.println();
        }
        selectorRead.selectedKeys().clear();
    }
}
```

> :boom:!!!**注意**：
>
> 1. register一定要在没有阻塞在select时调用，否则会失效。可以使用一个带超时的select
> 2. 如果客户端强制断开，注册在 selector 中的 channel 会不断触发 read 事件，此时调用 read() 方法去读数据会触发 IOException 异常，因此需要通过 try catch 捕捉处理，避免服务器因为异常而被强制停止。需要调用 cancel() 方法进行注销，从 selector 中注销触发异常 SelectionKey 所对应的 channel

正常运行：

![image-20211213105607078](imgs\netty\40.png)

# 参考

- [服务器网络编程之 IO 模型 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903812738596878)
- [https://tech.meituan.com/2016/11/04/nio.html](https://tech.meituan.com/2016/11/04/nio.html)
- [https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd](https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd)
- [https://www.zhihu.com/question/48161206](https://www.zhihu.com/question/48161206)
- [https://www.zhihu.com/question/57374068](https://www.zhihu.com/question/57374068)
- [https://www.cnblogs.com/baxianhua/p/9285102.html](https://www.cnblogs.com/baxianhua/p/9285102.html)
- [https://blog.csdn.net/Stephen___Qin/article/details/120466415](https://blog.csdn.net/Stephen___Qin/article/details/120466415)
- [https://blog.csdn.net/wangwei19871103/article/details/104080859](https://blog.csdn.net/wangwei19871103/article/details/104080859)
- [https://www.cnblogs.com/Anker/p/3265058.html](https://www.cnblogs.com/Anker/p/3265058.html)
- :star:[马士兵Netty](https://www.bilibili.com/video/BV1Af4y117ZK?p=1&spm_id_from=pageDriver)
