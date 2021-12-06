# LinkedList的源码自学



## 1、今日阅读部分

### 1.1 三个基本参数以及一个重要的节点内部类

- **size** 属性代表了 **LinkedList** 目前链表中元素的个数。
- 头尾指针就是为了辨别链表哪是头哪是尾。
- 这里用到了 **transient** 关键字，之前一直没用过这个关键字，查阅文档之后才知道被该关键字修饰的成员变量在保存对象时，该属性并不会被保存，也就是不能被序列化。

```java
// 被改关键字标注的属性不可被序列化
    // 代表了 LinkedList 对象目前元素的个数。
    transient int size = 0;

    /**
     * Pointer to first node.
     * 指向第一个节点的指针。
     * 注意，这玩意是就是第一个节点，并不是指向第一个节点。
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient java.util.LinkedList.Node<E> first;
    /**
     * Pointer to last node.
     * 指向最后一个节点的指针。
     * 注意，这玩意是就是最后一个节点，并不是指向最后一个节点。
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient java.util.LinkedList.Node<E> last;
    // 这俩兄弟就是为了辨别链表该从哪里遍历出现的。

    /**
     * 为啥这个内部类藏那么深，找了好久，淦
     * 这就是一个基本的链表节点的定义，双向链表，俩指针，
     * 数据存储也是泛型所以可以保存几乎所有的数据类型。
     * @param <E>
     */
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```



### 1.2、俩构造方法

- 一个是构建空链表。
- 一个是将指定元素列表或数组变成一个 LinkedList 的链表对象。

```java
 	/**
     * Constructs an empty list.
     * 构建一个空链表
     */
    public LinkedList() {
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     * 构造包含指定集合元素的列表，按集合迭代器返回的顺序排列。
     * @param  c the collection whose elements are to be placed into this list 可以实e和其子类
     * @throws NullPointerException if the specified collection is null
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```



### 1.3、增加链表节点的几个方法（重点）

- 提供了改变链表结构的尾插法，头插法，定位插入等修改链表操作方法。
- 同时在这几个方法之上又封装了对应的几个 **add** 方法。

```java
	 /**
     * Links e as first element.
     * 将 e 作为链表的第一个元素，头插法
     */
    private void linkFirst(E e) {

        // 先新建一个临时节点 l 保存 first 节点，方便接下来判断链表是否初始为空。
        final java.util.LinkedList.Node<E> f = first;
        // 再新建一个新的节点调用内部类 Node 的构造方法来进行节点的创建。
        // 此时的构造函数的意思就是将数据元素 e 插入，并且将这个新节点的下一个节点指向目前链表中的第一个节点，也就是头插法。
        // 此时连接上了双向链表从前往后方向的那个指针。
        final java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(null, e, f);

        // 移动 first 节点为当前插入的新节点。
        first = newNode;

        // 如果 **添加元素前的 first 节点**（临时节点 f）就是空的，那就说明当前添加的元素是链表的第一个元素。
        if (f == null) {
            // 将 last 节点保存到新插入的节点处，此时 first 和 last 节点都是相同的内容。
            last = newNode;
        } else {
            // 否则的话我们从后往前指的指针，就定义为 **添加元素前的 first 节点** 所在位置（临时节点 f）的 prev 等于他。
            f.prev = newNode;
        }

        // 到这里双向链表的两条线都连成功了，结束。
        // 链表中元素个数加一。
        size++;
        // 改变了数组结构，这个标志变量也要改变。
        modCount++;
    }

    /**
     * Links e as last element.
     * 将 e 作为链表的第一个元素，尾插法，add方法中基本都是调用尾插来保证链表key的有序性
     */
    void linkLast(E e) {
        // 尾插法开始
        // 先新建一个临时节点 l 保存 last 节点，方便接下来判断链表是否初始为空。
        final java.util.LinkedList.Node<E> l = last;
        // 再新建一个新的节点调用内部类 Node 的构造方法来进行节点的创建
        // 这里的三个参数，第一个就表示将这个要插入的新节点的上一个节点指向当前的 last 所在的位置，也就连接上了双向链表中从后往前方向的指向。
        // 我们此时还差从前往后指的指向。
        final java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(l, e, null);

        // 将新节点的一段连接完成之后，我们先将此时 last 指针的位置后移到目前链表中最后一个节点上。
        last = newNode;

        // 判断！如果这个链表中 **添加元素前的 last 节点**（临时节点 l），也就是链表中最后一个节点都为空了，那么链表肯定是一个空链表，
        if (l == null) {
            // 如果确实是空链表，那么此时插入的新元素就是链表第一个元素，将 first 指向第一个元素，此时 first 和 last 节点都是相同的内容。
            first = newNode;
        } else {
            // 否则的话我们从前往后指的指针，就定义为 **添加元素前的 last 节点** 所在位置（临时节点 l）的 next 等于他。
            l.next = newNode;
        }

        // 到这里双向链表的两条线都连成功了，结束。
        // 链表中元素个数加一。
        size++;
        // 改变了数组结构，这个标志变量也要改变。
        modCount++;
    }

    /**
     * Inserts element e before non-null Node succ.
     * 在非空节点succ之前插入元素e，也就是将一个节点插入到这个链表的指定节点位置之前。
     */
    void linkBefore(E e, java.util.LinkedList.Node<E> succ) {
        // assert succ != null;
        // 新建一个临时节点保存目的节点的上一个节点，防止找不到了。
        final java.util.LinkedList.Node<E> pred = succ.prev;
        // 再新建一个节点为要插入的节点，将其调用构造方法插入到 succ 节点之前，
        final java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(pred, e, succ);
        // 插入完成之后，将 succ 节点的前一个节点设置为新插入节点。
        succ.prev = newNode;

        // 如果 **插入新元素之前的 succ 节点的前一个节点** 是空的话。
        // 就说明，succ 节点在插入新节点之前就是 first 节点。
        if (pred == null) {
            // 所以插入新节点后，新的节点就变成了第一个节点。
            first = newNode;
        } else {
            // 否则 **插入新元素之前的 succ 节点的前一个节点** 的下一个节点就是新插入的节点。
            pred.next = newNode;
        }

        // 到这里双向链表的两条线都连成功了，结束。
        // 链表中元素个数加一。
        size++;
        // 改变了数组结构，这个标志变量也要改变。
        modCount++;
    }

    /**
     * Inserts the specified element at the beginning of this list.
     * 添加一个元素到链表表头。
     * @param e the element to add
     */
    public void addFirst(E e) {
        linkFirst(e);
    }

    /**
     * Appends the specified element to the end of this list.
     * 添加一个元素到链表表尾。
     * <p>This method is equivalent to {@link #add}.
     *
     * @param e the element to add
     */
    public void addLast(E e) {
        linkLast(e);
    }

    /**
     * Appends the specified element to the end of this list.
     * 向双向链表中添加一个新节点。
     * <p>This method is equivalent to {@link #addLast}.
     * 这个方法调用了尾插法。
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    /**
     * Inserts the specified element at the specified position in this list.
     * Shifts the element currently at that position (if any) and any
     * subsequent elements to the right (adds one to their indices).
     * 就是将指定索引下标处插入一个节点。
     * @param index index at which the specified element is to be inserted
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {
        // 检查下标是否合法。
        checkPositionIndex(index);

        // 如果此时下标就是元素个数。
        if (index == size) {
         	// 那么此时就是在尾部添加一个新节点就行了。
            linkLast(element);
        } else {
            // 否则就使用指定节点前添加新节点的方法来进行就行了。
            linkBefore(element, node(index)); 
        }
    }

    /*
        我们看这个 add(E e) 添加方法，我们可以看出来，linkedList提供的基础添加方法是尾插法，
        而且提供了多种添加思路，除此之外别的也没什么感悟了。
        就是好奇为什么 linkBefore(E e, java.util.LinkedList.Node<E> succ) 方法中定位下标没用二分查找。
     */
```



### 1.4、删除链表节点的几个方法（重点）

- 同添加节点对应，都有尾部删除，头部删除，定位删除等几个底层方法。
- 同时封装了这几个底层改动链表结构的方法。

```java
	/**
     * Unlinks non-null first node f.
     * 将头节点去除，就像队列，先入先出。
     */
    private E unlinkFirst(java.util.LinkedList.Node<E> f) {
        // assert f == first && f != null;
        // 先取出要删除节点的元素，准备返回。
        final E element = f.item;
        // 新建一个节点等于头节点的下一个节点。
        final java.util.LinkedList.Node<E> next = f.next;

        // 将当前头节点的数据域和指针域都置空让 GC 去回收。
        f.item = null;
        f.next = null; // help GC

        // 所以此时 first 节点就是删除之前头节点的下一个节点。
        first = next;
        // 如果它为空
        if (next == null) {
            // 那么当前链表中删除元素之后就没有任何元素了，first 和 last 节点也就都是 null 了。
            last = null;
        } else {
            // 否则将删除节点的从后往前方向的指针置空。
            // 到这里就完全删除了一个节点了。
            next.prev = null;
        }

        // 到这里双向链表的两条线都连成功了，结束。
        // 链表中元素个数加一。
        size++;
        // 改变了数组结构，这个标志变量也要改变。
        modCount++;

        return element;
    }

    /**
     * Unlinks non-null last node l.
     * 将尾节点去除，就像双端队列，先入先出，也可以用来模拟栈。
     */
    private E unlinkLast(java.util.LinkedList.Node<E> l) {
        // assert l == last && l != null;
        // 实现过程，就和头摘法差不多，我就不写注释了，看得懂就行了。
        final E element = l.item;
        final java.util.LinkedList.Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }

    /**
     * Unlinks non-null node x.
     * 删除指定非空节点x。
     */
    E unlink(java.util.LinkedList.Node<E> x) {
        // assert x != null;
        // 先存储要删除节点的数据域。
        final E element = x.item;

        // 新建俩节点来分别标志待删除节点的前后俩节点。
        final java.util.LinkedList.Node<E> next = x.next;
        final java.util.LinkedList.Node<E> prev = x.prev;

        // 如果待删除节点的前一个节点是空的，也就说明待删除节点为头节点。
        if (prev == null) {

            // 那么我们直接将 first 指针赋值为待删除节点的下一个节点。
            first = next;
        } else {

            // 否则，待删除节点的前一个节点的下一个节点直接跳过待删除节点指向待删除节点的下一个节点。
            prev.next = next;
            // 待删除节点摘除从后往前方向上的指针。
            x.prev = null;
        }

        // 如果待删除节点的后一个节点是空的，也就说明待删除节点为尾节点。
        if (next == null) {
            // 那么我们直接将 last 指针赋值为待删除节点的上一个节点。
            last = prev;
        } else {

            // 否则，待删除节点的下一个节点的上一个节点直接跳过待删除节点指向待删除节点的上一个节点。
            next.prev = prev;
            // 待删除节点摘除从前往后方向上的指针。
            x.next = null;
        }

        // 到这里两个方向的指针都删除，新的指针也连接了。
        // 将待删除节点的数据域置空，三个东西都是空的，GC来回收了。
        x.item = null;
        // 链表中元素个数减一。
        size--;
        // 改变了数组结构，这个标志变量也要改变。
        modCount++;
        return element;
    }

    /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If this list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * {@code i} such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
     * (if such an element exists).  Returns {@code true} if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * 我读下来的大致意思就是，emmmmm，删除一个节点，该节点数据域的内容等于传入的参数，从头到尾遍历，
     * 所以是删除遍历到的第一个数据域的内容等于传入参数的节点。
     *
     * @param o element to be removed from this list, if present
     * @return {@code true} if this list contained the specified element
     */
    public boolean remove(Object o) {

        // 如果传入的值为空。
        if (o == null) {

            // 从头遍历链表节点数据域。
            for (java.util.LinkedList.Node<E> x = first; x != null; x = x.next) {

                // 如果数据域为空，将这个节点删除。
                if (x.item == null) {
                    unlink(x);

                    // 返回删除成功。
                    return true;
                }
            }
        } else {
            // 否则，数据域不为空的话。
            for (java.util.LinkedList.Node<E> x = first; x != null; x = x.next) {

                // 比较这个元素与节点数据域的内容。
                if (o.equals(x.item)) {
                    // 相同的话，删除节点。
                    unlink(x);

                    // 返回删除成功。
                    return true;
                }
            }
        }

        // 到这一步就说明没有该链表中没有任何一个节点的数据域等于传入的参数。
        // 所以返回删除失败。
        return false;
    }

    /**
     * Removes and returns the first element from this list.
     * 移除头节点。
     * @return the first element from this list 返回头节点的数据域
     * @throws NoSuchElementException if this list is empty
     */
    public E removeFirst() {

        final java.util.LinkedList.Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        // 调用我们前面看到过的 unlinkFirst 方法来进行拆除。
        return unlinkFirst(f);
    }

    /**
     * Removes and returns the last element from this list.
     * 移除尾节点。
     * @return the last element from this list 返回尾节点数据域
     * @throws NoSuchElementException if this list is empty
     */
    public E removeLast() {
        final java.util.LinkedList.Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        // 调用我们前面看到过的 unlinkLast 方法来进行拆除。
        return unlinkLast(l);
    }

    /*
        remove方法全篇看下来，又算是给我双向链表的基础巩固了一遍，链表删除和增加太简单了，时间复杂度好低啊，
        值得注意的就是他们的封装思想，将移除一个节点这件事再细分封装成俩有层级调用关系的方法，私密性大大提升。
        这里面涉及指定一个数据删除对应节点的方法，是用强行遍历的方法，是因为没有办法确定下标，很无奈，时间复
        杂度只能维持在 O(n)。
     */
```



### 1.5、两个get衍生方法

- 这俩方法就只是看看，不对链表结构进行修改。

```java
/**
     * Returns the first element in this list.
     * 得到链表中的头节点的数据域，看下面的源码可以发现不会删除节点，就是纯纯的得到罢了。
     * @return the first element in this list
     * @throws NoSuchElementException if this list is empty
     */
    public E getFirst() {
        final java.util.LinkedList.Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

    /**
     * Returns the last element in this list.
     * 得到链表中的尾节点的数据域，看下面的源码可以发现不会删除节点，就是纯纯的得到罢了。
     * @return the last element in this list
     * @throws NoSuchElementException if this list is empty
     */
    public E getLast() {
        final java.util.LinkedList.Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
```



### 1.6、重要的下标查找方法（重点）

- 这里涉及到了二分法的变种。

```java
    /**
     * Returns the (non-null) Node at the specified element index.
     */
    // linkedlist 中很重要的按下标索引查询指定节点元素的查找方法！！！！
    // 注意该方法返回值是一个 Node 节点。
    java.util.LinkedList.Node<E> node(int index) {
        // assert isElementIndex(index);

        // 看到这个右移一位，我们就可以发现，这玩意就是我们的二分查找的变种实现嘛！
        // 此时 index 就是目标值。
        // 如果此时index小于size/2的话，说明要查找的下标在当前链表的左半段。
        if (index < (size >> 1)) {
            // 新建一个临时节点，也就是遍历开始的节点为当前链表的头节点。
            java.util.LinkedList.Node<E> x = first;
            // 从左往右遍历链表左半段。
            for (int i = 0; i < index; i++) {
                // 直到找到第 index 这个位置的节点。
                x = x.next;
            }
            // 将该节点返回。
            return x;

        } else {

            // 否则就是在链表的右半段。
            java.util.LinkedList.Node<E> x = last;
            // 新建一个临时节点，也就是遍历开始的节点为当前链表的尾节点。
            for (int i = size - 1; i > index; i--) {
                // 从后往前遍历直到找到第 index 这个位置的节点。
                x = x.prev;
            }

            // 将该节点返回
            return x;
        }
    }

    /**
     * Returns the element at the specified position in this list.
     * 重要的get方法，涉及二分变种查找。
     * @param index index of the element to return
     * @return the element at the specified position in this list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        // 检查下标是否合法。
        checkElementIndex(index);
        // 开始查找。
        return node(index).item;
    }
    
    /*
        为了提高查找速度，还是使用的二分法压缩时间复杂度了，我发现，只要涉及下标查找，都会用上二分法。
        有些指定位置删除和添加元素的方法并没有使用二分法查找到待插入和待删除的位置，是为什么呢？？？
     */
```



### 1.7、类头注释（重点）

```java
/**
 * Doubly-linked list implementation of the {@code List} and {@code Deque}
 * interfaces.  Implements all optional list operations, and permits all
 * elements (including {@code null}).
 * 双向链表，实现了List和Deque接口，并且实现了所有可选列表操作，允许所有类型的参数包括null
 *
 *
 * <p>All of the operations perform as could be expected for a doubly-linked
 * list.  Operations that index into the list will traverse the list from
 * the beginning or the end, whichever is closer to the specified index.
 * 所有的操作都按照双链表的要求执行操作，查找的时候是从头节点或者尾节点开始遍历，时间复杂度为 O(n)
 *
 *
 * <p><strong>Note that this implementation is not synchronized.</strong>
 * If multiple threads access a linked list concurrently, and at least
 * one of the threads modifies the list structurally, it <i>must</i> be
 * synchronized externally.  (A structural modification is any operation
 * that adds or deletes one or more elements; merely setting the value of
 * an element is not a structural modification.)  This is typically
 * accomplished by synchronizing on some object that naturally
 * encapsulates the list.
 * 这里可以看出是线程不安全的，得从外部控制。
 *
 *
 * If no such object exists, the list should be "wrapped" using the
 * {@link Collections#synchronizedList Collections.synchronizedList}
 * method.  This is best done at creation time, to prevent accidental
 * unsynchronized access to the list:<pre>
 *   List list = Collections.synchronizedList(new LinkedList(...));</pre>
 * 可以通过 Collections 创建线程同步的链表。
 *
 *
 * <p>The iterators returned by this class's {@code iterator} and
 * {@code listIterator} methods are <i>fail-fast</i>: if the list is
 * structurally modified at any time after the iterator is created, in
 * any way except through the Iterator's own {@code remove} or
 * {@code add} methods, the iterator will throw a {@link
 * ConcurrentModificationException}.  Thus, in the face of concurrent
 * modification, the iterator fails quickly and cleanly, rather than
 * risking arbitrary, non-deterministic behavior at an undetermined
 * time in the future.
 * 如果是创建了迭代器后迭代中任何地方修改元素的值比如说add，remove，而不是使用集合内部的迭代器，那么modCount就在这时起作用了，只要一
 * 不同步，他就会抛出异常也就是fail-fast快速失败，而不是继续再不确定的情况下冒着武断、不确定的行为风险继续迭代。
 *
 *
 * <p>Note that the fail-fast behavior of an iterator cannot be guaranteed
 * as it is, generally speaking, impossible to make any hard guarantees in the
 * presence of unsynchronized concurrent modification.  Fail-fast iterators
 * throw {@code ConcurrentModificationException} on a best-effort basis.
 * Therefore, it would be wrong to write a program that depended on this
 * exception for its correctness:   <i>the fail-fast behavior of iterators
 * should be used only to detect bugs.</i>
 * 说明了这个Fail-fast是只用于检测错误的，因为非同步线程在修改链表结构的时候，说不定就凑上了，这个modCount检测不到异常，
 * 就像俩线程，一个用迭代器迭代，一个在第一个迭代的过程中删除了一个节点，那么modCount此时就和expectedModCount属性不一致，
 * 就会抛出异常，我认为这并不是说只要你使用自建的迭代器也好还是集合内部类的迭代器，他都会有可能出现，前者是可能一个线程比另一个线程遍历得块，
 * 从而导致了modCount此时就和expectedModCount不一致，后者也可能有这个问题，但是后者的可能性比较小，所以这个快速失败机制，并不一定是绝对安全的，
 * 即使是用在了封装过的集合内部迭代器中。
 *
 * <p>This class is a member of the
 * <a href="{@docRoot}/../technotes/guides/collections/index.html">
 * Java Collections Framework</a>.
 *
 * @author  Josh Bloch
 * @see     List
 * @see     ArrayList
 * @since 1.2
 * @param <E> the type of elements held in this collection
 */
```



## 2、今日感受

- **LinkedList** 这个类啊，首先是线程不安全，得在外部控制，想要得到线程安全的链表那就得调用 Collections 创建线程同步的链表。

- 它没有了空间限制，由于是链表所以不用关心初始大小还有扩容的问题，随心所欲。

- 其次它是一个双向链表，再看看它实现的接口，直接就可以浮想联翩了，这个类太强大了。

  - 我们可以拿它来模拟栈和队列，而且比数组方便一点。
  - 所以刷题可以留意一下是否可以使用它。

  - 继承了 **AbstractSequentialList**， 它呢又继承 **AbstractList** 抽象类，看了一下它并不支持随机访问因为没有实现 **RandomAccess** 接口。就是一个简化版的List，实现了 **RandomAccess** 接口的类对象，遍历时使用 get 比 迭代器更快。而 **AbstractSequentialList** 只支持迭代器按顺序访问，不支持 **RandomAccess**，所以遍历 **AbstractSequentialList** 的子类，==使用 for 循环 get() 的效率要 <= 迭代器遍历的效率==。

- 在类头注释中我看到了一个叫快速失败的错误检查机制，查阅相关资料之后，发现这玩意就是用到 **modCount** 这个参数的地方。

  - 比如你将某一个集合类进行迭代，多线程下一个线程迭代，一个线程在第一个线程迭代的过程中，remove或者添加一个节点，这就会导致线程不同步问题，那么此时为了保持数据一致性，我们依赖 **modCount** 参数的错误检查机制就开始起作用，直接抛出异常，停止这种风险很大的操作。

  - 给我的感觉就是，好狠啊写这些东西的人，他们思考解决线程不同步问题的思想，很值得我去学习。

- 看到对链表结构进行改变的删除和添加的操作的时候，我发现他们把用户要对链表做的事情以及链表该怎么做这俩件事分开封装了，比如 **add** 方法调用实际对链表的操作的 **link** 开头的方法，如果要我来写一个双向链表的话，我真的没办法写的那么完备。

- 留下了一个疑问，为什么指定位置下标插入节点的时候，遍历那个位置下标的时候没用二分法，而在 **get** 方法中用了二分法呢？

  

​     
