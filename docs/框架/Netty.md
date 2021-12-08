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

## Java IO流程

> [https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd](https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd)

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



# 参考

- [服务器网络编程之 IO 模型 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903812738596878)
- [https://tech.meituan.com/2016/11/04/nio.html](https://tech.meituan.com/2016/11/04/nio.html)
- [https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd](https://mp.weixin.qq.com/s?__biz=MzkzNTEwOTAxMA==&mid=2247491660&idx=1&sn=a7d79ec4cc3f40e7b9a9018436a7377a&chksm=c2b1a8b1f5c621a7268ca298598a15c4ac575790628651e5651925b5efd96ebc0046796ef5b1&token=570732653&lang=zh_CN#rd)
- [https://www.zhihu.com/question/48161206](https://www.zhihu.com/question/48161206)

