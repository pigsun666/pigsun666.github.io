---
layout:     post
title:      HashMap与ConcurrentHashMap的故事
subtitle:   HashMap与ConcurrentHashMap的故事
date:       2020-6-10
author:     pigsun
header-img: img/post-bg-spring.jpeg
catalog: true
tags:
    - HashMap
    - ConcurrentHashMap
	- JDK7
	- JDK8
	- Java基础
---
# HashMap与ConcurrentHashMap的故事	

​		HashMap是我们工作学习中使用频率非常高的类，但是很多同学都是知其然，不知其所以然，并且现在面试难度越来越大，HashMap与ConcurrentHashMap几乎成为了必考题之一，所以让我们一起深入学习一下这两个类的过往版本迭代和底层原理。很多同学都知道在JDK8版本，对于它们两个类是一个重大版本，底层数据结构变化，逻辑优化，所以我们先从JDK7开始学习。

## JDK7的HashMap

从百度中搜索HashMap的面试题，统计出来一些结论，一一列举出来。

* HashMap是线程不安全的，非同步的
* HashMap的Key可以为null，Key值不能重复，如果重复则覆盖返回旧值
* HashMap的底层数据结构为数组+链表
* HashMap是无序的
* HashTable是线程安全的，为什么日常开发中不用HashTable而用ConcurrentHashMap
* HashMap的初始大小是16，扩容是直接乘以2

我们就以这些进行提问，在所有问题后面加个为什么？



### JDK7 HashMap源码解析

先来看看HashMap的类变量

```java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
  //默认容量大小
  static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
  //最大的容量
  static final int MAXIMUM_CAPACITY = 1 << 30;
  //负载因子
  static final float DEFAULT_LOAD_FACTOR = 0.75f;
  //空数组
  static final Entry<?,?>[] EMPTY_TABLE = {};
  //table代表HashMap里面的数组 默认为空数组
  transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
  //hashMap里面的元素数量
  transient int size;
  //容量阈值
  int threshold;
  //负载因子
  final float loadFactor;
  //修改次数（后文介绍）
  transient int modCount;
  //hash种子（后文介绍，为了让元素更加散列） 
  transient int hashSeed = 0;
  .........
}
```

再看看Entry的属性

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
  	//下一个元素
    Entry<K,V> next;
    int hash;

    /**
     * Creates new entry.
     */
    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }
}
```



#### 源码猜测

​		在阅读源码之前，我们先通过理解和看过的面试题，猜测一下HashMap的put逻辑。首先我们都知道HashMap是数组+链表的数据结构，每一个Key+Value对象都会封装成一个Entry对象。假设HashMap初始化的数组长度为8个，如果现在需要插入一个Entry对象，首先通过key算一个索引值，这个索引值要决定这个Entry对象放到哪个数组下标底下，可以通过hash值取余数组长度的方式。确定数组索引后，怎么放进去是下一个要思考的问题。链表的数据结构，就是当前对象都会保留下一个对象的引用，如果对象的next属性对象为null，则当前对象为该链条的尾节点。

![](https://pic-go-pigsun.oss-cn-shanghai.aliyuncs.com/20200604111033.png)

由上图其实可以发现，从链表的头部插入数据其实是最快最方便的，只需要把以前的头节点设置为当前元素的next元素。然后将当前元素直接放到数组的索引下标中，这样就完成了元素的插入操作。其实HashMap的实现方式大致也是这样的，只是做的更好。





测试代码

```java
public class HashMapTest {
    public static void main(String[] args) {

        HashMap<String, String> map = new HashMap<>();
        map.put("1","2");
        map.put("2","3");

        String s = map.get("1");
        System.out.println(s);
    }
}
```

#### 构造方法

先看下无参的构造方法

```java
public HashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}
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
        threshold = initialCapacity;
  			//没有实现逻辑，在LinkedHashMap有实现方法
        init();
    }
```

会初始化容量阈值和负载因子，DEFAULT_INITIAL_CAPACITY默认值是16，负载因子是0.75，构造方法比较简单。

#### put方法（重点）

接下来阅读Put方法（重点）

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```

前面的逻辑我们先暂时放下，等会回头再来看，直接看addEntry方法，

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
  	//扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }
		//创建entry对象设置到table里面
    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```

bucketIndex这个属性就是前面方法通过一些算法算出来的数组下标，这里的做法其实就是首先先把数组上当前下标的元素取出来，然后设置到新创建的Entry对象的next属性中，然后放到数组的指定数组下标上，容量+1。使用的头插法，和我们前文的 猜测是一样的。那么现在又有了一个问题，如果我put的key是重复值，为什么hashMap不会重复。带着这个问题，我们回到put方法的那个for循环里面

```` java
       for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
````

这个方法其实就是遍历链表的所有元素，如果有重复值，则直接替换旧值，并把旧值返回出去。那么这个数组下标到底是怎么算出来的呢，   

```java
int hash = hash(key);
int i = indexFor(hash, table.length);
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
```

hash方法其实就可以直接理解为hashCode方法，只是多了一些位运算和异或操作。

indexFor方法就是拿hashCode的值与数组长度-1进行与操作，这个写法和我们猜测的取余的方法都有共同的目的，得到一个数字并且这个数字不会超过数组的长度。

```java
//如果是空数组，则进行初始化
if (table == EMPTY_TABLE) {
  	//传入默认容量大小
    inflateTable(threshold);
}
//put key为null的值
if (key == null)
    return putForNullKey(value);
private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
  			//算数组长度
        int capacity = roundUpToPowerOf2(toSize);
				//计算阈值为 数组长度*负载因子
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
  			//数组的默认大小为16
        table = new Entry[capacity];
  			//初始化hash种子
        initHashSeedAsNeeded(capacity);
    }
private static int roundUpToPowerOf2(int number) {
        // assert number >= 0 : "number must be non-negative";
        return number >= MAXIMUM_CAPACITY
                ? MAXIMUM_CAPACITY
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }
private V putForNullKey(V value) {
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

Integer.highestOneBit((number-1) <<1)

这个函数的作用是取 i 这个数的二进制形式最左边的最高一位且高位后面全部补零，最后返回int型的结果。

比如传入的number是16   16-1=15 左移1位 等于30  然后取最高位，就是16.



结论：HashMap的默认数组长度为16 ，阈值是12，负载因子是0.75，如果put一个key是null的值，会把当前元素添加到数组的第一个位置。

最后我们再来讲HashMap的扩容。

##### 扩容时机

```java
if ((size >= threshold) && (null != table[bucketIndex])) {
  	//扩容
    resize(2 * table.length);
  	//重新计算hashCode
    hash = (null != key) ? hash(key) : 0;
  	//重新计算数组索引
    bucketIndex = indexFor(hash, table.length);
}
```

扩容的两个条件，当前容量超过阈值，并且当前要插入的数组索引上有元素（不为null），则触发扩容。

##### 扩容方法

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
  	//如果容量已达到了最大值，则不进行扩容了
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
		//创建一个新的Entry数组
    Entry[] newTable = new Entry[newCapacity];
  	//转移元素
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
  	//将新数组设置给类变量
    table = newTable;
  	//计算新的阈值
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
  			//遍历数组上的每一个元素
        for (Entry<K,V> e : table) {
            while(null != e) {
              	//暂存e.next 
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
              	//头插法
                e.next = newTable[i];
              	//把元素放在新数组上
                newTable[i] = e;
                e = next;
            }
        }
    }
```

大致逻辑：首先判断容量是否超过阈值，并且当前插入的数组位置已经有了元素，则会触发扩容。首先根据以前的容量直接乘以2，创建一个新的Entry数组，数组长度为原数组长度的两倍，然后进行转移元素。遍历所有数组上所有的链表，使用头插法，逐一放到新数组上面，和put方法类似，然后计算新的阈值，计算新的数组索引。



##### 扩容的问题

使用头插法进行元素转移，这样会导致链表元素的排列顺序调转，当然也没什么问题，因为当前链表上的元素有可能分布在新数组的任意位置。

多线程操作HashMap的时候会造成线程安全问题，会形成环形链表。因为table是类变量，多线程共享的一个变量。并且操作HashMap的方法都没有做任何的同步操作，所以当多线程的时候会造成线程安全问题。



### JDK7的HashMap总结

* 初始化容量为16，扩容是乘以2,默认的负载因子是0.75
* 插入元素使用头插法
* 扩容时机是当容量大于阈值，并且当前插入的数组索引位置已有元素，就是极限条件下，容量是可以超过16的
* HashMap是线程不安全的



## JDK7的ConcurrentHashMap



众所周知，ConcurrentHashMap是线程安全的。HashTable也是线程安全的，但是为什么不推荐使用呢？我们先看一下HashTable的源码。

```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }
  .....
}
public synchronized V get(Object key) {
        Entry tab[] = table;
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return e.value;
            }
        }
        return null;
    }
```

操作集合的方法中都添加了synchronized关键字进行了加锁。所以虽然保证了线程安全，但是执行的效率太低，比如我现在要往数组索引1号的位置插入一个Entry对象，又往数组索引2号的位置插入一个Entry对象，HashTable必须要等1号插完才能插入索引2号。但是实际上索引1号和索引2号是没有关系的。



### ConcurrentHashMap的特点

* ConcurrentHashMap是线程安全的
* 采用了分段锁的方式
* 以及采用的是数组+链表的数据结构
* 插入方式同样是头插法
* 初始化长度为16



### ConcurrentHashMap内部组成

我们前面学习了HashMap，HashMap的组成是Entry数组+Entry组成的链表 一起构成的

ConcurrentHashMap实现方式有所不同，多个Segment组成的数组，每个Segment对象里面维护一个HashEntry数组对象，这个HashEntry类似HashMap，也是数组+链表的数据结构。

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```

先看看ConcurrentHashMap的一些类变量

```java
//默认的初始化大小为16
static final int DEFAULT_INITIAL_CAPACITY = 16;
//默认的负载因子为0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//默认的并发级别为16
static final int DEFAULT_CONCURRENCY_LEVEL = 16;
//最大的容量
static final int MAXIMUM_CAPACITY = 1 << 30;
//最小的Segment数组个数
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
//最大的Segment个数
static final int MAX_SEGMENTS = 1 << 16; // slightly conservative
//与锁有关
static final int RETRIES_BEFORE_LOCK = 2;

final int segmentMask;
final int segmentShift;
final Segment<K,V>[] segments;
//集合中key的Set集合
transient Set<K> keySet;
//集合中Entry集合
transient Set<Map.Entry<K,V>> entrySet;
transient Collection<V> values;

```



### ConcurrentHashMap的构造方法

```java
public ConcurrentHashMap() {
  	//16，16，0.75
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
  			//使用循环找到大于等于concurrencyLevel的第一个2的n次方的数ssize
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
  			//默认情况下 ssize=16 sshift=4   
        this.segmentShift = 32 - sshift;
  			//segmentMask的各个二进制位都为1，目的是之后可以通过key的hash值与这个值做&运算确定Segment的索引
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
  			//创建首个Segment作为数组的第一个元素，原型的模式
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
  			//Unsafe  CAS操作
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
```

初始化所做的操作

* 验证参数合法性，抛出异常
* 并发级别不能超过最大segment个数
* 使用循环找到大于等于concurrencyLevel的第一个2的n次方的数ssize，这个数就是Segment数组的大小，并记录一共向左按位移动的次数sshift，并令segmentShift = 32 - sshift，并且segmentMask的值等于ssize - 1，segmentMask的各个二进制位都为1，目的是之后可以通过key的hash值与这个值做&运算确定Segment的索引
* 检查给的容量值是否大于允许的最大容量值，如果大于该值，设置为该值。最大容量值为static final int MAXIMUM_CAPACITY = 1 << 30;。
* 然后计算每个Segment平均应该放置多少个元素，这个值c是向上取整的值。比如初始容量为15，Segment个数为4，则每个Segment平均需要放置4个元素。
* 最后创建一个Segment实例，将其当做Segment数组的第一个元素。



### put方法

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
  	//计算hash
    int hash = hash(key);
  	//使用hash的高位与segment个数-1 取余 求出segment的索引值
    int j = (hash >>> segmentShift) & segmentMask;
  	//如果segment数组上的元素为null 则需要创建一个segment出来
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

put方法会把key进行三次hash操作，第一二次是为了求出segment的索引，第三次是找HashEntry所在的索引，先看看创建segment的方法是怎么实现的

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
  	//先通过cas读当前索引下的segment对象是否为空，然后先做一些准备工作
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
      	//创建HashEntry对象出来，再次cas读取
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck
          	//创建segment对象，但是还没有设置进数组
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
          	//自旋+CAS 保证只会有一个线程设置segment成功
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                  	//如果设置成功则会跳出循环，结束
                    break;
            }
        }
    }
    return seg;
}
```

如果已经把segment创建好了，或者segment对象本身就存在，才是真正把元素put进去的操作。Segment这个类本身就继承ReentrantLock，ReentrantLock是一个可重入锁

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  	//尝试加锁
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
      	//HashEntry数组对象
        HashEntry<K,V>[] tab = table;
     		//计算HashEntry的索引值
        int index = (tab.length - 1) & hash;
      	//取头节点
        HashEntry<K,V> first = entryAt(tab, index);
      	//遍历
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
              	//如果key相同 则覆盖原value
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                  	//如果putOnlyIfAbsent 则不修改
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
              	//如果获取锁的时候已经把HashEntry创建好了，直接把原来的头节点设置为当前节点的next节点
                if (node != null)
                    node.setNext(first);
                else
                  	//如果没有创建好HashEntry，
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
              	//如果超过阈值，则触发局部扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                  	//把当前节点头插法直接插到HashEntry上
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

Segment继承ReentrantLock类，ReentrantLock自带两个方法，lock方法和tryLock方法。lock方法会阻塞线程，直到拿到锁；tryLock方法不会阻塞，拿不到会返回false，一般会配合自旋，无限重复拿锁，但是非常消耗cpu。但是为什么这里会使用tryLock呢？

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
  	//获取头节点HashEntry
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
  	//重试次数
    int retries = -1; // negative while locating node
  	//反复尝试拿锁
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
      	//遍历HashEntry链表  如果有重复的key则设置重试次数为0 不做任何操作，如果遍历到尾节点了则创建node
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
      	//如果重试次数超过最大重试次数则阻塞拿锁
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
      	//每奇数次 轮训 且 头节点已经被另外一个线程修改的时候 
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

### 扩容机制

```java
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
  	//老数组长度
    int oldCapacity = oldTable.length;
  	//新数组长度*2
    int newCapacity = oldCapacity << 1;
  	//计算阈值
    threshold = (int)(newCapacity * loadFactor);
  	//新的数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
  	//通过sizeMask计算新的hash索引
    int sizeMask = newCapacity - 1;
  	//遍历数组
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
          	//单节点 直接赋值
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
              	//这个for循环 是为了快捷转移 如果链表的尾部一段元素都是统一的hash值，会记录这个点，等会一起拉过去
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
              	//移动到新数组 头插法
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
              	//遍历链表 排除lastRun节点以外的。 依旧采用头插法
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
  	//重新计算node的索引值
    int nodeIndex = node.hash & sizeMask; // add the new node
  	//
    node.setNext(newTable[nodeIndex]);
  	//头插法 插入
    newTable[nodeIndex] = node;
    table = newTable;
}
```



ConcurrentHashMap扩容是局部扩容，Segment的元素个数达到阈值，就会触发Segment内部HashEntry的扩容。ConcurrentHashMap初始化出来的时候只会创建一个Segment作为数组的首个元素，之后的Segment会以这个作为原型进行创建。

### JDK7的ConcurrentHashMap总结

* 初始化Segment（桶）数组的长度为16个，扩容是对单个桶进行扩容
* ConcurrentHashMap实现线程安全的方式主要是通过CAS+自旋+ReentrantLock，创建Segment的时候使用的是CAS+自旋，往Segment添加数据的时候是ReentrantLock。好处是尽可能的保证性能，分段锁，不像HashTable直接锁整个数组。
* 插入元素的方式依旧是头插法



## JDK8的HashMap

众所周知，JDK8的HashMap进行了较大的改动，数据结构由以前的数组+链表的方式修改成了数组+红黑树。所以在介绍HashMap之前，我们最好先学习一下红黑树，这里推荐一个讲的很生动的课程。

[什么是红黑树](https://zhuanlan.zhihu.com/p/31805309)

### 构造方法

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(int initialCapacity) {
  this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
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
```

构造方法没什么好说的，和1.7的基本一样，没有什么好说的

### 属性

```java
//默认初始化大小 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
//最大长度 
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//树化阈值 链表转换成红黑树的阈值 （树化的第一个条件）
static final int TREEIFY_THRESHOLD = 8;
//树转成链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//最小树化数组大小 （树化的第二个条件）
static final int MIN_TREEIFY_CAPACITY = 64;
//Node数组
transient Node<K,V>[] table;
//Entry Set集合
transient Set<Map.Entry<K,V>> entrySet;
//元素个数
transient int size;
//修改次数
transient int modCount;
//元素个数阈值
int threshold;
//负载因子
final float loadFactor;
```

### Node结构

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

### TreeNode结构

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
  	//父节点元素
    TreeNode<K,V> parent;  // red-black tree links
    //左边元素
  	TreeNode<K,V> left;
    //右边元素
  	TreeNode<K,V> right;
    //上一个元素
  	TreeNode<K,V> prev;    // needed to unlink next upon deletion
    //是否红节点
  	boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
}
static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
}

```

​		TreeNode就是红黑树数据结构的实现，但是实际上不仅仅只有红黑树，TreeNode继承LinkedHashMap.Entry, 这个Entry继承的还是Node类，所以TreeNode还有next属性。那么实际上TreeNode不仅是红黑树数据结构，它还是双向链表的数据结构。为什么要多这么一个数据结构呢？

 ### put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
  			//定义局部变量。tab为整个Node数组 p为当前要插入的数组下标的节点
  			//n为数组长度。 i为数组索引
        Node<K,V>[] tab; Node<K,V> p; int n, i;
  			//如果Node还没有初始化·这里会初始化数组
  			//resize 方法不仅仅只是初始化，他也是扩容的方法，所以现在先不看，当作初始化数组即可
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
  			//如果当前数组下标没有元素，则直接插入Node即可
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
          	//如果要插入的Node和数组下标的Node相同
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
          	//如果数组下标的Node是一个树节点 则要进行红黑树插入逻辑
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
              	//链表结构。遍历链表 尾插法
                for (int binCount = 0; ; ++binCount) {
                  	//当next为null的时候，就代表遍历到了链表结尾
                    if ((e = p.next) == null) {
                      	//创建一个节点 设置为尾节点的next
                        p.next = newNode(hash, key, value, null);
                      	//TREEIFY_THRESHOLD = 8 如果超过这个阈值，会触发链表转红黑树
                      	//注意执行顺序，如果要达到这个逻辑，是插入到第9个的时候，已经把第9个元素
                     		//插到了链表上，再转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                          	//树化链表
                            treeifyBin(tab, hash);
                        break;
                    }
                  	//如果找到一样的key 则跳出循环，后面的逻辑会直接赋值
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
          	//赋值value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
  			//超过阈值，扩容
        if (++size > threshold)
            resize();
  			//无实现
        afterNodeInsertion(evict);
        return null;
    }
```

这里大致简述一下执行逻辑

* 1:判断数组是否为空，如果为空则初始化数组
* 2:计算数组下标，然后判断当前数组下标Node是否为空，如果为空，则直接创建Node，赋值到数组。
*  
  * 如果要插入的key和数组下标上头节点的key的hashCode值一样，则直接返回，后面逻辑会直接替换值
  * 如果数组下标的头节点是一个TreeNode（树节点），则进行红黑树的新增逻辑
  * 上面都不成立则是链表结构，遍历链表，尾插法的方式插入元素。如果链表长度超过了阈值，则会转化成红黑树。
* 把value赋值给Node，然后判断容量是否超过阈值，如果超过则扩容



现在一层一层拨开HashMap的迷雾，先从红黑树的添加开始看

#### TreeNode新增

```java
e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```

我们前面有介绍过 p是通过hash计算出来的数组下标获取到的头节点，所以这个方法是TreeNode里面的方法了。

红黑树如何插入数据呢？应该是从root节点逐个判断，小于则与root节点的左节点对比大小，大于则再与root节点的右节点对比大小。

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
  	//拿到树的头节点
    TreeNode<K,V> root = (parent != null) ? root() : this;
  	//自旋 把root赋值给p
    for (TreeNode<K,V> p = root;;) {
      	//判断key
        int dir, ph; K pk;
        //ph为当前循环的key的hash值
      	//dir <=0 则会插在节点左边。 dir>0 则会插在节点的右边
      	if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
      	//如果key实现了Comparable接口，会用ComParable的比较逻辑
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }
				//开始插入逻辑
        TreeNode<K,V> xp = p;
      	//根据dir 取左右元素给p赋值，如果为空才代表到了红黑树的底部，否则继续循环对比
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
          	//注意 在这里xp就是p ，p已经变成了null
          	//取当前节点的next属性的值
            Node<K,V> xpn = xp.next;
         		//创建一个TreeNode
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
          	//根据dir 先直接插入到红黑树里面
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
          	//维护一下双向链表的关系
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
          	//调整红黑树
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

这段逻辑就是一个新的值，要插入到红黑树里面来，要从root节点逐一判断，直到空节点，然后直接插入。根据红黑树的知识，新插入的节点是红色的，极有可能当前的树是不符合红黑树数据结构的规则定义的，所以需要调整整个树。（左旋/右旋/变色）

#### 插入红黑树调整（balanceInsertion）

```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    //x是当前要插入的Node
  	//root是整个树
  	x.red = true;
  	//自旋   x是当前插入的节点 ，xp是x的父节点
  	//xpp 是xp的父节点。 xppl是xpp的左节点  xppr是xpp的右节点
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
      	//如果xp为空 则代表为根节点
        if ((xp = x.parent) == null) {
          	//直接设置为黑色。
            x.red = false;
          	//直接把当前节点作为root节点
            return x;
        }
      	//如果父节点为黑色，或者父节点为根节点，则表示不需要调整整个树
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
      	//走到这个逻辑，就表示父节点一定是个红节点
      	//如果父节点是祖父节点的左节点
        if (xp == (xppl = xpp.left)) {
          	//如果叔叔节点是红节点，则只需要把父节点和叔叔节点变成黑色，自己节点变成红色即可
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
						//如果叔叔节点是黑节点，则需要旋转+变色
          	else {
              	//如果当前插入的节点是父节点的右节点，则先进行左旋
              	//现在的情况是 父节点为红色，叔叔节点是黑色
                if (x == xp.right) {
                  	//由于旋转之后 xp成为了当前树的底部，所以对调xp和x 方便后面的操作
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                  	//设置为黑色
                    xp.red = false;
                  	//如果xp不为root节点 则设置xpp为红节点，再进行右旋
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
      	//如果父节点是祖父节点的右节点
        else {
          	//和上面逻辑一样 如果叔叔节点是红色变色即可
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
              	//如果当前插入的节点是父节点的左节点，则先进行右旋
              	//现在的情况是 父节点为红色，叔叔节点是黑色
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                  	//如果xp不为root节点 则设置xpp为红节点，再进行右旋
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

注意rotateRight和rotateLeft方法，这个方法有两个参数，第一个为整个树的Node，第二个是当前方法需要旋转的节点。上面方法的左旋和下面方法的左旋是不一样的，上面方法操作的是父节点的左旋。下面方法操作的是祖父节点的左旋。

左旋和右旋的方法就不看了，方法逻辑就是按照红黑树的规则，去调整这个树。



#### 将Node赋值到数组上，并且维护双向链表（moveRootToFront）

我们通过源码可以发现，HashMap的TreeNode节点不仅仅只是一个红黑树数据结构,还维护着一个双向链表的数据结构，我阅读源码之后的感想是为了方便操作元素，比如在扩容的时候，树可能会转换成链表的数据结构，这个时候因为有双向链表的结构，让这个操作变得简单了。

```java
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
      	//取出调整之前的数组上的TreeNode节点
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
      	//如果不等于 代表树的根节点已经改变 需要重新给数组执行下标赋值
        if (root != first) {
            Node<K,V> rn;
          	//把root设置到数组上
            tab[index] = root;
          	//这个root因为是刚调整之后的，所以root节点在调整之前不一定就一定是根节点，它有可能在调整之前只是某个叶子节点。要先确定这个逻辑，后面才好解释
          //取root的上一个元素rp
            TreeNode<K,V> rp = root.prev;
          	//取root的下一个元素rn  
            if ((rn = root.next) != null)
              	//把root抽离出来，rn指向rp
                ((TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
              	//rp指向rn
                rp.next = rn;
            if (first != null)
              	//以前的first指向root
                first.prev = root;
          	//现在的头节点指向原来的头节点
            root.next = first;
          	
            root.prev = null;
        }
        assert checkInvariants(root);
    }
}
```



#### 树化链表（treeifyBin）

默认情况下，把元素插入到链表的时候，当链表的长度大于等于8的且数组的长度大于64的时候会触发树化链表，会将链表的数据结构修改成红黑树的数据结构。

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
  	//如果数组为空，则初始化数组
  	//如果数组长度小于64。则只执行扩容操作，数组*2 然后转移元素
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
  	//计算hash索引 取出数组上的头节点元素
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
          	//用头节点元素 创建一个TreeNode 
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //这个循环是在红黑树的基础上 再维护双向链表的数据结构
          	if (tl == null)
                hd =  p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
          	//树化
            hd.treeify(tab);
    }
}
```

```java
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
  	//遍历链表
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        //下一个元素
      	next = (TreeNode<K,V>)x.next;
        //清空左右元素
      	x.left = x.right = null;
      	//初始化根节点
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
          	//自旋 下面逻辑在putTreeVal方法里面有 大致意思就是从根节点逐一对比 直到整棵树的底部 然后插入进去再调整树 最后把这个TreeNode节点放到数组上，重新维护双向链表关系
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```

树化链表的操作 简化来说就是取链表的头节点，把这个节点首先作为树的root节点，然后把链表的节点一个一个插入到树里面，树不断的调整，然后把这个树节点放进数组。



### 扩容/初始化方法（resize）

在JDK8的HashMap里面，初始化并不是在默认无参构造方法里面完成的。

```java
final Node<K,V>[] resize() {
  	//老数组
    Node<K,V>[] oldTab = table;
  	//老数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
  	//老阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
		//
  	if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
      	//新数组长度等于老数组长度左移1位
      	//如果新数组长度小于最大的数组长度 && 老数组长度 大于等于 默认的数组长度（16） 
      	//新的阈值等于老阈值左移1位 （*2）
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        //初始化长度和阈值
      	newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
  	//初始化新阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
  	//初始化数组
  	Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
  	//扩容逻辑
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
          	//取出老数组的节点
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
              	//如果当前数组的节点是单一节点 ，直接放到新数组上面去
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                  	//红黑树扩容逻辑
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                  	//链表扩容逻辑
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                      	//这里有个规律 因为oldCap都是2的指数，
                      	//所以这个结果只可能是0 或者非0 
                      	//还有一个扩容的规律，假设当前链表所在的数组索引是1，
                      	//那么扩容之后的索引 只会是1或者3
                      	//loHead 代表低位头节点。 loTail 代表低位尾节点
                      	//HiHead 代表高位头节点	 hiTail 代表高位尾节点
                      	//这段循环 循环当前链表，把链表里的元素分为了 高位和低位 两组链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
										//低位链表设置到新数组的低位数组中
                  	if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                  	//高位链表设置到新数组的高位数组上
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

再来看看红黑树的扩容，先想象一下，红黑树在扩容的时候，如果元素重新分配，新的树上面元素比较少，就没必要维护红黑树的数据结构，较短的链表查询效率会比红黑树更高。

```java
//tab代表新数组。bit代表旧数组长度
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
  	//这个方法是TreeNode的方法，所以this代表TreeNode本身
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
      	//这里有个规律 因为oldCap都是2的指数，
        //所以这个结果只可能是0 或者非0 
        //因为HashMap的TreeNode节点还维护了双向链表的结构，现在就很方便去拿所有节点
        //loHead 代表低位头节点。 loTail 代表低位尾节点
        //HiHead 代表高位头节点	 hiTail 代表高位尾节点
        //这段循环 循环当前链表，统计低位和高位的个数
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }
		//循环完成之后。一个树先被拆分成2个链表。 
    if (loHead != null) {
      	//如果链表长度小于默认的6个 就不需要使用红黑树
        if (lc <= UNTREEIFY_THRESHOLD)
          	//这里面 创建Node节点。把以前TreeNode的key和value 设置进Node  形成链表结构 设置到数组中
            tab[index] = loHead.untreeify(map);
        else {
          	//直接把当前TreeNode节点设置到数组里面
            tab[index] = loHead;
          	//如果高位头节点不为空 就代表红黑树结构发生了改变，有些元素需要转移到数组的其他地方，所以TreeNode需要调整
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
  	//下面也同理
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

