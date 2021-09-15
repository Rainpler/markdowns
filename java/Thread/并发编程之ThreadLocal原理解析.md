## ThreadLocal辨析
#### 与Synchonized的比较
ThreadLocal 和 Synchonized 都用于解决多线程并发访问。可是 ThreadLocal 与 synchronized 有本质的差别。synchronized 是利用锁的机制，使变量或代码块 在某一时该仅仅能被一个线程访问。而 ThreadLocal为每个线程都提供了**变量的副本**，使得每个线程在某一时间访问到的并非同一个对象，这样就**隔离**了多个线程对数据的数据共享。

#### ThreadLocal 的使用
ThreadLocal 类接口很简单，只有 4 个方法，我们先来了解一下：
- void set(Object value)
  设置当前线程的线程局部变量的值。
- public Object get()
  该方法返回当前线程所对应的线程局部变量。
- public void remove()
  将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是 JDK 5.0 新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动 被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它 可以加快内存回收的速度。
- protected Object initialValue()
  返回该线程局部变量的初始值，该方法是一个 protected 的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第 1 次调用 get() 或 set(Object)时才执行，并且仅执行 1 次。ThreadLocal 中的缺省实现直接返回一 个 null。
- public final static ThreadLocal<String> RESOURCE = new ThreadLocal<String>();
  RESOURCE代表一个能够存放String类型的ThreadLocal对象。 此时不论什么一个线程能够并发访问这个变量，对它进行写入、读取操作，都是线程安全的。

来看下面的例子：
```java
/**
 * 类说明：演示ThreadLocal的使用
 */
public class UseThreadLocal {

    //TODO
    private static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 1;
        }
    };
    /**
     * 运行3个线程
     */
    public void StartThreadArray() {
        Thread[] runs = new Thread[3];
        for (int i = 0; i < runs.length; i++) {
            runs[i] = new Thread(new TestThread(i));
        }
        for (int i = 0; i < runs.length; i++) {
            runs[i].start();
        }
    }

    /**
     * 类说明：测试线程，线程的工作是将ThreadLocal变量的值变化，并写回，看看线程之间是否会互相影响
     */
    public static class TestThread implements Runnable {
        int id;
        public TestThread(int id) {
            this.id = id;
        }

        public void run() {
            System.out.println(Thread.currentThread().getName() + ":start");
            Integer s = threadLocal.get();
            s = s + id;
            threadLocal.set(s);
            System.out.println(Thread.currentThread().getName() + ":" + threadLocal.get());
        }
    }

    public static void main(String[] args) {
        UseThreadLocal test = new UseThreadLocal();
        test.StartThreadArray();
    }
}
```
这里使用了一个``ThreadLocal<Integer>``静态变量，并初始化值为1，然后新启了三个线程去``get()``中``threadLocal``的值，跟id相加后，存放回``threadLocal``中。打印结果：
```Java
Thread-0:start
Thread-0:1
Thread-2:start
Thread-1:start
Thread-2:3
Thread-1:2
```
线程之间并不会互相影响，因为ThreadLocal为每个线程都提供了**变量的副本**，使得每个线程在某一时间访问到的并非同一个对象。那么，既然ThreadLocal在使用上那么方便，那么它又是如何实现的呢？
#### ThreadLocal的实现
既然要了解ThreadLocal的实现，那么就要从它的方法开始看，我们先来看``get()``方法：
```java
public T get() {
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null) {
           ThreadLocalMap.Entry e = map.getEntry(this);
           if (e != null) {
               @SuppressWarnings("unchecked")
               T result = (T)e.value;
               return result;
           }
       }
       return setInitialValue();
   }
```
进入``get()``方法，它首先拿到了当前调用方法的线程对象``t``，然后调用``getMap(t)``
```java
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
返回的是t里面的``threadLocals``对象，这是什么东西呢，我们再看看
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
``threadLocals``又是定义在ThreadLocal中的ThreadLocalMap对象，也就是说每个线程拥有一个这样的成员变量，我们再来看看ThreadLocalMap是什么东西。
```java
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }

            private Entry[] table;
            ......
        }
```
在ThreadLocalMap中又有一个Entry，就好像我们在使用HashMap的时候，由Key-Value构成。在这里key就是ThreadLocal，而value就是对象拥有的副本值。在ThreadLocalMap中还有一个Entry[]类型的变量``table``，为什么要定义那么一个数组类型的变量呢？

拿我们刚刚的例子来说，我们定义了一个``ThreadLocal<Integer>``类型的ThreadLocal，是不是还能再定义一个`ThreadLocal<Boolean>`类型的？那么对于同一个线程来说，他就会拥有了两个ThreadLocal副本，那么这个数组，就是用于存放线程所拥有的ThreadLocal对象的。

回到``get()``方法，拿到了线程所持有的ThreadLocalMap对象``map``后，又再继续通过``map.getEntry(this)``进一步拿到Entry对象``e``。如果线程尚未持有，则通过``setInitialValue()``方法为线程分配ThreadLocalMap。
#### ThreadLocal引发的内存泄漏分析
```java
/**
 * 类说明：ThreadLocal造成的内存泄漏演示
 */
public class ThreadLocalOOM {
    private static final int TASK_LOOP_SIZE = 500;

    final static ThreadPoolExecutor poolExecutor
            = new ThreadPoolExecutor(5, 5, 1,
            TimeUnit.MINUTES,
            new LinkedBlockingQueue<>());

    static class LocalVariable {
        private byte[] a = new byte[1024*1024*5];/*5M大小的数组*/
    }

    final static ThreadLocal<LocalVariable> localVariable
            = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < TASK_LOOP_SIZE; ++i) {
            poolExecutor.execute(new Runnable() {
                public void run() {
                  // localVariable.set(new LocalVariable()) 修改为
                    new LocalVariable();
                    System.out.println("use local varaible");
                    // localVariable.romove()
                }
            });

            Thread.sleep(100);
        }
        System.out.println("pool execute over");
    }
}
```
这里我们新建了一个线程池，执行500次线程任务，每次任务只是新建了一个5M的大小。我们通过Java VisualVM可以监视到程序运行时的内存变化情况，发现其峰值大概就是在25M左右。这是很合理的吧，因为线程池的最大线程数就是5个，数组大小是5，5x5=25M。

那么我们再来改一下代码，创建一个ThreadLocal变量，把``new LocalVariable()``，修改为``localVariable.set(new LocalVariable())``,看看又会发生什么。我们发现，程序的内存使用量一直在往上走，甚至达到了150M，达到了200M。难道ThreadLocal使用内存如此之大吗？我们再来变个神奇的魔术，加上``localVariable.romove()``再看看，内存使用情况就回复到了一开始的正常水平。从25~150M，其中发生了125M之多的内存泄漏，这又是为什么呢。

回到之前的源码，ThreadLocalMap在定义Entry类的时候，使用了一个**WeakReference**（所谓的弱引用）。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。
```java
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
          ......
        }
}
```

根据我们前面对ThreadLocal 的分析，我们可以知道每个Thread 维护一个ThreadLocalMap，这个映射表的key 是ThreadLocal 实例本身，value 是真正需要存储的Object，也就是说ThreadLocal 本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap 获取value。仔细观察ThreadLocalMap，这个map是使用ThreadLocal 的弱引用作为Key 的，弱引用的对象在GC 时会被回收这样，当把threadlocal 变量置为null 以后，没有任何强引用指向threadlocal实例，所以threadlocal 将会被gc 回收。

这样一来，ThreadLocalMap 中就会出现key 为null 的Entry，就没有办法访问这些key 为null 的Entry 的value，如果当前线程再迟迟不结束的话，这些key 为null 的Entry 的value 就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value，而这块value 永远不会被访问到了，所以存在着内存泄露。只有当前thread 结束以后，current thread 就不会存在栈中，强引用断开，Current Thread、Map value 将全部被GC 回收。最好的做法是不在需要使用ThreadLocal 变量后，都调用它的remove()方法，清除数据。

其实考察ThreadLocal 的实现，我们可以看见，无论是get()、set()在某些时候，调用了expungeStaleEntry 方法用来清除Entry 中Key 为null 的Value，但是这是不及时的，也不是每次都会执行的，所以一些情况下还是会发生内存泄露。只有remove()方法中显式调用了expungeStaleEntry 方法。
```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // 去掉对value的引用
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                //如果key为null,则去掉对value的引用。
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```
从表面上看内存泄漏的根源在于使用了弱引用，但是另一个问题也同样值得思考：为什么使用弱引用而不是强引用？下面我们分两种情况讨论：
- key 使用强引用：引用ThreadLocal 的对象被回收了，但是ThreadLocalMap还持有ThreadLocal 的强引用，如果没有手动删除，ThreadLocal 的对象实例不会被回收，导致Entry 内存泄漏。
- key 使用弱引用：引用的ThreadLocal 的对象被回收了，由于ThreadLocalMap持有ThreadLocal 的弱引用，即使没有手动删除，ThreadLocal 的对象实例也会被回收。value 在下一次ThreadLocalMap 调用set，get，remove 都有机会被回收。

比较两种情况，我们可以发：由于ThreadLocalMap 的生命周期跟Thread一样长，如果都没有手动删除对应key，都会导致内存泄漏，但是使用弱引用可以多一层保障。因此，ThreadLocal 内存泄漏的根源是：由于ThreadLocalMap 的生命周期跟Thread 一样长，如果没有手动删除对应key 就会导致内存泄漏，而不是因为弱引用。

**总结**
JVM 利用设置ThreadLocalMap 的Key 为弱引用，来避免内存泄露。JVM 利用调用remove、get、set 方法的时候，回收弱引用。当ThreadLocal 存储很多Key 为null 的Entry 的时候，而不再去调用remove、get、set 方法，那么将导致内存泄漏。使用线程池+ ThreadLocal 时要小心，因为这种情况下，线程是一直在不断的重复运行的，从而也就造成了value 可能造成累积的情况。

#### ThreadLocal的线程不安全
```java
/**
 * 类说明：ThreadLocal的线程不安全演示
 */
public class ThreadLocalUnsafe implements Runnable {

    public static Number number = new Number(0);

    public void run() {
        //每个线程计数加一
        number.setNum(number.getNum()+1);
      //将其存储到ThreadLocal中
        value.set(number);
        SleepTools.ms(2);
        //输出num值
        System.out.println(Thread.currentThread().getName()+"="+value.get().getNum());
    }

    public static ThreadLocal<Number> value = new ThreadLocal<Number>() {
    };

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new ThreadLocalUnsafe()).start();
        }
    }

    private static class Number {
        public Number(int num) {
            this.num = num;
        }

        private int num;

        public int getNum() {
            return num;
        }

        public void setNum(int num) {
            this.num = num;
        }

        @Override
        public String toString() {
            return "Number [num=" + num + "]";
        }
    }
}
```
为什么每个线程都输出5？难道他们没有独自保存自己的Number 副本吗？为什么其他线程还是能够修改这个值？仔细考察ThreadLocal和Thread的代码，因为``number``对象被定义为了static，所以ThreadLocalMap 中保存的其实是对象的一个引用，这样的话，当有其他线程对这个引用指向的对象实例做修改时，其实也同时影响了所有的线程持有的对象引用所指向的同一个对象实例。

这也就是为什么上面的程序为什么会输出一样的结果：5 个线程中保存的是同一Number 对象的引用，在线程睡眠的时候，其他线程将num 变量进行了修改，而修改的对象Number 的实例是同一份，因此它们最终输出的结果是相同的。而上面的程序要正常的工作，应该的用法是让每个线程中的ThreadLocal 都应该持有一个新的Number 对象
#### Spring中的ThreadLocal
Spring的事务就借助了ThreadLocal 类。Spring 会从数据库连接池中获得一个connection，然会把connection 放进ThreadLocal中，也就和线程绑定了，事务需 要提交或者回滚，只要从ThreadLocal 中拿到 connection进行操作。为何Spring的事务要借助ThreadLocal 类？以 JDBC为例，正常的事务代码可能如下：
- dbc = new DataBaseConnection();
- Connection con = dbc.getConnection();
- con.setAutoCommit(false);
- con.executeUpdate(...);
- con.executeUpdate(...);
- con.executeUpdate(...);
- con.commit();

上述代码，可以分成三个部分: 事务准备阶段：第1～3行业务处理阶段：第 4～6 行事务提交阶段：第7行可以很明显的看到，不管我们开启事务还是执行具体的sql都需要一个具体的数据库连接。

现在我们开发应用一般都采用三层结构，如果我们控制事务的代码都放在 DAO(DataAccessObject)对象中，在 DAO 对象的每个方法当中去打开事务和关闭事务，当 Service 对象在调用 DAO 时，如果只调用一个 DAO，那我们这样实现则 效果不错，但往往我们的 Service 会调用一系列的 DAO 对数据库进行 多次操作， 那么，这个时候我们就无法控制事务的边界了，因为实际应用当中，我们的 Service 调用的 DAO 的个数是不确定的，可根据需求而变化，而且还可能出现 Service调用 Service 的情况。

但是需要注意一个问题，如何让三个DAO使用同一个数据源连接呢？我们就必须为每个 DAO 传递同一个数据库连接，要么就是在 DAO 实例化的时候作为构造方法的参数传递，要么在每个DAO的实例方法中作为方法的参数传递。这两种方式无疑对我们的 Spring 框架或者开发人员来说都不合适。为了让这个数据库连接可以跨阶段传递，又不显示的进行参数传递，就必须使用别的办法。

Web 容器中，每个完整的请求周期会由一个线程来处理。因此，如果我们能将一些参数绑定到线程的话，就可以实现在软件架构中跨层次的参数共享（是隐式的共享）。而 JAVA 中恰好提供了绑定的方法——使用ThreadLocal。

结合使用 Spring 里的 IOC 和 AOP，就可以很好的解决这一点。只要将一个数据库连接放入 ThreadLocal 中，当前线程执行时只要有使用数据库连接的地方就从 ThreadLocal 获得就行了。

## 线程间的协作
线程之间相互配合，完成某项工作，比如：一个线程修改了一个对象的值，而另一个线程感知到了变化，然后进行相应的操作，整个过程开始于一个线程，而最终执行又是另一个线程。前者是生产者，后者就是消费者，这种模式隔离了“做什么”（what）和“怎么做”（How），简单的办法是让消费者线程不断地循环检查变量是否符合预期,在while 循环中设置不满足的条件，如果条件满足则退出while 循环，从而完成消费者的工作。但这存在如下问题：

- 难以确保及时性。
- 难以降低开销。如果降低睡眠的时间，比如休眠1 毫秒，这样消费者能更加迅速地发现条件变化，但是却可能消耗更多的处理器资源，造成了无端的浪费。
#### 等待/通知机制
在Java中，为我们提供了一种等待/通知机制，多个线程之间也可以实现通信。实现等待通知机制主要是用：``wait()/notify()``方法实现。

下面详细介绍一下这两个方法：

- wait()方法 ：wait()方法是使当前执行代码的线程进行等待，wait()方法是Object类的方法，该方法用来将当前线程置入“欲执行队列”中，并且在wait()所在的代码处停止执行，直到接到通知或被中断为止。在调用wait()方法之前，线程必须获得该对象的对象级别锁，即只能在同步方法或者同步块中调用wait()方法。在执行wait()方法后，当前线程释放锁。
- notify()方法：方法notify()也要在同步方法或同步块中调用，即在调用前，线程也必须获得该对象的对象级别锁。【但是如果使用了notify()，那么执行这个notify()的代码的线程会是什么状态？】

使用``wait()/notify()``的时候，我们必须遵循以下标准范式：

等待方遵循如下原则:
1. 获取对象的锁。
2. 如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件。
3. 条件满足则执行对应的逻辑。
```java
synchronized(obj){
  while(条件不满足){
    obj.wait(); //释放同步锁
  }
  逻辑处理
}
```
通知方遵循如下原则。
1. 获得对象的锁。
2. 改变条件。
3. 通知所有等待在对象上的线程。
```java
synchronized(obj){
  条件满足
  doSth.
  obj.notifyAll();
}
```

在调用``wait()``、``notify()``系列方法之前，线程必须要获得该对象的对象级别锁，即只能在同步方法或同步块中调用``wait()``方法、``notify()``系列方法，进入``wait()``方法后，当前线程释放锁，在从``wait()``返回前，线程与其他线程竞争重新获得锁，执行``notify()``系列方法的线程退出调用了``notifyAll()`` 的synchronized代码块的时候后，他们就会去竞争。如果其中一个线程获得了该对象锁，它就会继续往下执行，在它退出synchronized 代码块，释放锁后，其他的已经被唤醒的线程将会继续竞争获取该锁，一直进行下去，直到所有被唤醒的线程都执行完毕。

``notify()`` 和``notifyAll()`` 应该用谁?
尽可能用``notifyall()``，谨慎使用``notify()``，因为``notify()``只会唤醒一个线程，我们无法确保被唤醒的这个线程一定就是我们需要唤醒的线程。
#### 等待超时模式实现一个连接池
调用场景：调用一个方法时等待一段时间（一般来说是给定一个时间段），如果该方法能够在给定的时间段之内得到结果，那么将结果立刻返回，反之，超时返回默认结果。
```java
/**
 * 类说明：连接池的实现
 */
public class DBPool {

    /*容器 存放连接*/
    private static LinkedList<Connection> pool = new LinkedList<Connection>();

    public DBPool(int initialSize) {
        if (initialSize > 0) {
            for (int i = 0; i < initialSize; i++) {
                pool.addLast(SqlConnectImpl.fetchConnection());
            }
        }
    }

    /*释放连接，通知其他的等待连接的进程*/
    public void releaseConnection(Connection connection) {
        if (connection != null) {
            synchronized (pool) {
                pool.addLast(connection);
                pool.notifyAll();
            }
        }
    }

    /* 获取连接*/
    // 在mills内无法获取到连接，将会返回null
    public Connection fetchConnection(long mills) throws InterruptedException {
        synchronized (pool) {
            //永不超时
            if (mills < 0) {
                while (pool.isEmpty()) {
                    pool.wait();
                }
                return pool.removeFirst();
            } else {
                //超时时刻
                long future = System.currentTimeMillis() + mills;
                long remaining = mills;
                while (pool.isEmpty() && remaining > 0) {
                    pool.wait(remaining);
                    //唤醒一次，重新计算等待时间
                    remaining = future - System.currentTimeMillis();
                }
                Connection connection = null;
                if (!pool.isEmpty()) {
                    connection = pool.removeFirst();
                }
                return connection;
            }
        }
    }
}
```
#### 调用yield() 、sleep()、wait()、notify()等方法对锁有何影响？
``yield()`` 、``sleep()``被调用后，都不会释放当前线程所持有的锁。``调用wait()``方法后，会释放当前线程持有的锁，而且当前被唤醒后，会重新去竞争锁，锁竞争到后才会执行``wait()`` 方法后面的代码。调用``notify()``系列方法后，对锁无影响，线程只有在syn 同步代码执行完后才会自然而然的释放锁，``所以notify()``系列方法一般都是syn 同步代码的最后一行。
