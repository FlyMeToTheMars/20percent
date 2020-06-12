# 17/20第17课 读写锁 ReentrantReadWriteLock 深入分析

### 前言

上两篇的内容中已经介绍到了锁的实现主要有`ReentrantLock`和`ReentrantReadWriteLock`。

ReentrantLock 是重入锁，顾名思义就是支持重进入的锁，他表示该锁能够支持一个线程对资源的重复加锁，上文中已经提到在 **AQS** 中的同步状态 **state**，如果是`0`的话表示该资源没有被线程加锁，如果是`大于0`则表示该资源被当前线程重入的次数。

另外，我们还需要知道最简单的同步关键字 synchronized 也是支持锁重入的，但他和`ReentrantLock`相比，`synchronized`正如他的加锁和释放锁一样都是隐式的，在前几篇文章中也对比了关于`Lock`和`synchronized`的区别，这里比在赘述。

另外，`ReentrantLock`是支持公平锁和非公平锁的，提供了构造函数允许设置是否为公平锁，默认是非公平锁的，这是因为，根据统计衡量如果使用公平锁会有大量的线程上下文切换，而使用非公平锁的话相对较少一些，这也是为什么默认的是使用非公平锁。

上一篇文章在介绍到锁优化的时候，建议将锁分离使用读写锁，这一片我们就一起学习一下读写锁`ReentrantReadWriteLock`。

### ReentrantReadWriteLock 简介和代码结构

![这里写图片描述](https://img-blog.csdn.net/20171028153311250?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

上图中我们可以看到 ReentrantReadWriteLock 的大致结构。之所以称之为读写锁，是因为 ReentrantReadWriteLock 内部的两个内部类：ReadLock 和 WriteLock，这两个内部类实现了最基本的 Lock 接口。

读写锁维护了一对锁：一个读锁和一个写锁。通过分离读锁和写锁，使得并发性相比一般的排它锁有很大的性能提升。

ReentrantReadWriteLock 的特性：

![这里写图片描述](https://img-blog.csdn.net/20171028153913371?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

（1）ReentrantReadWriteLock 类结构和内部类：

![这里写图片描述](https://img-blog.csdn.net/20171028154414986?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

（2）ReentrantReadWriteLock 的内部类：

![这里写图片描述](https://img-blog.csdn.net/20171028154618287?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看出 ReentrantReadWriteLock 和 ReentrantLock 的区别是增加了 ReadLock 和 WriteLock，其他的主要是不同内部类的实现方法的不同，下边看一下 Lock 相关的接口：

（3）ReadWriteLock 接口：

![这里写图片描述](https://img-blog.csdn.net/20171028162137814?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

（4）ReadLock 和 WriteLock 接口：

![这里写图片描述](https://img-blog.csdn.net/20171028162239372?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

（5）ReadWriteLock 提供了`readLock（）`和`writeLock（）`方法，类似于**工厂方法模式**的**工厂接口**，而 Lock 就是返回的**产品接口**。而 ReentrantReadWriteLock 实现了 ReadWriteLock 接口，那么他就是具体的**工厂接口实现类**，ReadLock 和 WriteLock 就成了**具体产品的实现类**，一个简单的工厂方法模式使用案例，值得学习。

![这里写图片描述](https://img-blog.csdn.net/20171028163039110?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

根据上述几张图应该大致清楚了各个类和接口之间的关系了。

### ReentrantReadWriteLock 使用案例

通过一个使用 HashMap 实现的 Cache 来了解一下 ReentrantReadWriteLock 的使用，代码如下：

```
public class ReentrantReadWriteLockCache {

    private static Map<String, Object> map = new HashMap<>();
    private static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private static Lock readLock = readWriteLock.readLock();
    private static Lock writeLock = readWriteLock.writeLock();

    private static Object get(String key) {
        readLock.lock();
        Object result = map.get(key);
        readLock.unlock();
        return result;
    }

    private static void put(String key, Object value) {
        writeLock.lock();
        map.put(key, value);
        writeLock.unlock();
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            int index = i;
            executorService.execute(() -> {
                put(index + "", index);
            });
        }

        for (int i = 0; i < 10; i++) {
            int index = i;
            executorService.execute(() -> get(index + ""));
        }

        executorService.shutdown();
    }

}
```

### ReentrantReadWriteLock 原理分析

**1、读写锁同步状态的设计**

ReentrantLock 我们知道他是一个**排他锁**，使用的是 AQS 中的一个同步状态 **state** 表示当前共享资源是否被其他线程锁占用。如果为0则表示未被占用，其他值表示该锁被重入的次数。

ReentrantReadWriteLock 中如何使用一个整数来表示读写状态哪？

由 ReentrantReadWriteLock 读写锁的特性，我们应该知道需要在 AQS 的同步状态上维护多个读线程和一个写现成的状态。

如何在一个**整型变量**上维护多种状态，就需要"**按位切割使用**" 这个变量，读写锁将变量切分成两个部分，**高16位表示读，低16位表示写**，划分方式如下图：

![这里写图片描述](https://img-blog.csdn.net/20171028181910524?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当前状态表示一个线程已经获取了写锁，且重入了两次，同时也获取了两次读锁。那么读写锁是如何迅速确定读和写各自的状态那？答案就是"**位运算**" 。

如何通过位运算计算得出是读还是写获取到锁了那？

如果当前同步状态state不为0，那么先计算低16位写状态，如果低16为为0，也就是写状态为0则表示高16为不为0，也就是读状态不为0，则读获取到锁；如果此时低16为不为0则抹去高16位得出低16位的值，判断是否与 state 值相同，如果相同则表示写获取到锁。同样如果 state 不为0，低16为不为0，且低16位值不等于state，也可以通过 state 的值减去低16位的值计算出高16位的值。上述计算过程都是通过位运算计算出来的。

上图中为什么表示当前状态有一个线程已经获取了**写锁**，且重入了两次，同时也获取了两次读锁。这是因为：

**2、写锁的获取与释放**

写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者线程不是已经获取写锁的线程，则当前线程进入等待状态。

**3、读锁的获取与释放**

读锁是一个支持重进入的共享锁。它能够被多个线程同时获取，在没有其他线写线程访问（写状态为0）时，读锁总是会被成功获取，而所作的也只是增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。

**4、锁降级**

看到上述这两段似乎还是找不到为什么会出现高位和低位都不为0的情况怎样确定当前线程获取的是写锁的解答，这就需要我们从另一个需要注意的地方说起：**锁降级**, 何为锁降级，意思**主要是为了保证数据的可见性**，假如有一个线程 A 已经获取了写锁，并且修改了数据，如果当前线程 A 不获取读锁而直接释放写锁，此时，另一个线程 B 获取到了写锁并修改了数据，那么当前线程 A 无法感知线程 B 的数据更新。如果当前线程 A 获取读锁，即遵循降级的步骤，则线程 B 将会被阻塞，直到当前线程 A 使用数据并释放读锁之后，线程 B 才能获取写锁进行数据更新。

另外，**锁降级中读锁的获取是必要的**！！！

正是由于锁降级的存在，才会出现上图中高16位和低16为都不为0，但可以确定是写锁的问题。可以得出结论，如果高16为或者低16为为0，那么我们就可以判断获取到的是写锁或读锁；如果高16位和低16位都不为0那获取到的应该是写锁。**就是说如果当前线程已经获取到写锁的话，该线程也是可以通过 CAS 线程安全的增加读状态的，成功获取读锁。**

虽然，为了保证数据的可见性引入锁降级可以将写锁降级为读锁，但是却不可以锁升级，将读锁升级为写锁的，也就是不会出现：当前线程已经获取到读锁了，通过某种方式增加写状态获取到写锁的情况。不允许升级的原因也是保证数据的可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的。

**5、锁降级实例演示：**

举个例子更清楚一些，示例是：并发包中 ReentrantReadWriteLock 读写锁的锁降级模板，代码如下：

```
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private Lock readLock = lock.readLock();
    private Lock writeLock = lock.writeLock();

    private boolean update;

    public void processData() {
        readLock.lock(); //读锁获取
        if (!update) {
            readLock.unlock(); //必须先释放读锁
            writeLock.lock(); //锁降级从获取写锁开始
            try {
                if (!update) {
                    //准备数据流程（略）
                    update = true;
                }
                //获取读锁。在写锁持有期间获取读锁
                //此处获取读锁，是为了防止，当释放写锁后，又有一个线程T获取锁，对数据进行改变，
                //而当前线程下面对改变的数据无法感知。
                //如果获取了读锁，则线程T则被阻塞，直到当前线程释放了读锁，那个T线程才有可能获取写锁。
                readLock.lock();
            } finally {
                writeLock.unlock();//释放写锁
            }
            //锁降级完成
        }

        try {
            //使用数据的流程
        } finally {
            readLock.unlock(); //释放读锁
        }
    }
```

### LockSupport 的简要介绍

不管是 ReentrantReadWriteLock 还是 ReentrantLock，当需要阻塞或唤醒一个线程的时候，都会使用 LockSupport 工具类完成相应的工作。LockSupport 方法如下：

![这里写图片描述](https://img-blog.csdn.net/20171028220828161?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

LockSupport 以 park 开头的方法表示阻塞，以 unpark 开头的方法表示唤醒，具体含义如下：

![这里写图片描述](https://img-blog.csdn.net/20171028220942731?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGxnZW4xNTczODc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 总结

1、读锁的重入是允许多个申请读操作的线程的，而写锁同时只允许单个线程占有，该线程的写操作可以重入。

2、如果一个线程占有了写锁，在不释放写锁的情况下，它还能占有读锁，即写锁降级为读锁。

3、对于同时占有读锁和写锁的线程，如果完全释放了写锁，那么它就完全转换成了读锁，以后的写操作无法重入，在写锁未完全释放时写操作是可以重入的。

4、公平模式下无论读锁还是写锁的申请都必须按照 AQS 锁等待队列先进先出的顺序。非公平模式下读操作插队的条件是锁等待队列 head 节点后的下一个节点是 SHARED 型节点，写锁则无条件插队。

5、读锁不允许 newConditon 获取 Condition 接口，而写锁的 newCondition 接口实现方法同 ReentrantLock。

------

参考文章

http://www.jianshu.com/p/9f98299a17a5

http://www.cnblogs.com/shangxiaofei/p/5807692.html

部分截图参考资料《Java 并发编程的艺术》