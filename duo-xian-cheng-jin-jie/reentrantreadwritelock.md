# ReentrantReadWriteLock

前面提到ReentrantLock，是一种“排它锁”。也就是说，这些锁在同一时刻只允许一个线程进行访问。而读写锁可以再同一时刻允许多个读线程访问。Java提供了`ReentrantReadWriteLock`类作为读写锁的默认实现，内部维护了两个锁：一个读锁，一个写锁。

**ReadWriteLock接口**

这里面有两把锁，一把读锁，用来处理读相关的操作；一把写锁，用来处理写相关的操作。

```
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
```

**ReentrantReadWriteLock类**

这个类也是一个非抽象类，它是`ReadWriteLock`接口的JDK默认实现。它与ReentrantLock的功能类似，同样是可重入的，支持非公平锁和公平锁。不同的是，它还支持”读写锁“。

```
// 内部结构
private final ReentrantReadWriteLock.ReadLock readerLock;

private final ReentrantReadWriteLock.WriteLock writerLock;

final Sync sync;

abstract static class Sync extends AbstractQueuedSynchronizer {
    // 具体实现
}

static final class NonfairSync extends Sync {
    // 具体实现
}

static final class FairSync extends Sync {
    // 具体实现
}

public static class ReadLock implements Lock, java.io.Serializable {
    private final Sync sync;
    protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
    }
    // 具体实现
}

public static class WriteLock implements Lock, java.io.Serializable {
    private final Sync sync;
    protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
    }
    // 具体实现
}

// 构造方法，初始化两个锁
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

// 获取读锁和写锁的方法
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }

public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

**读读共享**

```
public class MyTask {

    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public void read() {
        try {
            lock.readLock().lock();
            System.out.println(Thread.currentThread().getName() + " start");
            Thread.sleep(10000);
            System.out.println(Thread.currentThread().getName() + " end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.readLock().unlock();
        }
    }
}

public class ReadReadTest {

    public static void main(String[] args) {
        final MyTask myTask = new MyTask();

        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.read();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.read();
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```

输出结果：

```
t2 start
t1 start
t1 end
t2 end
```

**写写互斥**

在同一时间只允许一个线程执行lock()方法后面的代码。

```
public class MyTask {

    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public void write() {
        try {
            lock.writeLock().lock();
            System.out.println(Thread.currentThread().getName() + " start");
            Thread.sleep(10000);
            System.out.println(Thread.currentThread().getName() + " end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.writeLock().unlock();
        }
    }
}

public class ReadReadTest {

    public static void main(String[] args) {
        final MyTask myTask = new MyTask();

        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.write();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.write();
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```

输出结果：

```
t1 start
t1 end
t2 start
t2 end
```

**读写互斥**

```
public class MyTask {

    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public void read() {
        try {
            lock.readLock().lock();
            System.out.println(Thread.currentThread().getName() + " start");
            Thread.sleep(10000);
            System.out.println(Thread.currentThread().getName() + " end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.readLock().unlock();
        }
    }

    public void write() {
        try {
            lock.writeLock().lock();
            System.out.println(Thread.currentThread().getName() + " start");
            Thread.sleep(10000);
            System.out.println(Thread.currentThread().getName() + " end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.writeLock().unlock();
        }
    }
}

public class ReadReadTest {

    public static void main(String[] args) {
        final MyTask myTask = new MyTask();

        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.read();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.write();
            }
        }, "t2");

        t1.start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        t2.start();
    }
}
```

输出结果：

```
t1 start
t1 end
t2 start
t2 end
```

**写读互斥**

```
public class ReadReadTest {

    public static void main(String[] args) {
        final MyTask myTask = new MyTask();

        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.write();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                myTask.read();
            }
        }, "t2");

        t1.start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        t2.start();
    }
}
```

输出结果：

```
t1 start
t1 end
t2 start
t2 end
```

**总结**

读操作的锁叫共享锁，写操作的锁叫排他锁。就是遇见写锁就需互斥。那么以此可得出读读共享，写写互斥，读写互斥，写读互斥。
