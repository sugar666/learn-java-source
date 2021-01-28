# 集合 -- TreeSet
底层组合的是`TreeMap`，迭代的时候，也可以按照 key 的排序顺序进行迭代

`NavigableMap<E,Object> m = new TreeSet<>()`
- 好处：可以直接的使用`NavigableMap`接口中简单的方法逻辑，反过来调用`TreeMap`中复杂的逻辑，实现代码的简化

```java
private transient NavigableMap<E,Object> m;

// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();

/**
 * Constructs a set backed by the specified navigable map.
 */
TreeSet(NavigableMap<E,Object> m) {
    this.m = m;
}

public TreeSet() {
    this(new TreeMap<E,Object>());
}
```

## add

```java
public boolean add(E e) {
    return m.put(e, PRESENT)==null;
}
```

## 迭代器

```java
// m.navigableKeySet() 是 TreeMap 写了一个子类实现了 NavigableSet接口，
// 实现了 TreeSet 定义的迭代规范
// iterator是keySet中的方法
public Iterator<E> iterator() {
    return m.navigableKeySet().iterator();
}

// TreeMap中的navigableKeySet方法
public final NavigableSet<K> navigableKeySet() {
    KeySet<K> nksv = navigableKeySetView;
    return (nksv != null) ? nksv :
        (navigableKeySetView = new TreeMap.KeySet<>(this));
}
```


`TreeMap`中的`keySet`

```java
static final class KeySet<E> extends AbstractSet<E> implements NavigableSet<E> {
    private final NavigableMap<E, ?> m;
    KeySet(NavigableMap<E,?> map) { m = map; }

    public Iterator<E> iterator() {
        if (m instanceof TreeMap)
            return ((TreeMap<E,?>)m).keyIterator();
        else
            return ((TreeMap.NavigableSubMap<E,?>)m).keyIterator();
    }

    public Iterator<E> descendingIterator() {
        if (m instanceof TreeMap)
            return ((TreeMap<E,?>)m).descendingKeyIterator();
        else
            return ((TreeMap.NavigableSubMap<E,?>)m).descendingKeyIterator();
    }

    public int size() { return m.size(); }
    public boolean isEmpty() { return m.isEmpty(); }
    public boolean contains(Object o) { return m.containsKey(o); }
    public void clear() { m.clear(); }
    public E lower(E e) { return m.lowerKey(e); }
    public E floor(E e) { return m.floorKey(e); }
    public E ceiling(E e) { return m.ceilingKey(e); }
    public E higher(E e) { return m.higherKey(e); }
    public E first() { return m.firstKey(); }
    public E last() { return m.lastKey(); }
    public Comparator<? super E> comparator() { return m.comparator(); }
    public E pollFirst() {
        Map.Entry<E,?> e = m.pollFirstEntry();
        return (e == null) ? null : e.getKey();
    }
    public E pollLast() {
        Map.Entry<E,?> e = m.pollLastEntry();
        return (e == null) ? null : e.getKey();
    }
    public boolean remove(Object o) {
        int oldSize = size();
        m.remove(o);
        return size() != oldSize;
    }
    public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                  E toElement,   boolean toInclusive) {
        return new KeySet<>(m.subMap(fromElement, fromInclusive,
                                      toElement,   toInclusive));
    }
    public NavigableSet<E> headSet(E toElement, boolean inclusive) {
        return new KeySet<>(m.headMap(toElement, inclusive));
    }
    public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
        return new KeySet<>(m.tailMap(fromElement, inclusive));
    }
    public SortedSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }
    public SortedSet<E> headSet(E toElement) {
        return headSet(toElement, false);
    }
    public SortedSet<E> tailSet(E fromElement) {
        return tailSet(fromElement, true);
    }
    public NavigableSet<E> descendingSet() {
        return new KeySet<>(m.descendingMap());
    }

    public Spliterator<E> spliterator() {
        return keySpliteratorFor(m);
    }
}
```

TreeMap 实现了 TreeSet 定义的各种特殊方法。

**这种思路是 TreeSet 定义了接口的规范，TreeMap 负责去实现，实现思路和思路一是相反的。**


    TreeSet 组合 TreeMap 实现的两种思路:
    1. TreeSet 直接使用 TreeMap 的某些功能，自己包装成新的 api。
    2. TreeSet 定义自己想要的 api，自己定义接口规范，让 TreeMap 去实现。

    
    TreeMap 去实现内部逻辑，TreeSet 负责接口定义，
    TreeMap 负责具体实现， 这样子的话因为接口是 TreeSet 定义的，
    所以实现一定是 TreeSet 最想要的，TreeSet 甚至都不用包装，可以直接把返回值吐出去都行。
    
    
-------


1. 像 add 这些简单的方法，我们直接使用的是思路 1，主要是 add 这些方法实现比较简单，没有复杂逻辑，所以 TreeSet 自己实现起来比较简单;
2. 思路 2 主要适用于复杂场景，比如说迭代场景，TreeSet 的场景复杂，比如要能从头开始迭代，比如要能取第一个值，比如要能取最后一个值，再加上 TreeMap 底层结构比较复杂， TreeSet 可能并不清楚 TreeMap 底层的复杂逻辑，这时候让 TreeSet 来实现如此复杂的场景逻辑，TreeSet 就搞不定了，不如接口让 TreeSet 来定义，让 TreeMap 去负责实现，TreeMap 对底层的复杂结构非常清楚，实现起来既准确又简单。