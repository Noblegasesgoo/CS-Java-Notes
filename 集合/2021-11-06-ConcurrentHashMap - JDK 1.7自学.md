# ConcurrentHashMap - JDK 1.7自学



## 为什么HashTable慢

- **HashTable** 虽然它是一个线程安全的类，但是它是使用 **synchronized** 关键字来修饰一些操作。
- **synchronized** 锁住了一整个哈希表。
- 导致同一时期只有一个线程可以进行对哈希表的操作。
- 所以 **HashTable** 虽然线程安全但是它慢，一个人操作所有人等待，并发值就是1。



## ConcurrentHashMap

- 在 **JDK1.5 ~JDK1.7** 中，**ConcurrentHashMap** 这个类它使用的是分段锁的机制来实现并发控制的。

  

## 数据结构

- 分段锁可以这么理解，我们原来的哈希表又在最外层套了一个数组，这个数组就被称作为 **段（Segment）**。
- 段数组中每个桶都包含了一个 **HashMap** 的哈希表，该哈希表在 **JDK1.8** 之前统一使用**数组+链表**的形式。
- 段数组中的每个段都可以独立加锁，段的默认大小为 **2^4** ，也就是默认可以有16个线程进行并发编程。
- **Segment** 通过继承 **ReentrantLock** 来进行加锁。
- 只要保证段中桶的线程安全，那么整个段也就是整个 **ConcurrentHashMap** 对象是线程安全的了。
- ==值得注意的是：**Segment**数组的长度是不可以被改变的，初始化如果不规定，那么就采用默认的 2^4==

![image-20211106094938292](C:\Users\noblegasesgoo\AppData\Roaming\Typora\typora-user-images\image-20211106094938292.png)



## 初始化

- 两个参数：
  - **initialCapacity** ： 
    - 初始容量，这个值指的是整个 **ConcurrentHashMap** 的初始容量，实际操作的时候需要平均分给每个 **Segment**。
    - 比如你初始容量是 **64**，**Segment **的容量为**16**，那么每个段中哈希表的初始容量就为 **64/16=4**。
  - **loadFactor**：
    - 这个负载因子是给 **段中哈希表** 扩容时候使用的。
- 其中某个构造函数的源码详解：

```java

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    // 非法参数判断。
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 并行级别越界重设。
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    
    // Find power-of-two sizes best matching arguments 找到2次幂大小的最佳匹配参数.
    int sshift = 0; // 偏移量吧因该是。
    int ssize = 1; // segment的数组长度。
    
    // 计算并行级别 ssize，因为要保持并行级别是 2 的 n 次方
    // 如果这里我们的并行级别是16的话。
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 到这一步，sshift = 4， ssize = 16。

    // 那么计算出 segmentShift 为 28，segmentMask 为 15，后面会用到这两个值
    this.segmentShift = 32 - sshift; // 段偏移量
    this.segmentMask = ssize - 1; // 段掩码

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    // initialCapacity 是设置整个 map 初始的大小，
    // 这里根据 initialCapacity 计算 Segment 数组中每个位置可以分到的大小。
    // 如 initialCapacity 为 64，那么每个 Segment 或称之为"槽"可以分到 4 个。
    // 那么此时段中哈希表的容量就为 4。
    int c = initialCapacity / ssize; // c 用于
    
    // 如果段中容量与segment的数组长度乘积小于了整个 ConcurrentHashMap 的初始容量，
    // 那我们就把目标容量（段中哈希表容量） + 1;
    if (c * ssize < initialCapacity)
        ++c;
    
    // 默认 MIN_SEGMENT_TABLE_CAPACITY 是 2，这个值也是有讲究的，因为这样的话，对于具体的槽上，
    // 插入一个元素不至于扩容，插入第二个的时候才会扩容
    int cap = MIN_SEGMENT_TABLE_CAPACITY; //先给此时的段中容量默认为 2 。
    // 如果段中默认容量小于目标容量的话就一直扩大到段中默认容量等于目标容量。
    while (cap < c)
        cap <<= 1;

    // 创建段数组的第一个元素 segment[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    // 创建段数组，数组长度就为 ssize 。
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // 往段数组中有序的写入 segment[0]。
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}

```

- 如果是无参构造的话，我们可以从中看到段中哈希表的 默认大小为**2**，负载因子是**0.75**，那么 扩容阈值就是**1.5**，插入第一个元素不会扩容。
- 那么到这整个 **ConcurrentHashMap** 对象就创建完毕了，但是有一个疑问：
  - 在末尾初始化了段表中的第一个位置，其他位置没有初始化，这是为什么呢？
  - 是因为每次创建一个 **Segment **对象要计算好几个值，初始化 **ConcurrentHashMap **的时候初始化了一个==s0==，只有再要初始化的 **Segment** 对象的时候，就拿==s0==当模板直接照搬参数就行，这样就会快一点。



## put方法及其过程分析

- 先来看看put方法的源码。

```java

public V put(K key, V value) {
    // 先建一个段的临时桶。
    Segment<K,V> s;
    // 如果要put的值为空的话就抛出异常。
    if (value == null)
        throw new NullPointerException();
    
    // 计算 key 的 hash 值
    int hash = hash(key);
    
    // 根据 hash 值找到段表中的位置 j。
    // hash 是 32 位，无符号右移 segmentShift(28) 位，剩下高 4 位，
    // 然后和 segmentMask(15) 做一次与操作，也就是说 j 是 hash 值的高 4 位，也就是槽的数组下标
    /*
    	0000 0000 0000 0000 0000 0000 0000 1101
    	0000 0000 0000 0000 0000 0000 0000 1111
    										&
    	----------------------------------------
    	0000 0000 0000 0000 0000 0000 0000 1101
    */
    
    int j = (hash >>> segmentShift) & segmentMask;
    // 刚刚说了，初始化的时候初始化了 segment[0]，但是其他位置还是 null。
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j); // 初始化 j 处的桶。
    // 插入新值到 s 中。
    return s.put(key, hash, value, false);
}

```

- 上面这个 **put方法** 是给用户调用的接口。
- 读完之后不难发现，他就做了一件事，根据待插入键值对的 **key** 的 **hash** 值找到相应的段表中应该插入的桶位置，并对该位置初始化。
- 再之后详细操作就是段的put操作了。

```java

final V put(K key, int hash, V value, boolean onlyIfAbsent) {

    // 再 put 到指定段中之前，我们得获取到当前段表中这个桶的独占的锁。
    // 以此来保证整个过程，只有我们一个线程在对这个桶做操作。
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
   
    V oldValue; // 用来存储被覆盖的值。
    
    try {
        // 这个是段表某个桶内部的哈希表。
        HashEntry<K,V>[] tab = table;
        
        // 再次利用待插入键值对 key 的 hash 值，求应该放置的数组下标。
        int index = (tab.length - 1) & hash;
        // first 是桶中哈希表的待插入桶处的链表的表头。
        HashEntry<K,V> first = entryAt(tab, index);

        // 遍历链表。
        for (HashEntry<K,V> e = first;;) {
            // 这个遍历主要是为了找出，key重复的情况。
            if (e != null) {
                K k;
                // 如果当前链表节点的key重复了的话。
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k)))
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        // 覆盖旧的值。
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
            
                e = e.next;
            }
            else {
                // node 到底是不是 null，这个要看获取锁的过程，不过和这里都没有关系。
                // 如果不为 null。
                if (node != null)
                    // 那就把它设置为链表表头，JDK1.7使用头插法。
                    node.setNext(first);
                else
                    // 否则如果是null，就先初始化它再设置为链表表头。
                    node = new HashEntry<K,V>(hash, key, value, first);

                int c = count + 1;
                // 如果超过了段表中该桶中哈希表的阈值，这个哈希表就需要扩容。
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node); // 扩容。
                else
                    // 没有达到阈值，将 node 放到哈希表的 index 位置，
                    // 其实就是将新的节点设置成原链表的表头，使用头插法。
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}
```

- put方法中有几个重要方法，我们下面来看看。



## ensureSegment方法（初始化段表中某个桶）

- **ConcurrentHashMap** 初始化的时候会初始化 **第一个桶segment[0]** 。
- 对于其他桶来说，在插入第一个值的时候进行初始化。
- 这里需要考虑并发，因为很可能会有多个线程同时进来初始化同一个槽 **segment[k]**，不过只要有一个成功了就可以。
- 该方法中使用了 **getObjectVolatile** 这个方法的意思就是：
  - 线程之间互相可看到一些透明的资源，一个线程要是对某的资源进行了操作，那么别的线程都可以看到操作后的结果。
  - 可以防止一个线程对资源进行 **6 - 1 = 5**，而别的线程还是看这个资源为 **6** 的情况。
  - 这个方法中比较有意思的还有最后一个 **if判断**。
    - CAS操作，这里的比较并交换在CPU里面就是一条指令，保证原子性的。
    - 也就是说，不会出现，我比较完了还没赋值，CPU就切换到下一个线程了。

```java

private Segment<K,V> ensureSegment(int k) {
    // 临时段表。
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    // 注意，这里是getObjectVolatile这个方法，这个方法的意思就是，别的线程要是修改了segment[k]，这个线程是可见的。
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 这里看到为什么之前要初始化 segment[0] 了，
        // segment[0] 就相当于一个初始化模板，
        // 使用当前 segment[0] 处的数组长度和负载因子来初始化 segment[k]，
        // 为什么要用“当前”，因为 segment[0] 可能早就扩容过了。
        Segment<K,V> proto = ss[0]; // 获取当前 segment[0] 作为初始化模板。
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);

        // 初始化 segment[k] 内部的哈希表。
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        // 再次检查一遍该槽是否被其他线程初始化了。
        // 也就是在做上面那些操作时，看看是否有别的线程操作过 segment[k]。
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { 
			
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab); // 构建新的 segment 对象。
       
            // 再次检查 segment [k] 是否为null。
            // 注意，这里是while，之前的if，也是起到如果下面操作失败，再次检查的作用。
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    // CAS操作，这里的比较并交换在CPU里面就是一条指令，保证原子性的。
                    // 不存在那种比较完毕之后的间隙，突然切换到别的线程来修改这个值的情况。
                    break;
            }
        }
    }
    
    // 返回 segment 对象。
    // 这里返回的seg可能是自己new的，也可能是别的线程new的，反正只要其中一个就好了。
    return seg;
}
```



## scanAndLockForPut方法（获取写入锁）

- 我们可以在上面的方法中看到这个方法的身影，这个方法也就是==并发环境下控制的重要方法之一==。

```java

private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    // 简单来说就是拿到段表中某个桶中的哈希表数组中通过hash计算的那个下标下的第一个节点。
    HashEntry<K,V> first = entryForHash(this, hash);
    // 备份一下当前节点的内容。
    HashEntry<K,V> e = first;
    // 辅助变量。
    HashEntry<K,V> node = null;
    int retries = -1; // 用于记录获取锁的次数。

    // 循环获取锁。
    // 如果获取失败，就会去进行一些准备工作。
    while (!tryLock()) {
        // 辅助变量用于重复检查，
        // 用来检查对应段表中那个桶上的哈希表数组中对应索引桶处，之前取出来的第一个节点是否还是我们之前取得那个。
        HashEntry<K,V> f; // to recheck first below
        
        // 准备工作。
        // 因为准备工作也不需要每次循环都去做对吧，最好的预期，做一次准备工作就够了。
        if (retries < 0) {
            // 判断段表中对应桶中的哈希表的对应桶上的节点 HashEntry 是不是还没被初始化。
            if (e == null) {
               
                if (node == null) // speculatively create node
                    // 进到这里说明数组该位置的链表是空的，没有任何元素。
                    // 当然，进到这里的另一个原因是 tryLock() 失败，所以该槽存在并发，不一定是该位置。
                    // 将我们即将插进去的元素，构建成一个HashEntry节点对象。
                    node = new HashEntry<K,V>(hash, key, value, null);
                
                // 将 retries 赋值为0，不让准备工作重复执行。
                retries = 0;
            }
            // 否则的话，判断 key 是否有重复的。
            else if (key.equals(e.key))
                // 将 retries 赋值为0，不让准备工作重复执行。
                retries = 0;
            else
                // 否则顺着链表往下走。
                e = e.next;
        }
        
        // 重试次数如果超过 MAX_SCAN_RETRIES(单核1多核64)，那么不抢了，进入到阻塞队列等待锁，避免cpu空转。
        // lock() 是阻塞方法，直到获取锁后返回。
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        else if ((retries & 1) == 0 && // 偶数次数才进行后面的判断。
                 // 这个时候出现问题了，那就是有新的元素进到了链表，成为了新的表头。
                 // 也可以说是链表的表头被其他线程改变了。
                 
                 // 所以这边的策略是，相当于重新走一遍这个 scanAndLockForPut 方法。
                 (f = entryForHash(this, hash)) != first) {
            
            // 此时怎么做呢，
            // 别的线程修改了该segment的节点，重新赋值e和first为最初值，和第一二行代码一样的效果。
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    
    // 将准备工作制作好的节点返回。
    return node;
}
```

- 源码是如何判断segment中某个地方被动过的呢？
  - JDK1.7中链表使用的是头插法，所以判断第一个节点是否不一样就对了。



## entryForHash（获取对应链表的节点）

- 该方法是用来获取==段表中指定桶中哈希表对应索引处的桶中的链表的第一个节点==。

```java
static final <K,V> HashEntry<K,V> entryForHash(Segment<K,V> seg, int h) {
    HashEntry<K,V>[] tab;//赋值变量，用于记录当前 segment 中的哈希表。
    return (seg == null || (tab = seg.table) == null) ? null :// 判断当前 HashEntry 数组是否为 null。
        (HashEntry<K,V>) UNSAFE.getObjectVolatile // 如果是 null，则直接返回 null。
        (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);// 通过 hash值计算下标，然后取出哈希表中对应下标的第一个HashEntry节点。
}
```



## rehash（扩容）

- 在 **ConcurrentHashMap** 中，扩容这个概念只针对于==段表中的每个桶中的哈希表==。
- **Segments段表** 是无法扩容的。
- 在 **put方法 **中被调用。
- 该方法不需要考虑并发，因为到这里的时候，是持有该桶的独占锁的。

```java

// 传入的参数 node 是这次扩容后，需要添加到新的数组中的数据。
private void rehash(HashEntry<K,V> node) {
    // 用来存储待扩容槽中的哈希表旧表。
    HashEntry<K,V>[] oldTable = table;
    // 用来存储待扩容槽中的哈希表旧表的容量。
    int oldCapacity = oldTable.length;
    // 新的容量为旧容量的 2 倍。
    int newCapacity = oldCapacity << 1;
    // 设置新的扩容阈值。
    threshold = (int)(newCapacity * loadFactor);
    // 用上面的参数创建新的哈希表。
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    
    // 新的掩码，如从 16 扩容到 32，那么 sizeMask 为 31，对应二进制 000...00011111。
    int sizeMask = newCapacity - 1;

    // 遍历原数组，老套路，将原哈希表索引 i 处的链表拆分到新哈希表索引 i 和 i+oldCap 两个位置，高位和低位。
    for (int i = 0; i < oldCapacity ; i++) {
        // e 是链表的第一个元素。
        HashEntry<K,V> e = oldTable[i];
       
        // 如果这条链不是空链的话。
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 计算当前节点应该放置在新数组中的位置，
            // 假设原哈希表长度为 16，e 在 oldTable[3] 处，那么 e 在新哈希表中的索引 idx 只可能是 3 或者是 3 + 16 = 19。
            int idx = e.hash & sizeMask;
            
            if (next == null)   // 该位置处只有一个元素。
                // 直接将该节点设置到这里。
                newTable[idx] = e;
            else { 
                // Reuse consecutive sequence at same slot
                // e 是链表表头。
                HashEntry<K,V> lastRun = e;
                // idx 是当前链表的头结点 e 的新位置。
                int lastIdx = idx;

                // 下面这个 for 循环会找到一个 lastRun 节点，这个节点之后的所有元素是将要放到一起的。
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
			
                // 将 lastRun 及其之后的所有节点组成的这个链表放到 lastIdx 这个位置。
                newTable[lastIdx] = lastRun;
                // 下面的操作是处理 lastRun 之前的节点，
                // 这些节点可能分配在另一个链表中，也可能分配到上面的那个链表中。
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
    
    // 将新来的 node 放到新数组中刚刚的 两个链表之一 的 头部
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```



## remove（删除）

- 整个操作是先定位到段，然后委托给段的remove操作。
- 当多个删除操作并发进行时，只要它们所在的段不相同，它们就可以同时进行。

```java
public V remove(Object key) {  
   // 得到待删除的key的哈希值。
   hash = hash(key.hashCode());   
    // 根据哈希值找到对应的段表中的桶，并对该桶进行删除操作。
   return segmentFor(hash).remove(key, hash, null);   
}
```

```java
V remove(Object key, int hash, Object value) {  
    // 上锁
    lock();  
     try {  
         
         int c = count - 1;  
         HashEntry<K,V>[] tab = table;  // 段表中对应桶中的哈希表。
         int index = hash & (tab.length - 1);  // 定位key在哈希表中的位置。
         HashEntry<K,V> first = tab[index];  // 记录链表头节点。
         HashEntry<K,V> e = first;  // 再备份链表头节点。
         
         // 链表不为空并且当前节点不为要删除节点的话遍历。
         // 遍历到要删除节点终止。
         while (e != null && (e.hash != hash || !key.equals(e.key)))  
             e = e.next;  
        
         V oldValue = null;  
         // 找到了这个要删除的节点。
         if (e != null) {  
             
             // 存待删节点的数据域。
             V v = e.value;  
            
             if (value == null || value.equals(v)) {  
                 // 开始删除。
                 oldValue = v;  
 
                 // All entries following removed node can stay  
                 // in list, but all preceding ones need to be  
                 // cloned.  
                 ++modCount; // 结构更改。
               
                 HashEntry<K,V> newFirst = e.next; // 存储要删除节点的下一个节点为新的头节点。
                
                 // 从链表表头开始遍历。
                 for (HashEntry<K,V> p = first; p != e; p = p.next)  
                     // 遍历到待删除结点。
                     // HashEntry 中的 next 是 final。
                     // 一经赋值以后就不可修改。
                     // 一局将待删除节点之前的节点都弄到连到待删除节点的下一个节点。
                     newFirst = new HashEntry<K,V>(p.key, p.hash,  
                                                   newFirst, p.value);  
                 
                 // 最后再将哈希表对应桶中的原来的链丢弃，将以待删除节点下一个节点为头节点的链表填入。
                 tab[index] = newFirst;  
                 count = c; // write-volatile  
             }  
         } 
         
         return oldValue;  
     } finally {  
         unlock();  
     }  
 }
```

- 至于节点为什么要设置为 **final** 不变性，这跟不变性的访问不需要同步从而节省时间有关。
- 这样节点在插入之后就不可能改变。



## ConcurrentHashMap - JDK 1.7 并发问题分析

- 看了这几个核心常用方法之后，发现唯一对结构进行了改变的也就只有 **put** 和 **remove** 方法，我们来讨论一下这俩方法的并发问题。

- **put** 方法的线程安全性：

  - 在插入的过程中，如果桶没被初始化的话，将用 **segment[0]** 来作为模板以 CAS 来初始化，保证在初始化的过程中不被别的线程打扰。
  - **JDK1.7** 中添加节点到链表的操作是插入到链表表头的，所以这个时候 **get** 方法已经遍历到链表中间，他是不会被影响的。
  - 但是如果，**get** 方法在 **put** 方法之后，需要保证刚刚插入表头的节点被读取，他们说这个依赖于 **setEntryAt** 方法中使用的 **UNSAFE.putOrderedObject** 来解决。
  - 扩容和 **HashMap** 一样是新创建了新的哈希表，然后进行迁移数据，最后面将 **newTable** 设置给属性 **table**。
  - 所以此时如果一个线程执行 **get** 方法的途中另一个线程执行 **put** 方法，那么也是没问题的，因为你要查找的肯定是旧的链表值当中的值。
  - 如果是 **put** 方法先做，那么也没有问题，因为 **table** 是被 **volatile** 关键字修饰，线程之间透明。

- **remove** 方法的线程安全性：

  - > 如果 **get** 方法的途中另一个线程执行 **remove** 方法，如果此节点是头结点，那么需要将头结点的 **next** 设置为数组该位置的元素，**table** 虽然使用了 volatile 修饰，但是 volatile 并不能提供数组内部操作的可见性保证，所以源码中使用了 **UNSAFE** 来操作数组，请看方法 **setEntryAt**。
    >
    > 著作权归https://pdai.tech所有。 链接：https://www.pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html

  - > 如果要删除的节点不是头结点，它会将要删除节点的后继节点接到前驱节点中，这里的并发保证就是 **next** 属性是 **volatile** 的。
    >
    > 著作权归https://pdai.tech所有。 链接：https://www.pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html

  - 但是如果先 **remove** 再 **get** 的话没有问题。

