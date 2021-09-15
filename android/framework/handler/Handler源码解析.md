1. 数据通信会带来开发中的什么问题

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

lanucher zygote 给每一个应用分配一个虚拟机 jvm  ActivityThread.main()



维持着android app 运行的框架


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
