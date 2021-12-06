# [返回](/)

# Netty

异步、基于事件驱动的网络应用框架

应用场景：RPC框架等

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

**客户端利用telnet建立连接：**

```sh
telnet 127.0.0.1 6666
```

![image-20211206205206839](imgs\netty\8.png)

大并发需要创建大量线程，消耗资源；read读不到数据会阻塞，造成系统资源的浪费

### 2、非阻塞 IO 模型（nonblocking IO）

![img](imgs\netty\3.png)

应用进程每次调用 recvfrom 即使没有数据准备好也不会阻塞，会继续往下执行，避免了进程阻塞在某个连接上的弊端

- 优点：代码编写相对简单，进程不会阻塞，可以在同一线程中处理所有连接
- 缺点：需要频繁的轮询，比较耗CPU，在并发量很大的时候将花费大量时间在没有任何数据的连接上轮询。所以该模型只在专门提供某种功能的系统中才会出现

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

# Java NIO

## Java NIO原理

> 参考：[https://tech.meituan.com/2016/11/04/nio.html](https://tech.meituan.com/2016/11/04/nio.html)



# 参考

- [服务器网络编程之 IO 模型 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903812738596878)
- [https://tech.meituan.com/2016/11/04/nio.html](https://tech.meituan.com/2016/11/04/nio.html)

