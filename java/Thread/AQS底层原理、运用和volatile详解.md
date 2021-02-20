## AbstractQueuedSynchronizer
>什么叫做AQS？从名字可以看出，AQS就是抽象队列同步器，是用来构建锁或者其他同步组件的基础框架。它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。并发包的大师（Doug Lea）期望它能够成为实现大部分同步需求的基础。
#### AQS使用方式和其中的设计模式
AQS的主要使用方式是继承，子类通过继承AQS并实现它的抽象方法来管理同步状态，在AQS里由一个int型的state来代表这个状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法``getState()``、``setState(int newState)``和``compareAndSetState(int expect,int update)``来进行操作，因为它们能够保证状态的改变是安全的。
```java
  /**
   * The synchronization state.
   */
  private volatile int state;
```
在实现上，子类推荐被定义为自定义同步组件的静态内部类，AQS自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器。可以这样理解二者之间的关系：**锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；**

同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

实现者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

##### 模板方法模式

同步器的设计基于模板方法模式。模板方法模式的意图是，定义一个操作中的算法的骨架，而将一些步骤的实现延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。我们最常见的就是Spring框架里的各种Template。

**来看这么一个例子：**

我们开了个蛋糕店，蛋糕店不能只卖一种蛋糕呀，于是我们决定先卖奶油蛋糕，芝士蛋糕和慕斯蛋糕。三种蛋糕在制作方式上一样，都包括造型，烘焙和涂抹蛋糕上的东西。所以可以定义一个抽象蛋糕模型。

```java
/**
 * 类说明：抽象蛋糕模型
 */
public abstract class AbstractCake {
    protected abstract void shape();
    protected abstract void apply();
    protected abstract void brake();

    /*模板方法*/
   public final void run(){
       this.shape();
       this.apply();
       this.brake();
   }
}
```
然后就可以批量生产三种蛋糕。
```java
/**
 * 类说明：芝士蛋糕
 */
public class CheeseCake  extends AbstractCake {

    @Override
    protected void shape() {
        System.out.println("芝士蛋糕造型");
    }

    @Override
    protected void apply() {
        System.out.println("芝士蛋糕涂抹");
    }

    @Override
    protected void brake() {
        System.out.println("芝士蛋糕烘焙");
    }
}

public class CreamCake extends AbstractCake {}
public class MouseCake extends AbstractCake {}
```
当我们使用的时候，则可以
```java
/**
 * 类说明：生产蛋糕
 */
public class MakeCake {
    public static void main(String[] args) {
        AbstractCake cake = new CheeseCake();
        AbstractCake cake2 = new CreamCake();
        //AbstractCake cake = new MouseCake();
        cake.run();
    }
}
```
这样一来，不但可以批量生产三种蛋糕，而且如果日后有扩展，只需要继承抽象蛋糕方法就可以了，十分方便，我们天天生意做得越来越赚钱。突然有一天，我们发现市面有一种最简单的小蛋糕销量很好，这种蛋糕就是简单烘烤成型就可以卖，并不需要涂抹什么食材，由于制作简单销售量大，这个品种也很赚钱，于是我们也想要生产这种蛋糕。

但是我们发现了一个问题，抽象蛋糕是定义了抽象的涂抹方法的，也就是说扩展的这种蛋糕是必须要实现涂抹方法，这就很鸡儿蛋疼了。怎么办？我们可以将原来的模板修改为带钩子的模板。
```java
 /*模板方法*/
    public final void run(){
        this.shape();
        if(this.shouldApply()){
            this.apply();
        }
        this.brake();
    }

    protected boolean shouldApply(){
        return true;
    }
```
做小蛋糕的时候通过flag来控制是否涂抹，其余已有的蛋糕制作不需要任何修改可以照常进行。
```java
/**
 * 类说明：小蛋糕
 */
public class SmallCake extends AbstractCake {

    private boolean flag = false;
    public void setFlag(boolean shouldApply){
        flag = shouldApply;
    }
    @Override
    protected boolean shouldApply() {
        return this.flag;
    }

}
```
#### 了解AQS其中的方法
实现自定义同步组件时，将会调用同步器提供的模板方法，而AQS为我们提供了如下所示的模版方法：
|方法名称|描述|
| ----------- | ---------------------------- |
| void acquire(int arg)   | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用重写的tryAcquire(intarg)方法            |
| void acquireInterruptibly(int arg)          | 与acquire(int arg)相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException 并返回 |
| boolean tryAcquireNanos(int arg,long nanos) | 在acquirenterpibyit arg)基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态。那么将会返回false. 如果获取到了返回true                  |
|       void acquireShared(int arg) |               共享式的获取同步状态，如果当前线程未获取到同步状态,将会进入同步队列等待，与独占式获取的主要区别是在同一时刻可以有多个线程获取到同步状态                  |      void acquireSharedInteruptibly(int arg)                  |与acquireShared(int arg)相同，该方法响应中断
|  boolean tryAcquireSbaredNanos(intarg, long nanos)|在acquiresharedntepriblylit arg)基硝上增加了超时限制
| boolean release(int arg)|独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第个节点包含的线程唤醒
| bboolean releaseShared(int arg)|共享式的释放同步状态
| ClelinrThreaea getQuevedTmeads()|获取等待在同步队列上的线程集合

这些模板方法同步器提供的模板方法基本上分为3类：独占式获取与释放同步状态、共享式获取与释放、同步状态和查询同步队列中的等待线程情况。

**可重写的方法**
| 方法名称        | 描述     |
| -------------------------------------- | -------------------- |
| protected boolean tryAcquiret(int arg) | 独占式获取同步状态，实现该方法需要查询当前状态,判断网步状态是否符合预期，然后再进行CAS设置同步状态
| protected boolean tryRelease(int arg) |独占式释放同少状态， 等待获取同步状态的线程将有机会获取同步状态
| protected int tryAcquireshared(in ag)| 共享式获取同步状态，返回大于等于0的值，表示获取成功，反之获取失败
| protected boolean tryReleaseShared(int arg) |共享式释放同步状态
| protected boolean isHeldExchusively() | 判断是否当前线程所独占

例如，我们来实现一个自己的独占锁，则可以基于以上方法。通过一个继承自AbstractQueuedSynchronizer的静态内部类，实现AQS提供的模版方法，并使用其作为代理作为独占锁的真实实现。
```java
public class SelfLock implements Lock {
// 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {

        /*判断处于占用状态*/
        @Override
        protected boolean isHeldExclusively() {
            return getState()==1;
        }

        /*获得锁*/
        @Override
        protected boolean tryAcquire(int arg) {
            if(compareAndSetState(0,1)){
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        /*释放锁*/
        @Override
        protected boolean tryRelease(int arg) {
            if(getState()==0){
                throw new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            //compareAndSetState(1,0);
            return true;
        }

        // 返回一个Condition，每个condition都包含了一个condition队列
        Condition newCondition() {
            return new ConditionObject();
        }
    }

    // 仅需要将操作代理到Sync上即可
    private final Sync sync = new Sync();

    @Override
    public void lock() {
    	System.out.println(Thread.currentThread().getName()+" ready get lock");
        sync.acquire(1);
        System.out.println(Thread.currentThread().getName()+" already got lock");
    }
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public void unlock() {
    	System.out.println(Thread.currentThread().getName()+" ready release lock");
        sync.release(1);
        System.out.println(Thread.currentThread().getName()+" already released lock");
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    @Override
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```
**访问或修改同步状态的方法**
重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。
- getState()：获取当前同步状态。
- setState(int newState)：设置当前同步状态。
- compareAndSetState(int expect,int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。

#### CLH队列锁
>CLH队列锁即Craig, Landin, and Hagersten (CLH) locks。

CLH队列锁也是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程仅仅在本地变量上自旋，它不断轮询前驱的状态，假设发现前驱释放了锁就结束自旋。

当线程A需要获取锁时：
1.	创建一个的QNode，将其中的locked设置为true表示需要获取锁，myPred表示对其前驱结点的引用

2.	线程A对tail域调用``getAndSet()``方法，使自己成为队列的尾部，同时获取一个指向其前驱结点的引用myPred

线程B需要获得锁，同样的流程再来一遍，myPred指向前驱结点A

3. 线程就在前驱结点的locked字段上旋转，直到前驱结点释放锁(前驱节点的锁值 locked == false)
4. 当一个线程需要释放锁时，将当前结点的locked域设置为false，同时回收前驱结点

前驱结点释放锁，线程A的myPred所指向的前驱结点的locked字段变为false，线程A就可以获取到锁。

CLH队列锁的优点是空间复杂度低（如果有n个线程，L个锁，每个线程每次只获取一个锁，那么需要的存储空间是O（L+n），n个线程有n个myNode，L个锁有L个tail）。CLH队列锁常用在SMP体系结构下。

**Java中的AQS是CLH队列锁的一种变体实现**。但是它的基本思想还是基于CLH队列锁实现的，只不过它的自旋并不是无限自旋，而是自旋一定次数后，就让线程进入一个阻塞的状态。

#### 实现ReentrantLock锁
在上面我们实现了一个独占锁，但是该独占锁只是简单的实现了同步锁的基本功能，但并没有实现公平于非公平，锁的可重入等机制。
##### 公平和非公平锁
ReentrantLock的构造函数中，默认的无参构造函数将会把Sync对象创建为NonfairSync对象，这是一个“非公平锁”；而另一个构造函数`ReentrantLock(boolean fair)`传入参数为true时将会把Sync对象创建为“公平锁”FairSync。

``nonfairTryAcquire(int acquires)``方法，只要CAS设置同步状态成功，则表示当前线程获取了锁。而公平锁则不同，``tryAcquire()``方法与``nonfairTryAcquire(int acquires)``比较，唯一不同的位置为判断条件多了``hasQueuedPredecessors()``方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。
```java
protected final boolean tryAcquire{
  ...
  if (!hasQueuedPredecessors() &&
    compareAndSetState(0, acquires)) {
    setExclusiveOwnerThread(current);
    return true;
  }
}
```

##### 锁的可重入
重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决以下两个问题。
1. 线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。例如线程访问A()方法需要获取一次锁，获取B()方法也需要获取一次锁，如果没有实现可重入的话，自己将会和自己造成死锁。
2. 锁的最终释放。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。

``nonfairTryAcquire()``方法增加了再次获取同步状态的处理逻辑：通过判断当前线程是否为获取锁的线程来决定获取操作是否成功，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回true，表示获取同步状态成功。同步状态表示锁被一个线程重复获取的次数。

如果该锁被获取了n次，那么前(n - 1)次``tryRelease(int releases)``方法必须返回false，而只有同步状态完全释放了，才能返回true。可以看到，该方法将同步状态是否为0作为最终释放的条件，当同步状态为0时，将占有线程设置为null，并返回true，表示释放成功。

所以，在之前的独占锁基础上要再实现可重入的话，我们需要将代码进行以下修改：
```java
/**
 *类说明：实现我们自己独占锁,可重入
 */
public class ReenterSelfLock implements Lock {
    // 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 当状态为0的时候获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }else if(getExclusiveOwnerThread() == Thread.currentThread()){
                setState(getState() + 1);
                return  true;
            }
            return false;
        }

        // 释放锁，将状态设置为0
        protected boolean tryRelease(int releases) {
            if(getExclusiveOwnerThread() != Thread.currentThread()){
                throw new IllegalMonitorStateException();
            }
            if (getState() == 0)
                throw new IllegalMonitorStateException();
            setState(getState() - 1);
            if(getState() == 0){
                setExclusiveOwnerThread(null);
            }
            return true;
        }
    }

}

```
我们来测试一下自己实现的可重入锁。
```java

/**
 * 类说明：
 */
public class TestReenterSelfLock {

    static final Lock lock = new ReenterSelfLock();

    public void reenter(int x) {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + ":递归层级:" + x);
            int y = x - 1;
            if (y == 0) return;
            else {
                reenter(y);
            }
        } finally {
            lock.unlock();
        }

    }

    public void test() {
        class Worker extends Thread {
            public void run() {
                System.out.println(Thread.currentThread().getName());
                SleepTools.second(1);
                reenter(3);
            }
        }
        // 启动3个子线程
        for (int i = 0; i < 3; i++) {
            Worker w = new Worker();
            w.start();
        }
        // 主线程每隔1秒换行
        for (int i = 0; i < 100; i++) {
            SleepTools.second(1);
        }
    }

    public static void main(String[] args) {
        TestReenterSelfLock testMyLock = new TestReenterSelfLock();
        testMyLock.test();
    }
}
```
## 深入理解并发编程归纳和总结
#### JMM基础-计算机原理
>Java内存模型即Java Memory Model，简称JMM。JMM定义了Java 虚拟机(JVM)在计算机内存(RAM)中的工作方式。JVM是整个计算机虚拟模型，所以JMM是隶属于JVM的。Java1.5版本对其进行了重构，现在的Java仍沿用了Java1.5的版本。物理计算机中的并发问题，物理机遇到的并发问题与虚拟机中的情况有不少相似之处，物理机对并发的处理方案对于虚拟机的实现也有相当大的参考意义。

物理计算机中的并发问题，物理机遇到的并发问题与虚拟机中的情况有不少相似之处，物理机对并发的处理方案对于虚拟机的实现也有相当大的参考意义。根据《Jeff Dean在Google全体工程大会的报告》，我们可以看到
| 操作                     | 响应时间  |
| ------------------------ | --------- |
| 打开一个站点             |  几秒      |
| 数据库查询条记录(有索引) | 十几毫秒  |
| 1.6G的CPU执行一 条指令   | 0.6纳秒   |
| 从机械磁盘顺序读取1M数据 | 2- 10毫秒 |
| 从SSD磁盘顺序读取1M数据  | 0.3毫秒   |
| 从内存连续读取1M数据     | 250微秒   |
| CPU读取一 次内存         | 100纳秒   |
| 1G网卡，网络传输2Kb数据  | 20微秒    |
- 1秒= 1000毫秒1毫秒=1000微秒  1微秒= 1000纳秒

如果从内存中读取1M的int型数据由CPU进行累加，耗时要多久？

做个简单的计算，1M的数据，Java里int型为32位，4个字节，共有1024\*1024/4 = 262144个整数 ，则CPU 计算耗时：262144 \*0.6 = 157 286 纳秒，而我们知道从内存读取1M数据需要250000纳秒，两者虽然有差距（当然这个差距并不小，十万纳秒的时间足够CPU执行将近二十万条指令了），但是还在一个数量级上。但是，没有任何缓存机制的情况下，意味着每个数都需要从内存中读取，这样加上CPU读取一次内存需要100纳秒，262144个整数从内存读取到CPU加上计算时间一共需要262144*100+250000 = 26 464 400 纳秒，这就存在着数量级上的差异了

而且现实情况中绝大多数的运算任务都不可能只靠处理器“计算”就能完成，处理器至少要与内存交互，如读取运算数据、存储运算结果等，这个I/O操作是基本上是无法消除的（无法仅靠寄存器来完成所有运算任务）。早期计算机中cpu和内存的速度是差不多的，但在现代计算机中，cpu的指令速度远超内存的存取速度,由于计算机的存储设备与处理器的运算速度有几个数量级的差距，所以现代计算机系统都不得不加入一层读写速度尽可能接近处理器运算速度的高速缓存（Cache）来作为内存与处理器之间的缓冲：将运算需要使用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就无须等待缓慢的内存读写了。

在计算机系统中，寄存器划是L0级缓存，接着依次是L1，L2，L3（接下来是内存，本地磁盘，远程存储）。越往上的缓存存储空间越小，速度越快，成本也更高；越往下的存储空间越大，速度更慢，成本也更低。从上至下，每一层都可以看做是更下一层的缓存，即：L0寄存器是L1一级缓存的缓存，L1是L2的缓存，依次类推；每一层的数据都是来至它的下一层，所以每一层的数据是下一层的数据的子集。

在现代CPU上，一般来说L0， L1，L2，L3都集成在CPU内部，而L1还分为一级数据缓存（Data Cache，D-Cache，L1d）和一级指令缓存（Instruction Cache，I-Cache，L1i），分别用于存放数据和执行数据的指令解码。每个核心拥有独立的运算处理单元、控制器、寄存器、L1、L2缓存，然后一个CPU的多个核心共享最后一层CPU缓存L3。

#### JMM-Java内存模型
从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

线程不允许直接访问主内存，也不允许去访问其他线程的本地内存，那么JMM会带来什么样的问题呢？我们来看这么一个例子：

主内存中有一个count变量，初始化为0，有A和B两个线程对count并行进行 +1 操作。由于工作内存的存在，A需要先读取count值，创建本地副本，进行+1后再写回主内存中。而同时，B也在进行这个操作，但是并不知道A已经修改过count的值了，于是最后count的值为1。跟我们想要的结果并不一致，产生了线程不安全的问题。

##### 可见性
可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

由于线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量，那么对于共享变量V，它们首先是在自己的工作内存，之后再同步到主内存。可是并不会及时的刷到主存中，而是会有一定时间差。很明显，这个时候线程 A 对变量 V 的操作对于线程 B 而言就不具备可见性了 。

要解决共享对象可见性这个问题，我们可以使用volatile关键字或者是加锁。

我们知道，使用volatile关键字可以在读写的时候强制线程去访问最新的主内存地址，但是仅仅使用volatile就够了吗？

##### 原子性
即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

我们都知道CPU资源的分配都是以线程为单位的,并且是分时调用,操作系统允许某个进程执行一小段时间，例如 50 毫秒，过了 50 毫秒操作系统就会重新选择一个进程来执行（我们称为“任务切换”），这个 50 毫秒称为“时间片”。而任务的切换大多数是在时间片段结束以后,

那么线程切换为什么会带来bug呢？因为操作系统做任务切换，可以发生在任何一条CPU 指令执行完！注意，是CPU 指令，CPU 指令，而不是高级语言里的一条语句。比如count++，在java里就是一句话，但高级语言里一条语句往往需要多条 CPU 指令完成。其实count++包含了多个CPU指令！

因此，由于count++操作不是原子操作的原因，当A和B线程多次调用count++，假如A已经完成了一次累加，count这时候为1，B读取到工作内存中，却刚好被操作系统剥夺了时间片，而A又再一次进行了count++，count变为2。对于B来说，当它重新分配到CPU时间片的时候，count的副本值仍然是1，这就产生了线程安全问题。

因为volatile不能保证原子性，所以我们可以使用synchronized关键字去保证线程的安全，因为它同时保证了可见性和原子性。

#### volatile详解
volatile变量自身具有下列特性：
可见性：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。

因此，可以把对volatile变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步。

volatile虽然能保证执行完及时把变量刷到主内存中，但对于count++这种非原子性、多指令的情况，由于线程切换，线程A刚把count=0加载到工作内存，线程B就可以开始工作了，这样就会导致线程A和B执行完的结果都是1，都写到主内存中，主内存的值还是1，而不是2。

##### volatile的实现原理
通过对OpenJDK中的unsafe.cpp源码的分析，会发现被volatile关键字修饰的变量会存在一个“Lock:”的前缀。

Lock不是一种内存屏障，但是它能完成类似内存屏障的功能，它会对CPU总线和高速缓存加锁，可以理解为CPU指令级的一种锁。同时该指令会将当前处理器缓存行的数据直接写会到系统内存中，且这个写回内存的操作会使在其他CPU里缓存了该地址的数据无效。

#### synchronized的实现原理
Synchronized在JVM里的实现都是基于进入和退出Monitor对象来实现方法同步和代码块同步，虽然具体实现细节不一样，但是都可以通过成对的MonitorEnter和MonitorExit指令来实现。

对同步块，MonitorEnter指令插入在同步代码块的开始位置，当代码执行到该指令时，将会尝试获取该对象Monitor的所有权，即尝试获得该对象的锁，而monitorExit指令则插入在方法结束处和异常处，JVM保证每个MonitorEnter必须有对应的MonitorExit。

对同步方法，从同步方法反编译的结果来看，方法的同步并没有通过指令monitorenter和monitorexit来实现，相对于普通方法，其常量池中多了ACC_SYNCHRONIZED标示符。

JVM就是根据该标示符来实现方法的同步的：当方法被调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。

synchronized使用的锁是存放在Java对象头里面。
| 长度     | 内容                   | 说明                            |
| -------- | ---------------------- | ------------------------------- |
| 32/64bit | Mark Word              | 存储对象的hashCode或锁信息等    |
| 32/64bit | Class Metadata Address | 存储到对象类型数据的指针        |
| 32/32bit | Array length           | 数组的长度 (如果当前对象是数组) |
具体位置是对象头里面的MarkWord，MarkWord里默认数据是存储对象的HashCode等信息，
| 锁状态   | 25bit          | 4bit         | 1bit: 是否是偏向锁 | 2bit锁标志位 |
| -------- | -------------- | ------------ | ------------------ | ------------ |
| 无锁状态 | 对象的Hashcode | 对象分代年岭 | 0                  | 01           |

但是会随着对象的运行改变而发生变化，不同的锁状态对应着不同的记录存储方式。
<table>
    <tr>
        <th rowspan="2">锁状态</th>
        <th colspan="2">25bit</th>
        <th rowspan="2">4bit</th>
        <th >1bit</th>
        <th>2bit</th>
    </tr>
    <tr>
        <td>23bit</td><td>2bit</td><td>是否偏向锁</td><td>锁标志位</td>
    </tr>
    <tr>
        <td>轻量级锁</td><td colspan="4">指向栈中锁记录的指针</td><td>00</td>
    </tr>
    <tr>
        <td>重量级锁</td><td colspan="4">指向互斥量（重量级锁）指针</td><td>10</td>
    </tr>
    <tr>
        <td>GC标记</td><td colspan="4">空</td><td>11</td>
    </tr>
    <tr>
        <td>偏向锁</td><td>线程ID</td><td>Epoch</td><td>对象分代年龄</td><td>1</td><td>01</td>
    </tr>

</table>

#### 了解各种锁
##### 自旋锁
**原理**
自旋锁原理非常简单，如果持有锁的线程能在很短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞挂起状态，它们只需要等一等（自旋），等持有锁的线程释放锁后即可立即获取锁，这样就避免用户线程和内核的切换的消耗。但是线程自旋是需要消耗CPU的，说白了就是让CPU在做无用功，线程不能一直占用CPU自旋做无用功，所以需要设定一个自旋等待的最大时间。

如果持有锁的线程执行的时间超过自旋等待的最大时间扔没有释放锁，就会导致其它争用锁的线程在最大等待时间内还是获取不到锁，这时争用线程会停止自旋进入阻塞状态。

**自旋锁的优缺点**
自旋锁尽可能的减少线程的阻塞，这对于锁的竞争不激烈，且占用锁时间非常短的代码块来说性能能大幅度的提升，因为自旋的消耗会小于线程阻塞挂起操作的消耗！

但是如果锁的竞争激烈，或者持有锁的线程需要长时间占用锁执行同步块，这时候就不适合使用自旋锁了，因为自旋锁在获取锁前一直都是占用cpu做无用功，占着XX不XX，线程自旋的消耗大于线程阻塞挂起操作的消耗，其它需要cup的线程又不能获取到cpu，造成cpu的浪费。

**自旋锁时间阈值**
自旋锁的目的是为了占着CPU的资源不释放，等到获取到锁立即进行处理。但是如何去选择自旋的执行时间呢？如果自旋执行时间太长，会有大量的线程处于自旋状态占用CPU资源，进而会影响整体系统的性能。因此自旋次数很重要。

JVM对于自旋次数的选择，jdk1.5默认为10次，在1.6引入了适应性自旋锁，适应性自旋锁意味着自旋的时间不在是固定的了，而是由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定，基本认为一个线程上下文切换的时间是最佳的一个时间。

JDK1.6中-XX:+UseSpinning开启自旋锁； JDK1.7后，去掉此参数，由jvm控制；

##### 锁的状态
一共有四种状态，无锁状态，偏向锁状态，轻量级锁状态和重量级锁状态，它会随着竞争情况逐渐升级。锁可以升级但不能降级，目的是为了提高获得锁和释放锁的效率。

##### 偏向锁
引入背景：大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁，减少不必要的CAS操作。

偏向锁，顾名思义，它会偏向于第一个访问锁的线程，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要触发同步的，减少加锁／解锁的一些CAS操作（比如等待队列的一些CAS操作），这种情况下，就会给线程加一个偏向锁。 如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁。它通过消除资源无竞争情况下的同步原语，进一步提高了程序的运行性能。

偏向锁获取过程：
- 步骤1、 访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01，确认为可偏向状态。
- 步骤2、 如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤5，否则进入步骤3。
- 步骤3、 如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行5；如果竞争失败，执行4。
- 步骤4、 如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。（撤销偏向锁的时候会导致stop the word）
- 步骤5、 执行同步代码。

**偏向锁的释放：**
偏向锁的撤销在上述第四步骤中有提到。偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放偏向锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

**偏向锁的适用场景**
始终只有一个线程在执行同步块，在它没有执行完释放锁之前，没有其它线程去执行同步块，在锁无竞争的情况下使用，一旦有了竞争就升级为轻量级锁，升级为轻量级锁的时候需要撤销偏向锁，撤销偏向锁的时候会导致stop the word操作；

那么什么是stop the word操作呢？比如我们有多个用户线程，有一个GC线程，如果一边发生GC，一边产生新的对象，那么肯定很让人恼火吧。这时候，操作系统就要求所有的用户线程先停下来，等到GC线程完成工作了，再恢复，这种就是所谓的stw操作。

在有锁的竞争时，偏向锁会多做很多额外操作，尤其是撤销偏向锁的时候会导致进入安全点，安全点会导致stw，导致性能下降，这种情况下应当禁用。

jvm开启/关闭偏向锁
开启偏向锁：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
关闭偏向锁：-XX:-UseBiasedLocking

##### 轻量级锁
轻量级锁是通过CAS操作来加锁和解锁，是由偏向锁升级来的，偏向锁运行在一个线程进入同步块的情况下，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁；

**轻量级锁的加锁过程：**
在代码进入同步块的时候，如果同步对象锁状态为无锁状态且不允许进行偏向（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。

拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤4，否则执行步骤5。
如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态。

如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，当竞争线程尝试占用轻量级锁失败多次之后，轻量级锁就会膨胀为重量级锁，重量级线程指针指向竞争线程，竞争线程也会阻塞，等待轻量级线程释放锁后唤醒他。锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。
| 锁      | 优点  | 缺点    | 适用场景   |
| -------- | ------------ | --------- | ---------- |
| 偏向锁   | 加锁和解锁不需要额外的消耗。和执行非同步方法比仅存在纳秒级的差距。 | 如果线程间存在锁竞争,会带来额外的锁撤销的消耗 | 适用于只有一一个线程访问同步块场景。      |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度。  | 如果始终得不到锁竞争的线程使用自旋会消耗CPU.  | 追求响应时间，     同步块执行速度非常快。 |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU. | 线程阻塞,响应时间缓慢| 追求吞吐量，同步块执行速度较长。  |
