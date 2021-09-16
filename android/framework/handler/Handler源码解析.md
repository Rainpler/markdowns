# Handler源码解析
## 前言
在操作系统中，线程与线程间离不开的就是数据通信，而`Handler`在Android中扮演着不可或缺的角色。我们都知道，在Android重为了保障线程安全，规定只能由主线程进行UI更新操作。作为线程之间的通讯媒介，子线程可以通过Handler通知主线程，按顺序一个个地完成更新UI的操作，从而保证了线程的安全。

那么Handler是如何实现线程间的通讯，又是如何保证了线程的安全的呢？下面我们就带着这些问题来对Handler进行学习。

## Handler的基本原理

### Handler源码解析

通过Handler进行线程通信，总是离不开主线程，那么主线程是怎么启动起来？我们先从一个APP的启动开始。

我们都知道，在Android中，zygote是整个系统创建新进程的核心进程。zygote进程在内部会先启动Dalvik虚拟机，继而加载一些必要的系统资源和系统类，最后进入一种监听状态,分裂出system_server并进行不断地ipc轮询。

在之后的运行中，当其他系统模块（比如AMS）希望创建新进程时，只需向zygote进程发出请求，zygote进程监听到该请求后，会相应地fork出新的进程，于是这个新进程就先天具有了自己的Dalvik虚拟机以及系统资源。

当你在桌面点击一个app图标时，并且这个app在内存中是无实例的。AMS会通知system_server，由system_server通知Zygote去fork出子进程并执行ActivityThread的`main()`方法，main方法的调用是在子进程的主线程中。

```java
public static void main(String[] args) {
    //开始追踪事件
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

    // 添加系统调用拦截
    AndroidOs.install();

    // CloseGuard 是一种资源清理机制，资源应该被显式关闭清理
    // setEnabled()明显就是使其失效，不过会在debug builds的时候重现变得有效。
    CloseGuard.setEnabled(false);

    //首先拿到当前进程的Id，然后初始化一个UserEnvironment实例。
    Environment.initForCurrentUser();

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    // Call per-process mainline module initialization.
    initializeMainlineModules();

    Process.setArgV0("<pre-initialized>");

    //1. 初始化主线程 Looper
    Looper.prepareMainLooper();

    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    ActivityThread thread = new ActivityThread();
    // 将AMS与ApplicationThread关联
    thread.attach(false, startSeq);

    //获取主线程的 Handler
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // 结束追踪事件
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    //2. 开启消息队列
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
在`ActivityThread.main()`方法中，最重要的两点就是**初始化了主线程**和将**AMS与ApplicationThread关联**。我们来看代码，注释1.处，调用了`Looper.prepareMainLooper()`，我们继续来看该方法。
```java
@Deprecated
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```
Looper类是用来为一个线程开启一个消息循环的。在`prepareMainLooper()`方法中，进一步调用了`prepare()`方法。
```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
在该方法中，有一个sThreadLocal的对象实例，我们都知道ThreadLocal通过每个线程持有自己的变量副本，从而实现了线程间的数据隔离。首先通过判断`sThreadLocal.get()`是否为空，如果不为空说明Looper已经被创建过了。否则，通过`ThreadLocal.set()`方法存入一个新的Loope实例。这样一来，主线程就拥有了自己唯一的Looper对象。

回到prepareMainLooper()方法，接下来在同步代码块中，先用synchronized锁住当前Looper实例，若sMainLooper不为空，则通过myLooper()方法，并调用ThreadLocal.get()将刚才创建的Looper实例赋值给它。

我们再来看Looper.loop()又干了些什么，这里的代码比较长，这里将代码进行了精简。
```java
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;

    // 死循环遍历
    for (;;) {
        // 从消息队列中获取一个消息
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        try {
            msg.target.dispatchMessage(msg);
        } catch (Exception exception) {
            throw exception;
        } finally {

        }

        // 重置msg
        msg.recycleUnchecked();
    }
}
```
在这个方法中，首先拿到了当前线程持有的Looper对象me，然后拿到消息队列queue，并通过一个死循环来遍历，从`queue.next()`获取一个消息，然后通过`msg.target.dispatchMessage()`将msg分发出去，最后通过`Message.recycleUnchecked()`重置消息的状态。通过查看Message的源码可以知道，target对象是一个Handler实例。那么什么时候才会跳出该死循环呢？只有当msg为null的时候。
###

##

源码  epoll
设计思路
设计模式
异步消息和同步消息
消息屏障

HandlerThread
IntentServer


源于生活高于生活
Choregrapher  编舞者


AMS
ActivityThread  围绕着Handler




handler工作流程

Hander.sendMessage
MessageQueue
enqueueMessage 队列入队操作

next  取消息
Looper.loop()

Handler.dispatchMessage
Handler.handlerMessage

老话常谈，进程是资源分配的最小单位，线程之间共享内存。而在整个消息传递的过程中，都是以Message对象为载体的，而Message的获取要么通过new Message或者是obtain()方法获取，从子线程到主线程的过程中，对象的存储地址并没有发生变化，因此Handler的通信机制实际上是依赖于内存共享的。


消息队列
单链表实现的优先级队列
先后顺序 排序  时间排序
先出


Looper源码
核心：构造函数、loop()、ThreadLocal


prepareMainLooper()
prepare()
ThreadLocal线程隔离

唯一的messageQueue  mQueue


面试题：
一个线程有几个Handler
一个线程有几个Looper

Handler内存泄漏原因，为什么其他的内部类没有说到这个

为何主线程可以new Handler()>?如果想要在子线程中创建要做哪些准备
子线程中维护的Looper，消息队列无消息的时候的处理方案是什么？有什么用

Looper.quit() 赋值  mQuiting  remove
既然可以存在多个 Handler 往 MessageQueue 中添加数据（发消息时各个 Handler 可能处于不同线程），那它内部是如何确保线程安全的？

我们使用 Message 时应该如何创建它？

Looper死循环为什么不会导致应用卡死


内部类持有外部类的对象


nativePollOnce/NativeWake
