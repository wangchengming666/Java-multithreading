# 线程安全和死锁

**什么叫线程安全？**

线程安全就是多线程访问时，采用了加锁机制，当一个线程访问该类的某个数据时，进行保护，其他线程不能进行访问直到该线程读取完，其他线程才可使用。不会出现数据不一致或者数据污染。

**如果线程不安全会出现什么情况？**

举个例子，正常情况下，一张火车票只能卖给一个人，但是在售票的核心代码块的地方你并没有加锁，就可能会出现线程不安全的情况，从而导致一张火车票可能会卖给多个人的情况发生。

```text
static class SellTickets implements Runnable {

        @Override
        public void run() {
            while (tickets > 0) {


                System.out.println(Thread.currentThread().getName() + "--->售出第：  " + tickets + " 票");
                tickets--;
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }

            if (tickets <= 0) {

                System.out.println(Thread.currentThread().getName() + "--->售票结束！");
            }
        }
    }

    public static void main(String[] args) {
        SellTickets sell = new SellTickets();

        Thread thread1 = new Thread(sell, "1号窗口");
        Thread thread2 = new Thread(sell, "2号窗口");
        Thread thread3 = new Thread(sell, "3号窗口");
        Thread thread4 = new Thread(sell, "4号窗口");

        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
    }
```

输出结果如下：

```text
1号窗口--->售出第：  10 票
4号窗口--->售出第：  10 票
3号窗口--->售出第：  10 票
2号窗口--->售出第：  10 票
3号窗口--->售出第：  6 票
2号窗口--->售出第：  6 票
4号窗口--->售出第：  6 票
1号窗口--->售出第：  6 票
3号窗口--->售出第：  2 票
4号窗口--->售出第：  2 票
2号窗口--->售出第：  2 票
1号窗口--->售出第：  2 票
3号窗口--->售票结束！
1号窗口--->售票结束！
4号窗口--->售票结束！
2号窗口--->售票结束！

Process finished with exit code 0
```

**如何实现线程安全**

* 使用自动锁synchronized，但是在大数据量的场景下，效率比较低。
* 使用安全类，比如 java.util.concurrent 下的类，使用原子类AtomicInteger，推荐。
* 使用手动锁 Lock，推荐。

那么修改一下上面的代码，采用lock\(\)加锁，unlock\(\)解锁，来保护指定的代码块。

```text
    static class SellTickets implements Runnable {

        Lock lock = new ReentrantLock();

        @Override
        public void run() {
            // Lock锁机制
            while (tickets > 0) {
                try {
                    lock.lock();
                    if (tickets <= 0) {
                        return;
                    }
                    System.out.println(Thread.currentThread().getName() + "--->售出第：  " + tickets + " 票");
                    tickets--;
                } catch (Exception e1) {
                    // TODO Auto-generated catch block
                    e1.printStackTrace();
                } finally {
                    lock.unlock();
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }

            if (tickets <= 0) {

                System.out.println(Thread.currentThread().getName() + "--->售票结束！");
            }
        }
    }
```

输出结果如下：

```text
1号窗口--->售出第：  10 票
2号窗口--->售出第：  9 票
3号窗口--->售出第：  8 票
4号窗口--->售出第：  7 票
3号窗口--->售出第：  6 票
2号窗口--->售出第：  5 票
1号窗口--->售出第：  4 票
4号窗口--->售出第：  3 票
3号窗口--->售出第：  2 票
1号窗口--->售出第：  1 票
4号窗口--->售票结束！
2号窗口--->售票结束！
3号窗口--->售票结束！
1号窗口--->售票结束！

Process finished with exit code 0
```

**什么是死锁**

死锁指的是当线程t1占有资源A，线程t2占有资源B，这个时候t1同时去请求资源B，t2同时去请求资源A，这是两个线程就会互相等待而进入死锁状态。

**形成死锁的四个必要条件是什么**

* 互斥条件：线程\(进程\)对于所分配到的资源具有排它性，即一个资源只能被一个线程\(进程\)占用，直到被该线程\(进程\)释放
* 请求与保持条件：一个线程\(进程\)因请求被占用资源而发生阻塞时，对已获得的资源保持不放。
* 不剥夺条件：线程\(进程\)已获得的资源在末使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。
* 循环等待条件：当发生死锁时，所等待的线程\(进程\)必定会形成一个环路（类似于死循环），造成永久阻塞

**如何避免线程死锁**

我们只要破坏产生死锁的四个条件中的其中一个就可以了。

* 破坏互斥条件

这个条件我们没有办法破坏，因为我们用锁本来就是想让他们互斥的（临界资源需要互斥访问）。

* 破坏请求与保持条件

一次性申请所有的资源。

* 破坏不剥夺条件

占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

* 破坏循环等待条件

靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。

