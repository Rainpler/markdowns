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

回到`prepareMainLooper()`方法，接下来在同步代码块中，先用synchronized锁住当前Looper实例，若sMainLooper不为空，则通过`myLooper()`方法，并调用`ThreadLocal.get()`将刚才创建的Looper实例赋值给它。

我们再来看`Looper.loop()`又干了些什么，这里的代码比较长，这里将代码进行了精简。
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
在这个方法中，首先拿到了当前线程持有的Looper对象me，然后拿到消息队列queue，并通过一个死循环来遍历，从`queue.next()`获取一个消息，然后通过`msg.target.dispatchMessage()`将msg分发出去，最后通过`Message.recycleUnchecked()`重置消息的状态。通过查看Message的源码可以知道，target对象是一个Handler实例。那么什么时候才会跳出该死循环呢？只有当msg为null的时候，那msg什么情况下才会为null呢，这个我们稍后再说。


到这里Handler在`ActivityThread.main()`中的流程就分析完了。总的来说就是以下两点
-  Looper.prepareMainLooper()  初始化主线程 Looper
-  Looper.loop()   开启消息队列


我们在使用Handler传递消息的时候，通常是使用其`Handler.sendMessage()`的相关方法，而调用`Handler.sendMessage()`中间的任何方法，最后都会走到`enqueueMessage()`方法中。
```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
在这里会Handler自己保存到msg的target中，最后再调用`MessageQueue.enqueueMessage()`将事件存入消息队列中。MessageMessageQueue实际上是一个单链表实现的优先级，遵循先进先出的原则。我们来看该方法：
```java
 boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        // 如果mQuitting，说明队列为空
        if (mQuitting) {

            msg.recycle();
            return false;
        }
        // 将时间记录到when中
        msg.when = when;
        Message p = mMessages;
        boolean needWake;

        // 如果队列为空 || 延迟为0 || 或着延迟小于队列首个元素
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {

            // 如果事件需要延时，则按when排序，插入到消息队列中
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        // 唤醒线程
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
mQuitting是MessageQueue中的一个标记变量，用于标记是否退出消息队列。如果为true，则会回收Message对象。否则，将消息传递时间保存到msg的when中，并将局部变量p记录消息队列的第一个元素。

如果首个元素为空，或者当前事件的延迟为0，又或者延迟小于首元素的延迟，则说明该元素要被立即执行，这时候就将next指向首元素这时候，并将该事件插入到消息队列的头，并将mBlocked赋值给needWake，用于后续唤醒线程（如果线程是阻塞的）。

如果该事件需要延迟处理，则将消息插入到消息队列中，通过比较when的值，越靠后则插入到后面去，这样就得到以时间排序的有序单链表。

在Looper.loop()方法中，可以看到如果要从消息队列中取出消息，就得看`MessageQueue.next()`方法了。
```java
Message next() {

    // mPtr是MessageQueue的一个long型成员变量，关联的是一个在C++层的MessageQueue，阻塞操作就是通过底层的这个MessageQueue来操作的；当队列被放弃的时候其变为0。
    final long ptr = mPtr;

    if (ptr == 0) {
        return null;
    }

    //
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    // 记录下一条消息唤醒的时间
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 阻塞方法，主要是通过native层的epoll监听文件描述符的写入事件来实现的。
        // 如果nextPollTimeoutMillis = -1，一直阻塞不会超时。
        // 如果nextPollTimeoutMillis = 0，不会阻塞，立即返回。
        // 如果nextPollTimeoutMillis > 0，最长阻塞nextPollTimeoutMillis毫秒(超时)，如果期间有程序唤醒会立即返回。
        nativePollOnce(ptr, nextPollTimeoutMillis);

        // 从消息队列中 取出下一条消息
        synchronized (this) {

            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;

            // 找到消息队列首个元素
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                //msg.target == null表示此消息为消息屏障（通过postSyncBarrier方法发送来的）
                //如果发现了一个消息屏障，会循环找出第一个异步消息（如果有异步消息的话），所有同步消息都将忽略（平常发送的一般都是同步消息）
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            // 找到消息
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    // 如果当前消息还未到延时时间，则计算需要阻塞的时间，等待下一次唤醒
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 取出消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有找到消息，阻塞消息队列
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }


            // 下面的代码都是IdleHandler的处理

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```
从`next()`方法中可以看出，该方法主要就是要从消息队列中取出一个消息，并返回。但是当第一次进来或者消息队列为空的时候，则会将nextPollTimeoutMillis置为-1，并在下一次循环中通过`nativePollOnce()`方法阻塞线程，并等待下一次唤醒。如果发现了消息屏障，则跳过后面的同步消息。如果拿到的消息还未到设定的延迟时间，则计算剩余延迟时间，并阻塞线程。否则则取出消息并返回。

在next()方法中，还涉及到了一个IdleHandler的使用，现在针对它的工作原理进行分析。消息队列中取完消息之后，就会走到IdleHandler的代码段。首先进行判断，如果`pendingIdleHandlerCount<0`且消息队列为空，或者消息需要延迟处理，就会计算IdleHandler的数量，并赋值给pendingIdleHandlerCount。由于首次进入的时候，pendingIdleHandlerCount初始化为-1，所以前一个条件必定满足。因此，IdleHandler只会在消息队列闲暇的时候才会起效，这也是名字的由来。然后创建了一个大小至少为4的IdleHandler数组。

接下来，会遍历mPendingIdleHandlers，拿到其引用并赋值给idler，并释放数组对应元素对对象的引用。然后调用IdleHandler的`queueIdle()`函数并返回boolean值给keep，最后利用`queueIdle()`的返回值来判断是否需要移除 IdleHandler, 如果keep为false，消息队列会移除该IdleHandler, 如果为true时，则继续保留。

如果`next()`方法成功获取到一个消息，会继续调用`Handler.dispatchMessage()`将消息分发。
```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
从这里看，如过Message中如果有Runnable接口实现callback，则会调用`handleCallback()`最后调用`Runnable.run()`。否则，如果初始化Handler的时候，传递了Callback，则会调用Callback.handlerMessage。如果两者皆为空，则会调用自身的`handlerMessage()`方法，我们在继承自Handler自定义类的时候，必须实现该方法。

我们通过上面的源码可以发现，Looper.loop()是开启了一个死循环，想要退出死循环只有在msg==null的情况下。再来看`quit()`方法，通过该方法可以结束掉一个消息队列。
```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```
在该方法中，首先通过mQuitAllowed判断是否允许结束消息队列，可以看到，主线程是不允许quit的，因为主线程的调度会一直伴随着整个APP的生命周期，该变量会在Looper调用`prepare()`的时候就会传递。然后在同步代码块中，将mQuitting设置为true，并调用`removeAllFutureMessagesLocked()`或者`removeAllMessagesLocked()`将消息队列的中的消息链针指向为空，最后通过`nativeWake()`唤醒线程。

在调用`Message.next()`方法的时候，阻塞的时候会阻塞在nativePollOnce()的代码处，然后继续往下执行
```java
if (mQuitting) {
    dispose();
    return null;
}
```
就会调用dispose()释放线程，并返回null，这就会导致在是`Looper.loop()`方法中判断`if(msg==null)`的时候条件成立，这是跳出死循环的唯一条件，所以如果我们要结束一个消息队列，必须调用`MessageQueue.quit()`方法。

观察MessageQueue的设计思路，其实这就是一个典型的生产者-消费者模式，通过维护Queue这个缓冲池，一方面生产者通过`enqueueMessage()`方法向队列中插入消息，一方面消费者通过`next()`方法从队列中取出消息，同时当队列为空或者消息延迟的时候，对线程进行阻塞。

又因为进程是资源分配的最小单位，线程之间共享内存。而在整个消息传递的过程中，都是以Message对象为载体的，而Message的获取要么通过`new Message()`或者是`obtain()`方法获取，从子线程到主线程的过程中，对象的存储地址并没有发生变化，因此Handler的通信机制实际上是依赖于内存共享的。

### 异步消息与同步消息、屏障消息


我们在分析Handler的源码解析的时候，在MessageQueue.next()的方法中，就有涉及到屏障消息的内容，但是没有细讲。要想更全面的掌握Handler的机制，那就必须掌握屏障消息的知识。接下来将从消息屏障的插入、删除和使用场景等角度分析它的工作原理。

Message中有三种分类，普通消息（也就是同步消息）、屏障消息（同步屏障）和异步消息。我们最常使用的就是普通消息，而屏障消息就是在消息队列中插入一个屏障，在屏障之后的普通消息都会被挡住，不能处理。异步消息则可以突破屏障，只有撤销了屏障消息之后，后续的同步消息才可以正常被处理。

#### 屏障消息

同步屏障是通过MessageQueue的`postSyncBarrier()`方法插入到消息队列的。
```java
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

分析上述方法，它具有以下特征：

1. 没有给target赋值，即不用handler分发处理，后续也会根据target是否为空来判断消息是否为消息屏障
2. 消息队列中可以插入多个消息屏障
3. 消息屏障也是可以有时间戳的，插入的时候也是按时间排序的，它只会影响它后面的消息，前面的不受影响
4. 消息队列中插入消息屏障，是不会唤醒线程的(插入同步或异步消息会唤醒消息队列)
5. 插入消息屏障后，会返回一个token，是消息屏障的序列号，后续撤除消息屏障时，需要根据token查找对 应的消息屏障
6. 发送屏障消息的API被隐藏，需要通过反射调用postSyncBarrier方法

如果需要移除消息屏障，则要调用MessageQueue的`removeSyncBarrier()`方法
```java
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

从代码看，移除同步屏障需要根据对应的token，在消息队列中找到对应的屏障删除，如果屏障不是在消息队列的头部的话，则会将屏障移除且不会唤醒线程。否则，除非后续还是消息屏障，不然总会唤醒线程。

#### 异步消息

异步消息相比同步消息，只不过它被设置setAsynchronous 为true。有了这个标志位，消息机制会对它有些特别的处理，

Handler有几个构造方法，可以传入async标志为true，这样构造的Handler发送的消息就是异步消息。不过可以看到，这些构造函数都是hide的，正常我们是不能调用的，不过利用反射机制可以使用@hide方法
```java
/**
 * @hide
 */
public Handler(boolean async) {}

/**
 * @hide
 */
public Handler(Callback callback, boolean async) { }

/**
 * @hide
 */
public Handler(Looper looper, Callback callback, boolean async) {}

```
当调用handler.sendMessage(msg)发送消息，会执行如下代码：
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);//把消息设置为异步消息
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
可以看到如果这个handler的mAsynchronous为true就把消息设置为异步消息，设置异步消息其实也就是设置msg内部的一个标志。而这个mAsynchronous就是构造handler时传入的async。除此之外，还有一个公开的方法：`Message.setAsynchronous(boolean);`

## 面试题：
接下来是一些经典面试题：
- **一个线程有几个Handler**

一个线程可以有多个Handler，本质上Handler只是消息的发送和接收的对象，实际上对消息的处理是在Looper中。

- **一个线程有几个Looper**

一个线程只能有一个Looper，而这个Looper是用ThreadLocal来确保每一个线程都只持有唯一的一个Looper对象。

- **Handler内存泄漏的原因是什么**

当在Activity中使用handler时，直接创建匿名内部类，会得到一个警告，意思是可能出现内存泄漏，推荐使用静态内部类。这也是面试时经常被问的一个问题，现在，我们就来解读一下为什么会出现这个警告，以及如何改进。

我们知道，Handler在使用时，通过post或者send的方式，可以把消息发送到MessageQueue队列中，期间Looper循环取出消息去交给对应的handler所在的线程去处理，有些消息还加上了延时发送，这些原因就可能会导致一个问题：当Activity销毁了，而主线程中一直运行的Looper会持有handler的引用，而我们在创建handler的时候用的是非静态匿名内部类，所以此handler会持有Activity的引用，导致Activity不会被回收，出现了内存泄漏。因此，编译器才会给我们一个这样的警告。

```java
 private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
在Handler.enqueueMessage()方法中，会将自身引用赋值到target中，又由于匿名内部类持有外部类的引用，也就是该Handler会持有Activity的引用，如果期间消息没有得到及时处理，而相应的activity又被销毁了，就会导致Activity不能被回收。

如果Looper.loop()中将消息分发出去处理，最后通过`Message.recycleUnchecked()`清空消息，释放掉对Handler的引用。

为了避免出现内存泄漏的情况，我们可以让Handler持有Activity的弱引用，根据java的gc机制，弱引用不会影响系统对该对象的回收。
```java
class MyHandler extends Handler{
		WeakReference<Activity> mActivity;
		public MyHandler(Activity activity){
			this.mActivity = new WeakReference<Activity>(activity);
		}

		public void handleMessage(Message msg) {
			if(msg.what == 1 ){

			}

		};
	}
```

- **为何主线程可以`new Handler()`，如果想要在子线程中创建要做哪些准备?**

我们观察Handler的构造方法：
```java
@Deprecated
public Handler() {
    this(null, false);
}
```
无参构造方法是一个过时的方法，会继续调用有参的构造方法
```java
 public Handler(@Nullable Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
可以看到，首先会调用`Looper.myLooper()`获取Looper对象，如果是主线程的话，由于在`ActivityThread.main()`方法中的时候，已经调用过`Looper.prepare()`初始化Looper，因此判空不成立。

而子线程如果直接调用new Handler()，则会因为Looper没有初始化导致异常抛出。因此，如果要想在子线程中创建Handler，必须先调用`Looper.prepare()`,然后调用`Looper.loop()`开始循环消息队列。


- **既然可以存在多个 Handler 往 MessageQueue 中添加数据（发消息时各个 Handler 可能处于不同线程），那它内部是如何确保线程安全的？**

在`MessageQueue.enqueueMessage()`中，通过synchronized关键字，将添加数据的代码段放入同步代码块中，由于使用的MessageQueue对象是从Looper中拿到的，而Looper又是通过ThreadLocal保证了每一个线程只有唯一的对象，因此在并发访问的时候，就会通过阻塞来确保线程安全。

- **我们使用 Message 时应该如何创建它？**

我们使用Message时，可以通过`new Message()`创建一个新的实例，或者通过`Message.obtain()`从消息队列中回收一个消息进行复用。推荐使用后者来拿到一个 Message，因为不断的去创建新对象的话，可能会导致垃圾回收区域中新生代被占满，从而触发 GC。

Message中的sPool就是用来存放被回收的 Message，当我们调用 obtain 后，会先查看是否有可复用的对象，如果真的没有才会去创建一个新的 Message 对象。

补充：主要的Message回收时机是：
1. 在MQ中remove Message 后；
2. 单次 loop 结束后；
3. 我们主动调用 Message 的 recycle 方法后

- **Looper死循环为什么不会导致应用卡死**

我们知道主线程在`ActivityThread.main()`执行的时候就会初始化，并开启`Looper.loop()`循环。如果消息队列中消息为空，或者延时时间未到，线程就会进入阻塞状态，但是如果有新消息来了，loop()方法就会及时将消息分发到Handler中去处理。如果在主线程中执行耗时操作，即对同一条消息处理时间过长，那么后续来的同步消息就得不到及时处理，无法响应手势操作，不能及时刷新UI，这才是导致ANR的真正原因。
