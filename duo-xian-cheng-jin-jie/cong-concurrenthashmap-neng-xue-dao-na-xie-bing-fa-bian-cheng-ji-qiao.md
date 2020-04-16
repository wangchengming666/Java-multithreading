# 从ConcurrentHashMap中学到的并发编程技巧

ConcurrentHashMap是Doug Lea所写。它在JDK 1.7中用的是“分段锁”的思想，但在JDK 1.8中几乎重写了一遍，把数据结构简化成与HashMap类似，减小了锁的粒度，大道至简，进一步提升了它在多线程下的性能。

这篇文章主要总结一下在ConcurrentHashMap源码学习过程中学到的一些并发编程的思想和技巧。

**使用volatile进行通信**

在ConcurrentHashMap中有一个很重要的属性是`sizeCtl`它被用于初始化和扩容的过程中。且不同阶段它的值有不同的含义。比如在扩容阶段，它是一个负数，绝对值就代表了参与扩容的线程的数量。它是一个`volatile`变量，定义如下：

```text
private transient volatile int sizeCtl;
```

在初始化或扩容等过程中，多个线程都会用到这个变量，它本身是volatile的，所以可以保证线程间的可见性。除此之外，多线程之间没有使用其它复杂的通信方式。

**使用CAS避免锁**

在之前的那篇put源码解析的文章我们可以看到，put过程中，如果判断数组的某个位置没有结点的话，就会使用casTabAt方法来存放新的结点。这个方法底层是使用的Unsafe类的`compareAndSetObject`方法，是一个native的CAS方法。

```text
// put过程
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
        break;                   // no lock when adding to empty bin
}

// casTabAt方法，底层使用CAS
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSetObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

CAS也是一种“无锁编程”的思想。在可预测的较小的粒度下，可以使用CAS代替锁来提升性能。

**尽量减小锁的粒度**

在面试中，可能经常会问到这个问题：HashMap, HashTable, ConcurrentHashMap有什么区别？稍微有点经验的人都能回答：“HashMap是线程不安全的，另外两个都是线程安全的，其中HashTable是在每个方法上使用了synchronized关键字，而ConcurrentHashMap使用了更小粒度的锁，ConcurrentHashMap在并发场景下性能更好。”

是的，ConcurrentHashMap为了提升性能，在JDK 1.7中使用了“分段锁”的概念，其核心思想是只锁住某一段，这样就不会影响其他段的数据，其它线程就可以并行地操作其它段的数据。“段”虽然相较于HashTable的整个实体来说，锁粒度有所降低，但仍然是一个较粗的粒度。

JDK 1.8对此作了进一步的改进，取消的“分段锁”的概念，直接以某个桶的头结点（如果是链表，就是首结点；如果是红黑树，就是根节点）来作为锁，使锁的粒度更低，能够更大程度地支持并发。

```text
synchronized (f) {
    // ...
}
```

**让空闲线程参与工作**

这是我觉得ConcurrentHashMap里面非常棒的一个思想。在Map进行扩容的时候，会锁住所有put、remove等操作。这个时候如果有其它线程进行put、remove操作，它不会阻塞在那里傻傻的等待。而是会先检查是否正在进行扩容，如果是，就参与进去帮助扩容。

与其让你闲着，不如来帮我先把扩容做完吧！

那这多个线程是怎么配合工作的呢？——仍然是上面提到的以头结点为锁，再配合自旋+CAS进行轻量级通信。

