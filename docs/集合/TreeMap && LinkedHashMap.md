# 集合 -- TreeMap && LinkedHashMap

# TreeMap

## java中常使用的两种排序的方式
[Comparable和Comparator接口的区别](https://blog.csdn.net/u010859650/article/details/85009595)

`Comparable`是排序的接口，是**内部的排序器**，使用`x.compareTo(y)`来比较x，y的大小。
- 比较的原则：
    - `x > y`: return 负数
    - `x == y`: return 0
    - `x < y`: return 正数

`Comparator`是比较器接口，是**外部比较器**，使用`compare(x,y)`进行比较，比较的方式与`compareTo()`类似



```java
public class TreeMapDemo {

    @Data
    class DTO implements Comparable<DTO> {

        private Integer id;

        public DTO(Integer id) {
            this.id = id;
        }

        @Override
        public int compareTo(DTO o) {
            //默认从小到大排序
            return id - o.getId();
        }
    }


    @Test
    public void testIterator() {
        TreeMap<String, String> m = new TreeMap<>();
        m.put("asdf", "nihao");
        m.put("sdf", "nihao");
        m.put("df", "nihao");
        m.keySet().iterator();
    }

    @Test
    public void testTwoComparable() {
        // 第一种排序，从小到大排序，实现 Comparable 的 compareTo 方法进行排序
        List<DTO> list = new ArrayList<>();
        for (int i = 5; i > 0; i--) {
            list.add(new DTO(i));
        }
        Collections.sort(list);
        log.info(JSON.toJSONString(list));

        // 第二种排序，从大到小排序，利用外部排序器 Comparator 进行排序
        Comparator comparator = (Comparator<DTO>) (o1, o2) -> o2.getId() - o1.getId();
        List<DTO> list2 = new ArrayList<>();
        for (int i = 5; i > 0; i--) {
            list2.add(new DTO(i));
        }
        // Collections.sort(list, comparator);
        Collections.sort(list, (o1,o2) -> o2.getId() - o1.getId());
        log.info(JSON.toJSONString(list2));
    }


}
```
## TreeMap
`TreeMap`底层的数据结构也是**红黑树**，与`HashMap`中红黑树的结构是一样的

`TreeMap`的特点：
- 利用了红黑树左节点小，右节点大的特点，使用key进行排序，每个插入的元素都位于正确的位置上，维护了key的大小关系，使用与key需要排序的场景中
- 底层使用的是平衡红黑树的结构，常用的方法例如：`put`，`get`，`containsKey`，`remove`的时间复杂度都是`O(logn)`

### 比较器
传进来外部的比较器，就是用外部的比较器，否则，使用内部实现的比较器

### add
`put`
```java

public V put(K key, V value) {
    Entry<K,V> t = root;
    // 没有root的时候，直接的重建红黑树
    if (t == null) {
        // compare中key不能是null
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    // 选择不同的比较器，先选择外部的比较器
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 自旋找到 key 应该新增的位置，就是应该挂载那个节点的头上
        do {
            parent = t;
            // 比较key的大小
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

`compare`
```java
// 外部比较器为空的时候，使用内部比较器
final int compare(Object k1, Object k2) {
    return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
        : comparator.compare((K)k1, (K)k2);
}
```


1. 新增节点时，就是利用了红黑树左小右大的特性，从根节点不断往下查找，直到找到节点是 null 为止，节点为 null 说明到达了叶子结点;
2. 查找过程中，发现 key 值已经存在，直接覆盖;
3. **TreeMap 是禁止 key 是 null 值的。**


# LinkedHashMap    
继承`HashMap`，拥有`HashMap`全部的特性，`put`方法直接的使用的是`HashMap`中的，独特的特征为:
- 可以按照插入的顺序进行访问
- 实现了访问最少先删除的功能，可以将很久都没有访问的key自动的进行删除（`removeEldestEntry`）


```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

## 按照插入的顺序进行访问

```java
/**
 * The head (eldest) of the doubly linked list.
 */
transient LinkedHashMap.Entry<K,V> head;

/**
 * The tail (youngest) of the doubly linked list.
 */
transient LinkedHashMap.Entry<K,V> tail;

/**
 * The iteration ordering method for this linked hash map: <tt>true</tt>
 * for access-order, <tt>false</tt> for insertion-order.
 *
 * @serial
 */
 // 控制两种访问模式的字段，默认 false
 // true 按照访问顺序，会把经常访问的 key 放到队尾 
 // false 按照插入顺序提供访问
final boolean accessOrder;

static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

```

类似于把`LinkedList`中的每个元素换成了`HashMap`中的Node --> 维护插入的顺序

`LinkedHashMap` 初始化时，默认 `accessOrder` 为 `false`，就是会按照插入顺序提供访问，插入方法使用的是父类 `HashMap` 的 `put`方法，不过覆写了 `put` 方法执行中调用的 `newNode/newTreeNode` 和 `afterNodeAccess`方法。

### 按顺序新增
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    // 将加入的元素增加到链表的尾部
    linkNodeLast(p);
    return p;
}

// link at the end of list
// 类似于LinkedList中链表的新增
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
**在put的时候，就维护了插入元素的顺序**

### 按顺序访问
只提供单向的访问，默认从头节点开始

通过迭代器进行访问，迭代器初始化的时候，默认从头节点开始访问，在迭代的过程 中，不断访问当前节点的 after 节点


```java
// Iterators

abstract class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next;
    LinkedHashMap.Entry<K,V> current;
    int expectedModCount;

    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }

    public final boolean hasNext() {
        return next != null;
    }

    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after;
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
```

### 访问最少的删除策略
LRU（Least recently used）,经常访问的元素本追加到了队尾，不经常访问的元素就自动的移动到了队首

**accessOrder为false，表示不启动LRU**
```java
@Test
public void testAccessOrder() {
    LinkedHashMap<Integer, Integer> map = new LinkedHashMap<Integer, Integer>(4, 0.75f, false) {
        {
            put(10, 10);
            put(9, 9);
            put(20, 20);
            put(1, 1);
        }

        // 在进行插入操作的时候LRU
        @Override
        protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
            return size() > 3;
        }
    };

    log.info("初始化：{}", JSON.toJSONString(map));
    Assert.assertNotNull(map.get(9));
    log.info("map.get(9)：{}", JSON.toJSONString(map));
    Assert.assertNotNull(map.get(20));
    log.info("map.get(20)：{}", JSON.toJSONString(map));

}
```
`accessOrder`为false，不会将访问的数据追加到队尾

```java
22:07:21.901 [main] INFO demo.two.LinkedHashMapDemo - 初始化：{9:9,20:20,1:1}
22:07:21.911 [main] INFO demo.two.LinkedHashMapDemo - map.get(9)：{9:9,20:20,1:1}
22:07:21.911 [main] INFO demo.two.LinkedHashMapDemo - map.get(20)：{9:9,20:20,1:1}
```
`accessOrder`为true
```java
22:08:42.672 [main] INFO demo.two.LinkedHashMapDemo - 初始化：{9:9,20:20,1:1}
22:08:42.683 [main] INFO demo.two.LinkedHashMapDemo - map.get(9)：{20:20,1:1,9:9}
22:08:42.683 [main] INFO demo.two.LinkedHashMapDemo - map.get(20)：{1:1,9:9,20:20}
```

## get

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        // 只有当accessOrder为true的时候，才会改变元素的顺序
        afterNodeAccess(e);
    return e.value;
}

void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
通过 afterNodeAccess 方法把当前访问节点移动到了队尾，其实不 仅仅是 get 方法，执行 getOrDefault、compute、computeIfAbsent、computeIfPresent、 merge 方法时，也会这么做，通过不断的把经常访问的节点移动到队尾，那么靠近队头的节点，就是很少被访问的元素了。

## 删除策略（在put的时候进行删除）

`HashMap`中的`put`方法

```java
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
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    // 调用删除策略
    afterNodeInsertion(evict);
    return null;
}
```

`afterNodeInsertion`
```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

```
可以自定义删除策略`removeEldestEntry`
需要去覆写
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```