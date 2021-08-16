# Java线程的状态

首先从一张图片来直观的了解一下线程的状态，图片来源于网络。 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200318105953491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size_16,color_FFFFFF,t_70) 

然后再看一下`Thread.State`这个枚举类，定义了线程的六种状态。

```text
public enum State {
    /**
     * 处于NEW状态的线程此时尚未启动。
     * 指的是线程建好了，但是并没有调用Thread实例的start()方法。
     */
    NEW,
    /**
     * 表示当前线程正在运行中。处于RUNNABLE状态的线程在Java虚拟机中运行，也有可能在等待其他系统资源（比如I/O）
     */
    RUNNABLE,
    /**
     * 阻塞状态。处于BLOCKED状态的线程正等待锁的释放以进入同步区。
     * 举个例子：比如你在小卖部窗口买东西（假设小卖部只有一个窗口），你前面有一个正在买东西，这个时候你必须要等待前面的人走了之后才可以买东西。
     * 假设你是线程t2，你前面的那个人是线程t1。此时t1占有了锁（小卖部唯一的窗口），t2正在等待锁的释放，所以此时t2就处于BLOCKED状态。
     */
    BLOCKED,
    /**
     * 等待状态。处于等待状态的线程变成RUNNABLE状态需要其他线程唤醒。
     * 举个例子：这个时候经过你几分钟的等待，前面的人（t1）买好东西走了，这个时候你（t2）获得了锁（就是来到了窗口前），
     * 但是有个人非常不友好的把你叫走了（假设他是t3），这个时候你就不得不释放掉刚刚得到的锁，此时你的状态就是WAITING状态。
     * 要是t3不主动唤醒你t2（notify、notifyAll..），可以说你t2只能一直等待了。
     */
    WAITING,
    /**
     * 超时等待状态。线程等待一个具体的时间，时间到后会被自动唤醒。
     * 举个例子：t3唤醒你之后，你来到了小卖部窗口前。但是不巧的是又来了个人（t4）说让你等5分钟再买，去帮他拿一下快递。这时你还是线程t2，让你拿快递的人是线程t4。
     * t4让t2等待了指定时间，t2先主动释放了锁。此时t2等待期间就属于TIMED_WATING状态。t2等待10分钟后，就自动唤醒，拥有了去争夺锁的资格。
     */
    TIMED_WAITING,
    /**
     * 终止状态。此时线程已执行完毕。
     * 举个例子：你已经买到了你想要的东西，整个线程结束。
     */
    TERMINATED;
}
```

**线程之间的转换**

![](https://img-blog.csdnimg.cn/2020031814324423.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size_16,color_FFFFFF,t_70)

* BLOCKED与RUNNABLE状态的转换

```text
public class ThreadState {

    public static void main(String[] args) throws InterruptedException {
        blockedTest();
    }

    public static void blockedTest() throws InterruptedException {

        Thread a = new Thread(new Runnable() {
            @Override
            public void run() {
                test();
            }
        }, "a");
        Thread b = new Thread(new Runnable() {
            @Override
            public void run() {
                test();
            }
        }, "b");

        a.start();
        b.start();
        System.out.println(a.getName() + ":" + a.getState()); // 输出？
        System.out.println(b.getName() + ":" + b.getState()); // 输出？
    }

    public static synchronized void test() {
        try {
            Thread.sleep(2000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果如下：

```text
a:TIMED_WAITING
b:BLOCKED
```

* WAITING状态与RUNNABLE状态的转换

使线程从RUNNABLE状态转为WAITING状态可以使用`Object.wait()`，`Thread.join()`方法

```text
public class ThreadState {

    public static void main(String[] args) throws InterruptedException {
        blockedTest();
    }

    public static void blockedTest() throws InterruptedException {

        Thread a = new Thread(new Runnable() {
            @Override
            public void run() {
                test();
            }
        }, "a");
        Thread b = new Thread(new Runnable() {
            @Override
            public void run() {
                test();
            }
        }, "b");

        a.start();
        a.join();// 这里调用join方法，join()方法不会释放锁，会一直等待当前线程执行完毕（转换为TERMINATED状态）。
        b.start();
        System.out.println(a.getName() + ":" + a.getState()); // 输出？
        System.out.println(b.getName() + ":" + b.getState()); // 输出？
    }

    public static synchronized void test() {
        try {
            Thread.sleep(2000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

```text
a:TERMINATED
b:TIMED_WAITING
```

* TIMED\_WAITING与RUNNABLE状态转换

  TIMED\_WAITING与WAITING状态类似，只是TIMED\_WAITING状态等待的时间是指定的。

  * Thread.sleep\(long\)：使当前线程睡眠指定时间。需要注意这里的“睡眠”只是暂时使线程停止执行，并不会释放锁。时间到后，线程会重新进入RUNNABLE状态。
  * Object.wait\(long\)：wait\(long\)方法使线程进入TIMED\_WAITING状态。这里的wait\(long\)方法与无参方法wait\(\)相同的地方是，都可以通过其他线程调用notify\(\)或notifyAll\(\)方法来唤醒。不同的地方是，有参方法wait\(long\)就算其他线程不来唤醒它，经过指定时间long之后它会自动唤醒，拥有去争夺锁的资格。
  * Thread.join\(long\)：join\(long\)使当前线程执行指定时间，并且使线程进入TIMED\_WAITING状态。

