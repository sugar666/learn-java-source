# 集合 -- HashMap

HashMap底层的数据结构主要是 ==数组 + 链表 + 红黑数==

- 当链表的长度大于等于8时，链表会转换成红黑树
- 当红黑树的长度小于等于6时，红黑树会转化成链表
- 允许null，是线程不安全的，`HashTable`是线程安全的
- `load factor`(影响因子)默认是0.75，在 需要的数组 / 数组容量 > 0.75 的时候就会触发扩容

==红黑树，链表转化的门槛为什么是8？==（在源码的类注释中有说明）
- 当链表长度大于等于 8，并且整个数组大小大于 64 时，才会转成红黑树，当数组大小小于 64 时，只会触发扩容，不会转化成红黑树。
- 链表查询的时间复杂度是 `O(n)`，红黑树的查询复杂度是 `O(log(n))`。在链表数据不多的时候，使用链表进行遍历也比较快，只有当链表数据比较多的时候，才会转化成红黑树，但红黑树需要的占用空间是链表的 2 倍，考虑到转化时间和空间损耗，所以我们需要定义出转化的边界值。
- 参考了泊松分布概率函数

```txt
* 0:    0.60653066
* 1:    0.30326533
* 2:    0.07581633
* 3:    0.01263606
* 4:    0.00157952
* 5:    0.00015795
* 6:    0.00001316
* 7:    0.00000094
* 8:    0.00000006
* more: less than 1 in ten million
```
- 当链表的长度是 8 的时候，出现的概率是 0.00000006，不到千万分之一，所以说正常情况下，链表的长度不可能到达 8 ，而一旦到达 8 时，肯定是 hash 算法出了问题，所以在这 种情况下，为了让 HashMap 仍然有较高的查询性能，所以让链表转化成红黑树，我们正常写代码，使用 HashMap 时，==几乎不会碰到链表转化成红黑树的情况==，毕竟概念只有千万分之 一。




## HashMap中常见的属性

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    // 初始容量
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
   // 负载因子默认值
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 容量大于等于8时，链表转化成红黑树
    static final int TREEIFY_THRESHOLD = 8;

    // 容量小于等于6时，红黑树转化成链表
    static final int UNTREEIFY_THRESHOLD = 6;

   // 容量最小64时才会转会成红黑树
    static final int MIN_TREEIFY_CAPACITY = 64;

    // 用于fail-fast的，记录HashMap结构发生变化(数量变化或rehash)的数目
    transient int modCount;

    // HashMap 的实际大小，可能不准(因为当你拿到这个值的时候，可能又发生了变化)
    transient int size;

    // 扩容的门槛，如果初始化时，给定数组大小的话，通过tableSizeFor 方法计算，永远接近于 2 的幂次方
    // 如果是通过 resize 方法进行扩容后，大小 = 数组容量 * 0.75
    int threshold;

    // 存放数据的数组
    transient Node<K,V>[] table;
}
```

## HashMap中常用的方法
### 构造函数


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
    
// 指定了初始容量是，使用tableSizeFor进行容量的设置，一般在进行扩容的时候，使用的是resize()方法，resize()进行扩容时，threshold = 数组大小 * 0.75
this.threshold = tableSizeFor(initialCapacity);
}

/**
* Constructs an empty <tt>HashMap</tt> with the specified initial
* capacity and the default load factor (0.75).
*
* @param  initialCapacity the initial capacity.
* @throws IllegalArgumentException if the initial capacity is negative.
*/
public HashMap(int initialCapacity) {
this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
* Constructs an empty <tt>HashMap</tt> with the default initial capacity
* (16) and the default load factor (0.75).
*/
public HashMap() {
this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

`tableSizeFor()`

```java
static final int tableSizeFor(int cap) {
int n = cap - 1;
n |= n >>> 1;
n |= n >>> 2;
n |= n >>> 4;
n |= n >>> 8;
n |= n >>> 16;
return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

`resize()`是进行双倍的扩容

```java
// 初始化或者双倍扩容，如果是空的，按照初始容量进行初始化
// 扩容是双倍扩容
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //老数组大小大于等于最大值，不扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //老数组大小2倍之后，仍然在最小值和最大值之间，扩容成功
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {
        // zero initial threshold signifies using defaults
        //初始化
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //这里也有问题，此时的table其实是个空值，get有可能是空的
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //节点只有一个值，直接计算索引位置赋值
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //规避了8版本以下的成环问题
                else { // preserve order
                    // loHead 表示老值,老值的意思是扩容后，该链表中计算出索引位置不变的元素
                    // hiHead 表示新值，新值的意思是扩容后，计算出索引位置发生变化的元素
                    // 举个例子，数组大小是 8 ，在数组索引位置是 1 的地方挂着两个值，两个值的 hashcode 是9和33。
                    // 当数组发生扩容时，新数组的大小是 16，此时 hashcode 是 33 的值计算出来的数组索引位置仍然是 1，我们称为老值
                    // hashcode 是 9 的值计算出来的数组索引位置是 9，就发生了变化，我们称为新值。
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // java 7 是在 while 循环里面，单个计算好数组索引位置后，单个的插入数组中，在多线程情况下，会有成环问题
                    // java 8 是等链表整个 while 循环结束后，才给数组赋值，所以多线程情况下，也不会成环
                    do {
                        next = e.next;
                        // (e.hash & oldCap) == 0 表示老值链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // (e.hash & oldCap) == 0 表示新值链表
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 老值链表赋值给原来的数组索引位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 新值链表赋值到新的数组索引位置
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

### `put`

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
// 1:空数组初始化。
// 2:key计算的数组索引下，如果没有值，直接新增赋值
// 3:如果hash冲突，分成2种，一个是链表，一个是红黑树
// 4:如果当前桶已经是红黑树了。调用红黑树新增的方法
// 5:如果是链表，递归循环
// 6:链表中的元素的key有和入参key相等的，允许覆盖值的话直接覆盖
// put方法默认覆盖
// 7:如果新增的元素在链表中不存在，则新增，新增到链表的尾部
// 8:新增时，判断如果链表的长度大于等于8时，转红黑树
// 9:如果数组的实际使用大小大于等于扩容的门槛，直接扩容
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果数组为空，初始化
    // onlyIfAbsent 默认是 false，put时会进行值的覆盖
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // hashCode的算法先右移16 在并上数组大小-1
    // 如果当前索引位置是空的，直接生成新的节点在当前索引位置上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果hash冲突，当前索引上有值
    else {
        Node<K,V> e; K k;
        // 如果key equals都相等，那么当前节点就是我们要新增的
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果是红黑树，使用红黑树的方式新增
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 是个链表
        else {
            for (int binCount = 0; ; ++binCount) {
                //如果是最后一个，还找不到和新增的元素相等的，直接新增
                //节点是新增到链表最后的
                if ((e = p.next) == null) {
                    //p.next是新增的节点，但是e仍然是null
                    //e和p.next都是持有对null的引用,即使p.next后来赋予了值
                    // 只是改变了p.next指向的引用，和e没有关系
                    p.next = newNode(hash, key, value, null);
                    //新增时，链表的长度大于等于8时，链表转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //链表中有元素和新增的元素相等，结束循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                //更改循环的当前元素
                p = e;
            }
        }
        //说明新增的元素table中原来就有
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 当前节点移动到队尾
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //如果kv的实际大小大于扩容的门槛，开始扩容
    if (++size > threshold)
        resize();
    // 删除不经常使用的元素
    afterNodeInsertion(evict);
    return null;
}
```
### `get`
首先根据hash值进行查找，在根据key进行对应值的匹配，因为有可能hash值是相同的

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

// 1:根据hashcode,算出数组的索引，找到槽点
// 2:槽点的key和查询的key相等，直接返回
// 3:槽点没有next，返回null
// 4:槽点有next，判断是红黑树还是链表
// 5:红黑树调用find，链表不断循环
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //数组不为空 && hash算出来的索引下标有值，
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //hash 和 key 的 hash 相等，直接返回
        if (first.hash == hash &&
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //hash不等，看看当前节点的 next 是否有值
        if ((e = first.next) != null) {
            // 使用红黑树的查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 采用自旋方式从链表中查找 key，e 为链表的头节点
            do {
                // 如果当前节点 hash == key 的 hash，并且 equals 相等，当前节点就是我们要找的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
                // 否则，把当前节点的下一个节点拿出来继续寻找
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
### `containsKey`

```java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}
```
### `containsValue`
进行数组遍历及数组中包含的链表的遍历

```java
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```
### `getOrDefault`
`get` 与 `containsKey`的结合使用
```java
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```
### `keySet`

```java
public Set<K> keySet() {
    Set<K> ks;
    return (ks = keySet) == null ? (keySet = new KeySet()) : ks;
}
```

## 红黑树中新增的流程

1. 首先判断新增的节点在红黑树上是不是已经存在，判断手段有如下两种: 
    1. 1.1. 如果节点没有实现 Comparable 接口，使用 equals 进行判断;
    2. 1.2. 如果节点自己实现了 Comparable 接口，使用 compareTo 进行判断。
2. 新增的节点如果已经在红黑树上，直接返回;不在的话，判断新增节点是在当前节点的左边 还是右边，左边值小，右边值大;
3. 自旋递归 1 和 2 步，直到当前节点的左边或者右边的节点为空时，停止自旋，当前节点即为 我们新增节点的父节点;
4. 把新增节点放到当前节点的左边或右边为空的地方，并于当前节点建立父子节点关系;
5. 进行着色和旋转，结束。

### 红黑树的原则

红黑树不一定是平衡二叉树。最大的高度是2logn。在复杂度中还是O(logn)
红黑树保持了黑平衡（从任意的一个节点出发到达叶子节点所经过黑色的节点数量是相同的）
红黑树的性质：
- 红黑树中含有红色和黑色两种颜色的节点；
- 根节点是黑色的；
- 所有的叶子节点（空节点）都是黑色的；
- 如果一个节点是红色的，那个他的两个子节点都是黑色的；
- 任意一个节点到叶子节点，经过的黑色的节点的个数是一样的；
- 红色的节点是向左倾斜的；




