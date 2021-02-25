# ArrayBlockingQueue
- 底层的数据结构是数组
- 初始的容量确定后，无法在改变大小，和ArrayList,hashMap是不一样的
- FIFO
- 队列满的时候，put会阻塞；队列空的时候，take会阻塞（这点和LinkedBlockQueue是一样的）

## 基础结构
- put，take的时候是根据对应的索引来拿的，**在进行操作的时候需要去维护索引值**


```java
// 队列存放在 object 的数组里面
// 数组大小必须在初始化的时候手动设置，没有默认大小
final Object[] items;
 
// 下次拿数据的时候的索引位置
int takeIndex;
 
// 下次放数据的索引位置
int putIndex;
 
// 当前已有元素的大小
int count;
 
// 可重入的锁
final ReentrantLock lock;
 
// take的队列
private final Condition notEmpty;
 
// put的队列
private final Condition notFull;
```

## 初始化

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    // 队列不为空 Condition，在 put 成功时使用
    notEmpty = lock.newCondition();
    // 队列不满 Condition，在 take 成功时使用
    notFull =  lock.newCondition();
}

```
- `ReentrantLock`在初始化的时候可以选择使用锁的类型
    - true：表示使用的是公平锁，比如在take数据的时候，如果发生了阻塞，后续执行的时候按照时间的先后顺序执行阻塞的线程
    - false:非公平锁，默认是非公平锁，和synchronized是一样的

集合初始化:

```java
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}   
```

## put

```java
// 新增，如果队列满，无限阻塞
public void put(E e) throws InterruptedException {
    // 元素不能为空
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 队列如果是满的，就无限等待
        // 一直等待队列中有数据被拿走时，自己被唤醒
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
 
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1; 同一时刻只能一个线程进行操作此方法
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    // putIndex 为本次插入的位置
    items[putIndex] = x;
    // ++ putIndex 计算下次插入的位置
    // 如果下次插入的位置，正好等于队尾，下次插入就从 0 开始
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    // 唤醒因为队列空导致的等待线程
    notEmpty.signal();
}
```

## take

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 如果队列为空，无限等待
        // 直到队列中有数据被 put 后，自己被唤醒
        while (count == 0)
            notEmpty.await();
        // 从队列中拿数据
        return dequeue();
    } finally {
        lock.unlock();
    }
}
 
private E dequeue() {
    final Object[] items = this.items;
    // takeIndex 代表本次拿数据的位置，是上一次拿数据时计算好的
    E x = (E) items[takeIndex];
    // 帮助 gc
    items[takeIndex] = null;
    // ++ takeIndex 计算下次拿数据的位置
    // 如果正好等于队尾的话，下次就从 0 开始拿数据
    if (++takeIndex == items.length)
        takeIndex = 0;
    // 队列实际大小减 1
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    // 唤醒被队列满所阻塞的线程
    notFull.signal();
    return x;
}

```

## 删除数据

```java
// 一共有两种情况：
// 1：删除位置和 takeIndex 的关系：删除位置和 takeIndex 一样，比如 takeIndex 是 2， 而要删除的位置正好也是 2，那么就把位置 2 的数据置为 null ,并重新计算 takeIndex 为 3。
// 2：找到要删除元素的下一个，计算删除元素和 putIndex 的关系
// 如果下一个元素不是 putIndex，就把下一个元素往前移动一位
// 如果下一个元素是 putIndex，把 putIndex 的值修改成删除的位置
void removeAt(final int removeIndex) {
    final Object[] items = this.items;
    // 情况1 如果删除位置正好等于下次要拿数据的位置
    if (removeIndex == takeIndex) {
        // 下次要拿数据的位置直接置空
        items[takeIndex] = null;
        // 要拿数据的位置往后移动一位
        if (++takeIndex == items.length)
            takeIndex = 0;
        // 当前数组的大小减一
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    // 情况 2
    } else {
        final int putIndex = this.putIndex;
        for (int i = removeIndex;;) {
            // 找到要删除元素的下一个
            int next = i + 1;
            if (next == items.length)
                next = 0;
            // 下一个元素不是 putIndex
            if (next != putIndex) {
                // 下一个元素往前移动一位
                items[i] = items[next];
                i = next;
            // 下一个元素是 putIndex
            } else {
                // 删除元素
                items[i] = null;
                // 下次放元素时，应该从本次删除的元素放
                this.putIndex = i;
                break;
            }
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}

```
- 删除数据的情况比较复杂，一共有两种情况，第一种情况是 takeIndex == removeIndex![remove](./media/16141734988343/remove.png)

- 第二种情况又分两种：
    - 如果 removeIndex + 1 != putIndex 的话，就把下一个元素往前移动一位
        - ![2.1](./media/16141734988343/2.1.png)

    - 如果 removeIndex + 1 == putIndex 的话，就把 putIndex 的值修改成删除的位置
        - ![2.2](./media/16141734988343/2.2.png)


