# ConcurrentHashMap - JDK 1.8自学



## 概述

- 在 **JDK1.7** 中我们的 **ConcurrentHashMap** 是靠**分段锁**来实现并发环境下线程安全的，采用的是 **数组 + 链表** 的数据结构。
- 在 **JDK1.8** 中我们的 **ConcurrentHashMap** 是靠 **数组+链表+红黑树** 的数据结构，用 **CAS** 操作来实现并发环境下的线程安全。



## 数据结构

- 采用 **数组 + 链表 + 红黑树** 的数据结构。
- 总体上和 **HashMap** 没有什么区别，就是需要实现并发环境下的线程安全，所以源码中要考虑的比 **HashMap** 要多，结构会更复杂一点。

| ![image-20211108134528797](C:\Users\noblegasesgoo\AppData\Roaming\Typora\typora-user-images\image-20211108134528797.png) |
| :----------------------------------------------------------: |
|                           **图示**                           |



## JDK1.8 中如何实现线程安全

- **CAS** 算法：
  - `首先它具有三个操作数，a、内存位置 V，预期值 A 和新值 B。如果在执行过程中，发现内存中的值 V 与预期值 A 相匹配，那么他会将 V 更新为新值 B。如果预期值 A和内存中的值 V 不相匹配，那么处理器就不会执行任何操作。`
- **ConcurrentHashMap** 它通过**CAS**算法又实现了3种原子操作。
- 原子操作就是线程安全的保障，能确保你在做这件事的时候别的线程没办法抢占 **CPU** 资源。



## 几个结点

- 一共有五种节点。



### Node 结点

- 他是其它四类节点的父类。
- 他就是默认连接到 **table** 中桶上的节点。
- 出现哈希冲突，他会以链表的形式存储，链表节点到一个阈值的话会转化为红黑树来加快查询速度。

```java
 /*
 * 普通的Entry结点, 以链表形式保存时才会使用, 存储实际的数据.
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; // 通过 key 的哈希值找到对应的桶。
    final K key;
    volatile V val;	// volatile修饰，保证可见性、有序性、但和原子性可没什么关系。
    volatile Node<K,V> next;  // volatile修饰，下一个节点。

    Node(int hash, K key, V val) {
        this.hash = hash;
        this.key = key;
        this.val = val;
    }

    Node(int hash, K key, V val, Node<K,V> next) {
        this(hash, key, val);
        this.next = next;
    }

    public final K getKey()     { return key; }
    public final V getValue()   { return val; }
    public final int hashCode() { return key.hashCode() ^ val.hashCode(); }
    public final String toString() { return Helpers.mapEntryToString(key, val); }
    public final V setValue(V value) { throw new UnsupportedOperationException(); }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }



```



### TreeNode 结点

- **TreeNode **是存储实际数据的红黑树节点。
- **TreeNode** 节点是不会直接链接到哈希表的桶上的。
- 有 **TreeBin** 节点来连接，**TreeBin** 会指向红黑树的根节点。

```java

static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links   红黑树链接
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    //删除链表的非头结点时，需要知道它的前驱结点才能删除，所以直接提供一个prev指针。
    TreeNode<K,V> prev; // needed to unlink next upon deletion 
    boolean red; // 节点颜色判断。

    TreeNode(int hash, K key, V val, Node<K,V> next , TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }

    Node<K,V> find(int h, Object k) {
        return findTreeNode(h, k, null);
    }

    //以当前结点（this）为根结点，开始遍历查找指定key.
    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {...}
}
        
```



### TreeBin 结点

- **TreeNode** 的封装结点；
- **TreeBin **会直接链接到哈希表的桶上。
- **该结点提供了一系列红黑树相关的操作，以及加锁、解锁操作**。

```java
/*
 TreeNode的代理结点（相当于封装了TreeNode的容器，提供针对红黑树的转换操作和锁控制）
 hash值固定为-2
*/
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;	// 根结点。
    volatile TreeNode<K,V> first; // 链表结构的头结点。
    volatile Thread waiter;	// 最近的一个设置 waiter 标识位的线程。
    
    volatile int lockState;	// 整体的锁状态标识位，初始时态为 0 。
    // values for lockState 锁定状态标识位的取值。
    static final int WRITER = 1; // 二进制001，红黑树的写锁状态。
    static final int WAITER = 2; // 二进制010，红黑树的等待获取写锁状态（优先锁，当有锁等待，读就不能增加了）。
    // 二进制100，红黑树的读锁状态，读可以并发，每多一个读线程，lockState 都加上一个 READER 值。
    static final int READER = 4; 

    // 在 hashCode 相等并且不是 Comparable类型时，用此方法判断大小。
    static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null ||
            (d = a.getClass().getName().
             compareTo(b.getClass().getName())) == 0)
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?  -1 : 1);
        return d;
    }

    // 将以b为头结点的链表转换为红黑树。
    TreeBin(TreeNode<K,V> b) {...}
    // 通过 lockState 属性,对红黑树的根结点加上写锁。
    private final void lockRoot() {
        if (!U.compareAndSetInt(this, LOCKSTATE, 0, WRITER))
            contendedLock(); // offload to separate method ，Possibly blocks awaiting root lock.
    }

    //释放写锁
    private final void unlockRoot() { lockState = 0; }

    //  从根结点开始遍历查找，找到“相等”的结点就返回它，没找到就返回null，当存在写锁时，以链表方式进行查找。
    final Node<K,V> find(int h, Object k) {... }

    /*
      	查找指定 key 对应的结点，如果树种没有该节点，那就插入它。
      	如果是插入成功就返回 null，否则就是待找节点。
    */
    final TreeNode<K,V> putTreeVal(int h, K k, V v) {...} 
    /*
    	红黑树节点删除：
    		如果红黑树规模太小，进行树转链表再删除的操作。
    		如果红黑树规模大，进行树节点摘除操作。
    */
    final boolean removeTreeNode(TreeNode<K,V> p) {...}

    // 以下是红黑树的经典操作方法，改编自《算法导论》。
    static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root , TreeNode<K,V> p) { ...}
    static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root , TreeNode<K,V> p) {...}
    static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root , TreeNode<K,V> x) {...}
    static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x) { ... }
    static <K,V> boolean checkInvariants(TreeNode<K,V> t) {...} // 递归检查红黑树的正确性。
    private static final long LOCKSTATE= U.objectFieldOffset(TreeBin.class, "lockState");
}

```



### ForwardingNode 结点

- 该节点只有在扩容的时候才使用得到。
- 相当于一个 **占位结点**，表示当前哈希表的这个桶正在进行扩容，当前线程可以尝试协助数据迁移。
- 哈希值固定为 **-1** ，不存在**实际数据的存储**！
- 如果旧桶中的全部数据都迁移到了新桶中，那么就在这个旧桶中放置一个 **ForwardingNode** 节点。
- 读操作碰到 **ForwardingNode** 节点时：
  - 将操作转到**扩容后**的**新哈希表**上去执行 **find** 方法。
- 写操作碰见 **ForwardingNode** 节点时：
  - 则尝试帮助扩容。

```java
static final class ForwardingNode<K, V> extends Node<K, V> {
    final Node<K, V>[] nextTable;

    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null);
        this.nextTable = tab;
    }
    
    // 在新的数组(nextTable)上进行查找。
    Node<K, V> find(int h, Object k) {...}
}
```



### **ReservationNode** 结点

- 保留节点，在 **computeIfAbsent** 和 **compute** 这俩函数式接口中使用的占位符节点加锁使用。
- 哈希值固定为 **-3**，不存在实际的数据存储。

```java
static final class ReservationNode<K, V> extends Node<K, V> {
    ReservationNode() {
        super(RESERVED, null, null, null);
    }
    Node<K, V> find(int h, Object k) {
        return null;
    }
}
```



## 初始化构造器

- **ConcurrentHashMap** 类提供了五个构造器。



### 空构造器

```java
public ConcurrentHashMap() { 
}
```



### 指定哈希表初始容量大小的构造器

- **tableSizeFor** 方法和 **HashMap** 中的一样，会返回大于**（initialCapacity + (initialCapacity >>> 1) + 1）**的最小2次幂值。
- 也就是说哈希表容量必须是2的整数倍，并且不能过小。

```java
public ConcurrentHashMap(int initialCapacity) {
    // 非法参数检查。
    if (initialCapacity < 0)
        throw new IllegalArgumentException();

    // 设定哈希表c容量。
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));

    this.sizeCtl = cap;
}	
```



### 根据已有的Map构造器

```java
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}	
```



### 指定哈希表初始容量和负载因子的构造器

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
```



### 指定哈希表初始容量、负载因子、并发级别的构造器

- **concurrencyLevel** 属性只是为了兼容**JDK1.8**以前的版本，并不是实际的并发级别，**loadFactor** 也不是实际的负载因子。
- 所以上面这俩兄弟只对初始容量有控制作用但是失去了原来的意义。

```java
	
		
	/**
	 * 指定table初始容量、负载因子、并发级别的构造器.
	 * 注意：concurrencyLevel只是为了兼容JDK1.8以前的版本，并不是实际的并发级别，loadFactor也不是实际的负载因子
	 * 这两个都失去了原有的意义，仅仅对初始容量有一定的控制作用
	 */
	public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
	    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
	        throw new IllegalArgumentException();
	
	    if (initialCapacity < concurrencyLevel)
	        initialCapacity = concurrencyLevel;
	
	    long size = (long) (1.0 + (long) initialCapacity / loadFactor);
	    int cap = (size >= (long) MAXIMUM_CAPACITY) ?
	        MAXIMUM_CAPACITY : tableSizeFor((int) size);
	    this.sizeCtl = cap;
	}

```



## 属性

- 一些 **ConcurrentHashMap** 类中的常量。



### 变量

![image-20211110211813935](C:\Users\noblegasesgoo\AppData\Roaming\Typora\typora-user-images\image-20211110211813935.png)

```java

// 哈希表，Node数组，标识整个Map，首次插入元素时创建。
// 也就是说不插入元素的话它不会存在。
// 大小总是 2的幂。
transient volatile Node<K, V>[] table;

// 扩容后的新的哈希表，只有在扩容时才非空。
// 线程可见，不可序列化。
private transient volatile Node<K, V>[] nextTable;

/**
 * sizeCtl：控制table的初始化和扩容。
 * =0 : 初始默认值。
 * -1 : 有线程正在进行哈希表的初始化。
 * >0 : table初始化时使用的容量，或初始化/扩容完成后的扩容阈值 threshold。
 *  = -(1 + nThreads) : 记录正在执行扩容任务的线程数。
 * 为正数时：
 *	数组没被初始化，那么存储的s数组的容量。
 *	数组被初始化，那么存储的是扩容阈值。
 */
private transient volatile int sizeCtl;

// 扩容的时候我们需要用到的桶下标变量。
// 用于控制迁移的位置。
// transfer方法是用来移动旧表中的数据到新表。
private transient volatile int transferIndex;

// 计数基值,没有出现并发冲突时也就是没有出现线程抢用的情况时，计数将直接加到该变量上。
private transient volatile long baseCount;

// 计数数组，出现并发冲突时使用，这时候计数不能直接加到baseCount变量上。
private transient volatile CounterCell[] counterCells;

// 自旋标识位，用于 CounterCell[] 扩容时使用。
private transient volatile int cellsBusy;

// 视图相关字段。
private transient KeySetView<K, V> keySet;
private transient ValuesView<K, V> values;
private transient EntrySetView<K, V> entrySet;

```



### 常量

```java

// 哈希表最大容量。
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 哈希表默认初始容量 16 = 1 << 4 
private static final int DEFAULT_CAPACITY = 16;

// 哈希表最大长度。
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

// 负载因子，为了兼容 JDK1.8 以前的版本而保留f
// JDK1.8中的 ConcurrentHashMap的负载因子 恒定为 0.75f。
private static final float LOAD_FACTOR = 0.75f;

/*
  	以下俩变量是关于：链表转换为红黑树的阈值。
  	在链表转换为红黑树之前会进行键值对的数量判断，
  	只有当前哈希表中键值对数量大于 MIN_TREEIFY_CAPACITY 时，才会进一步判断TREEIFY_THRESHOLD，
  	都满足才会发生转换。
    有效避免哈希表创建初期有多个键值对放入同一个链表而开始转换从而浪费时间。
*/
static final int TREEIFY_THRESHOLD = 8;
static final int MIN_TREEIFY_CAPACITY = 64;

/*
	以下俩变量是关于：红黑树转链表的阈值。
	在红黑树转为链表之前会进行键值对的数量判断，
	只有当前哈希表中键值对的数量小于 MIN_TRANSFER_STRIDE 时，才会进一步判断 UNTREEIFY_THRESHOLD，
	都满足才会发生转换。
*/
static final int UNTREEIFY_THRESHOLD = 6;
private static final int MIN_TRANSFER_STRIDE = 16;

// 用于在扩容时生成唯一的随机数，与默认哈希表容量一致（桶的数量）
// 变更容量时的标志位。
private static int RESIZE_STAMP_BITS = 16;

// 可同时进行扩容操作的最大线程数。
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

/**
 * The bit shift for recording size stamp in sizeCtl。
 */
// 记录 sizeCtl 中大小标志的位偏移量。
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;


// 给哈希值的意义。
static final int MOVED = -1;                // 标识 ForwardingNode 结点（在扩容时才会出现，不存储实际数据）。
static final int TREEBIN = -2;              // 标识红黑树的根结点。
static final int RESERVED = -3;             // 标识 ReservationNode 结点（）。
static final int HASH_BITS = 0x7fffffff;    // 标识 Node 结点，自然数0，1，2...

// CPU核心数，扩容时使用。
static final int NCPU = Runtime.getRuntime().availableProcessors();

```



## put 方法

- **put**方法是 **ConcurrentHashMap** 类的核心方法，所以我们一定要理解它。



### put

- 对外提供**put**接口，键值对都不能为 **null**。

```java
public V put(K key, V value) {  
    return putVal(key, value, false);
}
```



### putVal

- 内部插入操作。
- ==只有当前要插入的键值对的 **key** 不在当前的哈希表中才进行插入！==

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    
    // key 和 value 的非法参数判断。
    // 从这里我们直接看出键值对不能是空！
    if (key == null || value == null) throw new NullPointerException(); 
    // 为了防止并发情况的影响，再次计算一遍当前 要插入键值对key 的 哈希值。
    int hash = spread(key.hashCode()); 

    // 这个变量有俩个作用：
    // 使用链表保存的数据的时候，这个变量记录的是这个桶中的结点数量。
    // 使用红黑树保存数据的时候，这个变量会被赋值为 2 ，以此来保证 put 操作之后更改计数值的时候能够进行扩容检查，
    // 同时还不出发红黑树化操作。
    int binCount = 0;  // 用于记录相应链表的长度。
    
    // 自旋插入当前结点，直到成功。
    for (Node<K, V>[] tab = table; ; ) {      
        
        // 用来记录当前桶中第一个节点，方便后续开始处理哈希碰撞时比较。
        Node<K, V> f;
        
        // 哈希表的长度，以及对应 index。
        int n, i, fh;
        
        // 情况一：如果哈希表当前为空的化，进行哈希表初始化，懒加载。
        if (tab == null || (n = tab.length) == 0)                
            tab = initTable();
        
        // 情况二：哈希表中待插入数据桶的位置如果没有节点的化，直接插入就行了。
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {    
            // 条件中计算待插入键值对的索引计算的方式是 (n - 1) & hash)。
            // 使用这种索引计算方式可以实现 key 在哈希表中的尽量均匀分布，以此来相对减少哈希冲突。
            // 这也是为啥哈希表的容量必须为 2的幂 的原因。
            // tabAt(tab, i)方法是调用 Unsafe 类的方法查看值，保证每次获取到的值都是最新的。
            if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null))) // cas操作：插入一个链表结点，成功返回
                break;
            
            // 情况三: 发现 ForwardingNode 结点（通过MOVED标识ForwardingNode结点存在），
            // 说明此时table正在扩容，则尝试协助数据迁移。
        } else if ((fh = f.hash) == MOVED)                          
            tab = helpTransfer(tab, f);   //这个方法很重要。
        // 情况四: 不是空桶，出现哈希碰撞。
        else {    
            // 存储老的节点的数据域。
            V oldVal = null;
            
            // 对该桶d锁！
            synchronized (f) {         
                
                // 再次判断一遍，当前的桶中第一个节点是不是前面保存过的 f 节点。
                // 防止我们的头节点被其它线程在当前线程运行到这一步时修改过了。
                if (tabAt(tab, i) == f) { 
                    //static final int HASH_BITS = 0x7fffffff;  标识Node结点，自然数。
                   
                    // fh在情况三的判断条件中被赋值为当前节点的哈希值。
                    // 情况4.1：桶中装的是链表。
                    if (fh >= 0) {          
                        binCount = 1;  //记录结点数，超过阈值后，需要转为红黑树，提高查找效率。
                        // 循环遍历当前桶中元素。
                        for (Node<K, V> e = f; ; ++binCount) {
                            K ek;
                            
                            // 如果找到相同 key 的节点，判断是否需要更新 value值 （onlyIfAbsent为false则可以插入）
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;	// 记录 key冲突 的原节点对应的数据域。
                                if (!onlyIfAbsent)  // onlyIfAbsent 标志只有是 false不存在 时才插入。
                                    e.val = value;
                                break;
                            }
                            
                            // 如果没有找到冲突节点，直接进行尾插法添加节点。
                            // 从这里可以看出和 JDK1.7 中的区别，JDK1.7 用的是头插法，可能会出现死锁。
                            Node<K, V> pred = e;
                            // 如果是尾节点，直接插入。
                            if ((e = e.next) == null) {
                                pred.next = new Node<K, V>(hash, key,value, null);
                                break;
                            }
                        }
                        // 情况4.2: 桶中装的是红黑树节点。
                    } else if (f instanceof TreeBin) {  
                        Node<K, V> p;
                        binCount = 2;  // 红黑树对应的基值，包括了 treebin 和 root，所以为 2。
                        
                        // 调用红黑树的插值方法插入新节点。
                        if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key, value)) != null) { 
                            // 插入前保存旧的节点数据域。
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }  
                    }

            }
            
            // 链表是否为空。
            if (binCount != 0) {
                
                // 判断是否要将链表转换为红黑树。
                // 临界值 TREEIFY_THRESHOLD 和 HashMap 一样，也是 8。
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i); // 链表转化为红黑树。   
               
                // 判断是否只是进行了旧值替换，而没有插入新节点。
                // 表明本次操作增加了新的节点而不是老节点替换，计数值加1。
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



#### spread 方法

- 官方提示：由于表的边界，永远不要在索引计算中使用。

```java
static final int spread(int h) { 
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```



#### initTable 懒加载方法

- 官方说明：在构造函数中，哈希表数组并不会被初始化，只有到了第一个值插入的时候，才会初始化哈希表。

```java
private final Node<K,V>[] initTable() {
    
    // 存储初始化后的哈希表。
    Node<K,V>[] tab; 
    // 初始化容量。
    int sc;
    
    // 开始初始化。
    while ((tab = table) == null || tab.length == 0) {
        
        // 检测是否已经有其它线程初始化过了。
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin 当前线程正在忙，给我等着。
        // 等你扩容完了我再来用，我先挂起。
       
        // CAS 操作，将 sizeCtl 设置为 -1，代表抢到了锁，现在是当前线程进行哈希表的初始化。
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 初始化过程。
            try {
                // 当前哈希表是第一次初始化，判断是否初始化。
                if ((tab = table) == null || tab.length == 0) {
                    // 初始化容量设置。
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    // 用初始化容量来初始化哈希表。
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 将这个数组赋值给 table，table 是被 volatile 关键字修饰的，所以对别的线程透明。
                    table = tab = nt;
                    
                    // 如果 n 为 16 的话，那么这里 sc = 12。
                    // 其实就是 负载因子0.75 * 当前容量n。
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置 sizeCtl 为 sc，我们就当是 12 吧
                // sizeCtl>0 : table初始化时使用的容量，或初始化/扩容完成后的扩容阈值 threshold。
                sizeCtl = sc;
            }
            break;
        }
    }
    // 返回初始化后的哈希表。
    return tab;
}
```



#### helpTransfer 线程加入数据迁移方法

- 数据迁移伴常常随着扩容出现。
- 我的理解就是，当前线程帮助扩容。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    
    // 转移后的哈希表。
    Node<K,V>[] nextTab; 
    // 初始化容量。
    int sc;
    
    // 如果旧哈希表不为空，并且节点为 ForwardingNode 标记节点，并且标记节点指向的新哈希表不为空的话。
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        
        // 根据容量生成一个随机数来唯一标识本次扩容。
        int rs = resizeStamp(tab.length);
        
        // 循环迁移，一切条件都符合（包括当前sizeCtl控制变量为 -1）
        // sizeCtl = -1 : 有线程正在进行哈希表的初始化。
        while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 此时表示sizeCtl + 1 = -2，表示多一个线程进来协助扩容。
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {			
                transfer(tab, nextTab); // 扩容中的数据转移。
                break;
            }
        }
        
        return nextTab;
    }
    
    return table;
}
```



#### treeifyBin 链表转红黑树方法

- 它不一定就会进行红黑树转换，也可能是仅仅做哈希表扩容。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    
    Node<K,V> b; 
    // 用来存储哈希表长度以及，sizeCtl。
    int n, sc;
    
    // 如果当前哈希表有链表。
    if (tab != null) {
        // MIN_TREEIFY_CAPACITY 为 64。
        // 如果此时不满足转红黑树的条件，直接扩容。
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 实际扩容方法。
            tryPresize(n << 1);
        
        // b 此时是该桶中的头节点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            
            // 加锁。
            synchronized (b) {
				
                // 再次确认 b 是不是头节点，防止被其它线程更改。
                if (tabAt(tab, index) == b) {
                    // 下面就是遍历链表，建立一颗红黑树。
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
                    // 以 TreeBin结点类型进行包装，并链接到 table[index] 中
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```



## tryPresize 扩容方法

- 这里的扩容，是指的是将哈希表的容量扩大一倍。
- **JDK1.8 **中，**ConcurrentHashMap **最复杂的部分就是 **扩容、数据迁**。
- 扩容 **涉及多线程的合作和 rehash()方法**。
- **HashMap** 扩容规律：
  - 先扩大哈希表容量为原来的两倍，==整个过程只能一个线程去完成，不允许并发==！
  - 然后进行数据迁移。
    - 数据迁移涉及到 **key** 的 **rehash** 过程，因为容量变了，我们的下标计算范围就变了。

- **ConcurrentHashMap** 扩容规律：
  - 它在处理 **rehash** 的过程中，不一定会将每个 **key** 的哈希值重新计算。
  - 通过 **key.hash & table.length-1** 这种方式计算出的索引，当table扩容后，大小为原来的两倍，新的索引要么在原来的位置，要么是原来的位置。
  - 这也侧面说明了为什么哈希表的容量要为 **2的幂**。

```
例子：扩容前容量 16 = 0001 0000， 假设有俩哈希值：0001 0101、0000 0101
	扩容前，他俩所在的下标为: 0001 0000 - 1 = 0000 1111    0000 1111
										0001 0101    0000 0101
										 					&
										-----------------------
										0000 0101	 0000 0101
										
     扩容后，他俩所在的下标为: 0010 0000 - 1 = 0001 1111    0001 1111
     									 0001 0101    0000 0101
     									  					&
										-----------------------
										 0001 0101    0000 0101
我们可以看出虽然他俩原来在哈希表中的同一个桶，但是在扩容之后，就不在一起了。
```

```java
private final void tryPresize(int size) {
    // 方法参数 size 传进来的时候就已经翻了倍了，可以对照 treeifyBin 方法看。
    // 将当前哈希表的 size 与 MAXIMUM_CAPACITY/2 来比较 ， tableSizeFor求二次幂
    // 这里的 c 为 size 的 1.5 倍，再加 1，再往上取最近的 2 的 n 次方。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY : 
    	tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; 
        int n;
       
        // 情况一：哈希表还没被初始化。
        // 详情注释可以查看 initTable 方法。
        if (tab == null || (n = tab.length) == 0) {
		   // 开始初始化。
            // 取他俩中最大的那个作为当前哈希表的初始容量。
            n = (sc > c) ? sc : c; 
            
            // CAS 操作，将 sizeCtl 设置为 -1，代表抢到了锁，现在是当前线程进行哈希表的初始化。
            if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
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
        // 情况二: c <= sc 说明已经被其他线程扩容过了，
        // n >= MAXIMUM_CAPACITY 代表扩容已经到最大阈值。
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            // 既然都扩容过了，或者扩容的容量达到最大了，那就直接退出方法吧。
            break;
        
        // 情况三: 正式进行哈希表的扩容。
        else if (tab == table) {
            
            int rs = resizeStamp(n); // 根据容量生成一个随机数来唯一标识本次扩容。
            
            // 这里如果 sc < 0 就代表已经有线程在进行扩容了。
            if (sc < 0) {
                // 新的哈希表。
                Node<K,V>[] nt;
                // 如果当前线程无法协助数据转移就直接让该线程退出。
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 调用 CAS 方法把 sizeCtl 进行加一操作，正在执行任务的线程数+1，
                // 让当前线程协助数据转移。
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 否则如果当前扩容操作没有线程进行， sc 此时为负数，让当前线程成为第一个执行数据转移操作的线程。
            // 这个 CAS 操作可以保证，仅有一个线程会执行扩容。
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}

```

- 上述方法关键看情况三，情况三的分支，大意是这样的：

  - 如果 **sc < 0** 也就是 **sizeCtl < 0** 的时候，就代表了当前已经有线程开始了扩容。
  - 如果不是，此时就让 **sc** 为负数让当前线程成为第一个扩容的线程，再通过 **CAS** 和 **位运算** 来保证只有一个线程能发起扩容。

- **transfer(tab, nt)** 和 **transfer(tab, null)**

  - **nextTable** 表示扩容后的新数组，如果为 **null** ，就代表首次扩容，将再扩容的数组和扩容前的数组传入准备数据转移。

  - 否则就代表，已经有线程在进行扩容了，此时加入数据迁移大队。

    

## transfer 方法

- 通过 **tryPresize** 我们发现调用了 **transfer** 方法，该方法可以被多个线程同时调用，是“数据迁移”的核心操作方法。

```java
/**
* 数据转移和扩容，
* 每个调用 tranfer方法 的线程会对
* 当前旧哈希表中[transferIndex-stride, transferIndex-1]位置的结点进行迁移。
* transferIndex：扩容的时候我们需要用到的桶下标变量。
* @param tab     旧table数组
* @param nextTab 新table数组
*/
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    
    int n = tab.length // 原哈希表容量。
    // stride 可理解成步长，在数据迁移时，每个线程要负责旧table中的多少个桶。
    // 具体是根据CPU的核心数来决定的。
    int stride;
   
    // MIN_TRANSFER_STRIDE 的默认值为 16。
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range  
    
    // 情况一：如果 nextTab 为空的话，说明哈希表为第一次扩容。
    if (nextTab == null) {
        
        try {
            @SuppressWarnings("unchecked")
            // 重新创建新的哈希表，容量为原来的两倍。
            // n 在这里依旧没变化，还是为扩容前的哈希表的长度。
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;   
        } catch (Throwable ex) {  // 处理内存溢出（OOME）的情况。
            sizeCtl = Integer.MAX_VALUE;   // 将表示容量的 sizeCtl 设置为最大值，然后返回。
            return;
        }
        nextTable = nextTab; // 设置 nextTable 变量为扩容后的数组，nextTable 是 ConcurrentHashMap 中的属性。
        // [transferIndex-stride, transferIndex-1]：表示当前线程要进行数据迁移的桶区间，transferIndex 也是 ConcurrentHashMap 的属性。
        transferIndex = n;  
    }
    
    // 获得扩容后的哈希表的容量。
    int nextn = nextTab.length;
    // 新建 ForwardingNode 结点，‘
    // 当旧哈希表的某个桶中的所有结点都迁移完后，用该结点占据这个桶，说明这个桶进行了数据迁移。
    // 那么put方法的过程如果发现有个桶中有了 ForwardingNode 结点 就代表当前桶正在进行数据迁移，线程可以帮助数据迁移。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 标识一个桶的迁移工作是否完成，advance == true 表示可以进行下一个位置的迁移。
    boolean advance = true;
    // 最后一个数据迁移的线程将该值置为 true，并进行本轮扩容的收尾工作。
    boolean finishing = false; // to ensure sweep before committing nextTab 确保在提交下一个 tab 之前进行扫描。
    // i标识桶索引, bound标识边界。
    for (int i = 0, bound = 0;;) {
        
        // 新建节点用来存储当前桶的第一个节点。
        Node<K,V> f; 
        // 新建节点的哈希值。
        int fh;
        // 每一次自旋前的预处理，主要是为了定位当前线程本轮处理的桶区间。
        // 正常情况下，预处理完成后：i == transferIndex-1：右边界；bound == transferIndex-stride：左边界。
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            // 将 transferIndex 值赋给 nextIndex
            // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了。
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSetInt(this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ?
                                                                                     nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 情况一：当前是处理最后一个数据迁移的线程或出现扩容冲突。
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {  
                // 所有桶迁移均已完成。
                nextTable = null;
                table = nextTab;
                // 重新计算 sizeCtl: n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍。
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 扩容线程数减1,表示当前线程已完成自己的transfer任务
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 判断当前线程是否是本轮扩容中的最后一个线程，如果不是，则直接退出。
                // 之前我们说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2，
                // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1，
                // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务。
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit 扩容结束提交前最后检查一遍。
            }
            
            /*
             * 最后一个数据迁移线程要重新检查一次旧哈希表中的所有桶，看是否都被正确迁移到新表中了：
             * 正常情况下，重新检查时，旧的哈希表中的所有桶中应该都为 ForwardingNode 结点。
             * 特殊情况下，比如扩容冲突(多个线程申请到了同一个transfer任务)，此时当前线程领取的任务会作废，那么最后检查时，
             * 还要处理因为作废而没有被迁移的桶，把它们正确迁移到新表当中。
             */
        }
        
        // 情况二：要迁移的哈希表的这个桶本身就是 null，不用迁移，直接尝试放一个 ForwardingNode 结点告知迁移完毕。
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd); // CAS 设置成功则 advance 为 true ，表示该桶已经完成迁移。
       
        // 情况三：要迁移的哈希表的这个桶已经迁移完成。
        else if ((fh = f.hash) == MOVED)	
            advance = true; // already processed 这个旧桶已经被别的线程迁移过了，跳过。
        
        // 情况四：要迁移的哈希表的这个桶未迁移完成。
        else {		
            // 当前桶上锁！
            synchronized (f) {
                // 再次检查此时这个链表的头节点是不是还是 f，防止被别的线程修改。
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn; // 低位节点和高位结点。
                    // 情况4.1：桶的 hash>0，说明是链表迁移。
                    if (fh >= 0) {			
                        /**
                        * 下面的过程会将旧桶中的链表分成两部分：低位链和高位链
                        * ln链会插入到新哈希表的槽i中，hn链会插入到新哈希表的槽i+n中，至于为什么前面的扩容方法说明中写过。
                        */
                        int runBit = fh & n;	// 由于n是2的幂次，所以 runBit 要么是0，要么是1。
                        Node<K,V> lastRun = f; 	// lastRun指向最后一个相邻 runBit 不同的结点。
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        
                        // 以lastRun所指向的结点为分界，将链表拆成2个子链表ln、hn。
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);		// ln链表 存入新桶的索引 i 位置。
                        setTabAt(nextTab, i + n, hn); 	// hn链表 存入新桶的索引 i+n 位置。
                        // 将旧哈希表该桶位置用 ForwardingNode 结点占位用来表示该桶迁移完毕，告知别的线程不用对这个桶进行迁移了。
                        setTabAt(tab, i, fwd);			
                        advance = true;				// 表示当前旧桶的结点已迁移完毕。
                    }
                    // 情况4.2：红黑树迁移                      
                    else if (f instanceof TreeBin) {  
                        
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        // 遍历链表复制所有节点，根据高位低位分成俩链表。
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
                        
                        // 判断是否需要进行红黑树转为链表的操作。
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                                                
                        setTabAt(nextTab, i, ln);		// ln链表 存入新桶的索引 i 位置。
                        setTabAt(nextTab, i + n, hn); 	// hn链表 存入新桶的索引 i+n 位置。
                        // 将旧哈希表该桶位置用 ForwardingNode 结点占位用来表示该桶迁移完毕，告知别的线程不用对这个桶进行迁移了。
                        setTabAt(tab, i, fwd);			
                        advance = true;				// 表示当前旧桶的结点已迁移完毕。
                    }
                }
            }
        }
    }
}
```

- 通过该方法，我们可以发现，并且可以理解一些类属性和结点类的作用了。





