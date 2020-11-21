# Map中涉及到的问题

## `HashMap`底层的数据结构
`HashMap`底层是由**数组 + 链表 + 红黑树**
- 数组的作用主要是为了快速的查找，默认的大小是**16**，数据查询的时间复杂度是`O(1)`，key的hash的值是由`Object`中的`hashCode()`计算出来的。
    - `hashCode()`是native方法
- 当`key`的hash值是一样的时，单个node就会转化成为链表，链表查询的时间复杂度是`O(n)`，当链表的长度**大于等于8且数据的长度大于64时**，链表就会转化成为红黑树
    - 转化成为红黑树可以节省查找的时间
    - 但是红黑树的需要的储存的空间是链表的两倍
- 红黑树查询的时间复杂度是`O(logn)`，与红黑树的深度有关，当红黑树的长度**小于等于6**时，红黑树转换成为链表

## `HashMap`、`TreeMap`、`LinkedHashMap`三者的联系与区别

相同点：
- 在一定的条件下都会使用**红黑树**
- 在进行迭代的过程中，如果map的数据结构被改变了，都会出现抛出`ConcurrentModificationException`异常（**在remove的时候，需要使用迭代器**）

不同点：
- `HashMap`与`LinkedHashMap`底层都需要计算hash值，`TreeMap`不计算（LinkedHashMap继承HashMap）
- `HashMap`底层以数组为主，查询很快，`TreeMap`底层以红黑树为主，利用了红黑树左小右大的特点，可以实现key排序，适合key有序的场景，`LinkedHashMap`在`HashMap`的基础上增加了链表的结构，可以按照的插入的顺序方法和删除最少访问的数据
    - 一般的情况下都会使用`HashMap`

## `Map`中的hash算法

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
key在数组中的位置通过`tab[(n - 1) & hash]`计算出来

- 首先根据native方法`hashCode()`计算出key的hash值h
- 在将h**左移16位**
- `h ^ (h >>> 16)` : **计算出来的hash值比较的分散，不容易重复**
- 正常的思路：在计算完hash值后，需要**hash % 数组长度**，找出位于数组中的正确的位置，但是**这种取模的操作比较的慢**
    - 数学上有个公式，当 b 是 2 的幂次方时`，a % b = a &(b-1)`，因此，将取模的操作变成了**(n - 1) & hash**

### 为什么不用 key % 数组大小，而是需要用 key 的 hash 值 % 数组大小。（**因为key不一定是数字，还可能是其他的对象**）
如果 key 是数字，直接用 key % 数组大小是可以的，但key还有可能是字符串，是复杂对象，这时候用字符串或复杂对象 % 数组大小是不行的，所以需要先计算出 key 的 hash 值。

### 计算 hash 值时，为什么需要右移 16 位?（**使得计算出来的hash更加的分散，减少了碰撞的可能**）
hash 算法是 `h ^ (h >>> 16)`，为了使计算出的 hash **值更分散**，所以选择先将 h 无符号右移 16 位，然后再于 h 异或时，**就能达到 h 的高 16 位和低 16 位都能参与计算，减少了碰撞的可能性**。

### 为什么把取模换成了&操作
key.hashCode() 算出来的 hash 值还不是数组的索引下标，为了随机的计算出索引的下表位置，用 hash值和数组大小进行取模，`计算出来的索引下标比较均匀分布`。

取模操作处理器计算比较慢，处理器对 & 操作就比较擅长，换成了 & 操作，是有数学上证明的 支撑，为了提高了处理器处理的速度。

### 为什么要求数组的大小是2的幂次方（**a % b = a &(b-1)**）
以为只有在是2的幂次方的时候，取模的操作才等价于 &

## 解决hash冲突，有哪些方法
1. **一个好的hash算法**，在java中hashCode()是底层自己实现的，也可以覆写hash算法
2. **自动扩容**，当数组的容量达到阈值时（默认是0.75 * capacity），进行自动的扩容处理
3. hash冲突发生的时候，使用链表
4. hash冲突严重的时候，使用红黑树

## HashMap的源码细节
### HashMap是怎么实现扩容的
- put时，数组为空，初始化数据，默认的数组的大小是16
- 超出了阈值0.75后，就会触发扩容（`resize()`），扩容后数组的容量是之前的两倍
- 扩容的方法：新建一个数组，进行数组的复制，链表和红黑树都有自己的复制方法

### Hash冲突了怎么办
hash冲突指的是不同的key计算出来的hash值是相同的
- 当数组位置只有一个元素的时候，需要创建链表，然后将新增的元素添加到链表尾部；
- 当链表的长度大于等于8时
    - 如果数组的容量小于等于64，对数组进行扩容
    - 如果数组的容量大于64，将链表转换成为红黑树

**当数组容量小的时候，先进行数据的扩容，靠扩容解决hash冲突**

### 为什么链表长度大于等于8时需要进行扩容
链表中节点过多，在进行遍历的时候花费的时间越多，红黑树可以降低时间复杂度，通过泊松分布公式计算，正常情况下，链表个数出现8的概念不到千万分之一（源码用有介绍），在正常的情况下，链表是不会变成红黑树的

### 红黑树为什么要比变成链表
- 当红黑树的长度小于等于6的时候，红黑树变成链表
- 主要是考虑了红黑树占用的空间比较大

### HashMap在put的时候，相同的key，不希望对之前的key对应的value进行覆盖，怎么操作
- 可以选择`putIfAbsent`方法，就不会进行覆盖
    - HashMap中内置的属性`onlyIfAbsent`，为true，就不会覆盖，`put`方法，`onlyIfAbsent` 为 false，是允许覆盖的。
    - 不会覆盖 -- 表示当前的put操作是无效的

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

@Override
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 不会进行重新的赋值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 删除操作
`map.forEach((s, s2) -> map.remove("1"));`
会报错，一般在remove的时候建议使用迭代器

### 自定义的对象作为Map的key时，需要注意的问题
-  `HashMap` 的话，一定需要覆写 `equals` 和 `hashCode` 方法，因 为在 `get` 和 `put` 的时候，需要通过 `equals` 方法进行相等的判断
-  `TreeMap` 的话，需要实现 `Comparable` 接口，使用 `Comparable` 接口进行判断 `key` 的大小
-  `LinkedHashMap`和`HashMap`的要求一样，因为继承于HashMap

## LinkedHashMap中的LUR怎么实现的
`Least Recently userd` 最近最少访问。
- 在put的时候，可以覆写`removeEldestEntry()`设置删除的策略（boolean类型的方法），在put方法中，会调用`afterNodeInsertion()`方法，满足`removeEldestEntry`中的条件，就会执行删除操作，删除头结点
- 在`get`的时候，会调用`afterNodeAccess`方法，通过`accessOrder`开启该方法（设置成true，在构造函数中，默认是false），将访问的元素移动到链表的尾部，不经常访问的元素就自动的移动到了链表的首部，在进行`put`操作的时候，满足条件就会被删除


