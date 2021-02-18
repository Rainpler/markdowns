## AbstractQueuedSynchronizer
>什么叫做AQS？从名字可以看出，AQS就是抽象队列同步器，是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。并发包的大师（Doug Lea）期望它能够成为实现大部分同步需求的基础。
#### AQS使用方式和其中的设计模式
AQS的主要使用方式是继承，子类通过继承AQS并实现它的抽象方法来管理同步状态，在AQS里由一个int型的state来代表这个状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法``getState()``、``setState(int newState)``和``compareAndSetState(int expect,int update)``来进行操作，因为它们能够保证状态的改变是安全的。
```java
  /**
   * The synchronization state.
   */
  private volatile int state;
```
在实现上，子类推荐被定义为自定义同步组件的静态内部类，AQS自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。
             
#### 了解其中的方法
#### 实现ReentrantLock锁
