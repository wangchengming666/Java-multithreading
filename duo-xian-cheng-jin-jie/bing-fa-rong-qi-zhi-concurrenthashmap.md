# 并发容器之ConcurrentHashMap

我们知道在java.util包下提供了一些容器类，而Vector和HashTable是线程安全的容器类，但是这些容器实现同步的方式是通过对方法加锁\(sychronized\)方式实现的，这样读写均需要锁操作，导致性能低下。

**ConcurrentMap接口**

ConcurrentMap接口继承了Map接口，在Map接口的基础上又定义了四个方法：

```text
public interface ConcurrentMap<K, V> extends Map<K, V> {

    //插入元素，与原有put方法不同的是，putIfAbsent方法中如果插入的key相同，则不替换原有的value值；
    V putIfAbsent(K key, V value);

    //移除元素，与原有remove方法不同的是，新remove方法中增加了对value的判断，如果要删除的key-value不能与Map中原有的key-value对应上，则不会删除该元素;
    boolean remove(Object key, Object value);

    //替换元素，增加了对value值的判断，如果key-oldValue能与Map中原有的key-value对应上，才进行替换操作；
    boolean replace(K key, V oldValue, V newValue);

    //替换元素，与上面的replace不同的是，此replace不会对Map中原有的key-value进行比较，如果key存在则直接替换；
    V replace(K key, V value);
}
```

**ConcurrentHashMap类**

ConcurrentHashMap同HashMap一样也是基于散列表的map，但是它提供了一种与HashTable完全不同的加锁策略提供更高效的并发性和伸缩性。

在JDK 1.7中ConcurrentHashMap提供了一种粒度更细的加锁机制来实现在多线程下更高的性能，这种机制叫分段锁\(Lock Striping\)。

提供的优点是：在并发环境下将实现更高的吞吐量，而在单线程环境下只损失非常小的性能。

可以这样理解分段锁，就是将数据分段，对每一段数据分配一把锁。当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

有些方法需要跨段，比如size\(\)、isEmpty\(\)、containsValue\(\)，它们可能需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁ReentrantLock，HashEntry则用于存储键值对数据。

一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构， 一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构（同HashMap一样，它也会在长度达到8的时候转化为红黑树）的元素， 每个Segment守护者一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。

**ConcurrentHashMap的put过程，基于JDK1.8**

```text
public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // key 和 value都不能为空
        if (key == null || value == null) throw new NullPointerException();
        // 再哈希，防止用户自己写的哈希算法导致分布不均匀
        int hash = spread(key.hashCode());
        // 这个位置元素的个数
        int binCount = 0;
        // 死循环
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果没有初始化，就先进行初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
                // 根据hash，取table上相应位置的第一个结点f，判断它是否为空
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果为空，用cas添加进去
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果当前节点是一个forwarding节点，表示在resize过程中
            else if ((fh = f.hash) == MOVED)
                // 如果在resize过程中，帮助转换
                tab = helpTransfer(tab, f);
                // 检查第一个没有申请锁的结点
            else {
                V oldVal = null;
                // 节点上锁，hash值相同的节点组成的链表头结点
                synchronized (f) {
                        // 如果f是第一个结点
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            // 死循环，每次binCount加1
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果当前节点的hash等于key的hashCode，
                                    // 且当前节点的key equals key
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                     // 说明key已经存在，获取原来的值
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        // 替换这个节点的值
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 如果弄到最后还是为空，说明key不存在
                                if ((e = e.next) == null) {
                                    // 在链表尾部插入一个新的节点
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果是红黑树结点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            // 调用红黑树的put方法，其内部也是使用的CAS，并判断key是否已存在
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                // key存在，得到原来值
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                        // 设置新值
                                    p.val = value;
                            }
                        }
                    }
                }
                // 如果binCount不为0，插入成功了，否则会进入再一次的循环
                if (binCount != 0) {
                        // 如果大于转换红黑树的阈值
                    if (binCount >= TREEIFY_THRESHOLD)
                            // 转换成红黑树，内部以根结点为锁
                        treeifyBin(tab, i);
                        / 如果key已存在，有原值，返回原值
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 元素计数加1，根据binCount来检验是否需要检查和扩容
        // 如果<0，不会检查。这里不会小于0
        // 如果<=1，会检查uncontended。
        // 如果>1，会检查并尝试扩容
        addCount(1L, binCount);
        return null;
    }
```

总结一下put的过程：

1. 先检查key和value都不能为空，如果为空，就抛出空指针异常； 
2. 对key再哈希，防止用户自己写的哈希算法导致分布不均匀；
3. 检查是否已经进行了初始化，如果没有，要先初始化；
4. 根据再hash的值，计算在table上的位置的第一个结点f，如果f为空，用CAS直接插入；否则，进行第5步；
5. 检查f是否是一个forwarding结点，如果是表示正在扩容，当前线程就去帮助扩容；如果不是，进行第6步；
6. 检查f的key是不是等于要插入的key，如果是，获取原来的值并替换成新的值，如果不是，进行第7步；
7. 以第一个结点为锁；
8. 如果f不是红黑树结点，进行第9步，如果是红黑树结点，进行第11步；
9. 根据链表向下找，如果找到key相等的位置，获取原来的值并替换成新的值；
10. 如果一直到链表末尾也没有找到相等的key，就在链表尾部插入一个新节点；
11. 调用红黑树的putTreeVal方法，其内部是使用CAS，也会判断key是否已经存在并返回旧值；
12. 如果是保留的Node类型ReservationNode，直接抛出异常；
13. 根据binCount判断是否达到链表转红黑树的阈值，如果达到了，转成红黑树，这个过程会以根节点为锁；

