# [返回](/)

# JAVA IO

## IO流

### IO流分类

### 节点流

- 按数据单位：字节流（8bit）；字符流（16bit）
- 按流向：输入流；输出流
- 按角色：节点流（直接处理数据的流）；处理流（修饰/增强已有流的流）

![image-20210620175927750](imgs\JAVA IO\1.png)

四个基类，都是抽象类

![image-20210620224104082](imgs\JAVA IO\2.png)

> 垃圾回收机制只针对堆空间，对数据库连接、输入输出流、socket等无能为力，需要手动关闭

#### FileReader

对于文本test.txt，内容是”helloworld“

- 单个字符读取，返回字符的ASCII码，-1表示到文件末尾

```java
public static void main(String[] args)  {
    FileReader fr = null;
    int data;
    try{
        fr = new FileReader("test.txt");
        //  查看源码，这里会自动new一个File类的对象，new File("test.txt");
        while((data = fr.read()) != -1){
            System.out.print(data);
        }
    }catch (IOException e){
        e.printStackTrace();
    }finally {
        if(fr != null) {
            try {
                fr.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

> effective java推荐，try-with-resource写法

```java
public static void main(String[] args)  {
    int data;
    try(FileReader fr = new FileReader("test.txt")){
        while((data = fr.read()) != -1){
            System.out.print((char) data);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

- char数组读取

```java
public static void main(String[] args)  {
    char[] buf = new char[3];
    int len;
    StringBuilder sb = new StringBuilder();
    try(FileReader fr = new FileReader("test.txt")){
        while((len = fr.read(buf)) != -1){
            //  每次读取3个字符并返回读取的长度
            //  System.out.print(buf);  、
            //  上述这种方法会输出”helloworldrl“，因为最后一次只会读取一个字符，char数组并不会被清空
            sb.append(buf, 0, len);
        }
        System.out.println(sb);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

#### FileWriter

```java
public static void main(String[] args)  {
    char[] buf = new char[3];
    int len;
    StringBuilder sb = new StringBuilder();
    try(FileReader fr = new FileReader("test.txt");
        FileWriter fw = new FileWriter("test2.txt")){
        while((len = fr.read(buf)) != -1){
            sb.append(buf, 0, len);
            fw.write(buf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

```java
FileWriter fw = new FileWriter("test2.txt", false)
```

默认append参数为false，会覆盖之前的文件；为true，则会在之前的基础上添加

> 注意：字符流是无法处理图片等信息的

#### FileInputStream & FileOutputStream

```java
public static void main(String[] args)  {
    byte[] buf = new byte[3];
    int len;
    try(FileInputStream fr = new FileInputStream("1.jpg");
        FileOutputStream fw = new FileOutputStream("2.jpg")){
        while((len = fr.read(buf)) != -1){
            fw.write(buf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

> 注意，存中文的时候，由于是两个字节的，读取的时候会把中文分开，此时依此打印到控制台会显示乱码

### 处理流

#### 缓冲流

> 提升读写速度，包在节点流之外，属于处理流

```java
public static void main(String[] args)  {
    byte[] buf = new byte[1024];
    int len;
    try(FileInputStream fr = new FileInputStream("1.jpg");
        FileOutputStream fw = new FileOutputStream("2.jpg");
        BufferedInputStream bi = new BufferedInputStream(fr);
        BufferedOutputStream bo = new BufferedOutputStream(fw)
        ){
        while((len = bi.read(buf)) != -1){
            bo.write(buf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
```

> 内部提供了缓冲区，先读到缓冲区，读满再一次性写出

#### 转换流

- InputStreamReader：字节流→字符流

```java
InputStreamReader ir = new InputStreamReader(new FileInputStream("test.txt"), "UTF-8");
//  第二个参数不写的话，默认是系统的编码
```

- OutputStreamWriter：字符流→字节流

```java
OutputStreamWriter ow = new OutputStreamWriter(new FileOutputStream("test.txt");
```

> 字符集：
>
> - ANSI：英语操作系统默认为ISO8859-1；中文默认为GBK
> - Unicode是一种编码方式，具体实现是UTF-8/UTF-16

#### 标准输入/输出流

- static PrintStream err;
- static InputStream in;     // 实际实现是BufferedInputStream，read方法，阻塞读
- static PrintStream out;

> 可以通过System.setIn/setOut重新指定

练习：控制台输入字符串，将其转换成大写；如果输入exit，程序退出

1. Scanner

```java
public static void main(String[] args) {
    Scanner s = new Scanner(System.in);
    while(true){
        System.out.println("请输入字符串");
        String data = s.nextLine();
        if("exit".equals(data)){
            return;
        }
        System.out.println(data.toUpperCase());
    }
}
```

2. 直接使用System.in

```java
public static void main(String[] args) {
    try(BufferedReader r = new BufferedReader(new InputStreamReader(System.in))){
        while(true){
            System.out.println("请输入字符串");
            String data = r.readLine();
            if("exit".equals(data)){
                return;
            }
            System.out.println(data.toUpperCase());
        }
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

![image-20210621165141801](imgs\JAVA IO\3.png)

#### 数据流

> 用于读取/写入String和基本数据类型

```java
public static void main(String[] args) {
    try(FileOutputStream fo = new FileOutputStream("test.txt");
        FileInputStream fi = new FileInputStream("test.txt");
        DataOutputStream dos = new DataOutputStream(fo);
        DataInputStream dis = new DataInputStream(fi)
    ){
        dos.writeUTF("啦啦啦");
        dos.writeInt(123);
        dos.writeBoolean(true);
        System.out.println(dis.readUTF());
        System.out.println(dis.readInt());
        System.out.println(dis.readBoolean());
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

#### 对象流

> 用于读取/写入对象和基本数据类型

> 对象必须可序列化；内部属性也必须可序列化，或者声明为transient或static*（这两个是不能被序列化的，其中transient是特意声明为不序列化，static属于类不属于对象，因此不被序列化）*

```java
public class Test {
    public static void main(String[] args) {
        final String fileName = "test.txt";
        Student s = new Student(2020111, "张三");
        write(fileName, s);
        Student result = read(fileName);
        System.out.println(result);
    }
    static <T> T read(String fileName){
        try(ObjectInputStream ois = new ObjectInputStream(new FileInputStream(fileName))){
            return (T) ois.readObject();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
        throw new RuntimeException("出现异常");
    }
    static <T> void write(String fileName, T t){
        try(ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(fileName))){
            oos.writeObject(t);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
//  注意，不要将同一个文件同时作为输入流和输出流，会报EOFException
class Student implements Serializable{
    
    private static final long serialVersionUID = 121231232342342L;
    
    private Integer id;
    private String name;

    public Student(Integer id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```

> - JVM会在反序列化时将字节流中serialVersionUID与本地的实体类进行比较，如果不同，会出现反序列化版本不一致异常
> - 不显式定义的话，JVM会根据类的细节自动生成一个serialVersionUID，但如果实例发生改变可能导致serialVersionUID改变，反序列化不成功，因此**建议显式定义一个serialVersionUID**
>
> 例子：不加serialVersionUID，将对象写入文件，给对象的类增添一个属性，再从文件读取的时候，就会异常
>
> *java.io.InvalidClassException: Student; local class incompatible: stream classdesc serialVersionUID = -7252840388576400267, local class serialVersionUID = -2057941294771696647
> 	at java.base/java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:689)
> 	at java.base/java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1958)
> 	at java.base/java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1827)
> 	at java.base/java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2115)
> 	at java.base/java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1646)
> 	at java.base/java.io.ObjectInputStream.readObject(ObjectInputStream.java:464)
> 	at java.base/java.io.ObjectInputStream.readObject(ObjectInputStream.java:422)
> 	at Test.read(Test.java:16)
> 	at Test.main(Test.java:11)*
>
> 可以看到，是serialVersionUID版本不一致

#### 随机存取文件流

> - 任意位置读写
> - 同时继承DataInput/Output接口，可读可写

mode参数：

- r：只读
- rw：读写（写中途异常，已写的不会保存下来）
- rwd：读写并同步文件更新（写中途异常，已写的会保存下来）
- rws：读写并同步文件和元数据的更新

```java
//  test.txt和test2.txt都有一个字符串"hello world"
public static void main(String[] args) {
    try(RandomAccessFile raf1 = new RandomAccessFile("test.txt", "r");
    RandomAccessFile raf2 = new RandomAccessFile("test2.txt", "rw")){
        String str = raf1.readLine();  //  hello world
        System.out.println(str);
        //  raf2.seek(str.length());  //  seek方法，可以定位指针，就可以从指定位置开始覆盖了
        raf2.write("xyz".getBytes());  //  默认会从文件开头开始覆盖，变成"xyzlo world"
    }catch (IOException e){
        e.printStackTrace();
    }
}
```

## 网络编程

### 网络通信要素

- 地址：IP；端口号

> 公有IP；私有IP（192.168.0.0~192.168.255.255）

- 规则：OSI模型；TCP/IP协议

![image-20210622093421784](imgs\JAVA IO\4.png)

### TCP网络编程

```java
public class Test {
    @org.junit.Test
    public void client() {
        InetAddress inetAddress = null;
        try {
            inetAddress = InetAddress.getByName("127.0.0.1");
        }catch (UnknownHostException e){
            e.printStackTrace();
        }
        try(Socket socket = new Socket(inetAddress, 9999);
            OutputStream os = socket.getOutputStream();
            BufferedOutputStream bo = new BufferedOutputStream(os)) {
            bo.write("你好，我是客户端".getBytes());
        }catch (IOException e){
            e.printStackTrace();
        }
    }
    @org.junit.Test
    public void server(){
        try(ServerSocket socket = new ServerSocket(9999);
            InputStream is = socket.accept().getInputStream();
            BufferedReader br = new BufferedReader(new InputStreamReader(is))){
            char[] buf = new char[10];
            int len;
            StringBuilder sb = new StringBuilder();
            while((len = br.read(buf)) != -1){
                sb.append(buf, 0, len);
            }
            System.out.println(sb);
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

> socket.shutdownOutput/Input可以结束输出/输入，令read结束阻塞

### UDP网络编程

```java
public class Test {
    public static void main(String[] args) throws IOException {
        new Thread(()->send()).start();
        new Thread(()->receive()).start();
    }
    public static void send(){
        try(DatagramSocket socket = new DatagramSocket();
            FileInputStream fis = new FileInputStream("test.txt")){
            byte[] buf = new byte[3];
            int len;
            String str = "";
            while((len = fis.read(buf)) != -1){
                str += new String(buf, 0, len);
            }
            DatagramPacket packet = new DatagramPacket(str.getBytes(), 0, str.length(), InetAddress.getByName("127.0.0.1"), 8888);
            socket.send(packet);
        }catch (IOException e){
            e.printStackTrace();
        }
    }
    public static void receive(){
        try(DatagramSocket socket = new DatagramSocket(8888, InetAddress.getByName("127.0.0.1"));){
            byte[] buf = new byte[10];
            DatagramPacket packet = new DatagramPacket(buf, 0, 10);
            socket.receive(packet);
            byte[] result = packet.getData();
            for(byte b: result){
                System.out.print((char) b);
            }
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

### URL

> 利用URL来下载资源

```java
public class Test {
    public static void main(String[] args) throws IOException {
        final String URL = "http://www.xinhuanet.com/photo/2021-06/22/1127586141_16243281572321n.jpg";
        URL url = new URL(URL);
        URLConnection connection = url.openConnection();
        connection.connect();
        try(BufferedInputStream bis = new BufferedInputStream(connection.getInputStream());
            FileOutputStream fos = new FileOutputStream("download.jpg")
        ) {
            byte[] buf = new byte[1024];
            int len;
            while((len = bis.read(buf)) != -1){
                fos.write(buf, 0, len);
            }
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

