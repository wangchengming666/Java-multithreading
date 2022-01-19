# Java8中的并行计算

从Java 8 开始，我们可以使用`Stream`接口以及`lambda`表达式进行“流式计算”。它可以让我们对集合的操作更加简洁、更加可读、更加高效。

Stream接口有非常多用于集合计算的方法，比如判空操作empty、过滤操作filter、求最max值、查找操作findFirst和findAny等等。

**什么是并行计算**

假如我的计算机是一个多核计算机，我们在理论上能否利用多核来进行并行计算，提高计算效率呢？答案是可以的。

**比如我们在计算前两个元素1 + 2 = 3的时候，其实我们也可以同时在另一个核计算 3 + 4 = 7。然后等它们都计算完成之后，再计算 3 + 7 = 10的操作。** 这就是并行计算，是和串行完全相反的概念。

**Stream如何实现并行计算**

```
public class StreamParallelDemo {
    public static void main(String[] args) {
        Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .parallel()
                .reduce((a, b) -> {
                    System.out.println(String.format("%s: %d + %d = %d",
                            Thread.currentThread().getName(), a, b, a + b));
                    return a + b;
                })
                .ifPresent(System.out::println);
    }
}
```

输出结果如下：

```
ForkJoinPool.commonPool-worker-2: 8 + 9 = 17
ForkJoinPool.commonPool-worker-1: 3 + 4 = 7
ForkJoinPool.commonPool-worker-5: 1 + 2 = 3
ForkJoinPool.commonPool-worker-6: 5 + 6 = 11
ForkJoinPool.commonPool-worker-2: 7 + 17 = 24
ForkJoinPool.commonPool-worker-5: 3 + 7 = 10
ForkJoinPool.commonPool-worker-2: 11 + 24 = 35
ForkJoinPool.commonPool-worker-2: 10 + 35 = 45
45
```

可以很明显地看到，它使用的线程是`ForkJoinPool`里面的`commonPool`里面的worker线程。并且它们是并行计算的，并不是串行计算的。但由于Fork/Join框架的作用，它最终能很好的协调计算结果，使得计算结果完全正确。

如果我们用`Fork/Join`代码去实现这样一个功能，那无疑是非常复杂的。但Java8提供了并行式的流式计算，大大简化了我们的代码量，使得我们只需要写很少很简单的代码就可以利用计算机底层的多核资源。
