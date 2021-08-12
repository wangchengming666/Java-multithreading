# 阻塞队列

我们假设一种场景，生产者一直生产资源，消费者一直消费资源，资源存储在一个缓冲池中，生产者将生产的资源存进缓冲池中，消费者从缓冲池中拿到资源进行消费，这就是大名鼎鼎的**生产者-消费者**模式。

该模式能够简化开发过程，一方面消除了生产者类与消费者类之间的代码依赖性，另一方面将生产数据的过程与使用数据的过程解耦简化负载。

我们自己coding实现这个模式的时候，因为需要让多个线程操作共享变量（即资源），所以很容易引发线程安全问题，造成重复消费和死锁，尤其是生产者和消费者存在多个的情况。另外，当缓冲池空了，我们需要阻塞消费者，唤醒生产者；当缓冲池满了，我们需要阻塞生产者，唤醒消费者，这些个等待-唤醒逻辑都需要自己实现。

这么容易出错的事情，JDK当然帮我们做啦，这就是阻塞队列\(BlockingQueue\)，你只管往里面存、取就行，而不用担心多线程环境下存、取共享变量的线程安全问题。

BlockingQueue是Java util.concurrent包下重要的数据结构，区别于普通的队列，BlockingQueue提供了线程安全的队列访问方式，并发包下很多高级同步类的实现都是基于BlockingQueue实现的。

BlockingQueue一般用于生产者-消费者模式，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。BlockingQueue就是存放元素的容器。

**BlockingQueue的操作方法**

阻塞队列提供了四组不同的方法用于插入、移除、检查元素：

| 方法\处理方式 | 抛出异常 | 返回特殊值 | 一直阻塞 | 超时退出 |
| :--- | :--- | :--- | :--- | :--- |
| 插入方法 | add\(e\) | offer\(e\) | put\(e\) | offer\(e,time,unit\) |
| 移除方法 | remove\(\) | poll\(\) | take\(\) | poll\(time,unit\) |
| 检查方法 | element\(\) | peek\(\) | - | - |

* 抛出异常：如果试图的操作无法立即执行，抛异常。当阻塞队列满时候，再往队列里插入元素，会抛出IllegalStateException\("Queue full"\)异常。当队列为空时，从队列里获取元素时会抛出NoSuchElementException异常 。
* 返回特殊值：如果试图的操作无法立即执行，返回一个特殊值，通常是true / false。
* 一直阻塞：如果试图的操作无法立即执行，则一直阻塞或者响应中断。
* 超时退出：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，但等待时间不会超过给定值。返回一个特定值以告知该操作是否成功，通常是 true / false。

**BlockingQueue的实现类**

* ArrayBlockingQueue

由数组结构组成的有界阻塞队列。内部结构是数组，故具有数组的特性。

```text
public ArrayBlockingQueue(int capacity, boolean fair){
    //..省略代码
}
```

可以初始化队列大小， 且一旦初始化不能改变。构造方法中的fair表示控制对象的内部锁是否采用公平锁，默认是非公平锁。

* LinkedBlockingQueue

由链表结构组成的有界阻塞队列。内部结构是链表，具有链表的特性。默认队列的大小是Integer.MAX\_VALUE，也可以指定大小。此队列按照先进先出的原则对元素进行排序。

* SynchronousQueue

这个队列比较特殊，没有任何内部容量，甚至连一个队列的容量都没有。并且每个 put 必须等待一个 take，反之亦然。

需要区别容量为1的ArrayBlockingQueue、LinkedBlockingQueue。

以下方法的返回值，可以帮助理解这个队列：

* iterator\(\) 永远返回空，因为里面没有东西
* peek\(\) 永远返回null
* put\(\) 往queue放进去一个element以后就一直wait直到有其他thread进来把这个element取走。
* offer\(\) 往queue里放一个element后立即返回，如果碰巧这个element被另一个thread取走了，
* offer方法返回true，认为offer成功；否则返回false。
* take\(\) 取出并且remove掉queue里的element，取不到东西他会一直等。
* poll\(\) 取出并且remove掉queue里的element，只有到碰巧另外一个线程正在往queue里offer数据或者put数据的时候，该方法才会取到东西。否则立即返回null。
* isEmpty\(\) 永远返回true
* remove\(\)&removeAll\(\) 永远返回false

**线程池中使用阻塞队列**

```text
 public ThreadPoolExecutor(int corePoolSize,
                           int maximumPoolSize,
                           long keepAliveTime,
                           TimeUnit unit,
                           BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
}
```

Java中的线程池就是使用阻塞队列实现的，我们在了解阻塞队列之后，无论是使用Exectors类中已经提供的线程池，还是自己通过ThreadPoolExecutor实现线程池，都会更加得心应手。

