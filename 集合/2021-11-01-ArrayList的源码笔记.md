# ArrayList的源码自学



## 1、今日阅读部分

### 1.1、五个重要参数

- **DEFAULT_CAPACITY**（集合默认容量大小）。
- **EMPTY_ELEMENTDATA**（用于空实例的共享空数组实例）。
- **DEFAULTCAPACITY_EMPTY_ELEMENTDATA**（共享的空数组实例用于默认大小的空实例）。
- **elementData**（arraylist 中存数据的地方）。
- **size**（数据元素个数）。

```java
	/**
     * Default initial capacity.
     * 集合的默认容量为10
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     * 用于空实例的共享空数组实例。
     * 也就是说，创建为空集合的时候，集合未添加元素的时候，容量为空，此时用的就是它。
     * 所以空集合创建之后，size = 0 且 数组容量也等于空。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances.
     * 共享的空数组实例用于默认大小的空实例。
     * We distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when first element is added.
     * 我们将其与EMPTY_ELEMENTDATA区分开来，以知道在添加第一个元素时容量需要增加多少。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * 数组列表中存储元素的数组缓冲区。
     * The capacity of the ArrayList is the length of this array buffer.
     * 数组列表的容量就是这个数组缓冲区的长度。
     * Any empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     * 也就是说这个元素是保存集合中数据元素的数组。
     */
    // 所以它是 arraylist 中存数据的地方
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     * 这个不是集合的长度！是集合中元素的个数！不然一次扩容就是1.5倍，集合长度也就是某个数的倍数了。
     * @serial
     */
    private int size;
```



### 1.2、三个构造方法详读

- **ArrayList(int initialCapacity)** 指定初始容量的增长值使用。
- **ArrayList()** 构造一个空的集合，初始容量增长值用 **DEFAULT_CAPACITY** 属性来决定。
- **ArrayList(Collection<? extends E> c)** 把一个集合转换成 **Arraylist** 的对象。

```java
 	/**
     * Constructs an empty list with the specified initial capacity.
     * 构造一个具有指定初始容量的空列表。
     * @param  initialCapacity  the initial capacity of the list 列表初始化容量
     * @throws IllegalArgumentException if the specified initial capacity 非法参数异常
     *         is negative
     */
    public ArrayList(int initialCapacity) {

        // 如果给的初始容量大于0
        if (initialCapacity > 0) {
            // 那么我们给arraylist中最重要的数据元素存放属性 elementData 指向一个新建的容量为 initialCapacity 大小的 object 数组。
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {

            // 如果传入的初始容量为0
            // 我们就将此时的 elementData 指向他，说明集合中没有元素并且长度为空。
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            // 否则就是抛出一个非法参数异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }

    /**
     * Constructs an empty list with an initial capacity of ten.
     * 构造一个初始容量为10的空列表。
     */
    public ArrayList() {
        // 用到了我们的第二个属性
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Constructs a list containing the elements of the specified collection,
     * 构造包含指定集合元素的列表。
     * in the order they are returned by the collection's iterator.
     * 按集合迭代器返回的顺序排列。
     * @param c the collection whose elements are to be placed into this list 实现了Collection接口并且类型一定是E的子类。
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        // 这个 Collection 类中的 toArray 方法是把一个集合转化为一个 object 类数组。
        Object[] a = c.toArray();

        // 如果此时集合中元素的个数不为0的话。
        if ((size = a.length) != 0) {

            // 如果 c 这个对象运行时的类为 ArrayList 的话。
            if (c.getClass() == java.util.ArrayList.class) {

                // 这里的 elementData 就为传入的这个集合，直接赋值。
                elementData = a;
            } else {

                // 如果 c 这个对象运行时的类不为 ArrayList 的话。
                // 这里就把这个 c 集合转换为 object 数组再赋值。
                // 这里的 elementData 就复制指定的数组a，数组元素个数为size，新的类型为object类型。
                elementData = Arrays.copyOf(a, size, Object[].class);
            }
        } else {

            // replace with empty array.
            // 否则就返回一个空集合，用到了第二个属性。
            elementData = EMPTY_ELEMENTDATA;
        }
    }
```



### 1.3、添加方法详读

```java
    // ***********************************************************************
    /**
     * Appends the specified element to the end of this list.
     * 将指定的元素追加到列表的末尾。（重点add方法）
     * @param e element to be appended to this list 要追加的元素，万物皆可对象
     * @return <tt>true</tt> (as specified by {@link Collection#add}) 插入成功的话就返回真
     */
    public boolean add(E e) {

        // 先确保集合内部容量是否够用！之后再添加，如果这里不够用了，那么就是得给集合扩容了！！！
        // 先去扩容入口判断一下是否需要扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 得到最新容量之后的 elementData 数组就可以往末尾添加新元素了！！！
        // 这里也可以看出为什么 arraylist 是有序的！
        elementData[size++] = e;

        // 返回添加成功！
        return true;
    }
    /**
     * Inserts the specified element at the specified position in this
     * list. Shifts the element currently at that position (if any) and
     * any subsequent elements to the right (adds one to their indices).
     * 将指定的元素追加到列表的指定位置！！！（重点add方法）
     * @param index index at which the specified element is to be inserted 指定的位置！
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {

        // 检查下标是否合法
        rangeCheckForAdd(index);

        // 确保数组能够容纳插入值后的元素个数！！！
        // 再去扩容入口判断一下是否需要扩容
        // 涉及多线程参考变量！
        ensureCapacityInternal(size + 1);  // Increments modCount!!

        // 一个原生方法，意思就是我们正常的找到要插入数据的那个下标，并且将那个下标之后的所有值往后挪一位
        // 这意味着最大时间复杂度可能到达了 O(n)
        System.arraycopy(elementData, index, elementData, index + 1,
                size - index);

        // 最后将它插入到指定下标位置
        elementData[index] = element;

        // 此时elementData中容纳元素的个数再加一，我们就完成了这种指定下标添加数据的方法。
        size++;
    }

    /*
        对比这上面这两种添加方法有感：
            直接追加的话，速度还是看得过去的，O(1) 的时间复杂度吧。
            但是第二种最坏情况就是 O(n) 的时间复杂度，没办法毕竟是数组，不是链表。
     */
    // ***********************************************************************
```



### 1.4、扩容方法详读（重点）

```java
/**
     * 计算容量
     * @param elementData 原数组
     * @param minCapacity 传入新参数之后要求的最小容量
     * @return
     */
    private static int calculateCapacity(Object[] elementData, int minCapacity) {

        // 先判断 elementData 使用那种构造方法来赋值的。
        // 如果此时 elementData 属性是 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 的话，他就有10的容量增量。
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            // 返回默认容量和传入新参数之后要求的最小容量中最大的那个。
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        // 否则返回传入新参数之后要求的最小容量。
        return minCapacity;
    }

    /**
     *  扩容入口吧可以看作
     *  确保集合内部容量是否够用！！！！！！，此方法在 **add** 方法中被调用！！！
     * @param minCapacity 传入新参数之后要求的最小容量！！！
     */
    private void ensureCapacityInternal(int minCapacity) {

        // 计算完容量之后进入扩容阶段！！！
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++; // 给线程的标记，他在集合结构改变之后会加一，用于在多线程中判断一些事情。

        // overflow-conscious code
        // 考虑是否会溢出。
        // 如果此时 **传入新参数之后要求的最小容量** minCapacity 大于目前 elementData.length 值。
        // 注意这里比较的是 elementData 的容量！！！而不是元素的个数！！！
        if (minCapacity - elementData.length > 0)

            // 超出的话，我们就得给集合扩容！！！
            grow(minCapacity);
    }

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    // 要分配的数组的最大大小，如果容量大小超过int的最大大小，就将容量强行赋值为 int 的最大长度 - 8的长度也就是 2147483639
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     * 增加容量，以确保它能容纳至少当前需要的最小元素容量那么多的元素。
     * @param minCapacity the desired minimum capacity 传入新参数之后要求的最小容量！！！
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        // 继续考虑溢出！

        // 先记录下老容量，此时老容量就是 elementData 数组的长度。
        int oldCapacity = elementData.length;

        // 新的容量等于老容量加上老容量的值右移一位，这种位运算，我们这些学计算机的基础，这个右移一位也就是将原数 ÷ 2；
        // 所以我们这里得到的新容量为 老容量的 1.5 倍。
        int newCapacity = oldCapacity + (oldCapacity >> 1);

        // 如果扩容之后的新容量还比 **传入新参数之后要求的最小容量** 还要小的话
        if (newCapacity - minCapacity < 0) {
            // 那么新的容量不如就等于 **传入新参数之后要求的最小容量**
            newCapacity = minCapacity;
        }

        // 如果扩容之后的新容量比我们 arraylist 中的元素 MAX_ARRAY_SIZE 还要大的话！
        // MAX_ARRAY_SIZE 相当于一个 arraylist 对象所能存储的最大元素个数！
        if (newCapacity - MAX_ARRAY_SIZE > 0) {

            // 那么我们就判断一下此时 newCapacity 到底是得等于int的最大长度呢还是等于int的最大长度-8呢。
            newCapacity = hugeCapacity(minCapacity);
        }

        // minCapacity is usually close to size, so this is a win:
        // 因为 传入新参数之后要求的最小容量 非常接近于元素的个数，所以我们直接自己拷贝自己，但是唯一不同的就是，容量变成了新的容量！！！
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    // 这个方法就是给 grow 函数打辅助用的。
    // 我看下来是用来给 grow 函数的 newCapacity 属性超过了 arraylist 要分配的数组的最大大小之后进行调用的。
    private static int hugeCapacity(int minCapacity) {

        // 如果此时 **传入新参数之后要求的最小容量** 小于0
        if (minCapacity < 0) // overflow

            // 那就溢出了，抛出一个内存溢出异常。
            throw new OutOfMemoryError();

        // 否则如果 传入新参数之后要求的最小容量 大于 arraylist 可以提供的最大元素容量的话
        // 就返回 int 类型的最大容量给新容量 grow函数中的 newCapacity 属性
        // 否则就返回 arraylist 中可以提供的最大元素容量。
        // Integer.MAX_VALUE 和 MAX_ARRAY_SIZE 的不同之处就在于后者比前者少8！！
        return (minCapacity > MAX_ARRAY_SIZE) ?
                Integer.MAX_VALUE :
                MAX_ARRAY_SIZE;
    }
```



### 1.5、get和set方法的详读

- **get(int index)** 方法用来返回列表中指定下标处的元素。
- **set(int index, E element)** 方法用来将列表中指定位置的元素替换为指定的元素。
- 都没涉及到结构变化。

```java
/**
     * Returns the element at the specified position in this list.
     * 返回列表中指定位置的元素。
     * @param  index index of the element to return 目的下标索引。
     * @return the element at the specified position in this list 目的下表索引所对应的值。
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        // 先检查下标是否是非法的。
        rangeCheck(index);
        // 不是的话就直接返回目的下表索引所对应的值。
        // 这样查的好处就是，arraylist的底层数据结构是数组，靠数组索引查询，查询速度极快！不像 linkedlist 链表。
        return elementData(index);
    }

    /**
     * Replaces the element at the specified position in this list with the specified element.
     * 将列表中指定位置的元素替换为指定的元素
     *
     * @param index index of the element to replace 替换位置的下标
     * @param element element to be stored at the specified position 要替换的值
     * @return the element previously at the specified position 返回被替换的数
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {

        // 先检查下标是否是非法的
        rangeCheck(index);

        // 取出老的元素
        E oldValue = elementData(index);
        // 赋值
        elementData[index] = element;
        // 返回老元素
        return oldValue;
    }
```



## 2、今日感受



> 写写我自己得到的感触部分吧

- **ArrayList** 这个集合的底层实现是一个 **Object** 数组，这意味着他有着数组的优点和缺点。

  - 查询快，增删慢。
  - 但增删我也感觉是看情况，如果你是调用了插入到数组末尾的 **add(E e) **方法的话，除了涉及到扩容问题，个人感觉时间复杂度一般情况都在O(1)。
  - 但是如果调用的是 **add(int index, E element)** 方法的话，那么就得考虑最坏情况的时间复杂度为O(n)了，删除也是同理。
  - 所以我们这里面这个增删慢应该也是要看情况来说的。但肯定没有链表的优势大。

- 通过阅读该类的类头注释，也可以得出 **ArrayList** 它其实是线程不安全的。

- 它继承了 **AbstractList类**，实现了 **RandomAccess、Cloneable、Serializable** 这几个接口。

- 通过阅读 **add(E e)** 方法可以得出 **ArrayList** 它是有序的。

- 它的自动扩容机制，为了提高速度，用了位运算，每次扩容容量增长**1.5**倍。

- 今天仔细看了一下他的几个私有变量。发现一个问题，就是 **ArrayList** 的对象如果只是 new 了一个空的出来，那么它的初始容量并不是 **10** 而是 **0**！！

  - 证明的方法：

  - ```java
    /**
     * @author zhaolimin
     * @date 2021/11/1
     * @apiNote 测试一下我的猜想，因为调试构造函数跳不进去。
     */
    public class Example01 {
    
        public static void main(String[] args) {
    
            ArrayList<Object> objects1 = new ArrayList<>();
            // 查看这个新数组长度是多少
            System.out.println(getArrayListCapacity(objects1));
    
            ArrayList<Object> objects2 = new ArrayList<>(5);
            for (int i = 0; i < 11; i++) {
                // 我们这里弄一个正好是到第一次扩容大小
                objects2.add(0);
            }
            // 查看指定长度之后的长度
            System.out.println(getArrayListCapacity(objects2));
    
            // 我们再新建一个，插入一个数据之后再查看容量
            ArrayList<Object> objects3 = new ArrayList<>();
            objects3.add(0);
            System.out.println(getArrayListCapacity(objects3));
        }
    
        // 因为 elementData 是 Arraylist 私有的属性，我也不敢乱改源码。
        // 所以利用反射获取 elementData 的长度来验证一下对象创立之初是否是空的没有容量还是初始自带10容量。
        public static int getArrayListCapacity(ArrayList<?> arrayList) {
    
            Class<ArrayList> arrayListClass = ArrayList.class;
    
            try {
                Field field = arrayListClass.getDeclaredField("elementData");
                // 由于调用的是私有属性。
                field.setAccessible(true);
                Object[] objects = (Object[])field.get(arrayList);
    
                return objects.length;
            } catch (Exception e) {
    
                e.printStackTrace();
                return -1;
            }
        }
    }
    ```

  - 通过上面这个我测出来了，再仔细阅读那几个私有属性，他们确实一开始都没有大小。

  - 所以**如果初始化数组不带任何初始容量并且也不添加新元素的话，arraylist对象此时的容量就是0！当你添加第一个数据进去之后，才是进行了一个数组扩容！**

