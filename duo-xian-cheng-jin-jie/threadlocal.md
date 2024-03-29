# ThreadLocal

**什么是ThreadLocal**

ThreadLocal提供了线程内部的局部变量，每个线程都可以通过`get()`和`set()`来对这个局部变量进行操作，但不会和其他线程的局部变量进行冲突，保证了多线程环境下数据的独立性，**实现了线程的数据隔离**～。

以下是**ThreadLocal常用的API**

* `threadLocal.get()`方法，取当前线程存放在`ThreadLocal`里的数据&#x20;
* `threadLocal.set(T value)`方法，设置当前线程在`ThreadLocal`里的数据
* `threadLocal.remove()`方法，移除当前线程在`ThreadLocal`里的数据
* `threadLocal.initialValue()`，返回当前线程在`ThreadLocal`里的初始值&#x20;

**ThreadLocal的原理**

每个`Thread`维护一个`ThreadLocalMap`哈希表，这个哈希表的`key`是`ThreadLocal`实例本身，`value`才是真正要存储的值`Object`。

```
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

 **ThreadLocal的get()方法**

```
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程维护的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    // 如果map不为空
    if (map != null) {
        // 则以当前的ThreadLocal为key，调用ThreadLocalMap中的getEntry方法获取对应的存储实体e。
        // 找到对应的存储实体e，获取存储实体e对应的value值，即为我们想要的当前线程对应此ThreadLocal的值，返回结果值。
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 如果ThreadLocalMap为空，则返回一个新的ThreadLocalMap
    return setInitialValue();
}
```

* setInitiaValue方法如下

```
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
} 
```

* createMap方法如下

```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

 **ThreadLocal的set()方法**

```
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程维护的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    // 如果map不为空
    if (map != null)
        // 赋值
        map.set(this, value);
    else
        // 初始化ThreadLocalMap，并将此实体entry作为第一个值存放至ThreadLocalMap中
        createMap(t, value);
}
```

 **ThreadLocal的remove()方法**

```
public void remove() {
    // 获取当前线程维护的ThreadLocalMap
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        // 如果m不为空，以当前ThreadLocal为key移除
        m.remove(this);
}
```

 **ThreadLocal中的内存泄漏问题**

内存泄漏的定义**：** 对象不再被程序使用，但是因为它们仍在被引用导致垃圾收集器无法删除它们。

通过源码可以发现，`ThreadLocalMap`中的存储实体`Entry`使用`ThreadLocal`作为`key`，但这个`Entry`是继承弱引用`WeakReference`的。

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

`ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收。这样一来，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，就没有办法访问这些`key`为`null`的`Entry`的`value`，如果当前线程再迟迟不结束的话，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：`Thread Ref ->Thread ->ThreaLocalMap ->Entry ->value`永远无法回收，造成内存泄漏。

其实，`ThreadLocalMap`的设计中已经考虑到这种情况，也加上了一些防护措施：在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。

**ThreadLocal最佳实践**

那么我们既然知道了这个问题，就一定存在解决方案的。

* 每次使用完`ThreadLocal`，都调用它的`remove()`方法，清除数据。
* 使用`ThreadLocal`，建议用`static`修饰 `static ThreadLocal<Object> threadLocal = new` `ThreadLocal()`。

**跨线程通信的问题**

考虑到ThreadLocal可以实现单线程的数据隔离，但是想要做线程之间的通信，JDK提供了[`InheritableThreadLocal`](https://docs.oracle.com/javase/10/docs/api/java/lang/InheritableThreadLocal.html)可以使用。但是如果线程是通过线程池创建的，那么主线程和子线程之间的值传递就显得毫无意义了。针对这种情况阿里巴巴提供了解决方案：`transmittable-thread-local`，`transmittable-thread-local`继承了`InheritableThreadLocal`[参考资料](https://github.com/alibaba/transmittable-thread-local)。
