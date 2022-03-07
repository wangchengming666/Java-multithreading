# CAS

**CAS的概念**

CAS的全称是：比较并交换（Compare And Swap）。在CAS中，有这样三个值：

* V：要更新的变量(var)
* E：预期值(expected)
* N：新值(new)

比较并交换的过程如下：

判断V是否等于E，如果等于，将V的值设置为N；如果不等，说明已经有其它线程更新了V，则当前线程放弃更新，什么都不做。

**CAS的过程**

我们以一个简单的例子来解释这个过程：

*   如果有一个多个线程共享的变量i原本等于5，我现在在线程A中，想把它设置为新的值6;

    我们使用CAS来做这个事情；
* 首先我们用i去与5对比，发现它等于5，说明没有被其它线程改过，那我就把它设置为新的值6，此次CAS成功，i的值被设置成了6；
* 如果不等于5，说明i被其它线程改过了（比如现在i的值为2），那么我就什么也不做，此次CAS失败，i的值仍然为2。
* 在这个例子中，i就是V，5就是E，6就是N

那有没有可能我在判断了i为5之后，正准备更新它的新值的时候，被其它线程更改了i的值呢？

不会的。因为`CAS是一种原子操作`，它是一种系统原语，是一条CPU的原子指令，从CPU层面保证它的原子性

**当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作。**

**通过AtomicInteger看CAS实现原理**

首先我们看一下`java.util.concurrent.atomic`这个包，这个包里有很多类 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200319103630885.png?x-oss-process=image/watermark,type\_ZmFuZ3poZW5naGVpdGk,shadow\_10,text\_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size\_16,color\_FFFFFF,t\_70)&#x20;

这里我们以`AtomicInteger`类的`getAndSet(int newValue)`方法为例，来看看Java是如何实现原子操作的。

```
    /**
     * Atomically sets to the given value and returns the old value.
     *
     * @param newValue the new value
     * @return the previous value
     */
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
```

`unsafe`是`sun.misc`包里面的一个类。

```
    package sun.misc;

    import java.lang.reflect.Field;
    import java.lang.reflect.Modifier;
    import java.security.ProtectionDomain;
    import sun.reflect.CallerSensitive;
    import sun.reflect.Reflection;

    public final class Unsafe { }
```

所以说`getAndSet()`方法是调用的`unsafe`的`getAndSetInt()`方法。在`Unsafe`类中有很多native方法，众所周知，native方法Java就不负责具体实现它，而是交给底层的JVM使用c或者c++去实现。`Unsafe`中对CAS的实现是C++写的，它的具体实现和操作系统、CPU都有关系。

```
    public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }
```

我们来一步步解析这段源码。首先，对象var1是一个AtomicInteger对象。然后var2是一个常量VALUE，这个常量是在AtomicInteger类中声明的，var4是新值。

继续看源码。前面我们讲到，CAS是"无锁"的基础，它允许更新失败。所以经常会与while循环搭配，在失败后不断去重试。

这里声明了一个var5，也就是要返回的值。从getAndAddInt来看，它返回的应该是原来的值，而新的值的等于var5 + var4。

这里使用的是do-while循环。这种循环不多见，它的目的是保证循环体内的语句至少会被执行一遍。这样才能保证return的var5值是我们期望的值。

循环体的条件是一个CAS方法：

```
final boolean weakCompareAndSetInt(Object o, long offset,
                                          int expected,
                                          int x) {
    return compareAndSetInt(o, offset, expected, x);
}

public final native boolean compareAndSetInt(Object o, long offset,
                                             int expected,
                                             int x);
```

简单来说，`weakCompareAndSet`操作仅保留了`volatile`自身变量的特性，而除去了happens-before规则带来的内存语义。也就是说，`weakCompareAndSet`**无法保证处理操作目标的volatile变量外的其他变量的执行顺序( 编译器和处理器为了优化程序性能而对指令序列进行重新排序 )，同时也无法保证这些变量的可见性。**这在一定程度上可以提高性能。

再回到循环条件上来，可以看到它是在不断尝试去用CAS更新。如果更新失败，就继续重试。那为什么要把获取“旧值”v的操作放到循环体内呢？其实这也很好理解。前面我们说了，CAS如果旧值V不等于预期值E，它就会更新失败。说明旧的值发生了变化。那我们当然需要返回的是被其他线程改变之后的旧值了，因此放在了do循环体内。

**ABA问题**

所谓ABA问题，就是一个值原来是A，变成了B，又变回了A。这个时候使用CAS是检查不出变化的，但实际上却被更新了两次。

ABA问题的解决思路是在变量前面追加上`版本号`或者`时间戳`。从JDK 1.5开始，JDK的atomic包里提供了一个类`AtomicStampedReference`类来解决ABA问题。

这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果二者都相等，才使用CAS设置为新的值和标志。

```
    /**
     * Atomically sets the value of both the reference and stamp
     * to the given update values if the
     * current reference is {@code ==} to the expected reference
     * and the current stamp is equal to the expected stamp.
     *
     * @param expectedReference the expected value of the reference
     * @param newReference the new value for the reference
     * @param expectedStamp the expected value of the stamp
     * @param newStamp the new value for the stamp
     * @return {@code true} if successful
     */
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```

**循环时间长开销大问题**

CAS多与自旋结合。如果自旋CAS长时间不成功，会占用大量的CPU资源。

解决思路是让JVM支持处理器提供的pause指令。

pause指令能让自旋失败时cpu睡眠一小段时间再继续自旋，从而使得读操作的频率低很多,为解决内存顺序冲突而导致的CPU流水线重排的代价也会小很多。

**只能保证一个共享变量的原子操作**

这个问题你可能已经知道怎么解决了。有两种解决方案：

* 使用JDK 1.5开始就提供的AtomicReference类保证对象之间的原子性，把多个变量放到一个对象里面进行CAS操作；
* 使用锁。锁内的临界区代码可以保证只有当前线程能操作。
