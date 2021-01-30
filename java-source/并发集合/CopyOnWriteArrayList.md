# CopyOnWriteArrayList
位置：`java.util.concurrent`

> ArrayList是线程不安全的，如果要并发的使用，可以外部加锁，或者直接的使用`Collections.synchronizeList`（不是直接的在方法上使用synchronized，而是在方法内部使用）

```java
static class SynchronizedList<E>
    extends SynchronizedCollection<E>
    implements List<E> {
    private static final long serialVersionUID = -7754090372962971524L;

    final List<E> list;

    SynchronizedList(List<E> list) {
        super(list);
        this.list = list;
    }
    SynchronizedList(List<E> list, Object mutex) {
        super(list, mutex);
        this.list = list;
    }

    public boolean equals(Object o) {
        if (this == o)
            return true;
        synchronized (mutex) {return list.equals(o);}
    }
    public int hashCode() {
        synchronized (mutex) {return list.hashCode();}
    }

    public E get(int index) {
        synchronized (mutex) {return list.get(index);}
    }
    public E set(int index, E element) {
        synchronized (mutex) {return list.set(index, element);}
    }
    public void add(int index, E element) {
        synchronized (mutex) {list.add(index, element);}
    }
    public E remove(int index) {
        synchronized (mutex) {return list.remove(index);}
    }

    public int indexOf(Object o) {
        synchronized (mutex) {return list.indexOf(o);}
    }
    public int lastIndexOf(Object o) {
        synchronized (mutex) {return list.lastIndexOf(o);}
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        synchronized (mutex) {return list.addAll(index, c);}
    }

    public ListIterator<E> listIterator() {
        return list.listIterator(); // Must be manually synched by user
    }

    public ListIterator<E> listIterator(int index) {
        return list.listIterator(index); // Must be manually synched by user
    }

    public List<E> subList(int fromIndex, int toIndex) {
        synchronized (mutex) {
            return new SynchronizedList<>(list.subList(fromIndex, toIndex),
                                        mutex);
        }
    }
```

## 实现机制
> 写入时复制（CopyOnWrite，简称COW）思想是计算机程序设计领域中的一种优化策略。其核心思想是，如果有多个调用者（Callers）同时要求相同的资源（如内存或者是磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者视图修改资源内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变

CopyOnWriteArrayList在底层的实现与ArrayList是类似的，底层的数据结构都是数组，但是在CopyWriteArrayList中没有默认的数组的大小，在进行add，remove等操作的时候，一般有四个步骤：
- 加锁
- 从原数组中拷贝出新的数组
- 在新的数组上进行操作，并将新数组赋值给底层的数据容器
- 解锁

CopyOnWriteArrayList的底层的数据被`volatile`关键字修饰，==只要数据的内存地址被修改，其他线程就可以知道，保证了线程之间的可见性==

### 线程安全的原因
- 底层的数据结构使用了volatile关键字
- 对所有的方法都进行了加锁，解锁的操作（使用finally保证了一定是可以被解锁的）
- 对数据的修改是在新的数组上进行的

## `add`

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
- `Arrays.copyOf`：底层也是调用了native的方法，System.arraycopy
- `final ReentrantLock lock = this.lock;`：使用final关键字进行修饰保证了在加锁的过程中，锁的内存地址不会被修改
- 为什么要使用volatile关键字：
    - 如果只是简单的在原数据上修改元素，是不会触发可见性的
    - ==必须要去修改内存地址==
    - volatile修饰的成员变量在每次被线程访问时，都强迫从主内存中重读该成员变量的值。而且，==当成员变量发生变化时，强迫线程将变化值回写到主内存==。这样在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。


### 在任意index处add

```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
```
- 首先去判断index的位置，在尾部，直接的进行插入；不在，分两段进行数据的拷贝

## get
在读取的时候是不用加锁的，读的时候不需要加锁，如果读的时候有多个线程正在向CopyOnWriteArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的CopyOnWriteArrayList。

```java
public E get(int index) {
    return get(getArray(), index);
}
```
## remove

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

### 批量删除
在批量删除的时候，将不需要删除的元素放在新的数据中，而不是每进行一次删除就会拷贝一次数据，优化了程序的性能

```java
public boolean removeAll(Collection<?> c) {
    if (c == null) throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (len != 0) {
            // temp array holds those elements we know we want to keep
            int newlen = 0;
            Object[] temp = new Object[len];
            for (int i = 0; i < len; ++i) {
                Object element = elements[i];
                if (!c.contains(element))
                    temp[newlen++] = element;
            }
            if (newlen != len) {
                setArray(Arrays.copyOf(temp, newlen));
                return true;
            }
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```

## ==迭代==
在迭代的过程中，原数组的值被改变，也不会抛出`ConcurrentModificationException`，因为修改是在新数据上进行操作的，在迭代器中引用的数据的内存地址是不发生变化的


```java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}


static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    
    // 快照是一个final的对象，内存地址是不会发生变化的
    private final Object[] snapshot;
    
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public boolean hasPrevious() {
        return cursor > 0;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }

    public int nextIndex() {
        return cursor;
    }
}
```

## 使用场景与缺点
使用了并发环境下的读多写少的场景

### 缺点
存在两个问题，内存占用问题和数据一致性问题：
- 内存占用问题。因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，**旧的对象和新写入的对象**（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，**那么这个时候很有可能造成频繁的Yong GC和Full GC**。
    - 针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。
- 数据一致性问题。CopyOnWrite容器**只能保证数据的最终一致性，不能保证数据的实时一致性**。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。
    - 比如在迭代器中，不能保证数据的事实一致性

> 内存占用：可能导致频繁的GC
> 数据的一致性的问题：不能保证所有的线程访问的数据都是实时一致的