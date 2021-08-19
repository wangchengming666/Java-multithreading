# 乐观锁和悲观锁

锁可以从不同的角度分类。其中，乐观锁和悲观锁是一种分类方式。

**悲观锁**

悲观锁就是我们常说的锁。对于悲观锁来说，它总是认为每次访问共享资源时会发生冲突，所以必须对每次数据操作加上锁，以保证临界区的程序同一时间只能有一个线程在执行。

**乐观锁**

乐观锁又称为"无锁"，顾名思义，它是乐观派。乐观锁总是假设对共享资源的访问没有冲突，线程可以不停地执行，无需加锁也无需等待。而一旦多个线程发生冲突，乐观锁通常是使用一种称为CAS的技术来保证线程执行的安全性。

由于无锁操作中没有锁的存在，因此不可能出现死锁的情况，也就是说乐观锁天生免疫死锁。

**两种锁的使用场景**

乐观锁多用于“读多写少“的环境，避免频繁加锁影响性能；而悲观锁多用于”写多读少“的环境，避免频繁失败和重试影响性能。

**悲观锁常见的实现方式**

* 使用`synchronized`

```text
public synchronized void testMethod() {
      //操作同步资源 
 }
```

* 使用`Lock`

```text
Private ReentrantLock lock = new ReentrantLock(); 

Public void modifyPublicResource() {
    lock.lock();
    //操作同步资源
    lock.unlock();
}
```

**乐观锁常见的实现方式**

* 版本号机制

一般是在数据表中加上一个数据版本号version字段，表示数据被修改的次数，当数据被修改时，version值会加一。当线程A要更新数据值时，在读取数据的同时也会读取version值，在提交更新时，若刚才读取到的version值为当前数据库中的version值相等时才更新，否则重试更新操作，直到更新成功。

* CAS算法

Java.util.concurrent包中的原子类就是通过CAS实现了乐观锁。

```text
private AtomicInteger atomicInteger = new AtomicInteger();  

AtomicInteger.incrementAndGet();
```

