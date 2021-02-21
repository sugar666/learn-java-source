# Thread

- 每个线程都有优先级，从低到高为1 ~ 10，默认的线程的优先级是5
- 父线程创建子线程后，子线程的优先级，是否是守护线程，与父类是一样的（子线程继承父线程的特性）
- JVM 启动时，通常都启动 MAIN 非守护线程，以下任意一个情况发生时，线程就会停止
    - 退出方法被调用，并且安全机制允许这么做（比如调用 Thread.interrupt 方法）；
    - 所有的非守护线程都结束了
- 每个线程都有名字，可以自定义，默认采用计数器来命名

## 线程的状态
- 初始状态（NEW），线程创建成功了，还没有进行start之前，都是NEW状态
- 运行状态：线程调用了`start`方法，变成运行状态
- 阻塞状态：线程在等待monitor lock 锁，比如在等待进入 synchronized 修饰的代码块或方法时，会从 RUNNABLE 变成 BLOCKED
- 等待状态：都表示在遇到 Object#wait、Thread#join、LockSupport#park 这些方法时，线程就会等待另一个线程执行完特定的动作之后，才能结束等待
- 超时等待状态：在等待状态的时候，可以设置超时的时间，等待设置的时间后，就可以取消等待状态
- 终止状态：线程运行结束

![线程的6中状态](./media/16139226712954/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-02-22%2000.11.53.png)

## 守护进程与非守护进程
**默认创建的都是非守护的进程**。在Thread中可daemon 属性设置成 true，变成守护进程。**守护进程的优先级比较低**，JVM在退出的时候，不关心守护进程是否执行完成
- 可能会写一些工具做一些监控的工作，这时我们都是用守护子线程去做，这样即使监控抛出异常，但因为是子线程，所以也不会影响到业务主线程，因为是守护线程，所以 JVM 也无需关注监控是否正在运行，该退出就退出，所以对业务不会产生任何影响。

#### 守护线程和非守护线程的区别？如果我想在项目启动的时候收集代码信息，请问是守护线程好，还是非守护线程好，为什么？
区别：在JVM退出的时候，如果非守护进程还在运行，就不会退出，不会管守护进程

> 如果需要在项目启动的时候收集代码信息，就需要看收集工作是否重要了，如果不太重要，又很耗时，就应该选择守护线程，这样不会妨碍 JVM 的退出，如果收集工作非常重要的话，那么就需要非守护进程，这样即使启动时发生未知异常，JVM 也会等到代码收集信息线程结束后才会退出，不会影响收集工作。


## 线程初始化的方法
线程的初始化，分为两种：
- 无参初始化
- 有参初始化

### 无参初始化
#### 自定义类，继承Thread，实现run方法

```java
// 继承 Thread，实现其 run 方法
class MyThread extends Thread{
  @Override
  public void run() {
    log.info(Thread.currentThread().getName());
  }
}
@Test
// 调用 start 方法即可，会自动调用到 run 方法的
public void extendThreadInit(){
  new MyThread().start();
}
```

start的源码：

```java
// 该方法可以创建一个新的线程出来
public synchronized void start() {
    // 如果没有初始化，抛异常
    if (threadStatus != 0)
        throw new IllegalThreadStateException();
    group.add(this);
    // started 是个标识符，我们在做一些事情的时候，经常这么写
    // 动作发生之前标识符是 false，发生完成之后变成 true
    boolean started = false;
    try {
        // 这里会创建一个新的线程，执行完成之后，新的线程已经在运行了，既 target 的内容已经在运行了
        start0();
        // 这里执行的还是主线程
        started = true;
    } finally {
        try {
            // 如果失败，把线程从线程组中删除
            if (!started) {
                group.threadStartFailed(this);
            }
         // Throwable 可以捕捉一些 Exception 捕捉不到的异常，比如说子线程抛出的异常
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
              it will be passed up the call stack */
        }
    }
}
// 开启新线程使用的是 native 方法
private native void start0();
```
#### 实现Runnable接口，并使用new Thread(Runnable)初始化Thread对象

```java
Thread thread = new Thread(new Runnable() {
  @Override
  public void run() {
    log.info("{} begin run",Thread.currentThread().getName());
  }
});
// 开一个子线程去执行
thread.start();
// 不会新起线程，是在当前主线程上继续运行
thread.run();

```

直接的去调用run方法，还是使用的是主线程：
run：

```java
// 简单的运行，不会新起线程，target 是 Runnable
public void run() {
    if (target != null) {
        target.run();
    }
}
```

**线程都是通过Thread进行实现的，调用start方法去开启子线程，在run方法中具体实现线程中的功能**

### 有参初始化
使用`FutureTask`，不在使用Runnable，而是使用有返回值的Callable接口，并重写call方法
**FutureTask中将Runnable转化成了Callable**
```java
// 首先我们创建了一个线程池
ThreadPoolExecutor executor = new ThreadPoolExecutor(3, 3, 0L, TimeUnit.MILLISECONDS,
                                                     new LinkedBlockingQueue<>());
// futureTask 我们叫做线程任务，构造器的入参是 Callable
FutureTask futureTask = new FutureTask(new Callable<String> () {
  @Override
  public String call() throws Exception {
    Thread.sleep(3000);
    // 返回一句话
    return "我是子线程"+Thread.currentThread().getName();
  }
});
// 把任务提交到线程池中，线程池会分配线程帮我们执行任务
executor.submit(futureTask);
// 得到任务执行的结果
String result = (String) futureTask.get();
log.info("result is {}",result);

```
- Callable有返回值
- Callable作为FutureTask的入参
- 通过 FutureTask.get() 得到子线程的计算结果。

#### Callable
和 Runnable 一样，不过这个线程任务是有返回值的
```java
public interface Callable<V> {
    V call() throws Exception;
}
```

#### FutureTask
继承了`RunnableFuture`， RunnableFuture 又实现了 Runnable, Future 两个接口
```java
public class FutureTask<V> implements RunnableFuture<V> {
}
```

##### Future
- 异步计算的接口，提供了计算是否完成的 check、等待完成和取回等多种方法
- 使用 get 方法，此方法(无参方法)会一直阻塞到异步任务计算完，有参方法可以设置获取超时的时间
- 可以使用cancel取消任务

![屏幕快照 2021-02-22 00.39.46](./media/16139226712954/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-02-22%2000.39.46.png)

```java
// 如果任务已经成功了，或已经取消了，是无法再取消的，会直接返回取消成功(true)
// 如果任务还没有开始进行时，发起取消，是可以取消成功的。
// 如果取消时，任务已经在运行了，mayInterruptIfRunning 为 true 的话，就可以打断运行中的线程
// mayInterruptIfRunning 为 false，表示不能打断直接返回
boolean cancel(boolean mayInterruptIfRunning);
 
// 返回线程是否已经被取消了，true 表示已经被取消了
// 如果线程已经运行结束了，isCancelled 和 isDone 返回的都是 true
boolean isCancelled();
 
// 线程是否已经运行结束了
boolean isDone();
 
// 等待结果返回
// 如果任务被取消了，抛 CancellationException 异常
// 如果等待过程中被打断了，抛 InterruptedException 异常
V get() throws InterruptedException, ExecutionException;
 
// 等待，但是带有超时时间的，如果超时时间外仍然没有响应，抛 TimeoutException 异常
V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;

```

##### RunnableFuture

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
**通过Future实现了对Runnable接口的管理，可以取消任务，查看任务时候已完成**

##### 统一 Callable 和 Runnable
新建任务有两种方式，一种是无返回值的 Runnable，一种是有返回值的 Callable，FutureTask 实现了 RunnableFuture 接口，又集合了 Callable（Callable 是 FutureTask 的属性），还提供了两者一系列的转化方法，这样 FutureTask 就统一了 Callable 和 Runnable

##### FutureTask 的构造器

```java
// 使用 Callable 进行初始化
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    // 任务状态初始化
    this.state = NEW;       // ensure visibility of callable
}
 
// 使用 Runnable 初始化，并传入 result 作为返回结果。
// Runnable 是没有返回值的，所以 result 一般没有用，置为 null 就好了
public FutureTask(Runnable runnable, V result) {
    // Executors.callable 方法把 runnable 适配成 RunnableAdapter，RunnableAdapter 实现了 callable，所以也就是把 runnable 直接适配成了 callable。
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}

```

把入参都转化成 Callable，主要是因为 Callable 的功能比 Runnnable 丰富，Callable 有返回值，而 Runnnable 没有。

入参是 Runnable 的构造器，会使用 Executors.callable 方法来把 Runnnable 转化成 Callable，Runnnable 和 Callable 两者都是接口，两者之间是无法进行转化的，所以 Java 新建了一个转化类：`RunnableAdapter` 来进行转化

```java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}

```


```java
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```

##### FutureTask 对 Future 接口方法的实现

###### get

```java
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    // 如果任务已经在执行中了，并且等待一定的时间后，仍然在执行中，直接抛出异常
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    // 任务执行成功，返回执行的结果
    return report(s);
}
// 等待任务执行完成
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    // 计算等待终止时间，如果一直等待的话，终止时间为 0
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    // 不排队
    boolean queued = false;
    // 无限循环
    for (;;) {
        // 如果线程已经被打断了，删除，抛异常
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
        // 当前任务状态
        int s = state;
        // 当前任务已经执行完了，返回
        if (s > COMPLETING) {
            // 当前任务的线程置空
            if (q != null)
                q.thread = null;
            return s;
        }
        // 如果正在执行，当前线程让出 cpu，重新竞争，防止 cpu 飙高
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
            // 如果第一次运行，新建 waitNode，当前线程就是 waitNode 的属性
        else if (q == null)
            q = new WaitNode();
            // 默认第一次都会执行这里，执行成功之后，queued 就为 true，就不会再执行了
            // 把当前 waitNode 当做 waiters 链表的第一个
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
            // 如果设置了超时时间，并过了超时时间的话，从 waiters 链表中删除当前 wait
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 没有过超时时间，线程进入 TIMED_WAITING 状态
            LockSupport.parkNanos(this, nanos);
        }
        // 没有设置超时时间，进入 WAITING 状态
        else
            LockSupport.park(this);
    }
}

```
###### run

```java
/**
 * run 方法可以直接被调用
 * 也可以开启新的线程进行调用
 */
public void run() {
    // 状态不是任务创建，或者当前任务已经有线程在执行了，直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        // Callable 不为空，并且已经初始化完成
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 调用执行
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            // 给 outcome 赋值
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}

```
###### cancel

```java
// 取消任务，如果正在运行，尝试去打断
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&//任务状态不是创建 并且不能把 new 状态置为取消，直接返回 false
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    // 进行取消操作，打断可能会抛出异常，选择 try finally 的结构
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                //状态设置成已打断
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        // 清理线程
        finishCompletion();
    }
    return true;
}

```

## 线程的初始化

```java
// 无参构造器，线程名字自动生成
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
// g 代表线程组，线程组可以对组内的线程进行批量的操作，比如批量的打断 interrupt
// target 是我们要运行的对象
// name 我们可以自己传，如果不传默认是 "Thread-" + nextThreadNum()，nextThreadNum 方法返回的是自增的数字
// stackSize 可以设置堆栈的大小
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }
 
    this.name = name.toCharArray();
    // 当前线程作为父线程
    Thread parent = currentThread();
    this.group = g;
    // 子线程会继承父线程的守护属性
    this.daemon = parent.isDaemon();
    // 子线程继承父线程的优先级属性
    this.priority = parent.getPriority();
    // classLoader
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext =
            acc != null ? acc : AccessController.getContext();
    this.target = target;
    setPriority(priority);
    // 当父线程的 inheritableThreadLocals 的属性值不为空时
    // 会把 inheritableThreadLocals 里面的值全部传递给子线程
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    this.stackSize = stackSize;
    /* Set thread ID */
    // 线程 id 自增
    tid = nextThreadID();
}
```

## join
当前的线程等待其他的线程执行完成后，在去该线程下面的操作：

```java
@Test
public void testJoin2() throws Exception {
  	Thread thread2 = new Thread(new Runnable() {
    	@Override
    	public void run() {
      		log.info("我是子线程 2,开始沉睡");
      		try {
        		Thread.sleep(2000L);
      		} catch (InterruptedException e) {
        		e.printStackTrace();
      		}
      		log.info("我是子线程 2，执行完成");
    	}
  	});
  
	Thread thread1 = new Thread(new Runnable() {
  		@Override
  			public void run() {
      			log.info("我是子线程 1，开始运行");
      			try {
      				log.info("我是子线程 1，我在等待子线程 2");
      				// 这里是代码关键  
      				thread2.join();
      				log.info("我是子线程 1，子线程 2 执行完成，我继续执行");
      			} catch (InterruptedException e) {
        			e.printStackTrace();
      			}
      			log.info("我是子线程 1，执行完成");
    		}
  	});
  	thread1.start();
  	thread2.start();
  	Thread.sleep(100000);
}
```

## yield
当前的线程进行让步，主动的放弃当前cpu的占用，然后再和其他的线程去进行CPU的竞争

- 写 while 死循环的时候，预计短时间内 while 死循环可以结束的话，可以在循环里面使用 yield 方法，防止 cpu 一直被 while 死循环霸占。

**在重新的竞争CPU的时候，主动使用yield的线程也有可能在次的获取CPU的资源**

## sleep与wait的联系与区别
相同点：
- 都可以释放当前线程占用的CPU的资源
- 都可以设置超时等待，到达设定的时候后，主动的释放

不同点：
- sleep()方法属于Thread中的，wait()是Object中的方法
- sleep()不会释放锁，比如调用synchronized方法，sleep的时候不会释放占用的锁
- wait()方法的时候，线程会放弃对象锁，进入等待此对象的等待锁定池，只有针对此对象调用notify()方法后本线程才进入对象锁定池准备

> Thread.Sleep(1000) 意思是在未来的1000毫秒内本线程不参与CPU竞争，1000毫秒过去之后，这时候也许另外一个线程正在使用CPU，那么这时候操作系统是不会重新分配CPU的，直到那个线程挂起或结束，即使这个时候恰巧轮到操作系统进行CPU 分配，那么当前线程也不一定就是总优先级最高的那个，CPU还是可能被其他线程抢占去。另外值得一提的是Thread.Sleep(0)的作用，就是触发操作系统立刻重新进行一次CPU竞争，竞争的结果也许是当前线程仍然获得CPU控制权，也许会换成别的线程获得CPU控制权。
> 
> wait(1000)表示将锁释放1000毫秒，到时间后如果锁没有被其他线程占用，则再次得到锁，然后wait方法结束，执行后面的代码，如果锁被其他线程占用，则等待其他线程释放锁。注意，设置了超时时间的wait方法一旦过了超时时间，并不需要其他线程执行notify也能自动解除阻塞，但是如果没设置超时时间的wait方法必须等待其他线程执行notify。

## interrupt
可以中断线程

- Object#wait ()、Thread#join ()、Thread#sleep (long) 这些方法运行后，线程的状态是 WAITING 或 TIMED_WAITING，这时候打断这些线程，就会抛出 InterruptedException 异常，使线程的状态直接到 TERMINATED；
- 如果 I/O 操作被阻塞了，主动打断当前线程，连接会被关闭，并抛出 ClosedByInterruptException 异常；


## 面试题
#### 创建子线程时，子线程是得不到父线程的 ThreadLocal，有什么办法可以解决这个问题？
可以使用 InheritableThreadLocal 来代替 ThreadLocal，ThreadLocal 和 InheritableThreadLocal 都是线程的属性，所以可以做到线程之间的数据隔离，在多线程环境下我们经常使用，但在有子线程被创建的情况下，父线程 ThreadLocal 是无法传递给子线程的，但 InheritableThreadLocal 可以，主要是因为在线程创建的过程中，会把InheritableThreadLocal 里面的所有值传递给子线程：

```java
// 当父线程的 inheritableThreadLocals 的值不为空时
// 会把 inheritableThreadLocals 里面的值全部传递给子线程
if (parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals =
        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

```
#### 线程创建的方式
- 无参
    - 继承Thread
    - 使用Runnable 作为new Thread的入参
- 有参
    - FutureTask，使用Callable作为入参
    - **Callable无法直接作为Thread的入参，需要使用FutureTask变相的进行传递**

```java
@Test
public void testThreadByCallable() throws ExecutionException, InterruptedException {
  FutureTask futureTask = new FutureTask(new Callable<String> () {
    @Override
    public String call() throws Exception {
      Thread.sleep(3000);
      String result = "我是子线程"+Thread.currentThread().getName();
      log.info("子线程正在运行：{}",Thread.currentThread().getName());
      return result;
    }
  });
  new Thread(futureTask).start();
  log.info("返回的结果是 {}",futureTask.get());
}

```
#### 线程 start 和 run 之间的区别。
调用 Thread.start 方法会开一个新的线程，run 方法不会

#### Thread、Runnable、Callable 三者之间的区别。
Thread 实现了 Runnable，本身就是 Runnable，但同时负责线程创建、线程状态变更等操作。

Runnable 是无返回值任务接口，Callable 是有返回值任务接口，如果任务需要跑起来，必须需要 Thread 的支持才行，Runnable 和 Callable 只是任务的定义，具体执行还需要靠 Thread

#### 线程池 submit 有两个方法，方法一可接受 Runnable，方法二可接受 Callable，但两个方法底层的逻辑却是同一套，这是如何适配的。
Runnable 和 Callable 是通过 FutureTask 进行统一的，FutureTask 有个属性是 Callable，同时也实现了 Runnable 接口，两者的统一转化是在 FutureTask 的构造器里实现的，FutureTask 的最终目标是把 Runnable 和 Callable 都转化成 Callable，Runnable 转化成 Callable 是通过 RunnableAdapter 适配器进行实现的。

线程池的 submit 底层的逻辑只认 FutureTask，不认 Runnable 和 Callable 的差异，所以只要都转化成 FutureTask，底层实现都会是同一套。
#### Callable 能否丢给 Thread 去执行
可以的，可以新建 Callable，并作为 FutureTask 的构造器入参，然后把 FutureTask 丢给 Thread 去执行即可。

#### FutureTask 有什么作用(谈谈对 FutureTask 的理解)。
- 组合了 Callable，实现了 Runnable，把 Callable 和 Runnnable 串联了起来。
- 统一了有参任务和无参任务两种定义方式，方便了使用。
- 实现了 Future 的所有方法，对任务有一定的管理功能，比如说拿到任务执行结果，取消任务，打断任务等等



