# HashTable的源码自学



## 简述

- 他是线程安全的，不过使用 **synchronized** 关键字修饰方法，就带来一个问题，并发情况下效率低下。

```java
public synchronized V put(K key, V value);
public synchronized V get(Object key);
// ...
```



## 数据结构

- **数组 + 链表**，==没有红黑树==。



## 属性

```java
// 底层数据结构。
private transient Entry<?,?>[] table;

// 集合中键值对个数。
private transient int count;

// 扩容阈值。
private int threshold;

// 加载因子。
private float loadFactor;

// 快速失败检测变量。
private transient int modCount = 0;

// 序列化UID。
private static final long serialVersionUID = 1421746759512286392L;
```



## 初始化

- 指定容量以及负载因子构造函数。

```java
public Hashtable(int initialCapacity, float loadFactor) {
    // 容量合法性检查。
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    
    this.loadFactor = loadFactor;
    table = new Entry<?,?>[initialCapacity];
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}
```



### 指定容量构造函数

- 负载因子默认 **0.75f**。

```java
public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}
```



### 无参构造函数

- 默认容量为 **11**，默认负载因子为 **0.75f**。

```java
public Hashtable() {
    this(11, 0.75f);
}
```



### 指定集合转为 HashTable 构造函数

```java
public Hashtable(Map<? extends K, ? extends V> t) {
    this(Math.max(2*t.size(), 11), 0.75f);
    putAll(t);
}
```



## 添加元素有关方法



### put 方法

- 与 **HashMap** 不同，它不允许 **value** 为空。
- 同步方法。

```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    
    // 索引计算方式也不同，没有全用位运算，降低了性能。
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    
    // 循环查找替换，如果是 key 已经存在，那就用新 value 换旧 value。
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    // 如果没有 key 重复，那就添加到链表表尾。
    addEntry(hash, key, value, index);
    return null;
}
```



### addEntry 方法

- 用来添加新结点的方法。

```java
private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        // 如果我们此时插入数据后的总容量大于扩容阈值，那就扩容。
        rehash();

        // 扩容后的哈希表。
        tab = table;
        hash = key.hashCode();
        // 计算索引。
        // 计算索引时将哈希值值先与上 0x7FFFFFFF ,这是为了保证hash值始终为正数。
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    // 将新结点插入扩容后的哈希表中。
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```



## 扩容



### rehash 方法

- **HashTable** 的扩容方法。

```java
protected void rehash() {
    // 记录旧表容量。
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    // 计算新表容量为 旧表容量*2 + 1。
    int newCapacity = (oldCapacity << 1) + 1;
    // 扩容后容量合法性检查。
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    
    // 用扩容后容量，初始化新表。
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    // 计算新的扩容阈值。
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    
    // 旧表换新表。
    table = newMap;

    // 遍历将旧表中的元素添加到新表中。
    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            // 计算索引时将哈希值值先与上 0x7FFFFFFF ,这是为了保证hash值始终为正数。
            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```





## 总结

- 在数据迁移阶段，我们就可以看出 **HashaTable** 和 **HashMap** 的区别。

  - **HashaTable** 是全部元素重新计算下标，而且是模运算，性能不高，并且牺牲了一部分的均匀分布。
  - **HashMap** 将原链表或树一分为二，高位链表在新表中 **i + n** 处，低位链表在新表中的位置不变。

- **HashaTable** 是线程安全的。

- **HashaTable** 的键与值不能为 **null** ！**key** 调用的是原生的 **hashCode** 方法，为空也会报错。

  

  

