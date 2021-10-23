# 线程池的使用

**为什么要使用线程池**

* 创建/销毁线程需要消耗系统资源，线程池可以复用已创建的线程。
* 控制并发的数量。并发数量过多，可能会导致资源消耗过多，从而造成服务器崩溃。（主要原因）。
* 可以对线程做统一管理。

**线程池的原理**

Java中的线程池顶层接口是`Executor`接口，`ThreadPoolExecutor`是这个接口的实现类。

* 我们先看一下`Executor`。

```
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

* 我们再看看`ThreadPoolExecutor`类。构造函数如下：

```
// 五个参数的构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue)

// 六个参数的构造函数-1
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory)

// 六个参数的构造函数-2
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler)

// 七个参数的构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

*   关于构造函数的参数的含义

    * **int corePoolSize**：该线程池中核心线程数最大值。线程池中有两类线程，核心线程和非核心线程。核心线程默认情况下会一直存在于线程池中，即使这个核心线程什么都不干（铁饭碗），而非核心线程如果长时间的闲置，就会被销毁（临时工）。
    * **int maximumPoolSize**：该线程池中线程总数最大值 。该值等于核心线程数量 + 非核心线程数量。
    * **long keepAliveTime**：非核心线程闲置超时时长。非核心线程如果处于闲置状态超过该值，就会被销毁。
    * **TimeUnit unit**：keepAliveTime的单位。
    * **BlockingQueue workQueue**：阻塞队列，维护着等待执行的Runnable任务对象。



    > 常用的几个阻塞队列：
    >
    > &#x20;  1\. LinkedBlockingQueue
    >
    > 链式阻塞队列，底层数据结构是链表，默认大小是`Integer.MAX_VALUE`，也可以指定大小。
    >
    > &#x20;  2\. ArrayBlockingQueue
    >
    > 数组阻塞队列，底层数据结构是数组，需要指定队列的大小。
    >
    > &#x20;  3\. SynchronousQueue
    >
    > 同步队列，内部容量为0，每个put操作必须等待一个take操作，反之亦然。
    >
    > &#x20;  4\. DelayQueue
    >
    > 延迟队列，该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素 。
* 线程池主要的任务处理流程

处理任务的核心方法是execute，我们看看 JDK 1.8 源码中ThreadPoolExecutor是如何处理线程任务的：

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException(); 
    // clt 记录着runState workerCount  
    int c = ctl.get();
    // 1.当前线程数小于corePoolSize，则调用addWorker创建核心线程执行任务
    if (workerCountOf(c) < corePoolSize) {
       if (addWorker(command, true))
           return;
       c = ctl.get();
    }
    // 2.如果不小于corePoolSize，则将任务添加到workQueue队列。
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 2.1 如果isRunning返回false(状态检查)，则remove这个任务，然后执行拒绝策略。
        if (! isRunning(recheck) && remove(command))
            reject(command);
            // 2.2 线程池处于running状态，但是没有线程，则创建线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3.如果放入workQueue失败，则创建非核心线程执行任务，
    // 如果这时创建非核心线程失败(当前线程总数不小于maximumPoolSize时)，就会执行拒绝策略。
    else if (!addWorker(command, false))
         reject(command);
}
```

总结一下处理流程

1. 线程总数量 < corePoolSize，无论线程是否空闲，都会新建一个核心线程执行任务（让核心线程数量快速达到corePoolSize，在核心线程数量 < corePoolSize时）。注意，这一步需要获得全局锁。
2. 线程总数量 >= corePoolSize时，新来的线程任务会进入任务队列中等待，然后空闲的核心线程会依次去缓存队列中取任务来执行（体现了线程复用）。
3. 当缓存队列满了，说明这个时候任务已经多到爆棚，需要一些“临时工”来执行这些任务了。于是会创建非核心线程去执行这个任务。注意，这一步需要获得全局锁。
4. 缓存队列满了， 且总线程数达到了maximumPoolSize，则会采取拒绝策略进行处理。

* 线程池四种拒绝策略

当线程池的任务缓存队列已满并且线程池中的线程数目达到maximumPoolSize，如果还有任务到来 就会采取任务拒绝策略。`RejectedExecutionHandler`接口定义如下

```
public interface RejectedExecutionHandler {

    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

通过源码可以看到，线程池一共有四种拒绝策略，如下图所示 &#x20;

![](https://img-blog.csdnimg.cn/20200530224546771.png?x-oss-process=image/watermark,type\_ZmFuZ3poZW5naGVpdGk,shadow\_10,text\_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size\_16,color\_FFFFFF,t\_70)

`ThreadPoolExecutor.AbortPolicy`是线程池的默认决绝策略，丢弃任务并抛出`RejectedExecutionException`异常。

```
public static class AbortPolicy implements RejectedExecutionHandler {
    /**
     * Creates an {@code AbortPolicy}.
     */
    public AbortPolicy() { }

    /**
     * Always throws RejectedExecutionException.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     * @throws RejectedExecutionException always
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```

`ThreadPoolExecutor.DiscardPolicy`的策略是丢弃任务，但是不抛出异常。

```
public static class DiscardPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardPolicy}.
     */
    public DiscardPolicy() { }

    /**
     * Does nothing, which has the effect of discarding task r.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```

`ThreadPoolExecutor.DiscardOldestPolicy`的策略是丢弃队列最前面的任务，然后重新尝试执行任务并重复此过程。

```
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardOldestPolicy} for the given executor.
     */
    public DiscardOldestPolicy() { }

    /**
     * Obtains and ignores the next task that the executor
     * would otherwise execute, if one is immediately available,
     * and then retries execution of task r, unless the executor
     * is shut down, in which case task r is instead discarded.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```

`ThreadPoolExecutor.CallerRunsPolicy`的策略是由调用线程处理该任务。

```
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code CallerRunsPolicy}.
     */
    public CallerRunsPolicy() { }

    /**
     * Executes task r in the caller's thread, unless the executor
     * has been shut down, in which case the task is discarded.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```

**线程池如何实现复用**

可以先看一下线程池复用的流程图，接下来我们通过源码对线程复用的原理做详细的分析。&#x20;

![](https://img-blog.csdnimg.cn/20200531134819427.png?x-oss-process=image/watermark,type\_ZmFuZ3poZW5naGVpdGk,shadow\_10,text\_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size\_16,color\_FFFFFF,t\_70)

ThreadPoolExecutor在创建线程时，会将线程封装成工作线程worker，并放入工作线程组中，然后这个worker反复从阻塞队列中拿任务去执行。

简单的说，线程池就是一组工人，任务是放在队列Queue里，一共就这么几个工人，当有空闲的工人，就会去队列里领取下一个任务，所以通过这种手段限制的总工人（线程）数量，即为复用。接下来我们通过源码来分析一下线程池复用的原理。

首先看一下`ThreadPoolExecutor.addWorker`

```
private boolean addWorker(Runnable firstTask, boolean core) {
    //...这里有一段CAS代码，通过双重循环目的是通过CAS增加线程池线程个数
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        //...省略部分代码
        workers.add(w);
        //...省略部分代码
        workerAdded = true;
        if (workerAdded) {
            t.start();
            workerStarted = true;
        }
    }
}
```

源代码比较长，这里省略了一部分。过程主要分成两步，第一步是一段CAS代码通过双重循环检查状态并为当前线程数扩容 +1，第二部是将任务包装成worker对象，用线程安全的方式添加到 HashSet() 里，并开始执行线程。

接下来看一下`Worker`的部分源码。`Worker`类实现了`Runnable`接口，所以`Worker`也是一个线程任务。在构造方法中，创建了一个线程，线程的任务就是自己。故`addWorker`方法中的`t.start`，会触发`Worker`类的run方法被JVM调用。

```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
    final Thread thread;
    Runnable firstTask;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 新建一个线程
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
         runWorker(this);
    }
    //其余代码略...
}
```

继续来看runWorker()方法

```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    //省略代码
        while (task != null || (task = getTask()) != null) {
            //省略代码
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (Exception x) {
                    thrown = x; throw x;
                }
            //省略代码
        }
    //省略代码
}
```

这里有一个大的while循环，当我们的task不为空的时候它就永远在循环，并且会源源不断的调用getTask()来获取新的任务，然后调用task.run()执行任务，从而达到复用线程的目的。

继续跟踪getTask()方法，这里主要是在workQueue中拉取任务

```
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        //..省略

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
       //..省略

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

以上源码就是线程池复用的整个流程。总结一下最核心的一点就是：新建一个Worker内部类就会建一个线程，并且会把这个内部类本身传进去当作任务去执行，这个内部类的run方法里实现了一个while循环，当任务队列没有任务时结束这个循环，则这个线程就结束。

**常见的线程池**

* newSingleThreadExecutor

```
public static ExecutorService newSingleThreadExecutor() {
  return new FinalizableDelegatedExecutorService
      (new ThreadPoolExecutor(1, 1,
                              0L, TimeUnit.MILLISECONDS,
                              new LinkedBlockingQueue<Runnable>()));
}
```

有且仅有一个核心线程（ corePoolSize == maximumPoolSize=1），使用了LinkedBlockingQueue（容量很大），所以，不会创建非核心线程。所有任务按照先来先执行的顺序执行。如果这个唯一的线程不空闲，那么新来的任务会存储在任务队列里等待执行。

* newFixedThreadPool

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```

核心线程数量和总线程数量相等，都是传入的参数nThreads，所以只能创建核心线程，不能创建非核心线程。因为LinkedBlockingQueue的默认大小是Integer.MAX\_VALUE，故如果核心线程空闲，则交给核心线程处理；如果核心线程不空闲，则入列等待，直到核心线程空闲。

* newCachedThreadPool

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

运行流程如下：

1. 提交任务进线程池。
2. 因为corePoolSize为0的关系，不创建核心线程，线程池最大为Integer.MAX\_VALUE。
3. 尝试将任务添加到SynchronousQueue队列。
4. 如果SynchronousQueue入列成功，等待被当前运行的线程空闲后拉取执行。如果当前没有空闲线程，那么就创建一个非核心线程，然后从SynchronousQueue拉取任务并在当前线程执行。
5. 如果SynchronousQueue已有任务在等待，入列操作将会阻塞。

当需要执行很多短时间的任务时，CacheThreadPool的线程复用率比较高， 会显著的提高性能。而且线程60s后会回收，意味着即使没有任务进来，CacheThreadPool并不会占用很多资源。

* newScheduledThreadPool

创建一个定长线程池，支持定时及周期性任务执行。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

//ScheduledThreadPoolExecutor():
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```
