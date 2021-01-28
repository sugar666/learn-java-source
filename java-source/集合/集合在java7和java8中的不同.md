# 集合在java7和java8中的不同

## 所有的集合中都新增了`forEach`方法

继承的`Iterable`接口，方法的入参`Consumer`，是函数式的接口（消费式接口，输入参数，但是没有返回值）

`ArrayList`中的`forEach`

```java
@Override
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
    for (int i=0; modCount == expectedModCount && i < size; i++) {
    	// accept就是需要执行的操作，是函数型接口中的内容
        action.accept(elementData[i]);
    }
    // 在进行循环的时候，不能进行集合的修改，如果修改就会抛出异常
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}


@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * Returns a composed {@code Consumer} that performs, in sequence, this
     * operation followed by the {@code after} operation. If performing either
     * operation throws an exception, it is relayed to the caller of the
     * composed operation.  If performing this operation throws an exception,
     * the {@code after} operation will not be performed.
     *
     * @param after the operation to perform after this operation
     * @return a composed {@code Consumer} that performs in sequence this
     * operation followed by the {@code after} operation
     * @throws NullPointerException if {@code after} is null
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```


```java
@Test
public void testForEach() {
    List<Integer> list = new ArrayList<Integer>() {{
        add(1);
        add(3);
        add(2);
        add(4);
    }};
    // value 是每次循环的入参，就是 list 中的每个元素
    list.forEach(value -> log.info("当前值为：{}", value));
}
```

`forEach` 方法上有 `@Override` 注解，说明该方法是被继承实现的，该方法是被定义在 `Iterable `接口上的，Java 7 和 8 的 `ArrayList` 都实现了该接口，但我们在 Java 7 的 `ArrayList` 并没有发现有实现该方法，这个主要是因为 `Iterable` 接口的 `forEach` 方法被加上了 `default` 关键字，这个关键字只会出现在接口类中，被该关键字修饰的方法无需强制要求子类继承

## List的区别
在`ArrayList`进行**无参**初始化的时候，java7是直接的赋值成为`10`，但是，在java8中，**初始化的时候，底层的数组是空数组，只有在第一次进行add的时候，数组才会拥有容量，默认的容量是10**

## HashMap的区别
- 与`ArrayList`一样，并不是在进行无参初始化的时候就赋予了初始的容量，**在第一次put的时候，才会扩充数组的容量**
- java7与java8中，hash算法不同，java8更加的简单，代码更加的简洁
- java8中底层增加了红黑树，在java7中，底层的数据结构是数组 + 链表

java8中的HashMap几乎全写了一次

新增的一些方法：
#### `getOrDefault`

```java
@Override
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```

#### `putIfAbsent`
map中存在key，就不会put新的值

```java
@Override
public V putIfAbsent(K key, V value) {
	// onlyIfAbsent  == true
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

#### `compute`

`compute` 方法，意思是允许我们把 key 和 value 的值进行计算后，再 put 到 map 中，为防止 key 值不存在造成未知错误，map 还提供了 `computeIfPresent` 方法，表示只有在 key 存在的时候，才执行计算


```java
@Override
public V compute(K key,
                 BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
    if (remappingFunction == null)
        throw new NullPointerException();
    int hash = hash(key);
    Node<K,V>[] tab; Node<K,V> first; int n, i;
    int binCount = 0;
    TreeNode<K,V> t = null;
    Node<K,V> old = null;
    if (size > threshold || (tab = table) == null ||
        (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((first = tab[i = (n - 1) & hash]) != null) {
        if (first instanceof TreeNode)
            old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
        else {
            Node<K,V> e = first; K k;
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    old = e;
                    break;
                }
                ++binCount;
            } while ((e = e.next) != null);
        }
    }
    V oldValue = (old == null) ? null : old.value;
    V v = remappingFunction.apply(key, oldValue);
    if (old != null) {
        if (v != null) {
            old.value = v;
            afterNodeAccess(old);
        }
        else
            removeNode(hash, key, null, false, true);
    }
    else if (v != null) {
        if (t != null)
            t.putTreeVal(this, tab, hash, key, v);
        else {
            tab[i] = newNode(hash, key, v, first);
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);
        }
        ++modCount;
        ++size;
        afterNodeInsertion(true);
    }
    return v;
}
```

## Arrays 提供了很多 parallel 开头的方法
Java 8 的 Arrays 提供了一些 parallel 开头的方法，这些方法支持并行的计算，在数据量大的时候，会充分利用 CPU ，提高计算效率，比如说 parallelSort 方法，方法底层有判断，只有数据量大于 8192 时，才会真正走并行的实现，在实际的实验中，并行计算的确能够快速的提高计算速度。

## 面试题
#### Java 8 在 List、Map 接口上新增了很多方法，为什么 Java 7 中这些接口的实现者不需要强制实现这些方法呢？

新增加的方法使用的是`default`关键字进行修饰，子类无需强制的继承该方法

#### java8中新增的方法有哪些
`getOrDefault`、`putIfAbsent`、`computeIfPresent`

#### 怎么使用`computeIfPresent`
computeIfPresent 是可以对 key 和 value 进行计算后，把计算的结果重新赋值给 key，并且如果 key 不存在时，不会报空指针，会返回 null 值。

#### HashMap 8 和 7 有啥区别？
新增了红黑树，修改了底层数据逻辑，修改了 hash 算法，几乎所有底层数组变动的方法都重写了一遍，可以说 Java 8 的 HashMap 几乎重新了一遍。
