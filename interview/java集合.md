# java集合

## Map
![Map的类关系图](./media/16117374856458/Map%E7%9A%84%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

#### HashTable
HashTable是线程安全的，因为里面的get，put，remove是有synchronized关键字的，保证了线程的安全性

==在HashMap中key与value都可以是null，但是在HashTable中key与value不能是null==


```java
public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
```
index的计算没有使用 & 还是使用的取模的操作

#### HashTable中的先关的问题
##### HashTable在扩容的时候为什么是2倍 + 1
HashTable中数据的长度尽量是==素数或者是奇数==，HashTable中计算key的hash值是直接的使用hashCode(),在采用取模的操作获取对应的下标index，==这样减少Hash碰撞，计算出来的数组下标更加均匀==

##### HashTable为什么采用头插法迁移数据
头插法的效率高，在JDK8之后，HashMap在进行resize()的时候采用的是尾插法，需要先去遍历链表中的所有的元素，需要消耗时间

##### JDK1.8前HashMap也是采用头插法迁移数据，多线程情况下会造成死循环，JDK1.8对HashMap做出了优化，为什么JDK1.8Hashtable还是采用头插法的方式迁移数据？
HashMap中采用头插法时，高并发时，会导致环，但是在HashTable中，使用了synchronized，是线程安全的，不会出现并发的问题

##### 为什么HashTable的使用率不是很高
使用了synchronized后，性能会降低

#### HashMap与HashTable的区别
- HashMap是线程不安全的，HashTable是线程安全的
- HashMap中key的hash值`h == k.hashCode() ^ h >>> 16` ，但是在HashTable中key的hash值是直接的使用 `int hash = key.hashCode();`
- 在HashMap中key与value都可以是null，但是在HashTable中key与value不能是null
    - 在进行put的时候会进行null的校验，为null就会出现null异常
    - 在ArrayQueue中，也不能去添加null
    - 在LinkedList中，可以为null
    - ArrayList也是可以的，但是不能模仿队列，在LinkedList中是可以去模仿队列的
- 初始容量: 在HashMap中初始的容量是16，扩容会变成之前的2倍，
    - 在HashTable中初始的容量是11，扩容的时候容量会变成之前的2倍 + 1
        - `int newCapacity = (oldCapacity << 1) + 1;`


### HashMap
HashMap在容量进行初始化的时候，要求容量是2的倍数，如果之定义的容量不是2的倍数，会使用`tableSizeFor`初始化为2的倍数，保证了`hash & (n - 1)`的正确性，有利于提高HashMap的效率

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

```
#### JDK8在 resize 的时候，通过巧妙的设计，减少了 rehash 的性能消耗。
16 -- 32 (10000)看第5位上是不是1，是1，就在之前的index上 + 16

==这个设计的巧妙之处在于，节省了一部分重新计算hash的时间，同时新增的一位为0或1的概率可以认为是均等的，所以在resize 的过程中就将原来碰撞的节点又均匀分布到了两个bucket里==

##### 怎么去知道上一次的hash值?
在Node中保存了

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

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
==每个key的hash值是不变的，在生成对应的Node的时候就进行保存了，不需要在进行更新==，这里就不需要在进行==&==操作了

#### resize()

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // oldCap > 0，是扩容而不是初始化
    if (oldCap > 0) {
        // 超过最大值就不再扩充了，并且把阈值增大到Integer.MAX_VALUE
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // oldCap 为0，oldThr 不为0，第一次初始化 table，一般是通过带参数的构造器
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // oldCap 和 oldThr 都为0的初始化，一般是通过无参构造器生成，用默认值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果到这里 newThr 还没计算，则计算新的阈值threshold
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
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
                        // 原索引（e.hash 和类似 0010000 按位与，结果为0，说明原高位为0）
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引+oldCap放到bucket里
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
#### 如果new HashMap<>(19)，bucket数组多大
==32==，32 * 0.75 = 24 储存的容量大于24的时候，就会进行resize()

#### HashMap中什么时候为数组分配内存
在第一次put的时候，如果cap == 0，就会进行resize()。在resize中，oldCap == 0，就会赋值cap == 默认值（在没有指定初始容量时），==int threshold; 决定能放入的数据量，一般情况下等于 Capacity * LoadFactor==

> ArrayList也是在第一次进行add的时候才会去分配空间，默认的大小是10，扩容是变成之前的1.5倍


```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                     // cap扩大两倍，门槛也扩大两倍
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
        // 之前的oldThr > 0表示指定了初始的容量
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
        // 没有指定初始的容量，初始的容量为0
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 对新的门槛进行重新的计算 newCap * loadFactor
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
```

初始化
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

```

#### hash算法
在JDK7与JDK8中的hash算法是不一样的
> 在JDK7中
> 
> hash为了防止只有 hashCode() 的低 bit 位参与散列容易碰撞，也采用了位移异或，只不过不是高低16bit，而是如下代码中多次位移异或。
JKD7的 hash 中存在一个开关：hashSeed。开关打开(hashSeed不为0)的时候，对 String 类型的key 采用sun.misc.Hashing.stringHash32的 hash 算法；对非 String 类型的 key，多一次和hashSeed的异或，也可以一定程度上减少碰撞的概率。
JDK 7u40以后，hashSeed 被移除，在 JDK8中也没有再采用，因为stringHash32()的算法基于MurMur哈希，其中hashSeed的产生使用了Romdum.nextInt()实现。Rondom.nextInt()使用AtomicLong，它的操作是CAS的（Compare And Swap）。这个CAS操作当有多个CPU核心时，会存在许多性能问题。因此，这个替代函数在多核处理器中表现出了糟糕的性能。

JDK7
```java
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

/**
 * 下标计算依然是使用按位与
 */
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```

[JDK7与JDK8中HashMap的比较，写的很好](https://yq.aliyun.com/articles/225660?spm=5176.10695662.1996646101.searchclickresult.12ba15f2bZ8eoS)

#### 虽然在JDK8在扩容的时候解决了死循环和数据丢失的问题
但是甚至还会出现红黑树死循环(==在并发的场景下，尽力的不要去使用HashMap==)

#### WeakHashMap与HashMap的区别是什么?
WeakHashMap 的工作与正常的 HashMap 类似，但是使用==弱引用==作为 key，意思就是当 key 对象没有任何引用时，key/value 将会被回收。

#### HashMap 初始容量设置为 10000 时，放入 10000 条数据是否需要扩容；如果初始容量设置为 1000 时，放入 1000 条数据是否需要扩容？
> 初始容量设置为10000时，hashmap的数组长度应该为16384（大于10000的2的幂次方），加载因子0.75，则threshold为12288>10000,所以不需要扩容；当初始容量设置为1000时，hashamp的数组长度应该为1024，threshold为768<1000,所以需要扩容。


## List与Set
![list和set的类结构](media/16117374856458/list%E5%92%8Cset%E7%9A%84%E7%B1%BB%E7%BB%93%E6%9E%84.png)
==map没有去继承Collection:==
- 尽管Map接口和它的实现也是集合框架的一部分，但Map不是集合，集合也不是Map。因此，Map继承Collection毫无意义，反之亦然。
- 如果Map继承Collection接口，那么元素去哪儿？Map包含key-value对，它提供抽取key或value列表集合的方法，但是它不适合“一组对象”规范。



#### ArrayList与LinkedList的联系与区别
相同点:
- 都是List接口的实现类，在功能上是相近的，存取数据都是有序的，数据是可以重复的，但是都是线程不安全的类

不同点:
- ArrayList底层是基于数组实现的
- LinkedList底层是基于双向链表实现的

#### ArrayList与LinkedList分别使用的场景
- ArrayList：基于数组实现的，在get的时候直接的可以使用数据的索引，但是在remove，插入的时候需要频繁的去复制数据，**适合读多写少的场景**
- LinkedList：底层是基于双向链表实现的，在进行get的时候需要去**遍历链表**，性能较低；在进行删除，插入的时候，只需要改变前后链表之间的指向的关系就可以了。适合于**写多读少的场景**

#### 线程安全的List类
- Vector ：底层结构类似于ArrayList，与StringBuffer，HashTable一样，是在所有的方法上都加上了synchronized关键字
- CopyOnWriteList：通过加锁和通过数组复制的方法来保证线程安全的。

#### remove与poll的区别
- poll在获取元素失败，是会返回null
- remove操作失败，会报异常


```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```



