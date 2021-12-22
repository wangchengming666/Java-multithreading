# ReentrantLock和Condition

Java在`java.util.concurrent.locks`包下，还为我们提供了几个关于锁的类和接口，相对于`synchronized`它们有更强大的功能或更高的性能。&#x20;

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200319152940625.png?x-oss-process=image/watermark,type\_ZmFuZ3poZW5naGVpdGk,shadow\_10,text\_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size\_16,color\_FFFFFF,t\_70)

**锁的分类**

* 可重入锁和非可重入锁

所谓重入锁，顾名思义。就是支持重新进入的锁，也就是说这个锁支持一个线程对资源重复加锁。

`synchronized`关键字就是使用的重入锁。比如说，你在一个synchronized实例方法里面调用另一个本实例的`synchronized`实例方法，它可以重新进入这个锁，不会出现任何异常。

如果我们自己在继承AQS实现同步器的时候，没有考虑到占有锁的线程再次获取锁的场景，可能就会导致线程阻塞，那这个就是一个“非可重入锁”。

`ReentrantLock`的中文意思就是可重入锁。也说本文后续要介绍的重点类。

* 公平锁与非公平锁

公平锁指的就是严格按照时间线的先后来工作。也就是说严格按照“先来后到”的理论工作，对于先对锁获取请求的线程一定会先被满足，后对锁获取请求的线程后被满足，那这个锁就是公平的。反之，那就是不公平的。

一般情况下，**非公平锁能提升一定的效率。但是非公平锁可能会发生线程饥饿（有一些线程长时间得不到锁）的情况**。所以要根据实际的需求来选择非公平锁和公平锁。

`ReentrantLock`支持非公平锁和公平锁两种。

* 读写锁和排它锁

我们前面知道的`synchronized`用的锁和`ReentrantLock`，其实都是“排它锁”。也就是说，这些锁在同一时刻只允许一个线程进行访问。

而读写锁可以再同一时刻允许多个读线程访问。Java提供了`ReentrantReadWriteLock`类作为读写锁的默认实现，内部维护了两个锁：一个读锁，一个写锁。通过分离读锁和写锁，使得在“读多写少”的环境下，大大地提高了性能。

**ReentrantLock解读**

ReentrantLock是一个非抽象类，它是Lock接口的JDK默认实现，实现了锁的基本功能。从名字上看，它是一个"可重入"锁，从源码上看，它内部有一个抽象类`Sync`，是继承了`AQS`，自己实现的一个同步器。同时，ReentrantLock内部有两个非抽象类NonfairSync和FairSync，它们都继承了Sync。从名字上看得出，分别是”非公平同步器“和”公平同步器“的意思。这意味着ReentrantLock可以支持”公平锁“和”非公平锁“。

通过看着两个同步器的源码可以发现，它们的实现都是”独占“的。都调用了AOS的`setExclusiveOwnerThread`方法，所以`ReentrantLock`的锁的”独占“的，也就是说，它的锁都是”排他锁“，不能共享。

在ReentrantLock的构造方法里，可以传入一个boolean类型的参数，来指定它是否是一个公平锁，默认情况下是非公平的。这个参数一旦实例化后就不能修改，只能通过isFair()方法来查看。

```
   //获取锁，获取不到lock就不罢休，不可被打断,即使当前线程被中断，线程也一直阻塞，直到拿到锁， 比较无赖的做法。
    void lock();

   /**
   *获取锁，可中断，如果获取锁之前当前线程被interrupt了，
   *获取锁之后会抛出InterruptedException，并且停止当前线程；
   *优先响应中断
   */
    void lockInterruptibly() throws InterruptedException;

    //立即返回结果；尝试获得锁,如果获得锁立即返回ture,失败立即返回false
    boolean tryLock();

    //尝试拿锁，可设置超时时间，超时返回false，即过时不候
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    //释放锁
    void unlock();

    //返回当前线程的Condition ，可多次调用
    Condition newCondition();
```

通常的使用方法如下：

```
    private static void test() {
        ReentrantLock lock = new ReentrantLock();
        Thread thread1 = new Thread(() -> {
            try {
                lock.lock();
                // todo...
            } catch (Exception e) {

            } finally {
                lock.unlock();
            }
        });
        thread1.start();
    }
```

**Condition接口**

* 提供了类似Object监视器的方法，通过与Lock配合来实现等待/通知模式。（可以代替`Object`的`wait`/`notify`），Condition和Object的wait/notify基本相似。
* Condition的signal/signalAll方法则对应Object的notify/notifyAll()。

```
public interface Condition {
    /**
    *Condition线程进入阻塞状态,调用signal()或者signalAll()再次唤醒，
    *允许中断如果在阻塞时锁持有线程中断，会抛出异常；
    *重要一点是：在当前持有Lock的线程中，当外部调用会await()后，ReentrantLock就允许其他线程来抢夺锁当前锁，
    *注意：通过创建Condition对象来使线程wait，必须先执行lock.lock方法获得锁
    */
    void await() throws InterruptedException;

    //Condition线程进入阻塞状态,调用signal()或者signalAll()再次唤醒，不允许中断，如果在阻塞时锁持有线程中断，继续等待唤醒
    void awaitUninterruptibly();

    //设置阻塞时间，超时继续，超时时间单位为纳秒，其他同await()；返回时间大于零，表示是被唤醒，等待时间并且可以作为等待时间期望值，小于零表示超时
    long awaitNanos(long nanosTimeout) throws InterruptedException;

    //类似awaitNanos(long nanosTimeout);返回值：被唤醒true，超时false
    boolean await(long time, TimeUnit unit) throws InterruptedException;

   //类似await(long time, TimeUnit unit) 
    boolean awaitUntil(Date deadline) throws InterruptedException;

   //唤醒指定线程
    void signal();

    //唤醒全部线程
    void signalAll();
}
```

* 生产者-消费者模型的交替打印英文字母和数字

```
private static void alternateTask() {
        ReentrantLock lock = new ReentrantLock();
        Condition condition1 = lock.newCondition();
        Condition condition2 = lock.newCondition();
        Thread thread1 = new Thread(() -> {
            try {
                lock.lock();
                for (int i = 65; i < 91; i++) {
                    System.out.println("----------thread1------- " + (char) i);
                    condition2.signal();
                    condition1.await();
                }
                condition2.signal();
            } catch (Exception e) {
            } finally {
                lock.unlock();
            }
        });
        Thread thread2 = new Thread(() -> {
            try {
                lock.lock();
                for (int i = 0; i < 26; i++) {
                    System.out.println("----------thread2------- " + i);
                    condition1.signal();
                    condition2.await();
                }
                condition1.signal();
            } catch (Exception e) {
            } finally {
                lock.unlock();
            }
        });
        thread1.start();
        thread2.start();
    }
```

输出结果：

```
----------thread1------- A
----------thread2------- 0
----------thread1------- B
----------thread2------- 1
----------thread1------- C
----------thread2------- 2
----------thread1------- D
----------thread2------- 3
----------thread1------- E
----------thread2------- 4
----------thread1------- F
----------thread2------- 5
----------thread1------- G
----------thread2------- 6
----------thread1------- H
----------thread2------- 7
----------thread1------- I
----------thread2------- 8
----------thread1------- J
----------thread2------- 9
----------thread1------- K
----------thread2------- 10
----------thread1------- L
----------thread2------- 11
----------thread1------- M
----------thread2------- 12
----------thread1------- N
----------thread2------- 13
----------thread1------- O
----------thread2------- 14
----------thread1------- P
----------thread2------- 15
----------thread1------- Q
----------thread2------- 16
----------thread1------- R
----------thread2------- 17
----------thread1------- S
----------thread2------- 18
----------thread1------- T
----------thread2------- 19
----------thread1------- U
----------thread2------- 20
----------thread1------- V
----------thread2------- 21
----------thread1------- W
----------thread2------- 22
----------thread1------- X
----------thread2------- 23
----------thread1------- Y
----------thread2------- 24
----------thread1------- Z
----------thread2------- 25

Process finished with exit code 0
```
