## 并发基础知识补全
#### Callable、Future和FutureTask
在前文（线程基础、线程之间的共享与协作）中提到过中，新启线程的方式只有两种，一种就是扩展自``Thread``类，然后重写``run()``方法，另一种就是实现``Runnable``接口，实现``run()``方法。

那么``Callable``接口这种方式，又是怎么回事呢。我们先来观察Thread类中的构造方法，并没有可以接受一个callable这种参数的构造方法。我们使用``Callable``的时候，首先要把它包装成``FutureTask``，而它又实现了``RunnableFuture``接口。

```java
public class FutureTask<V> implements RunnableFuture<V> {

    public FutureTask(Callable<V> callable) {
          if (callable == null)
              throw new NullPointerException();
          this.callable = callable;
          this.state = NEW;  // ensure visibility of callable
      }

    public void run() {
      Callable<V> c = callable;
      if (c != null && state == NEW) {
          V result;
          boolean ran;
          try {
              result = c.call();
          }catch (Throwable ex) {}
        }
    }
}
```
而``RunnableFuture``接口实际上又是继承了``Runnable``和``Future``接口。也就是说到底，``Callable``交给线程去执行的时候，实际上还是包装成了``Runnable``交给线程去执行。
```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
#### 阻塞方法sleep()和wait()的区别
之前提到线程生命周期的时候，无论是``sleep``还是``wait()``,都会进入阻塞状态，但是如果细分的话，虽然两者都会暂停线程的执行，实际上两者进入的线程状态是不一样的。
**我们先来看sleep()的源码:**
```java
/**
     * Causes the currently executing thread to sleep (temporarily cease
     * execution) for the specified number of milliseconds, subject to
     * the precision and accuracy of system timers and schedulers. The thread
     * does not lose ownership of any monitors.
     */
    public static native void sleep(long millis) throws InterruptedException;
```
sleep()方法是Thread类中的一个native方法，它会在指定的时间内阻塞线程的执行。而且从其注释中可知，并不会失去对任何监视器(monitors)的所有权，也就是说不会释放锁，仅仅会让出cpu的执行权。

**我们再来看wait()的源码:**
无论是``wait()``，还是``wait(long timeout, int nanos)``走的都是native的``wait()``方法。
```java
/**
* This method should only be called by a thread that is the owner
* of this object's monitor. See the {@code notify} method for a
* description of the ways in which a thread can become the owner of
* a monitor.
*/
public final native void wait(long timeout) throws InterruptedException;
```
根据注释可以看出，此方法调用的前提是当前线程已经获取了对象监视器monitor的所有权。

该方法会调用后不仅会让出cpu的执行权，还会释放锁(即monitor的所有权)，并且进入wait set中，直到其他线程调用``notify()``或者``notifyall()``方法，或者指定的timeout到了，才会从wait set中出来，并重新竞争锁。

**区别**
两者最主要的区别就是释放锁(monitor的所有权)与否，但是两个方法都会抛出InterruptedException。

#### 线程阻塞BLOCKED和等待WAITING的区别
**阻塞BLOCKED：**
阻塞表示线程在等待对象的monitor锁，试图通过**synchronized**去获取某个锁，但是此时其他线程已经独占了monitor锁，那么当前线程就会进入等待状态。

**等待WAITING**
当前线程等待其他线程执行某些操作，典型场景就是生产者消费者模式，在任务条件不满足时，等待其他线程的操作从而使得条件满足。可以通过``wait()``方法或者``Thread.join()``方法都会使线程进入等待状态。

实际上不用可以区分两者, 因为两者都会暂停线程的执行。两者的区别是: 进入WAITING状态是线程主动的, 而进入BLOCKED状态是被动的。更进一步的说, 进入BLOCKED状态是在同步(synchronized代码之外), 而进入WAITING状态是在同步代码之内。

例如：
```java
synchronized(obj){
  obj.wait()
}
```
在这个同步代码块中，我们通过synchronize关键字去获取obj对象的同步锁，如果没有获取到，这时候被动就会进入BLOCKED状态。直到获取到了锁，从阻塞状态进入就绪/运行状态，然后调用``obj.wait()``，主动进入WAITING状态进如状态，直到其他线程在同步代码块中调用了``obj.notify()/obj.notifyAll()``，又会从WAITING状态进入进入就绪/运行状态。


#### 死锁
死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力的作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁。

来看这个例子：
小明和小王都去买饺子皮和馅料，但是它们都各自只有一份，这时候小明抢到了饺子皮，小王抢到了馅料，但他们都各自不肯松手，但是只有一份原料，又都包不成饺子。所以两个人都只能僵持着，这就是所谓的死锁。

所以，死锁有这些特点：
- 多个操作者(M>=2)，争夺多个资源(N>=2)，且N<=M
- 争夺资源的顺序不对
- 拿到资源后不放手

**用凝练一点的语言来描述，那就是：**
1.  互斥：一个时间同一个资源只能由一个进程持有
2.  持有并等待：进程保持至少一个资源，并等待其他进程持有的额外资源
3.  不剥夺：进程持有资源之后不会被其他进程剥夺
4.  循环等待：进程互相等待各自的资源

我们用一段代码来进行死锁的演示：
```java
/**
 * 类说明：演示死锁的产生
 */
public class DeadLock {

    private static Object lock1 = new Object();//第一个锁
    private static Object lock2 = new Object();//第二个锁

    //第一个拿锁的方法
    private static void firstDo() throws InterruptedException {
        String threadName = Thread.currentThread().getName();
        synchronized (lock1) {
            System.out.println(threadName + " get lock-1");
            Thread.sleep(100);
            synchronized (lock2) {
                System.out.println(threadName + " get lock-2");
            }
        }
    }

    //第二个拿锁的方法
    private static void secondDo() throws InterruptedException {
        String threadName = Thread.currentThread().getName();
        synchronized (lock2) {
            System.out.println(threadName + " get lock-2");
            Thread.sleep(100);
            synchronized (lock1) {
                System.out.println(threadName + " get lock-1");
            }
        }
    }

    private static class TestThread extends Thread {

        private String name;

        public TestThread(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            Thread.currentThread().setName(name);
            try {
                firstDo();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread.currentThread().setName("Thread A");
        TestThread testThread = new TestThread("Thread 2");
        testThread.start();
        secondDo();
    }
}
```

在实际开发过程中，我们应避免死锁的产生，因为死锁会进入一个等待的状态，并不会抛出异常，不利于我们查找错误。那么怎么来避免死锁呢？

由于多个操作者来争抢多个资源，这是由业务逻辑来决定的，所以我们一般从后者入手，比如以正确的顺序去争夺资源，或者拿到资源后允许放手，都可以解决死锁。

**ReentrantLock**中的``tryLock()``就可以用于死锁的解决。它可以对显式锁尝试获取，并返回boolean值。代码段可以这么写
```java
  if (lock.tryLock()) {
      try {
          //TODO
      } finally {
          lock.unlock();
      }
  } else {
      // perform alternative actions
  }
```
#### 活锁
在上文中，我们使用``tryLock()``来解决死锁，但是不正当使用的，有可能会造成活锁的产生。来看这么一段代码：
```java

/**
 *类说明：演示尝试拿锁解决死锁
 */
public class TryLock {
    private static Lock lock1 = new ReentrantLock();//第一个锁
    private static Lock lock2 = new ReentrantLock();//第二个锁

    //先尝试拿lock1 锁，再尝试拿lock2锁，lock1锁没拿到，连同lock2锁一起释放掉
    private static void firstToSecond() throws InterruptedException {
        String threadName = Thread.currentThread().getName();
        Random r = new Random();
        while(true){
            if(lock1.tryLock()){
                System.out.println(threadName +" get lock1");
                try{
                    if(lock2.tryLock()){
                        try{
                            System.out.println(threadName +" get lock2");
                            System.out.println("firstToSecond do work------------");
                            break;
                        }finally{
                            System.out.println(threadName +" release lock2");
                            lock2.unlock();
                        }
                    }
                }finally {
                    System.out.println(threadName +" release lock1");
                    lock1.unlock();
                }

            }
            //Thread.sleep(r.nextInt(3));
        }
    }

    //先尝试拿lock2 锁，再尝试拿lock1锁，lock2锁没拿到，连同lock1锁一起释放掉
    private static void SecondToFirst() throws InterruptedException {
        String threadName = Thread.currentThread().getName();
        Random r = new Random();
        while(true){
            if(lock2.tryLock()){
                System.out.println(threadName +" get lock2");
                try{
                    if(lock1.tryLock()){
                        try{
                            System.out.println(threadName +" get lock1");
                            System.out.println("SecondToFirst do work------------");
                            break;
                        }finally{
                            System.out.println(threadName +" release lock1");
                            lock1.unlock();
                        }
                    }
                }finally {
                    System.out.println(threadName +" release lock2");
                    lock2.unlock();
                }

            }
            //Thread.sleep(r.nextInt(3));
        }
    }

    private static class TestThread extends Thread{

        private String name;

        public TestThread(String name) {
            this.name = name;
        }

        public void run(){
            Thread.currentThread().setName(name);
            try {
                SecondToFirst();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        Thread.currentThread().setName("Thread A");
        TestThread testThread = new TestThread("Thread B");
        testThread.start();
        try {
            firstToSecond();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
在我们没有加上``Thread.sleep(r.nextInt(3));``之前，线程之间的相互等待的过程可能会被急剧拉长。为什么呢？比如马路中间有条小桥，只能容纳一辆车经过，桥两
头开来两辆车A和B，A比较礼貌，示意B先过，B也比较礼貌，示意A先过，结果两人一直谦让谁也过不去。

在上面的代码中，有线程A、B，分别去获取锁1和2
1.   A先竞争到1，然后尝试去竞争2
2.   B先竞争到2，然后尝试去竞争1
3.   A释放锁1，B释放锁2
4.   A再竞争到1，然后尝试去竞争2
5.   B再竞争到2，然后尝试去竞争1

这样一来，A和B线程就一直在相互竞争中循环等待着，但是跟死活不一样，进入了死锁的线程，是不进行工作的，而进入活锁的线程，是在忙碌的工作着。而我们加上休眠，可以让两个线程在竞争锁的时候，时间错开一点，避免了活锁。

## CAS基本操作（Compare And Swap）
#### 原子操作
我们都知道，在没有发现电子、原子核之前，科学界所认识到物质的最小单位就是原子，原子就是不可再分的。那么反映并发编程中，所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，不会发生上下文的切换。

#### 如何实现原子操作
被``synchronized``包围的代码块，其实就是一个原子操作。但是，使用``synchronized``是一个很消耗性能的操作，因为会涉及到线程的状态变化，没有抢到内置锁的线程，会进入等待队列里面并等待。但假如我们的同步代码块里面只有简单的``i++``，那么使用``synchronized``是不是就太大题小做了，有没有一种更轻量级的同步机制呢？

为了解决这个问题，在现代CPU里面，提供了一种Compare And Swap指令，简称CAS

#### CAS指令原理
Compare And Swap，从名字来看，就是包含比较并交换两个步骤，但是在CPU提供的CAS指令已经包含这两个操作了，由CPU去保证这两个步骤是原子的。意思就是说，比较和交换两个动作，要么全都完成，要么就全都不执行。

那么CAS指令是如何保证线程的同步的呢，我们就拿简单的``i++``来看。假如使用``synchronized``，那就是谁抢到了锁，谁就去执行``i++``。那么在CAS指令中，是怎么操作的呢。

假如i初始值为0，有A~D四个线程，都要执行``i++``这个操作。首先，四个线程都从内存里面取出i=0，然后在自己的方法栈上，进行i++，i变为1，这时候要重新写回内存的时候，同一时间只允许一个线程进行操作。假设A线程拿到了这个权限，这时候它会再次从内存中取出i的值，如果i==0，则把计算得到的值写回去，这个线程就执行完了，轮到其他线程来执行。这时候B线程从内存中取出i，悲催地发现i已经等于1了，说明i的值已经被人读写过了，那么就应该重新执行``i++``。其他线程也一样。

所以说，CAS其实就是不断重复这个指令（自旋），直到成功为止。

在这里涉及到了悲观锁跟乐观锁的概念。在使用``synchronized``同步关键字的时候，线程会**悲观**的认为，总有其他线程想来害它自己，不如先用锁把代码块锁起来，``i++``这个过程只有自己能做，直到我自己做完了，才让别人去接着干这个事情。而对于CAS来说，线程会**乐观**的认为，没人会来改自己的东西，我先把值取出来，先改了再说。但它也不傻，还是会去检查结果，发现已经被改过，那也没办法，只能再来一遍。

#### CAS指令问题
既然CAS指令在执行效率要高于``synchronized``，那么是不是可以替代``synchronized``呢？

答案是不可以，因为CAS存在以下问题：
1. ABA问题
2. 开销问题
3. 只能保证一个共享变量的原子操作

**我们来看以下思考：**

假设有线程1和线程2，并且有一个变量A，对于1来说，它要把A改为B，假设线程2跑的更快，它先把A改为C，再改回A。那么对于线程1来说，在执行CAS指令的时候，发现A的值没有变化吧，并没有修改过，然后放心的将它改写为B。

但实际上，这个A已经不是原来的A了，已经被线程2修改过。但是CAS操作并没有办法去发现。拿一个实际的例子来说，你的水杯里面装满了你最喜欢的可乐，这时候你有事去打了一个电话，你的女朋友口渴了，突然喝了一口，然后她怕被你发现，又将水杯给重新倒满了。你打完电话回来，一看水杯是满的，并没有人喝过，然后继续高兴的吃你的炸鸡。这个就是ABA问题。

那么要怎么解决ABA问题呢，我们可以加上一个版本戳，每次修改，都会对更新当前变量的版本。

而开销问题就是说，因为CAS指令是基于自旋来实现的，但是线程如果长时间不能成功执行，会给CPU带来非常大的执行开销。

CAS指令是通过比较内存中某个变量的值，来决定是否能进行交换操作，对于计算机来说，一个地址只能存放一个变量。所以CAS指令一次只能保证一个共享变量的原子操作。假如有ABC三个变量，需要保证它们的读写是一个原子操作，那么CAS就不能实现了，而使用``synchronized``就能很好的执行。从Java 1.5开始，JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

#### Java中的原子操作类
Jdk中为我们提供了如下相关原子操作类：
- 更新基本类型类：AtomicBoolean，AtomicInteger，AtomicLong
- 更新数组类：AtomicIngerArray，AtomicLongArray，AtomicReferenceArray
- 更新引用类型：AtomicReference，AtomicMarkableReference，AtomicStampedReference
