# Callable、Future与FutureTask

通常来说，我们使用`Runnable`和`Thread`来创建一个新的线程，但是它们是没有返回值的。而有时候我们希望开启一个线程去执行一个任务，并且这个任务执行完成后有一个返回值。

那么JDK提供了`Callable`接口与`Future`类为我们解决这个问题，这也是所谓的“异步”模型。

**Callable接口**

可以看到Callable是一个函数式接口。同时`Callable`是有返回值的，并且支持泛型。

```
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

那一般是怎么使用Callable的呢？Callable一般是配合线程池工具ExecutorService来使用的。这里举例介绍ExecutorService可以使用submit方法来让一个Callable接口执行。它会返回一个Future，我们后续的程序可以通过这个Future的get方法得到结果。

```
public class Task implements Callable {

    @Override
    public String call() throws Exception {
        return "call()方法";
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        Task task = new Task();
        Future<String> submit = executorService.submit(task);
        System.out.println(submit.get());
    }
}
```

输出结果：`call()方法`

**Future接口**

`Future`和`Callable`有着类似的地方，同样也是支持泛型和有返回结果。 `Future`接口只有几个比较简单的方法：

```
public interface Future<V> {

    /**
     * Attempts to cancel execution of this task.  This attempt will
     * fail if the task has already completed, has already been cancelled,
     * or could not be cancelled for some other reason. If successful,
     * and this task has not started when {@code cancel} is called,
     * this task should never run.  If the task has already started,
     * then the {@code mayInterruptIfRunning} parameter determines
     * whether the thread executing this task should be interrupted in
     * an attempt to stop the task.
     *
     * <p>After this method returns, subsequent calls to {@link #isDone} will
     * always return {@code true}.  Subsequent calls to {@link #isCancelled}
     * will always return {@code true} if this method returned {@code true}.
     *
     * @param mayInterruptIfRunning {@code true} if the thread executing this
     * task should be interrupted; otherwise, in-progress tasks are allowed
     * to complete
     * @return {@code false} if the task could not be cancelled,
     * typically because it has already completed normally;
     * {@code true} otherwise
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * Returns {@code true} if this task was cancelled before it completed
     * normally.
     *
     * @return {@code true} if this task was cancelled before it completed
     */
    boolean isCancelled();

    /**
     * Returns {@code true} if this task completed.
     *
     * Completion may be due to normal termination, an exception, or
     * cancellation -- in all of these cases, this method will return
     * {@code true}.
     *
     * @return {@code true} if this task completed
     */
    boolean isDone();

    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     *
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * Waits if necessary for at most the given time for the computation
     * to complete, and then retrieves its result, if available.
     *
     * @param timeout the maximum time to wait
     * @param unit the time unit of the timeout argument
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     * @throws TimeoutException if the wait timed out
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

* `cancel()`注意是试图取消，并不一定能取消成功。因为任务可能已完成、已取消、或者一些其它因素不能取消，存在取消失败的可能。boolean类型的返回值是“是否取消成功”的意思。参数paramBoolean表示是否采用中断的方式取消线程执行。

**FutureTask类**

`FutureTask`的几个状态：

```
    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

* 先看一下`FutureTask`类

可以看到`FutureTask`是实现了`RunnableFuture`接口的。

```
public class FutureTask<V> implements RunnableFuture<V> { }
```

* 再看一下`RunnableFuture`接口

可以看到`RunnableFuture`接口同时继承了`Runnable`接口和`Future`接口

```
/**
 * A {@link Future} that is {@link Runnable}. Successful execution of
 * the {@code run} method causes completion of the {@code Future}
 * and allows access to its results.
 * @see FutureTask
 * @see Executor
 * @since 1.6
 * @author Doug Lea
 * @param <V> The result type returned by this Future's {@code get} method
 */
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

* 那FutureTask类有什么用？为什么要有一个FutureTask类？前面说到了Future只是一个接口，而它里面的cancel，get，isDone等方法要自己实现起来都是非常复杂的。所以JDK提供了一个FutureTask类来供我们使用。
* 修改一下上面的代码，改用`FutureTask`

使用上与第一个Demo有一点小的区别。首先，调用submit方法是没有返回值的。这里实际上是调用的submit(Runnable task)方法，而上面的Demo，调用的是submit(Callable task)方法。

然后，这里是使用FutureTask直接取get取值，而上面的Demo是通过submit方法返回的Future去取值。

```
public class Task implements Callable {

    @Override
    public String call() throws Exception {
        return "call()方法";
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        FutureTask futureTask = new FutureTask(new Task());
        executorService.submit(futureTask);
        System.out.println(futureTask.get());
    }
}
```
