# CountDownLatch

**CountDownLatch介绍**

先来解读一下CountDownLatch这个类名字的意义。CountDown代表计数递减，Latch是“门闩”的意思。也有人把它称为“屏障”。而CountDownLatch这个类的作用也很贴合这个名字的意义，假设某个线程在执行任务之前，需要等待其它线程完成一些前置任务，必须等所有的前置任务都完成，才能开始执行本线程的任务。

CountDownLatch的方法也很简单，如下：

```text
// 构造方法：
public CountDownLatch(int count)

public void await() // 等待
public boolean await(long timeout, TimeUnit unit) // 超时等待
public void countDown() // count - 1
public long getCount() // 获取当前还有多少count
```

**CountDownLatch案例**

我们知道，玩游戏的时候，在游戏真正开始之前，一般会等待一些前置任务完成，比如“加载地图数据”，“加载人物模型”，“加载背景音乐”等等。只有当所有的东西都加载完成后，玩家才能真正进入游戏。下面我们就来模拟一下这个demo。

```text
public class PhaserDemo {
    static class PreTaskThread implements Runnable {

        private String task;
        private Phaser phaser;

        public PreTaskThread(String task, Phaser phaser) {
            this.task = task;
            this.phaser = phaser;
        }

        @Override
        public void run() {
            for (int i = 1; i < 4; i++) {
                try {
                    // 第二次关卡起不加载NPC，跳过
                    if (i >= 2 && "加载新手教程".equals(task)) {
                        continue;
                    }
                    Random random = new Random();
                    Thread.sleep(random.nextInt(1000));
                    System.out.println(String.format("关卡%d，需要加载%d个模块，当前模块【%s】",
                            i, phaser.getRegisteredParties(), task));

                    // 从第二个关卡起，不加载NPC
                    if (i == 1 && "加载新手教程".equals(task)) {
                        System.out.println("下次关卡移除加载【新手教程】模块");
                        phaser.arriveAndDeregister(); // 移除一个模块
                    } else {
                        phaser.arriveAndAwaitAdvance();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) {
        Phaser phaser = new Phaser(4) {
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println(String.format("第%d次关卡准备完成", phase + 1));
                return phase == 3 || registeredParties == 0;
            }
        };

        new Thread(new PreTaskThread("加载地图数据", phaser)).start();
        new Thread(new PreTaskThread("加载人物模型", phaser)).start();
        new Thread(new PreTaskThread("加载背景音乐", phaser)).start();
        new Thread(new PreTaskThread("加载新手教程", phaser)).start();
    }
}
```

输出：

```text
关卡1，需要加载4个模块，当前模块【加载背景音乐】
关卡1，需要加载4个模块，当前模块【加载新手教程】
下次关卡移除加载【新手教程】模块
关卡1，需要加载3个模块，当前模块【加载地图数据】
关卡1，需要加载3个模块，当前模块【加载人物模型】
第1次关卡准备完成
关卡2，需要加载3个模块，当前模块【加载地图数据】
关卡2，需要加载3个模块，当前模块【加载背景音乐】
关卡2，需要加载3个模块，当前模块【加载人物模型】
第2次关卡准备完成
关卡3，需要加载3个模块，当前模块【加载人物模型】
关卡3，需要加载3个模块，当前模块【加载地图数据】
关卡3，需要加载3个模块，当前模块【加载背景音乐】
第3次关卡准备完成
```

**Phaser原理**

请各位看官移步[这里](https://www.jianshu.com/p/a9a713cba61a)

