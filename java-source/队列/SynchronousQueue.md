# SynchronousQueue
SynchronousQueue 是比较独特的队列，其本身是没有容量大小，比如放一个数据到队列中，是不能够立马返回的，必须等待别人把我放进去的数据消费掉了，才能够返回。SynchronousQueue 在消息队列技术中间件中被大量使用，

> SynchronousQueue 的整体设计比较抽象，在内部抽象出了两种算法实现，一种是先入先出的队列，一种是后入先出的堆栈，两种算法被两个内部类实现，而直接对外的 put，take 方法的实现就非常简单，都是直接调用两个内部类的 transfer 方法进行实现

## 类注释
- 队列不存储数据，所以没有大小，也无法迭代；
- 插入操作的返回必须等待另一个线程完成对应数据的删除操作，反之亦然；
- 队列由两种数据结构组成，分别是后入先出的堆栈和先入先出的队列，堆栈是非公平的（FILO），队列是公平的（FIFO）。

## 基础结构

```java
// 堆栈和队列共同的接口
// 负责执行 put or take
abstract static class Transferer<E> {
    // e 为空的，会直接返回特殊值，不为空会传递给消费者
    // timed 为 true，说明会有超时时间
    abstract E transfer(E e, boolean timed, long nanos);
}
 
// 堆栈 后入先出 非公平
// Scherer-Scott 算法
static final class TransferStack<E> extends Transferer<E> {
}
 
// 队列 先入先出 公平
static final class TransferQueue<E> extends Transferer<E> {
}
 
private transient volatile Transferer<E> transferer;
 
// 无参构造器默认为非公平的
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}

```