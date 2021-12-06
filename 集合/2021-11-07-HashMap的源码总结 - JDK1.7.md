# HashMap的源码总结 - JDK1.7



## 数据结构

- **JDK1.7** 中的 **HashMap** 的底层结构是数组加单向链表实现的。
- 将 **key的hash值** 进行取模获取哈希表下标，既即将存放的元素的数组的位置，然后到对应的链表中进行 **put** 和 **get** 操作。

![image-20211107125432773](C:\Users\noblegasesgoo\AppData\Roaming\Typora\typora-user-images\image-20211107125432773.png)

## 加链表的好处与坏处

- 链表的引入，一定程度上解决了哈希碰撞带来的问题。
- 但是遍历寻找某个桶中的值的时候，如果链表过长，那么查询效率不高，查询时间复杂度为 **O(n)**。



## Entry 链表节点类

- 源码

```java
   static class Entry<K,V> implements Map.Entry<K,V> {
        final K key; // 节点 key。
        V value; // 节点数据域。
        Entry<K,V> next; // 下一个阶段。
        int hash; // 存储对应 key 的哈希值。
 
        /**
         *   这里保证始终插入的节点都是最新数据.
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }
    }
```



## 几个重要属性

- 和 **JDK1.8** 版本的没什么区别。

```java
// 默认初始化化容量,即16。  
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

// 最大容量，即2的30次方。  
static final int MAXIMUM_CAPACITY = 1 << 30;  

// 默认负载因子。  
static final float DEFAULT_LOAD_FACTOR = 0.75f;  

// HashMap内部的存储结构是一个数组，此处数组为空，即没有初始化之前的状态。
static final Entry<?,?>[] EMPTY_TABLE = {};  

// 空的存储实体。
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;  

// 实际存储的key-value键值对的个数。
transient int size;

// 阈值，当table == {}时，该值为初始容量（初始容量默认为16）；
// 当table被填充了，也就是为table分配内存空间后，threshold一般为 capacity*loadFactory。HashMap在进行扩容时需要参考threshold
int threshold;

// 负载因子，代表了table的填充度有多少，默认是0.75
final float loadFactor;

// 用于快速失败，由于HashMap非线程安全，在对HashMap进行迭代时，如果期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），需要抛出异常ConcurrentModificationException
transient int modCount;

// 默认的threshold值  
static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;
```



## 初始化

- 一共有四个构造方法。

```java
// 计算Hash值时的key。  
transient int hashSeed = 0;  

// 通过初始容量和负载因子构造 HashMap。  
public HashMap(int initialCapacity, float loadFactor) {  
    // 容量合法性检查。
    if (initialCapacity < 0) 
        throw new IllegalArgumentException("Illegal initial capacity: " +  initialCapacity);  
    if (initialCapacity > MAXIMUM_CAPACITY) // 最大容量限定。  
        initialCapacity = MAXIMUM_CAPACITY;  
    if (loadFactor <= 0 || Float.isNaN(loadFactor)) // 负载因子合法检查。
        throw new IllegalArgumentException("Illegal load factor: " +  
                                           loadFactor);  

    this.loadFactor = loadFactor;  
    threshold = initialCapacity;  
    init(); // init方法 在 HashMap 中没有实际实现，不过在其子类如 linkedHashMap 中就会有对应实现。
}  

// 通过初始容量构造 HashMap , 负载因子使用默认值，即 0.75f。  
public HashMap(int initialCapacity) {  
    this(initialCapacity, DEFAULT_LOAD_FACTOR);  
}  

// 负载因子取0.75，默认容量取16，构造HashMap。  
public HashMap() {  
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);  
}  

//通过其他 Map 来初始化 HashMap,容量通过其他 Map 的size来计算，装载因子取0.75。  
public HashMap(Map<? extends K, ? extends V> m) {  
    // 这里初始化，如果是 Map 的 size 更大的话默认容量就选择 Map 的 Size。
    this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1, DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);  
    inflateTable(threshold);// 初始化HashMap底层的数组结构。  
    putAllForCreate(m);// 添加m中的元素。  
}  
```



## put 方法

- 下面逐层分析 **put方法** 中的流程以及方法内调用的别的方法。

```java

public V put(K key, V value) {
    // 初始化数组和容量。
    // 所以在这里我们可以看到，如果你新建一个新的 HashMap 它一开始是不会有内存空间的，容量为null，而不是默认的16
    // 直到第一个元素进入哈希表才进行内存分配。
    // 如果table数组为空数组{}，进行数组填充（为table分配实际内存空间），入参为threshold，此时threshold为initialCapacity 默认是1<<4(=16)
    if (table == EMPTY_TABLE) {
        inflateTable(threshold); // 分配数组空间。
    }
    
    // 如果 key 为 null，存储位置为 table[0] 或 table[0] 的冲突链上。
    // 所以 HashMap 在 JDK1.7 中也是允许 key 可以为空的。
    if (key == null)
        return putForNullKey(value);
    
    // 计算 key 对应的哈希值，确保散列均匀。
    int hash = hash(key);
    // 寻找 key 在哈希表中所在的桶的下标。
    int i = indexFor(hash, table.length);
    
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        // 如果该对应数据已存在，执行覆盖操作。用新value替换旧value，并返回旧value。
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this); // 调用value的回调函数，其实这个函数也为空实现
            return oldValue;
        }
    }
    
    modCount++;
    addEntry(hash, key, value, i); // 新增一个节点
    return null;
}
```



### **inflateTable **方法

```java
private void inflateTable(int toSize) {
    // 这一步就是获取比当前传入长度大的最小二次幂数。
    // 比如 toSize = 5 的话 capacity = 2^3 = 8。
    int capacity = roundUpToPowerOf2(toSize);
    // 扩容阈值取默认和当前初始化过俩个中最小的那个。
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity]; // 开辟数组空间。
    initHashSeedAsNeeded(capacity);
}
```



 **roundUpToPowerOf2** 方法

```java
private static int roundUpToPowerOf2(int number) {
    // assert number >= 0 : "number must be non-negative";
    // 如果目前需要容量大于最大值，那么就返回最大值。
    // 否则目前需要容量是否是1，是的话就返回1，
    // 否则就返找到小于或等于目前传入容量的一个2的幂次方数，并且让他作为初始容量。
    return number >= MAXIMUM_CAPACITY
        ? MAXIMUM_CAPACITY
        : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}
```



### **Integer.highestOneBit** 方法

- 给定一个数字，找到小于或等于这个数字的一个2的幂次方数。

```java
public static int highestOneBit(int i) {
    // HD, Figure 3-1
    /*
    	如果此时 capacity = 18 = 10010
    	第一步：10010 或 01001 = 11011
    	第二步：00110 或 11011 = 11111
    	第三步：00001 或 11111 = 11111
    	第四步：00000 或 11111 = 11111
    	第五步：00000 或 11111 = 11111
    	最后返回：11111 - (11111 >> 1) = 10000 = 16
    */
    i |= (i >> 1);
    i |= (i >> 2);
    i |= (i >> 4);
    i |= (i >> 8);
    i |= (i >> 16);
    // return 语句中的i有这么个特点: 
    // 从最高位的1到最低位, 都是1, 所以i>>>1之后原先的最高位变成了0, 那么i - (i>>>1)的结果只剩下最高位的1了, 返回该值;
    return i - (i >>> 1);
}
```



### hash方法

- 用位运算来取 **key** 对应的尽可能符合均匀分布的哈希值。

```java
final int hash(Object k) {
    // 用了很多的异或，移位等运算，对 key 的 hashcode 进一步进行计算以及二进制位的调整等来保证最终获取的存储位置尽量分布均匀。
    int h = hashSeed;
    
    // 这里针对 String 优化了 hash 函数，是否使用新的 hash 函数和 hash因子有关。
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```



### indexFor方法

- 用来确定 **key** 对应的 **桶**。

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```



### addEntry方法

- 往对应桶的链表中添加新元素。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // 如果此时 size 超过扩容阈值 threshold，并且即将发生哈希冲突时进行扩容。
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);// 进行哈希表扩容，容量为之前的两倍。
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length); // 扩容后重新计算插入的位置下标。
    }

    // 把元素放入哈希表对应的桶的对应位置。
    createEntry(hash, key, value, bucketIndex);
}
```



### createEntry方法

- 往指定链表中头插法插入新节点。

```java
void createEntry(int hash, K key, V value, int bucketIndex) {  
    Entry<K,V> e = table[bucketIndex];  //获取待插入位置元素
    table[bucketIndex] = new Entry<>(hash, key, value, e);//这里执行链接操作，使得新插入的元素指向原有元素。
    //这保证了新插入的元素总是在链表的头  
    size++;//元素个数+1  
}  
```



## resize扩容

- 下面逐层分析扩容方法。

```java
void resize(int newCapacity) {  
    
    Entry[] oldTable = table; // 记录旧的哈希表。
    int oldCapacity = oldTable.length; // 获取旧哈希表的最大容量（数组长度）。
    // 如果旧的容量等于最大容量值。
    if (oldCapacity == MAXIMUM_CAPACITY) { 
        // 那就将扩容阈值修改为最大。
        threshold = Integer.MAX_VALUE;
        // 此时无法扩容了，直接返回原来的。
        return;  
    }  
    
    // 否则建立新表。
    Entry[] newTable = new Entry[newCapacity];  
    // 将旧表中的数据移动到新表中。
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    // 修改当前table的底层数组为扩容后的数组。
    table = newTable;
    // 修改数组之后设置新表的扩容阈值。
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1); 
}  
```



### transfer方法

- 移动旧表中的数据到新表。
- 注意这里我们的 **table** 还是为**旧表**，因为调用该方法的 **resize** 方法还没到覆盖底层数组那句代码。

```java
 void transfer(Entry[] newTable, boolean rehash) {  
    int newCapacity = newTable.length; // 存储新表的容量。  
    // 遍历旧表中的所有桶。
    for (Entry<K,V> e : table) { 
        // 如果不是空桶的话就取出链表放到新桶中。
        while(null != e) {
            // 保存下一次循环的 entry 节点对象。
            Entry<K,V> next = e.next;  
            // 如果 rehash 为 true,就通过 e的key值 计算 e的新hash值。
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);  
            }  
            
            // 定位到新表中的桶。
            int i = indexFor(e.hash, newCapacity);
            // 链表头插法，一个一个把原来的链表元素插入到新桶中。
            e.next = newTable[i];
            newTable[i] = e; // newTable[i]的值总是最新插入的值
            e = next;//继续下一个元素  
        }  
    }  
}  
```



### 并发情况下的分析

- 假设一个场景，一个线程正在进行扩容，运行到 **transfer** 方法的 `Entry<K,V> next = e.next;` 这条语句上时，
- 突然被别的线程夺取了资源，夺取资源之后执行了 `Entry<K,V> next = e.next;` 及以下的代码语句完成了扩容后的链表移植。
- 移植后的链表由于是头插法，所以全为倒叙，那么此时线程切换回来，就会出现死锁的情况，即自旋操作。
- 假设一个线程在做 **get** 操作，另一个线程正在扩容，那么也会导致 **get** 到的内容不是你想要的内容。



## remove方法

- 其实看下来无非就是把链表某个位置摘链。
- 与 **JDK1.8** 不同的是，**JDK1.8** 在删除时需要判断一下删除节点是红黑树节点还是普通链表节点。
  - 如果是红黑树节点，那么就得进行树的节点删除，很麻烦，性能消耗也大。
  - 还得判断这棵红黑树删了当前节点是否得变成普通链表，这一部分的性能消耗肯定没有直接删除链表节点的小。



## 总结

- **JDK1.7 **中 **HashMap** 的数据结构采用 **数组+单向链表** 进行存储数据,
  - 好处是便于解决hash冲突。
  - 坏处是可能增大get元素的遍历成本，查询时间复杂度为 **O(n)**。
- 插入链表数据时采用头插法来进行插入。
- 扩容时，其它线程更改扩容对象，是有可能发生死锁的。
- **put** 操作没有加锁,会导致 **get** 操作时不一定取到对应的值。
  

