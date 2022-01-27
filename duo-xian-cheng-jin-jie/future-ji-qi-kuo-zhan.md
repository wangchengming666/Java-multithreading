# Future及其扩展

今天这里主要介绍`CompetableFuture`类，因为它是JDK自带的，相比于`RxJava`项目来说也比较轻量。功能也比较齐全，基本上满足绝大多数的异步编程需求。

先看一下使用线程池的`submit`方法是怎么写的，它底层是使用的`FutureTask`

```
// 定义一个线程池并让它执行一个异步任务
ExecutorService executor = Executors.newCachedThreadPool();
Future<Integer> result = executor.submit( () -> {
    TimeUnit.SECONDS.sleep(5);
    return 1;
});
```

然后取值的时候，有这样一些方式：

```
// 取值
System.out.println(result.get()); // 阻塞
System.out.println(result.get(3, TimeUnit.SECONDS)); // 阻塞至超时
for(;;) { // 自旋验证
    if (result.isDone()) {
        System.out.println(result.get());
        break;
    }
}
```

我们在使用`Callable`而不是`Runnable`的时候，就说明了我们希望得到线程返回的结果，并对它进行下一步操作。但这样通过阻塞去取或自旋不断去验证的方式，显然不如回调。

如果我们希望得到结果并把它打印出来，使用`CompetableFuture`可以这样做：

```
CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(5);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 1;
}).whenComplete((result, exception) -> {
    System.out.println(result);
});
```

这里的`whenComplete`方法，就是给他传一个回调函数进去，它会在`supplyAsync`中定义的任务完成并返回后，调用这个回调函数。

`CompetableFuture`源码大概有2900行，方法非常多。这里做一个分类和总结，以便以后可以快速查阅。

先介绍一下`CompletionStage`这个类：`CompletionStage`代表异步计算过程中的某一个阶段，一个阶段完成以后可能会触发另外一个阶段。

#### 创建异步任务

`CompletableFuture` 提供了四个静态方法来创建一个异步任务。如果没有指定`Executor`的方法会使用`ForkJoinPool.commonPool()` 作为它的线程池执行异步代码。如果指定线程池，则使用指定的线程池运行。以下所有的方法都类同。

```
public static CompletableFuture<Void> runAsync(Runnable runnable);
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor);
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs);
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs);
```

其中，`runAsync`方法不支持返回值，`supplyAsync`可以支持返回值。

#### 计算结果完成时的回调方法

当`CompletableFuture`的计算结果完成，或者抛出异常的时候，可以执行特定的`Action`

```
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action);
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action);
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor);
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn);
```

其中`whenComplete` 和 `whenCompleteAsync` 的区别（下面所有方法类似）：

* `whenComplete`：是执行当前任务的线程执行继续执行 whenComplete 这个任务。
* `whenCompleteAsync`：是执行把 whenCompleteAsync 这个任务继续提交给线程池来进行执行。

`handle` 方法 跟 `whenComplete` 类似，但是返回的是一个`CompletionStage`。

```
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

`thenAccept`方法跟 `handle` 方法类似，不过 `handle` 方法是有返回值的，如果我们只是消费任务完成的结果，**不需要返回值**，可以使用 `thenAccept` 方法：

```
public CompletionStage<Void> thenAccept(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
```

`thenRun`跟 `thenAccept` 方法不一样的是，**不关心任务的处理结果**。只要上面的任务执行完成，就开始执行。

```
public CompletionStage<Void> thenRun(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action,Executor executor);
```

#### 线程串行化

`thenCompose` 方法允许你对两个 `CompletionStage` 进行流水线操作，第一个操作完成时，将其结果作为参数传递给第二个操作。

```
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn);
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn);
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor);
```

当一个线程依赖另一个线程时，可以使用 `thenApply` 方法来把这两个线程**串行化**。

```
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn);
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor);
```

其中`T`是上一个任务返回结果的类型，`U`是当前任务的返回值类型。

都是串行化，`thenCompose` 与 `thenApply` 不同之处在于，`thenCompose` 传入的是一个`CompletionStage`， 而`thenApply`可以直接使用上一个任务返回的结果。`thenApply` 可以“链式调用”。

#### 等两个任务都执行完

`thenCombine` 会等两个 `CompletionStage` 的任务都执行完成后，把两个任务的结果合并到一块来处理。

```
public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

而 `thenAcceptBoth` 跟 `thenCombine` 类似，不同的是它只是消费结果，但没有返回值。

```
public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action,     Executor executor);
```

`runAfterBoth` 类似，只是不关心上一个任务的返回值：

```
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

#### 看哪个任务执行得更快

使用 `applyToEither` 方法的作用是：两个`CompletionStage`，谁执行返回的结果快，我就用那个`CompletionStage`的结果进行下一步的转化操作。

```
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);
```

同样，它也有对应的**只是消费**的方法 `acceptEither`：

```
public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action,Executor executor);
```

也有**不关心上一个任务的返回值**，任意一个任务跑完了都会执行下一步操作的：

```
public CompletionStage<Void> runAfterEither(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

### 总结

其实是有一些命名规则是比较通用的。这里大概总结一下：

* 涉及消费者的，都是`accept`
* 不关心之前结果的，都是`run`
* 看哪个跑得快，就是`either`
* 两个都要跑完，就是`both`
