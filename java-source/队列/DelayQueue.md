# DelayQueue
延迟队列，介绍的队列都是在`package java.util.concurrent;`

阻塞队列是在有条件限制的情况下才会阻塞，DelayQueue都会进行统一的延迟的操作，可以手动的去设置延迟的时间

## 类注释
- 底层使用PriorityQueue，实现延迟时间的排序
- 队列中元素将在过期时被执行，越靠近队头，越早过期
- 没有过期的元素不能被take
- 不能放null元素

## 基础结构

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader = null;

    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();
```

- 使用DelayQueue时，数据类型需要继承Delayed接口
    - 需要覆写getDelay()与compareTo()


```java
public interface Delayed extends Comparable<Delayed> {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
}
```

## put
put:
```java
public void put(E e) {
    offer(e);
}

public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    // 上锁
    lock.lock();
    try {
        // 使用 PriorityQueue 的扩容，排序等能力
        q.offer(e);
        // 如果恰好刚放进去的元素正好在队列头
        // 立马唤醒 take 的阻塞线程，执行 take 操作
        // 如果元素需要延迟执行的话，可以使其更快的沉睡计时
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}

```

PriorityQueue中的offer ：

```java
// 新增元素
public boolean offer(E e) {
    // 如果是空元素的话，抛异常
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    // 队列实际大小大于容量时，进行扩容
    // 扩容策略是：如果老容量小于 64，2 倍扩容，如果大于 64，50 % 扩容
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    // 如果队列为空，当前元素正好处于队头
    if (i == 0)
        queue[0] = e;
    else
    // 如果队列不为空，需要根据优先级进行排序
        siftUp(i, e);
    return true;
}

// 按照从小到大的顺序排列
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    // k 是当前队列实际大小的位置
    while (k > 0) {
        // 对 k 进行减倍
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        // 如果 x 比 e 大，退出，把 x 放在 k 位置上
        if (key.compareTo((E) e) >= 0)
            break;
        // x 比 e 小，继续循环，直到找到 x 比队列中元素大的位置
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}

```


### compareTo
需要在 compareTo 方法里面实现：通过每个元素的过期时间进行排序，如下：

```java
(int) (this.getDelay(TimeUnit.MILLISECONDS) - o.getDelay(TimeUnit.MILLISECONDS));
```

## take
有元素的过期时间到了，就能拿出数据来，如果没有过期元素，那么线程就会一直阻塞
```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return q.poll();
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

### poll没有阻塞，没有过期时间到的元素，就返回null

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll();
    } finally {
        lock.unlock();
    }
}

```



