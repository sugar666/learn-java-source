# 集合 -- LinkedList
LinkedList的底层是一个双向链表

Node节点的属性：

```java
private static class Node<E> {
    // 节点的val
    E item;
    
    // 下一个节点
    Node<E> next;
    
    // 上一个节点
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

在进行LinkedList的操作的时候，需要去维护next，prev


```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    transient int size = 0;

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
}
```
- `first`是双向链表的头结点
- `last`是双向链表的尾结点
- 当链表中没有数据的时候，last，first指向同一个节点，`next`，`prev`都是null


## LinkedList中包含的操作

| 方法 | 抛出异常 | Deque中的实现方法 |
| :-: | :-- | :-- |
| 增加 | add(E e) 成功返回true 基于linkLast(e)实现 | offer(E e) 底层直接的调用add(E e) |
| 删除 | remove() 底层调用removeFirst(),删除头结点的值，头结点是null的话，会抛出异常 | poll()可以删除null |
| 获取 | element() 底层调用getFirst()，头结点是null的话，会抛出异常  | peek() 可以返回null |


- Queue 接口注释建议 add 方法操作失败时抛出异常，但 LinkedList 实现的 add 方法返回 true。
- LinkedList 也实现了 Deque 接口，对新增、删除和查找都提供从头开始，还是从尾开始两种方向的方法，比如 `remove` 方法，Deque 提供了 `removeFirst` 和 `removeLast` 两种方向的使用方式，但当链表为空时的表现都和 remove 方法一样，都会抛出异常


## 增加

### `boolean add(E e)` 添加到尾部
```java
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    // 链表为空的时候需要对first进行赋值
    if (l == null)
        first = newNode;
    else
        // 使用next建立链表连接
        l.next = newNode;
    size++;
    modCount++;
}
```

### 在指定的index处添加元素 void add(int index, E element）

```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        // 直接在尾部进行添加
        linkLast(element);
    else
        // 在指定的位置进行添加
        linkBefore(element, node(index));
}

// 找到指定index的节点
Node<E> node(int index) {
    // assert isElementIndex(index);

    // 使用二分查找，判断从first还是last开始查找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

// 在制定的node前插入节点   
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    
    // 建立prev连接
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

### 在头结点进行添加 `void addFirst(E e)`

```java
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

### 在尾部进行添加（与直接的使用add一样）`void addLast(E e)`

```java
public void addLast(E e) {
    linkLast(e);
}
```

###  offer
底层实现与add一样
```java
// 在尾部添加
public boolean offer(E e) {
    return add(e);
}

// 添加头结点
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

// 添加尾结点
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

### push()

```java
public void push(E e) {
    addFirst(e);
}
```
## 删除

### 删除头结点 `E remove()`
==不能删除null==
```java
public E remove() {
    return removeFirst();
}

public E removeFirst() {
    final Node<E> f = first;
    // first为空时，会抛出异常
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
    
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    
    if (next == null)
        // 更新last
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

### 删除指定索引的值 `E remove(int index)`

```java
public E remove(int index) {
    checkElementIndex(index);
    // node(index)找到指定索引处的node
    return unlink(node(index));
}

E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        // 删除头结点
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        // 删除尾结点，维护last
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}

```
### `removeLast` && `removeFirst`类似
略


### `E poll()`

```java
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

```

## 查找

### `E element()`
```java
public E element() {
    return getFirst();
}

public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

```

### `E get(int index)`

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

```

## Iterator
LinkedList 要实现双向的迭代访问，所以使用 Iterator 接口肯定不行了，因为 `Iterator` 只支持从头到尾的访问。
Java 新增了一个迭代接口: `ListIterator`

```java
private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;
    private Node<E> next;
    private int nextIndex;
    private int expectedModCount = modCount;

    ListItr(int index) {
        // assert isPositionIndex(index);
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        checkForComodification();
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() {
        checkForComodification();
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;
        unlink(lastReturned);
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }
}
```