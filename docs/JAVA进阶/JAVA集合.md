# [返回](/)

# JAVA集合

## 体系概述

- Collection：单列数据
  - List：有序、可重复
  - Set：无序、不可重复
- Map：双列数据，如Key-Value对这种形式

## List

### ArrayList

- ArrayList
- LinkedList
- Vector

#### JDK 7.0 源码分析

**底层实现**

```java
private transient Object[] elementData;
```

> 为什么要transient？
>
> ```java
> private void writeObject(java.io.ObjectOutputStream s)
>     throws java.io.IOException{
>     // Write out element count, and any hidden stuff
>     int expectedModCount = modCount;
>     s.defaultWriteObject();
> 
>     // Write out size as capacity for behavioural compatibility with clone()
>     s.writeInt(size);
> 
>     // Write out all elements in the proper order.
>     for (int i=0; i<size; i++) {
>         s.writeObject(elementData[i]);
>     }
> 
>     if (modCount != expectedModCount) {
>         throw new ConcurrentModificationException();
>     }
> }
> ```
>
> 序列化时，首先调用默认的writeObject，接着遍历数组，只序列化存在元素的部分，这样可以加快序列化且减小序列化后体积

**初始化**

- ArrayList()：无参构造器底层创建长度为10的Object类型数组，this(10);

  ![image-20210609111505538](imgs\JAVA集合\5.png)

**添加**

![image-20210609105846591](imgs\JAVA集合\1.png)

- ensureCapacityInternal

  ![image-20210609105940238](imgs\JAVA集合\2.png)

  需要的容量大于数组长度，调用grow方法扩容

- grow

  ![image-20210609110101131](imgs\JAVA集合\3.png)

  1. 旧容量 x 1.5
  2. 若还是小于需要的容量，则直接等于需要的容量
  3. 如果大于MAX_ARRAY_SIZE，调用hugeCapacity。超过Integer.MAX_VALUE，抛出OOM

  ![image-20210609110853375](imgs\JAVA集合\4.png)

#### JDK 8.0 源码分析

**底层实现**

```java
transient Object[] elementData; // non-private to simplify nested class access
```

去掉了private，简化内部类的访问

**初始化**

- 无参

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

- 带参

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

private static final Object[] EMPTY_ELEMENTDATA = {};    
```

**添加**

![image-20210609105846591](imgs\JAVA集合\1.png)

- ensureCapacityInternal

```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

​		第一次添加，则取minCapacity和DEFAULT_CAPACITY*(=10)*中的最大值

- ensureExplicitCapacity

![image-20210609112644288](imgs\JAVA集合\6.png)

-  grow

  需要的容量大于数组长度，调用grow方法扩容

![image-20210609113329410](imgs\JAVA集合\7.png)

​		第一次调用add方法时才生成数组，通过copyOf方法，第一次长度为10，即Math.max(DEFAULT_CAPACITY, minCapacity)

> 后续的添加和扩容与JDK 7 无异

#### Java中ArrayList最大容量为什么是Integer.MAX_VALUE-8?

[链接](https://www.zhihu.com/question/27999759)

### LinkedList

#### JDK 8.0 源码分析

**底层实现**

![image-20210610094734143](imgs\JAVA集合\8.png)

用双向链表实现，first头节点，last尾节点

**初始化**

![image-20210610095453134](imgs\JAVA集合\10.png)

**添加**

![image-20210610095539936](imgs\JAVA集合\11.png)

调用LinkLast

![image-20210610095012259](imgs\JAVA集合\9.png)

l指向last，newNode的prev引用指向l，next指向null；令last指向newNode，l就指向上一个last

- 上一个last为null，说明链表只有当前这一个元素，令first=last=newNode
- 上一个last不为null，令上一个last指向newNode

### Vector

#### 源码分析

**底层实现**

```java
protected Object[] elementData;
```

**初始化**

```java
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}

/**
 * Constructs an empty vector with the specified initial capacity and
 * with its capacity increment equal to zero.
 *
 * @param   initialCapacity   the initial capacity of the vector
 * @throws IllegalArgumentException if the specified initial capacity
 *         is negative
 */
public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}

/**
 * Constructs an empty vector so that its internal data array
 * has size {@code 10} and its standard capacity increment is
 * zero.
 */
public Vector() {
    this(10);
}
```

默认capacityIncrement为0，可以在初始化时手动指定，意义是设置扩容时增加的长度

**添加**

![image-20210610103112325](imgs\JAVA集合\12.png)

elementData：底层数组；elementCount：元素数目

![image-20210610103345527](imgs\JAVA集合\13.png)

如果数组已经装满，用grow方法扩容并获得新数组，新数组怎么得到的？点进grow

![image-20210610103527217](imgs\JAVA集合\14.png)

minCapacity=原数目+1

![image-20210610103624533](imgs\JAVA集合\15.png)

如果初始化时没设置capacityIncrement，则为旧容量 x 2；否则为旧容量 + capacityIncrement

## Set

- HashSet：主要实现类；线程不安全；底层用HashMap实现，可存储null
- LinkedHashSet：HashSet的子类
- TreeSet：按顺序存储

> 无序性：不等于随机性，按哈希值来确定索引位置

### HashSet

JDK 7.0之前：数组+链表

> 拉链法解决散列冲突，JDK 7.0 采用头插法，即新元素在链表头；JDK 8.0用尾插法，即第一个出现在某位置的元素为链表头

#### 为什么重写equals要重写hashCode？

**因为散列表（HashSet、HashTable、HashMap等）判断时，会先判断hash值，再判断equals；非散列表中hashCode()是没有用的**

添加元素流程：

1. 根据元素hashCode()计算哈希值（hash()方法）
2. 根据哈希值计算索引位置
3. 如果此位置没有元素，则添加成功
4. 如果此位置有其他元素，比较哈希值
   - 如果哈希值不同，则向链表添加成功
   - 如果哈希值相同
     - equals()返回true，添加失败
     - equals()返回false，向链表添加成功

> 用IDEA自己生成的重写hashCode()，会出现如下
>
> ![image-20210610223201574](imgs\JAVA集合\16.png)
>
> 点进去
>
> ![image-20210610223231341](imgs\JAVA集合\17.png)
>
> 再点进去
>
> ![image-20210610223312445](imgs\JAVA集合\18.png)
>
> 重复result*31+属性值的hashCode，为什么这里选取31？
>
> - 31只占用5bits，相乘溢出可能性较小
> - i * 31，可由 (i << 5) - 1来表示，虚拟机针对此做了优化
> - 用素数乘，造成冲突概率小

**相等的对象必须要有相等的散列码**

### LinkedHashSet

- **遍历顺序和添加时一致**

- **每个数据维护一个指向前一个数据和指向后一个数据的引用，频繁遍历的时候效率较高**

### Treeset

- 需要添加相同类的对象
- 添加的对象需要继承Comparable接口；或者在初始化TreeSet的时候设置好Comparator
- 采用红黑树的结构

## Map

### HashMap

#### 概述

线程不安全；效率高；可存储null的键或值

**底层**

- JDK 7.0及之前：数组+链表
- JDK 8：数组 + 链表 + 红黑树

![image-20210615195012571](imgs\JAVA集合\19.png)

**结构**

- Map中的key：无序、不可重复，使用Set存储所有的key

  > key所对应的类要重写equals()和hashCode()

- Map中的value：无序、可重复，使用Collection存储所有的value

- key-value构成一个Entry对象，无序、不可重复，使用Set存储所有的entry

#### JDK 7.0 源码分析

> 版本：JDK 7u60

**初始化**

- 无参

初始化时创建一个长度为16的一维数组Entry[] table

- 带参

<img src="imgs\JAVA集合\20.png" alt="image-20210615213856731" style="zoom: 50%;" />

其中DEFAULT_INITIAL_CAPACITY为16，DEFAULT_LOAD_FACTOR为0.75，threshold为12

> 注意：并不是按initialCapacity进行初始化的，而是大于initialCapacity的某个数：2 ^ n

**添加**

```java
public V put(K key, V value) {
    //  key == null，使用该方法专门处理
    if (key == null)
        return putForNullKey(value);
    //  计算hash
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    //  遍历该索引上的链表
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        //  如果hash值相同且equals也相同，则新值覆盖旧值，并返回旧值
        //  如果hash值不同，或hash相同但不equals，则在循环结束后进入addEntry方法添加数据
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
	//  modCound，用于fast-fail
    modCount++;
    //  使用这个方法在链表上添加数据
    addEntry(hash, key, value, i);
    return null;
}
```

- 获取索引：hashCode→hash→indexFor*（一般都是(table.length - 1) & hash）*

  > 为什么table长度是2的n次幂？
  >
  > 用于快速寻址，hash & (table.length - 1)，可以快速将hash映射到0 ~ length-1的索引位置上，位运算的速度比取模快

- 添加一个数据(k1, v1)的过程：

1. 根据k1的hashCode，通过hash()计算散列值再得到索引
2. 该索引位置上没有数据，添加成功
3. 该索引上有数据
   - 哈希值不同，添加成功
   - 哈希值相同
     - equals不同，添加成功
     - equals相同，覆盖之前的数据

```java
private V putForNullKey(V value) {
    //  可以看出，null键都是放在table[0]这个位置
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```

看看关键的添加数据的addEntry方法

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    //  threshold为12，当size大于等于12且要添加数据的索引位置上已经有数据时，进行扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        //  扩容为2倍
        resize(2 * table.length);
        //  计算hash，并根据新的table.length计算索引
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }
	//  在该索引位置上creatEntry
    createEntry(hash, key, value, bucketIndex);
}
```

看一看createEntry

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    //  在索引位置new了一个Entry
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    ......
    /**
     * Creates new entry.
     */
    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        //  这一步可以看到，是把原有的table[bucketIndex]，作为自己的next
        //  也就是说是头插法，新插入的元素作为table[bucketIndex]，即链表头
        next = n;
        key = k;
        hash = h;
    }
    ......
}
```

**扩容**

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    //  如果旧容量已经到达MAXIMUM_CAPACITY，就不扩容了，令threshold==Integer.MAX_VALUE
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
	//  根据新长度创建新数组
    Entry[] newTable = new Entry[newCapacity];
    //  旧数组是否采用了useAltHashing优化
    boolean oldAltHashing = useAltHashing;
    /**
     *  ① useAltHashing为true；② 虚拟机isBooted并且新容量大于设置好的
     		ALTERNATIVE_HASHING_THRESHOLD。两个条件满足其一，就采用优化
     **/
    useAltHashing |= sun.misc.VM.isBooted() &&
        (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    //  旧数组和新数组优化策略不一致，则rehash为true，需要重新计算hash
    boolean rehash = oldAltHashing ^ useAltHashing;
    //  将旧数组中的数据放入新数组
    transfer(newTable, rehash);
    //  令table指向新数组
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            //  如果rehash为true，则需要重新计算哈希；为false，只需要通过indexFor改变索引
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            //  计算新索引
            int i = indexFor(e.hash, newCapacity);
            //  将数据放入新数组的某个新下标
            /**
             * 旧下标：hash & (oldLength - 1)
             * 新下标：hash & (oldLength * 2 - 1)
             *  假设旧长度为8，旧哈希为0xxx
             		旧索引：0xxx & (8-1) == 0xxx & 0111 == 0xxx
             		新索引：0xxx & (16-1) == 0xxx & 1111 == 0xxx
             	索引没变
             	假设旧长度为8，旧哈希为1xxx
             		旧索引：1xxx & (8-1) == 1xxx & 0111 == 0xxx
             		新索引：1xxx & (16-1) == 1xxx & 1111 == 1xxx
             	索引+8
             	所以新索引要么没变，要么加length
             **/
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

**哈希**

回过头看一看hash方法

```java
final int hash(Object k) {
    int h = 0;
    //  如果useAltHashing为true
    //  执行String类型键的替代哈希（内部哈希算法stringHash32）以减少由于哈希码计算较弱导致的冲突发生率
    //  如果不是String类型的键，就与一个哈希种子异或，也能降低冲突概率
    if (useAltHashing) {
        if (k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        h = hashSeed;
        //  transient final int hashSeed = sun.misc.Hashing.randomHashSeed(this);
        //  hashSeed是通过将this传入randomHashSeed并计算得到的，每个对象都有个固定值
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

**最后一句是“扰动函数”**

> 我们都知道，indexFor相当于一个低位掩码，只取小于table.length那一部分
>
> 如果hash方法只是简单的取hashCode，然后indexFor取后几位，则很容易碰撞；比如对于长度为8的数组，对于hashCode为1、9、17...这些数据都会放在同一个下标位置，疯狂碰撞
>
> 因此加入扰动函数，一方面大大降低碰撞概率，另一方面也融入了高位数据的信息

**useAltHashing**

针对useAltHashing变量，它是一种降低哈希冲突概率的优化措施；当虚拟机开启isBooted且HashMap的容量大于Holder.ALTERNATIVE_HASHING_THRESHOLD时，useAltHashing为true

```java
useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
```

看一看Holder

```java
/**
 * holds values which can't be initialized until after VM is booted.
 */
private static class Holder {

    // Unsafe mechanics
    /**
      * Unsafe utilities
      */
    static final sun.misc.Unsafe UNSAFE;

    /**
      * Offset of "final" hashSeed field we must set in readObject() method.
      */
    static final long HASHSEED_OFFSET;

    /**
      * Table capacity above which to switch to use alternative hashing.
      */
    static final int ALTERNATIVE_HASHING_THRESHOLD;

    static {
        //  通过设置虚拟机参数-Djdk.map.althashing.threshold来设置
        String altThreshold = java.security.AccessController.doPrivileged(
            new sun.security.action.GetPropertyAction(
                "jdk.map.althashing.threshold"));

        int threshold;
        try {
            //  如果没有设置-Djdk.map.althashing.threshold，使用默认的
            //  ALTERNATIVE_HASHING_THRESHOLD_DEFAULT==Integer.MAX_VALUE
            //  由于capacity一定不大于MAX_VALUE，所以默认情况就是不开启该优化
            threshold = (null != altThreshold)
                ? Integer.parseInt(altThreshold)
                : ALTERNATIVE_HASHING_THRESHOLD_DEFAULT;

            // disable alternative hashing if -1
            //  相当于一个开关，可通过-Djdk.map.althashing.threshold=-1来设置
            //  表示关闭该优化，实际上和默认情况一样
            if (threshold == -1) {
                threshold = Integer.MAX_VALUE;
            }

            if (threshold < 0) {
                throw new IllegalArgumentException("value must be positive integer.");
            }
        } catch(IllegalArgumentException failed) {
            throw new Error("Illegal value for 'jdk.map.althashing.threshold'", failed);
        }
        ALTERNATIVE_HASHING_THRESHOLD = threshold;

        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            HASHSEED_OFFSET = UNSAFE.objectFieldOffset(
                HashMap.class.getDeclaredField("hashSeed"));
        } catch (NoSuchFieldException | SecurityException e) {
            throw new Error("Failed to record hashSeed offset", e);
        }
    }
}
```

总结：

- -Djdk.map.althashing.threshold=-1：表示不做优化（不配置这个值作用一样）
- -Djdk.map.althashing.threshold<0：报错
- -Djdk.map.althashing.threshold=1：表示总是启用随机hashSeed
- -Djdk.map.althashing.threshold>=0：HashMap的table长度超过该值就异或一个随机hashSeed，减少碰撞

#### JDK 8.0 源码分析

> 版本：JDK 8u60

**初始化**

```java
transient Node<K,V>[] table;
```

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    ......
}
```

底层是一个Node数组，结构和JDK 7.0的HashMap.Entry比较像

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

初始化时并没有创建一个数组，而是给loadFactor和threshold赋值

DEFAULT_LOAD_FACTOR==0.75f；MAXIMUM_CAPACITY = 1 << 30

```java
this.threshold = tableSizeFor(initialCapacity);
```

来看看tableSizeFor

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= 1 << 30) ? 1 << 30 : n + 1;
}
```

目的是获取大于且最接近initialCapacity的二次幂

> - 对于cap > 1，n的高位一定有一个1
>
> 00001xxxxxxx
>
> 右移一位并|，即00001xxxxxxx | 000001xxxxxx == 000011xxxxxx，最高位和次高位变成了1
>
> 右移二位并|，即000011xxxxxx | 00000011xxxx == 00001111xxxx，从最高位数，前四位变成了1
>
> 接着就是前8、16、32位变成1，如果大于1 << 30，则令n等于1 << 30，为最大值
>
> 再加一，就变成二次幂了
>
> - 对于cap <= 1
>   - n == 0，最后返回1
>   - n < 0，最后同样返回1

**添加**

```java
public V put(K key, V value) {
    //  计算hash值，传入putVal
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    //  新的扰动函数，同样融合进了高位的信息，比JDK7.0简洁
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

点进putVal

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //  如果table没有初始化，或table长度为0，调用resize()进行初始化，并把长度赋给n
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //  (n - 1) & hash获得下标，首先令p指向该处。如果下标处为null，直接调用newNode，生成新节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //  else包含的就是table已经初始化，且下标处已经存在数据的情况
    else {
        Node<K,V> e; K k;
        //  key的hash相同，且equals为true，则令e=p，方便下面新值替换旧值
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //  如果p是TreeNode，则按树的方法添加，这里是红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //  key的hash不同或equals为false，并且p为Node链表节点的情况
        else {
            for (int binCount = 0; ; ++binCount) {
                //  遍历节点，如果下一个节点为空，则生成新节点
                if ((e = p.next) == null) {
                    //  令next指向新节点，为尾插法
                    p.next = newNode(hash, key, value, null);
                    //  如果链表深度大于8时，调用treeifyBin方法
                    //  ps：链表深度 == binCount + 2
                    //  static final int TREEIFY_THRESHOLD = 8;
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //  遍历节点，如果某一处hash和equals都相同，直接break，跳到下面新值替换旧值
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //  新值替换旧值，并返回旧值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //  如果size大于threshold，则resize
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**扩容**

resize方法

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //  threshold此时有两种可能：
    // ① 无参初始化，threshold=0；
    // ② 有参初始化，threshold=2^n，为最接近initialCapacity的二次幂
    int oldThr = threshold;
    int newCap, newThr = 0;
    //  1、如果table不为空/null
    if (oldCap > 0) {
        //  如果旧容量大于等于MAXIMUM_CAPACITY，即1 << 30，无法扩容，返回旧table
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //  static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
        //  否则扩容为两倍；如果旧容量大于等于16，则threshold x 2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //  2、如果table为空/null
    //  2.1、如果是有参初始化，threshold=2^n > 0，则令newCap=threshold
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //  2.2、如果是无参初始化，则令newCap=16，newThr=12
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //  如果table为空/null且是有参初始化，则重置threshold，从2^n变为newCap * loadFactor
    //  是上面的情况2.1的补充
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //  令threshold = newThr，此时threshold要么等于Cap * loadFactor，要么等于Integer.MAX_VALUE
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //  此处才创建Node数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //  将oldTab中的数据放入newTab
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //  假设oldCap = 8 = 1000，则newCap = 10000
                        /**
                         *  ① e.hash = 0xxx，0xxx & 1000 = 0
                         	0xxx & (oldCap - 1) = 0xxx
                         	0xxx & (newCap - 1) = 0xxx
                         	扩容后下标位置不变
                         	下面就是这种情况
                        **/ 
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        /**
                         *  ② e.hash = 1xxx，0xxx & 1000 != 0
                         	0xxx & (oldCap - 1) = 0xxx
                         	0xxx & (newCap - 1) = 1xxx
                         	扩容后下标位置+oldCap
                         	下面就是这种情况
                        **/ 
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        // 按原位置放
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 按原位置+oldCap放
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

**红黑树**

treeifyBin方法

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //  如果链表深度 > 8，但table长度 < MIN_TREEIFY_CAPACITY = 64时，只作扩容，不转换红黑树
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //  否则遍历节点并转换成红黑树
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

[参考](https://blog.csdn.net/qq_31709249/article/details/106951491)

> 需要注意的是，红黑树转换成链表的操作，在resize中和remove中是不同的，resize中是当红黑树中的节点少于6个时就将红黑树转为链表，而remove函数中，是当红黑树的根节点的左孩子或右孩子为空时，才将红黑树转换成链表的

#### JDK 7.0 和 8.0中的区别

1. **7中初始化时就创建底层数组，8中put时调用resize方法时才创建**
2. **7中底层为Entry数组，8中为Node数组**
3. **8中链表深度超阈值会转换成红黑树，Node→TreeNode，7中不会**
4. **7中threshold初始化时就等于cap * loadFacotor，8中初始化时会先根据initialCapacity计算threshold=2^n，在resize的时候才会令threshold = cap * loadFactor**
5. **扰动函数不一致，8中较简洁**
6. 7中会使用useAltHashing结合hashSeed和StringHashing32方法在扩容时优化哈希，8中去掉了这个
7. tableSizeFor方法不一致，7中是从最高位不断使用自|来计算，8中先数清楚cap-1数字高位有多少个0，再用-1*(FFFFFFFF)*右移这么多位，得到的就是最接近cap的2次幂
8. ......

#### HashMap为什么是线程不安全的？

主要集中在：①数据丢失；②循环链表；③数据覆盖

- 数据丢失和循环链表

> 这两个问题主要出现在JDK 1.7及以前，1.8中已经得到解决

扩容时的数据丢失和循环链表，是由于头插法的弊端：扩容后链表元素顺序会反转

- 数据覆盖

线程同时put，后面的覆盖的前面的，且由于判定索引处为null，所以不会返回旧值

例如：在并发条件下，第一个线程判断桶中无元素，则new一个新节点，但此时该线程被挂起。然后另一个线程直接在此处放了一个新节点进去，然后线程1被唤醒后，不知道此位置已经有新元素了，会直接放数据进去，此时会将原来线程放的数据覆盖；还有++size的时候

[链接](https://blog.csdn.net/swpu_ocean/article/details/88917958?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-2.control)

### LinkedHashMap

可以按添加的顺序遍历

> 在原有的底层实现上添加了一对引用，对于频繁的遍历有用

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    //  before指向前一个，after指向后一个
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

### TreeMap

按照添加的key排序

底层用红黑树实现

### Hashtable

> 线程安全的，效率低；用ConcurrentHashMap（分段锁）代替

#### Properties

```java
FileInputStream in = new FileInputStream("xxx/test.properties");
Properties properties = new Properties();
properties.load(in);
System.out.println(properties.get("test"));
```

### ConcurrentHashMap

#### JDK 7.0 源码分析

**底层结构**

先来看看ConcurrentHashMap底层最重要的两个结构：

首先是底层具体存储数据的结构：`final Segment<K,V>[] segments;`

也就是说，ConcurrentHashMap底层是由Segment数组组成的，那么Segment是什么？

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
    ......
    transient volatile HashEntry<K,V>[] table;
    ......
}
```

注意到，Segment类继承ReentrantLock，而Segment底层又是由HashEntry构成的，看到这个table变量的命名，有没有联想到HashMap中的Node<K, V> table？

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;

    HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    /**
     * Sets next field with volatile write semantics.  (See above
     * about use of putOrderedObject.)
     */
    final void setNext(HashEntry<K,V> n) {
        UNSAFE.putOrderedObject(this, nextOffset, n);
    }

    // Unsafe mechanics
    static final sun.misc.Unsafe UNSAFE;
    static final long nextOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class k = HashEntry.class;
            nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

可以看到，实际上，HashEntry确实和Node非常像，只不过其中多出一个nextOffset，是通过UnSafe的方法直接定位了next引用的内存地址，并后续通过CAS操作next

- 常量

定义的几个常量

```java
static final int DEFAULT_INITIAL_CAPACITY = 16;  // 所有的HashEntry（每个Segment的加起来）数目的的初始值
static final float DEFAULT_LOAD_FACTOR = 0.75f;  // 负载因子
static final int DEFAULT_CONCURRENCY_LEVEL = 16;  // 默认并发等级
static final int MAXIMUM_CAPACITY = 1 << 30;  // 所有HashEntry数目的最大值
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;  // Segment的table数组的最小长度
static final int MAX_SEGMENTS = 1 << 16; // Segment数组最大长度
static final int RETRIES_BEFORE_LOCK = 2;  // 重试次数
```

- 成员变量

```java
final int segmentMask; 
final int segmentShift;  
final Segment<K,V>[] segments;

transient Set<K> keySet;
transient Set<Map.Entry<K,V>> entrySet;
transient Collection<V> values;
```

- Segment

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
    ......
    transient volatile HashEntry<K,V>[] table;
    ......
}
```

可以看到，Segment继承ReentrantLock，也就是说，**每个Segment对象都相当于一把锁，这个Segment对象中的HashEntry数组依赖这同一把锁，不同HashEntry读写之间互不干扰；此外，每次只有一个线程能获取锁，其他线程会通过继承的tryLock方法来尝试获取锁，不断重试，而不是死等**。举一个例子，假如Segment数组长度为n，那么就可以同时进行n个线程的并发操作，性能比Hashtable提升n倍以上

- Segment的成员变量

```JAVA
static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;  // 重试次数
transient volatile HashEntry<K,V>[] table;
transient int count;  // HashEntry数组元素个数
transient int modCount;  // 修改次数，用于安全失败
transient int threshold;  // 扩容阈值
final float loadFactor;  // 负载因子
Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
    this.loadFactor = lf;
    this.threshold = threshold;
    this.table = tab;
}
```

- Segment的成员方法

![image-20210712111527773](imgs\JAVA集合\22.png)

> 其中scanAndLockForPut和scanAndLock：线程没有获取锁时调用这两个方法，用于尝试获取锁的期间进行一些准备工作

**初始化**

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;  // 不能超过最高并发等级
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    /**
     * 假设并发等级为16，那么ssize就等于16；假如并发等级为17，那么ssize就等于32
     * 采用这种方法令ssize为2的n次幂
    **/
    while (ssize < concurrencyLevel) {
        ++sshift;  // 2的n次幂，那个“n”的值
        ssize <<= 1;  // 乘2
    }
    this.segmentShift = 32 - sshift;  // 用于确定参与散列运算的位数，减去segmentShift就能获得掩码位数
    this.segmentMask = ssize - 1;  // 掩码，快速定址
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // c为所有HashEntry的数目除以ssize，也就是Segment长度，也就是每个Segment里面HashEntry的数目
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 同样的，令为2的n次幂
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                    (HashEntry<K,V>[])new HashEntry[cap]);  // 初始化第一个元素
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];  // 初始化Segment数组
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

![image-20210720095255476](imgs\JAVA集合\24.png)

> 对于putOrderedObject：
>
> ```java
> Class sc = Segment[].class;
> SBASE = UNSAFE.arrayBaseOffset(sc);
> ```
>
> SBASE可看作数组首地址，是一个long型偏移量
>
> 利用UNSAFE的native方法将s0放入ss的首地址

**添加**

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    /**
     * 获取索引j位置的segment
     * 这里这么绕，到底是什么意思？
     * 其实是因为java数组存放的是引用，它的大小不确定，有可能是4字节也有可能是8字节，看JVM设置(指针压缩：UseCompressedOops)
     * SSHIFT就是用来计算引用大小，如果引用为4个字节，则SSHIFT为2；如果为8，SSHIFT为3
     * SSHIFT = 31 - Integer.numberOfLeadingZeros(ss)，ss为引用大小
     * 相当于数组偏移索引*每个数组元素大小+首地址，获取的自然是该处的元素
     * 主要是UNSAFE运用的集中体现
     */
    //  对于这里的hash >>> segmentShift，我的理解是，反正前面的segmentShift位都不参与运算了，不如直接把它略过去
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}

public V putIfAbsent(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    
    if ((s = (Segment<K,V>)UNSAFE.getObject
            (segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);
    return s.put(key, hash, value, true);
}
```

除了最后一行，这两个方法的其他代码都是一致的

首先看put，前面跟HashMap差不多，都是hash之后利用掩码计算索引，之后获取j索引位置的元素*（相关参考链接：https://blog.csdn.net/qq_27114677/article/details/110121328）*，调用ensureSegment取出元素

```java
private Segment<K,V> ensureSegment(int k) {
    // 取出索引位置的Segment对象seg
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    // 如果为null，则创建HashEntry数组
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 使用第一个segment元素作为一个模板
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 利用CAS+自旋锁更新该位置的HashEntry
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                    == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

![image-20210720101651283](imgs\JAVA集合\25.png)

取到了segment之后，就要调用segment的put方法了

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 如果tryLock获得锁，则node为null，否则调用scanAndLockForPut方法等待锁
    /**
     * 逻辑在于：node是用于临时存放将要put的key和value的
     * 如果直接获取到了锁，则node会在后面直接创建，并头插法插入链表
     * 如果没有获取到锁，则会调用scanAndLockForPut预创建一个node，用于后续使用，而不是只傻傻的等待锁
     * 这里有一个问题：如果链表中已存在相同key的，那这个预创建的node不就没用了吗
     * 其实这个问题在scanAndLockForPut中是有所解决的，在scanAndLockForPut中会不断判断key是否已存在
    **/
    HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        // index处HashEntry的头节点
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    // 如果onlyIfAbsent为true，则不修改，否则用新值覆盖
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                // 直接使用预创建的node
                if (node != null)
                    node.setNext(first);
                else
                    // tryLock直接获取到了锁，直接创建node，使用头插法
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);  // 扩容
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

来看scanAndLockForPut方法

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    // 自旋，不断去尝试获取锁
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            // 链表头节点为null，预创建node
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            // 如果链表已存在key，则下次不会进入该循环
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        // 循环拿锁超过阈值，循环拿锁失败，采用阻塞式拿锁
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        // 扫描过程中发现链表被其他线程修改了，在这里需要处理一下，重新从头节点开始扫描
        else if ((retries & 1) == 0 &&
                (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed，令e为头节点，重新扫描
            retries = -1;
        }
    }
    return node;
}
```

总结起来：

![image-20210720135827190](imgs\JAVA集合\26.png)

**扩容**

```java
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                // 记录lastRun，有可能是高位(即扩容后位置变成高位)，也可能是低位(即扩容后位置不变)
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                // 从头节点→lastRun遍历链表
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    // 利用头插法插入新节点
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

#### JDK 8.0 源码分析

**底层结构**

**初始化**

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    // tableSizeFor(initialCapacity * 1.5 + 1)，即如果initialCapacity为32，cap == 64
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    // 初始化并发等级为1，注意，默认为DEFAULT_CONCURRENCY_LEVEL == 16，只不过这里用的是1
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    // tableSizeFor(initialCapacity / loadFactor + 1)
    // 注意这里和单参数的过程又不一样，如果是32，则为tableSizeFor(32 / 0.75 + 1) == tableSizeFor(43) = 64
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

喜闻乐见，又是jdk8的懒人模式，都是懒加载，这次的无参构造方法直接成空方法了

**sizeCtl**

1. sizeCtl为0，代表数组未初始化，DEFAULT_CAPACITY=16

2. sizeCtl为正数，如果数组未初始化，则记录的是初始容量；否则是扩容阈值，即初始容量 * 0.75

3. sizeCtl为-1，说明正在进行初始化

4. sizeCtl < 0且不为-1，说明正在扩容

   > 目前网上很多中文博客，对这个值的解释都是错的，然后再互相copy也不检查，就一错再错，只能说六个字：......
   >
   > 实际上，sizeCtl的高16位存储了当前容量的信息，低16位代表线程数 ；0-15位用于统计线程数， 16-31位用于记录扩容时容器的大小，令低16位为M，就是存在M-1个扩容线程
   >
   > [链接](https://blog.csdn.net/Unknownfuture/article/details/105350537)

**put**

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    /**
     *     static final int spread(int h) {
     *         return (h ^ (h >>> 16)) & HASH_BITS;
     *     }
     *     相当于hashmap的扰动函数与0x7FFFFFFF相与
     *     作用是去取最高的符号位，保证必须为整数
     *	   主要原因可能是防止和MOVED，TREEBIN，RESERVED这些预留hard-code的hash为负数的节点产生碰撞
     *     方便后面类型判断
     */
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // 详细见下面
            tab = initTable();
        // 如果tab相应的索引位置为空，则通过cas添加进去
        /**
         *    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
         *        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
         *    }
         *    通过ABASE首地址 + i * ASHIFT(Node引用的偏移量)确定地址
         */
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 基础的CAS操作
            if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果头节点的hash值为MOVED，表示为FORWARD节点，说明有线程正在扩容，于是本线程协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 锁住该桶位置处的头节点，其他桶不受影响
            synchronized (f) {
                // 为什么这里要double check？是为了防止某次插入之后变成树
                if (tabAt(tab, i) == f) {
                    // 头节点hash >= 0，说明为普通的链表结构
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    // 否则判断是否为树结构
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 树化，详细见下面
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

首先，如果table没有初始化，就进入initTable()的逻辑

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 有其他线程正在初始化，释放cpu
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 否则，CAS改变sizeCtl的值为-1，表示本线程已经去进行初始化了
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 个人猜想，这里double check的目的可能是为了避免重复初始化
                // 想象的一个例子：A、B线程同时到达CAS这块儿
                // A线程将sc(初始容量)改成-1并初始化完毕
                // B卡住了
                // C将其扩容，sc记录的扩容阈值等于之前的初始容量，产生ABA问题
                // 此时B恢复了，如果没有这句，相当于又初始化了一次
                if ((tab = table) == null || tab.length == 0) {
                    // ① sc == 0，说明数组未初始化，令n = DEFAULT_CAPACITY == 16
                    // ② sc > 0，如果数组未初始化，则sc代表初始容量，否则是扩容阈值。令n = sc
                    // 这里确定数组未初始化，所以n为16或者初始容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 已经初始化了，所以sc = n - 0.25n = 0.75n，为扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

![image-20210722112703215](imgs\JAVA集合\27.png)

> 两路逻辑：table未初始化时，和sizeCtl密切相关；已经初始化了，和头节点的hash值密切相关

**扩容**

> 补充知识：LongAdder
>
> 作用：实现原子的加减操作
>
> 源码分析：
>
> 重要成员变量
>
> ```java
> /** Number of CPUS, to place bound on table size */
> static final int NCPU = Runtime.getRuntime().availableProcessors();
> 
> /**
>  * Table of cells. When non-null, size is a power of 2.
>  */
> transient volatile Cell[] cells;
> 
> /**
>  * Base value, used mainly when there is no contention, but also as
>  * a fallback during table initialization races. Updated via CAS.
>  */
> transient volatile long base;
> 
> /**
>  * Spinlock (locked via CAS) used when resizing and/or creating Cells.
>  */
> transient volatile int cellsBusy;
> ```
>
> 重要内部类
>
> ```java
> // 缓存填充，解决伪共享问题
> @jdk.internal.vm.annotation.Contended static final class Cell {
>     volatile long value;
>     Cell(long x) { value = x; }
>     final boolean cas(long cmp, long val) {
>         return VALUE.compareAndSet(this, cmp, val);
>     }
>     final void reset() {
>         VALUE.setVolatile(this, 0L);
>     }
>     final void reset(long identity) {
>         VALUE.setVolatile(this, identity);
>     }
>     final long getAndSet(long val) {
>         return (long)VALUE.getAndSet(this, val);
>     }
> 
>     // VarHandle mechanics
>     private static final VarHandle VALUE;
>     static {
>         try {
>             MethodHandles.Lookup l = MethodHandles.lookup();
>             VALUE = l.findVarHandle(Cell.class, "value", long.class);
>         } catch (ReflectiveOperationException e) {
>             throw new ExceptionInInitializerError(e);
>         }
>     }
> }
> ```
>
> 重要方法
>
> ```java
> public void add(long x) {
>     Cell[] cs; long b, v; int m; Cell c;
>     // 如果cells为null并且通过cas令base + x成功，则不进入if判断
>     // 如果cells为null，但cas base失败，则进入if判断
>     // 若cells不为null，直接进入if逻辑
>     /**
>      * 这里优先判断了cell数组是否为空，之后才判断base字段的cas累加
>      * 意味着如果线程不发生竞争，cell数组一直为空，那么所有的累加操作都会累加到base上
>      * 而一旦发生过一次竞争导致cell数组不为空，那么所有的累加操作都会优先作用于数组中的对象上
>      */
>     if ((cs = cells) != null || !casBase(b = base, b + x)) {
>         boolean uncontended = true;
>         // ① 首先判断cells是否未初始化或者length == 0
>         // ② 接着通过当前线程哈希探针getProbe和掩码确定线程要在Cell[]数组的哪一个位置进行操作，判断该位是否为null
>         	// m相当于hashmap定址时的掩码，合理猜测，getProbe是不是和hash值差不多的存在？
>             /** 实际上还是有一定区别的，这是线程的探针哈希值。
>              *  在这里，这个探针哈希值的作用是哈希线程，将线程和数组中的不用元素对应起来，尽量避免线程争用同一数组元素
>              *  探针哈希值和 map 里使用的哈希值的区别是，当线程发生数组元素争用后，可以改变线程的探针哈希值，
>              *  让线程去使用另一个数组元素，而 map 中 key 对象的哈希值，由于有定位 value 的需求，所以它是一定不能变的
>              *  https://www.cnblogs.com/snail-gao/p/13605061.html
>              */
>         // ③ 若该位置不为空，则尝试通过cas改变该处cell的value的值
>         // 上述几个条件只要有一个符合，则进入longAccumulate 
>         // 只有获取到指定位置的Cell并成功cas改变其value，才不会进入longAccumulate
>         if (cs == null || (m = cs.length - 1) < 0 ||
>             (c = cs[getProbe() & m]) == null ||
>             !(uncontended = c.cas(v = c.value, v + x)))
>             longAccumulate(x, null, uncontended);
>     }
> }
> 
> final void longAccumulate(long x, LongBinaryOperator fn,
>                               boolean wasUncontended) {
>     int h;
>     // 如果线程哈希探针为0，说明Thread类的threadLocalRandomSeed和threadLocalRandomProbe没有初始化
>     // 是在ThreadLocalRandom类中进行初始化的
>     if ((h = getProbe()) == 0) {
>         // 对hash值进行初始化
>         ThreadLocalRandom.current(); // force initialization
>         h = getProbe();
>         // 这里令为true的原因是，通过getProbe rehash了一次，所以就认为下标处不会冲突的可能性比较大
>         // 所以在下面会直接通过CAS在下标处进行操作，跳过额外自旋并rehash那一步
>         wasUncontended = true;
>     }
>     /**
>      * 用来标记某个线程在上一次循环中找到的数组下标是否已经有Cell对象了
>      * 如果为true，则表示数组下标处为null
>      */
>     boolean collide = false;                // True if last slot nonempty
>     done: for (;;) {
>         Cell[] cs; Cell c; int n; long v;
>         // 如果cells没有初始化，跳转到“@init(自己定义的，方便看，在下面)”处的逻辑
>         // 下面是cells已经初始化的逻辑
>         if ((cs = cells) != null && (n = cs.length) > 0) {
>             // cells已经初始化，但指定位置的cell没有初始化
>             if ((c = cs[(n - 1) & h]) == null) {
>                 // cellsBusy == 0，说明这一刻没有线程正在操作cells
>                 if (cellsBusy == 0) {       // Try to attach new Cell
>                     // 创建新cell节点
>                     Cell r = new Cell(x);   // Optimistically create
>                     // 竞争cellsBusy锁
>                     if (cellsBusy == 0 && casCellsBusy()) {
>                         try {               // Recheck under lock
>                             Cell[] rs; int m, j;
>                             if ((rs = cells) != null &&
>                                 (m = rs.length) > 0 &&
>                                 rs[j = (m - 1) & h] == null) {
>                                 // 赋值并退出循环
>                                 rs[j] = r;
>                                 break done;
>                             }
>                         } finally {
>                             cellsBusy = 0;
>                         }
>                         // 如果跳到这儿，说明rs[j = (m - 1) & h] != null
>                         // 说明在判断c为null和拿锁之间，其他线程初始化好了，于是重回循环
>                         continue;           // Slot is now non-empty
>                     }
>                 }
>                 // 本次循环的时候下标处还没有初始化
>                 collide = false;
>             }
>             // cells已经初始化，指定位置的cell也已初始化，这时会尝试rehash一次，争取找到另一个位置
>             /**
>              * 这个字段如果为false，说明之前已经和其他线程发过了竞争
>              * 即使此时可以直接取尝试cas操作，但是在高并发场景下
>              * 这2个线程之后依然可能发生竞争，而每次竞争都需要自旋的话会很浪费cpu资源
>              * 因此在这里先直接增加自旋一次，在for的最后会做一次rehash
>              * 使得线程尽快地找到自己独占的数组下标
>              */
>             else if (!wasUncontended)       // CAS already known to fail
>                 wasUncontended = true;      // Continue after rehash
>             // rehash之后，还是发生冲突了
>             /**
>              * 尝试给hash对应的Cell累加，如果这一步成功了，那么就返回
>              * 如果这一步依然失败了，说明此时整体的并发竞争非常激烈
>              * 那就可能需要考虑扩容数组了
>              * （因为数组初始化容量为2，如果此时有10个线程在并发运行，那就很难避免竞争的发生了）
>              */
>             else if (c.cas(v = c.value,
>                            (fn == null) ? v + x : fn.applyAsLong(v, x)))
>                 break;
>             /**
>              * 这里判断下cpu的核数，因为即使有100个线程
>              * 能同时并行运行的线程数等于cpu数
>              * 因此如果数组的长度已经大于cpu数目了，那就不应当再扩容了
>              */
>             // 只要collide在这里赋了false，就一定走不到扩容那一步
>             else if (n >= NCPU || cells != cs)
>                 collide = false;            // At max size or stale
>             /**
>              * 走到这里，说明当前循环中根据线程hash值找到的数组下标已经有元素了
>              * 如果此时collide为false，说明上一次循环中找到的下标是没有元素的
>              * 那么就自旋一次并rehash，争取找到另一个位置
>              * 如果再次运行到这里，并且collide为true，就说明明竞争非常激烈，应当扩容了
>              */
>             else if (!collide)
>                 collide = true;
>             
>             else if (cellsBusy == 0 && casCellsBusy()) {
>                 try {
>                     if (cells == cs)        // Expand table unless stale
>                         cells = Arrays.copyOf(cs, n << 1);
>                 } finally {
>                     cellsBusy = 0;
>                 }
>                 collide = false;
>                 continue;                   // Retry with expanded table
>             }
>             /**
>              * 做一个rehash，使得线程在下一个循环中可能找到独占的数组下标
>              */
>             h = advanceProbe(h);
>         }
>         @init
>         // 如果cellsBusy == 0，说明没有其他线程正在修改cells；
>         // cells == cs，说明跳转过程中没有其他线程对cells进行修改；
>         // casCellsBusy为true，说明把cellsBusy通过CAS由0改到1成功
>         else if (cellsBusy == 0 && cells == cs && casCellsBusy()) {
>             try {                           // Initialize table
>                 // double check，防止ABA问题导致的重复初始化
>                 if (cells == cs) {
>                     // 初始化cells数组并令cell[h & cells.length - 1]的value等于x
>                     Cell[] rs = new Cell[2];
>                     rs[h & 1] = new Cell(x);
>                     cells = rs;
>                     // 退出循环
>                     break done;
>                 }
>             } finally {
>                 // 释放自旋锁
>                 cellsBusy = 0;
>             }
>         }
>         // Fall back on using base
>         // 竞争锁来初始化，没竞争到，再次尝试修改base，修改失败返回自旋
>         else if (casBase(v = base,
>                          (fn == null) ? v + x : fn.applyAsLong(v, x)))
>             break done;
>     }
> }
> 
> public long sum() {
>     Cell[] cs = cells;
>     long sum = base;
>     if (cs != null) {
>         for (Cell c : cs)
>             if (c != null)
>                 sum += c.value;
>     }
>     return sum;
> }
> ```
>
> [链接](https://www.cnblogs.com/tera/p/13886856.html)

```java
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            // 计数，记录hashmap元素个数
            s = sumCount();
        }
    	// check >= 0，添加节点成功
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // s大于等于sizeCtl，即扩容阈值，需要扩容
            // table为null的话，循环等待直到初始化完成
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                /**
                 *     static final int resizeStamp(int n) {
                 *         return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
                 *     }
                 */
                // private static int RESIZE_STAMP_BITS = 16;
                /**
                 *     以n = tab.length == 8为例来解析一下几个重要的值
                 *     n = 8，sc = sizeCtl == n - (n >> 2) = 6
                 *     numberOfLeadingZeros(n)   == 28，注意，只要容量不同，这个值就不同
                 *     numberOfLeadingZeros(n)   == 0000 0000 0000 0000 0000 0000 0001 1100
                 *     1<<(RESIZE_STAMP_BITS-1)  == 0000 0000 0000 0000 1000 0000 0000 0000
                 *     rs = resizeStamp(n)       == 0000 0000 0000 0000 1000 0000 0001 1100
                 *     先以第一个线程进来为例，sc > 0，将sizeCtl从sc改成(rs << RESIZE_STAMP_SHIFT) + 2
                 *     (rs<<RESIZE_STAMP_SHIFT)+2== 1000 0000 0001 1100 0000 0000 0000 0010
                 *	       取低16位为2，说明有2-1=1个线程在进行扩容
                 *     tansfer完成之后，重新计数一次，返回while循环，判断是否大于阈值
                 *     如果进来时发现，sc < 0，则将sizeCtl从sc改成sc + 1
      			 *         这样的话，低16位+1，表示参与扩容的线程+1
                 */
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
}
```

**transfer**

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
    	// 如果单线程，就不对步长进行划分，否则划分为(length / 8) / NCPU，最小值为MIN_TRANSFER_STRIDE，16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
    	// 当前无线程正在扩容，因为nextTab还未创建
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                // 创建一个容量为两倍的新数组
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            // 旧数组的长度，也是新数组比旧数组多出来的部分的第一个元素的索引，可以看作旧数组的末尾
            transferIndex = n;
        }
        int nextn = nextTab.length;
        // 为什么初始化时要传入nextTab，因为扩容还没完成时，调用get方法，遇到该节点会调用find方法在新表找
    	// 此外，在其他线程协助扩容时，也需要通过forwarding节点中的引用找到nextTable
        // 因为虽然扩容还未完成，但这里的节点已经迁移完成了
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    	// 用来标识当前线程是否需要去寻找下一个节点
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        // i指当前迁移任务的开始下标，bound为当前迁移任务的结束下标，注意，是从后往前遍历，所以范围是bound~i
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // 第一次会进来这儿，将bound改为末尾下标减步长
                // 将transferIndex改成bound，其他线程进来后，就会在这里设置bound~i
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    // 此时的i为旧数组末尾的下标
                    i = nextIndex - 1;
                    // advance令为false，退出该循环
                    advance = false;
                }
            }
            // 完成旧数组所有迁移
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 所有线程都完成了任务
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    // sizeCtl改为旧容量的1.5倍，为扩容阈值
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 首先，令sc - 1，即扩容线程数-1
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 如果(sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT，说明还有其他线程没完成扩容工作
                    // 直接return
                    // 如果==，说明当前线程是最后一个线程
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 所有线程都完成了扩容任务
                    finishing = advance = true;
                    // 进入finishing逻辑
                    i = n; // recheck before commit
                }
            }
            // 如果发现要处理的这个桶上没有元素，则将头节点置为ForwardingNode，表示这个节点有线程正在处理
            // ForwardingNode的hash值为MOVED==-1，后面线程判断出-1，就知道有线程正在处理扩容，于是也加入，进行协助扩容
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 探测到这个节点正在被其他线程处理，于是advance置为true，更新bound和i
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                // 锁住当前节点进行处理
                synchronized (f) {
                    // check一次，看看节点有没有被改变
                    if (tabAt(tab, i) == f) {
                        // 个人理解：low node，下标不变的；hign node，下标加length的
                        Node<K,V> ln, hn;
                        // fh个人理解是“first hash”，表示头节点的hash
                        if (fh >= 0) {
                            // 使用fh&n可以快速把链表中的元素区分成两类，A类是hash值的第X位为0，B类是hash值的第X位为1
                            // A类下标不变，B类下标会加上旧数组的length
                            int runBit = fh & n;
                            // 记录最后需要处理的节点
                            Node<K,V> lastRun = f;
                            // 遍历桶处链表
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            // 下面两个判断，确定lastRun的类型，是A类还是B类
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    // 头插法
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // A类下标不变
                            setTabAt(nextTab, i, ln);
                            // B类下标+n
                            setTabAt(nextTab, i + n, hn);
                            // 处理完后，在原table的此处标记为MOVED，表示已处理过
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
}
```

> 通过i和transferIndex确定边界，这是个volatile的变量

具体可看这位大佬的总结：[https://blog.csdn.net/qq_31709249/article/details/106952137](https://blog.csdn.net/qq_31709249/article/details/106952137)

![在这里插入图片描述](imgs\JAVA集合\28.png)

- **ln链**：和原来相比，变成了倒序
- **hn链：**hn链，在`lastRun`下标之前的节点被逆序了，`lastRun`之后，顺序没变

**helpTransfer**

发现forwarding节点，就进入helpTransfer方法

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        // 根据forwarding节点找到nextTable
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 参与扩容的线程数加一
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

**treeifyBin**

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n;
    if (tab != null) 
        // 跟hashMap一样，如果tab.length < 64，先扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 将b节点锁住，然后进行红黑树插入
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 当前位置放一个TreeBin，TreeBin的hash为-2，且有一个first引用，指向hd，也就是树真正意义上的根节点
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

**tryPresize**

```java
 private final void tryPresize(int size) {
     int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
     tableSizeFor(size + (size >>> 1) + 1);
     // 新长度c == tableSizeFor*(size * 1.5 + 1)，0.75 * c一定大于原size
     int sc;
     // 若sc > 0，表示没有线程正在扩容
     while ((sc = sizeCtl) >= 0) {
         Node<K,V>[] tab = table; int n;
         if (tab == null || (n = tab.length) == 0) {
             n = (sc > c) ? sc : c;
             // 如果未初始化，将sizeCtl改成-1，并进行初始化
             if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                 try {
                     if (table == tab) {
                         @SuppressWarnings("unchecked")
                         Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                         table = nt;
                         sc = n - (n >>> 2);
                     }
                 } finally {
                     sizeCtl = sc;
                 }
             }
         }
         // c <= sc，说明阈值变大了，可能有其他线程做了扩容
         else if (c <= sc || n >= MAXIMUM_CAPACITY)
             break;
         else if (tab == table) {
             int rs = resizeStamp(n);
             // sc < 0，说明是辅助扩容
             if (sc < 0) {
                 Node<K,V>[] nt;
                 if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                     sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                     transferIndex <= 0)
                     break;
                 // 扩容
                 if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                     transfer(tab, nt);
             }
             else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                          (rs << RESIZE_STAMP_SHIFT) + 2))
                 transfer(tab, null);
         }
     }
 }
```

### 几个细节

#### ConcurrentHashMap的弱一致性

> 遍历过程中做修改操作，会安全失败，而不是util包的快速失败

get、clear、iterator等方法，都没有加锁，可能出现弱一致性

[https://blog.csdn.net/mawming/article/details/51982112](https://blog.csdn.net/mawming/article/details/51982112)

#### JDK 1.7的size

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            // 一开始不进入该循环，不加锁
            // 如果重试次数达到RETRIES_BEFORE_LOCK==2，就给所有segment加锁
            // 也就是说，只要有一次比对modCount和expectedModCount错误，就直接加锁了
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            // 给每个segment计数，并记录modCount
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    // size += c，每个segment的count相加
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            // 第一遍last会为0，如果第二遍检查，modCount的值未改变，说明没有修改，跳出循环，返回size
            // 如果改变了，则不退出循环，直到重试次数大于阈值，给所有segment加锁并计算size，这样的话，其他线程就无法修改了
            // 保证了一致性
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        // 如果加锁的话，到最后当然要解锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

**所以JDK 1.7中的size是强一致性的**

需要注意的是，**加锁过程中会强制创建所有的不存在的Segment**

JDK 1.8中利用baseCount和CountCells再加上各种自旋、扩容算法，如果有A线程正在进行put操作，之后触发了扩容或者红黑树转置，那么立即就会synchronized锁定root节点。之后开始进行对应的操作，这个操作是需要时间的。但是这个时候，如果线程B来调用size方法，那么size方法由于没有任何锁机制，肯定是能够返回的，此时返回的size就是put之前的值。那么这个结果就导致了弱一致性
[链接](https://blog.csdn.net/dhaibo1986/article/details/108281950)

## 参考

[1] [尚硅谷JAVA——宋红康](https://www.bilibili.com/video/BV1Kb411W75N)

[2] [https://my.oschina.net/huangy/blog/1619144——听柳](https://my.oschina.net/huangy/blog/1619144)

[3] [https://www.zhihu.com/question/20733617](https://www.zhihu.com/question/20733617)

[4] [https://blog.csdn.net/swpu_ocean/article/details/88917958?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-2.control——swpu_ocean](https://blog.csdn.net/swpu_ocean/article/details/88917958?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-2.control)

[5] [https://www.zhihu.com/question/27999759](https://www.zhihu.com/question/27999759)

[6] [https://blog.csdn.net/qq_27114677/article/details/110121328——宿命小乐](https://blog.csdn.net/qq_27114677/article/details/110121328)

[7] [https://www.cnblogs.com/snail-gao/p/13605061.html——snail](https://www.cnblogs.com/snail-gao/p/13605061.html)

[8] [https://www.cnblogs.com/tera/p/13886856.html——tera](https://www.cnblogs.com/tera/p/13886856.html)

[9] [https://blog.csdn.net/Unknownfuture/article/details/105350537——DatDreamer](https://blog.csdn.net/Unknownfuture/article/details/105350537)

[10] [https://blog.csdn.net/qq_31709249/article/details/106951491——MeteorChenBo](https://blog.csdn.net/qq_31709249/article/details/106951491)

[11] [https://blog.csdn.net/qq_31709249/article/details/106952137——MeteorChenBo](https://blog.csdn.net/qq_31709249/article/details/106952137)

[12] [https://blog.csdn.net/mawming/article/details/51982112](https://blog.csdn.net/mawming/article/details/51982112)

[13] [https://blog.csdn.net/dhaibo1986/article/details/108281950——冬天里的懒猫](https://blog.csdn.net/dhaibo1986/article/details/108281950)
