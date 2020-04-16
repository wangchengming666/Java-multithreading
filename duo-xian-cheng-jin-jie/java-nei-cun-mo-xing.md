# 共享内存并发模型

**首先先了解一下Java运行时内存的划分：**

![](https://img-blog.csdnimg.cn/20200328164222468.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size_16,color_FFFFFF,t_70)

对于每一个线程来说，栈都是私有的，而堆是共有的。而在堆中的变量是共享的，称为共享变量。所以，内存可见性是针对的**共享变量**。

**JMM（**Java内存模型**）通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证**。

![](https://img-blog.csdnimg.cn/20200330091315859.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdjaGVuZ21pbmcx,size_16,color_FFFFFF,t_70)

在这里可以看到所有的共享变量都是在主内存中的。如果线程B想去获取线程A的变量，线程A需要刷新变量到主内存中，然后线程B再去主内存中读取线程A刚刚刷新的共享变量，所以说线程A和线程B之间的通信，必须要经过主内存才可以实现。

根据JMM的规定，**线程对共享变量的所有操作都必须在自己的本地内存中进行，不能直接从主内存中读取**。所以线程B并不是直接去主内存中读取共享变量的值，而是先在本地内存B中找到这个共享变量，发现这个共享变量已经被更新了，然后本地内存B去主内存中读取这个共享变量的新值，并拷贝到本地内存B中，最后线程B再读取本地内存B中的新值。

