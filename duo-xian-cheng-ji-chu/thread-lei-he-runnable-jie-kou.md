# Thread类和Runnable接口

如果我们需要有一个“线程”类，JDK提供了`Thread`类和`Runnalble`接口来让我们实现自己的“线程”类。

* 继承Thread类，并重写run方法（注意：Thread类实现了Runnable接口）

```text
public class Thread implements Runnable { }
```

* 实现Runnable接口的run方法

**继承Thread类**

```text
    static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("MyThread");
        }
    }

    public static void main(String[] args) {
        Thread t1 = new MyThread();
        t1.start();
        System.out.println(t1.getName() + "运行结束");
    }
```

使用`start()`方法启动一个线程，注意不能多次调用`start()`方法，否则会抛出异常。

**实现Runnable接口**

```text
    static class MyThread1 implements Runnable {
        @Override
        public void run() {
            System.out.println("MyThread1");
        }
    }

    public static void main(String[] args) {
        new Thread(new MyThread1()).start();
        System.out.println("运行结束");
    }
```

**Thread类的几个常用方法**

* `currentThread()`：静态方法，返回对当前正在执行的线程对象的引用

该方法可以可以返回代码段正在被哪个线程调用的信息。实例代码如下：

```text
    public static void main(String[] args) {
         System.out.println(Thread.currentThread().getName());
    }
```

结果会在控制台打印`main`，证明main方法正在被名字叫main的线程调用。 修改代码如下：

```text
    static class MyThread extends Thread {

        public MyThread() {
            System.out.println("构造方法打印：" + Thread.currentThread().getName());
        }

        @Override
        public void run() {
            System.out.println("run方法打印：" + Thread.currentThread().getName());
        }
    }
    public static void main(String[] args) {
        Thread t1 = new MyThread();
        t1.start();
    }
```

输出结果如下，证明构造函数是被main线程调用的，而`run()`方法是被名叫“Thread-0”调用的。

```text
构造方法打印：main
run方法打印：Thread-0
```

再次修改代码如下：

```text
    static class MyThread extends Thread {

        public MyThread() {
            System.out.println("构造方法打印：" + Thread.currentThread().getName());
        }

        @Override
        public void run() {
            System.out.println("run方法打印：" + Thread.currentThread().getName());
        }
    }

    static class MyThread1 implements Runnable {
        @Override
        public void run() {
            System.out.println("MyThread1");
        }
    }

    public static void main(String[] args) {
        Thread t1 = new MyThread();
        // t1.start();
        t1.run();
    }
```

输出结果如下，证明两个线程都是被main调用的。

```text
构造方法打印：main
run方法打印：main
```

* `isAlive()`方法：是判断当前线程是不是出于活动状态
* `sleep()`方法：静态方法，使当前线程睡眠一段时间
* `start()`：开始执行线程的方法，java虚拟机会调用线程内的run\(\)方法
* `yield()`：yield在英语里有放弃的意思，同样，这里的yield\(\)指的是当前线程愿意让出对当前处理器的占用。

```text
    static class MyThread extends Thread {
        @Override
        public void run() {
            long beginTime = System.currentTimeMillis();
            int count = 0;
            for (int i = 0; i < 5000000; i++) {
                Thread.yield();
                count = count + (i + 1);
            }
            long endTime = System.currentTimeMillis();
            System.out.println("用时：" + (endTime - beginTime));
        }
    }

    public static void main(String[] args) {
        Thread t1 = new MyThread();
        t1.start();
    }
```

当 `Thread.yield();`这句代码被注释掉之后，输出结果为`用时：4`。 当`Thread.yield();`这句代码没有被注释掉之后，输出结果为`用时：1515`。所以证明`yield()`方法会放弃当前资源，将CPU让给其他资源做事情，所以导致速度变慢。

* `Thread.stop()`：暴力停止线程， 不推荐这么做。
* `Thread.interrupt()`：推荐使用此方法。此方法是在当前线程中打印一个停止的标记，并不是真正的停止线程。
  * this.interrupted\(\)，测试当前线程是否已经中断，执行后具有将状态标志清除为false的功能
  * this.isInterrupted\(\)，测试线程Thread对象是否已经是中断状态，但不会清除状态标志

**守护线程**

User Thread（用户线程）和Daemon Thread（守护线程）从本质上来说并没有什么区别，唯一的不同之处就在于虚拟机的离开：如果用户线程已经全部退出运行了，只剩下守护线程存在了，虚拟机也就退出了。 因为没有了被守护者，守护线程也就没有工作可做了，也就没有继续运行程序的必要了。

**线程的优先级**

首先看下JDK对线程优先级的设置有哪些：

```text
    /**
     * The minimum priority that a thread can have.
     */
    public final static int MIN_PRIORITY = 1;

   /**
     * The default priority that is assigned to a thread.
     */
    public final static int NORM_PRIORITY = 5;

    /**
     * The maximum priority that a thread can have.
     */
    public final static int MAX_PRIORITY = 10;
```

那么JDK是通过什么方法设置线程的优先级呢？答案是通过`setPriority(int newPriority)`这个方法设置优先级，参数`newPriority`越大，优先级越高。但是需要注意的是优先级虽然高，占得CPU资源较多，但是也不能保证优先级高的线程全部执行完，因为`优先级具有随机性`。

**Thread类与Runnable接口的比较**

实现一个自定义的线程类，可以有继承Thread类或者实现Runnable接口这两种方式，它们之间有什么优劣呢？

* 由于Java“单继承，多实现”的特性，Runnable接口使用起来比Thread更灵活。
* Runnable接口出现更符合面向对象，将线程单独进行对象的封装。
* Runnable接口出现，降低了线程对象和线程任务的耦合性。
* 如果使用线程时不需要使用Thread类的诸多方法，显然使用Runnable接口更为轻量。

所以，我们通常优先使用“实现Runnable接口”这种方式来自定义线程类。

**参考文章**

* [Java中的守护线程](https://www.cnblogs.com/yanggb/p/11702843.html)
* 《Java多线程编程核心技术》

