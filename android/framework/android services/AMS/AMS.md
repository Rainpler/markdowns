## 前言

ActivityManagerService是Android系统中一个特别重要的系统服务，也是我们上层APP打交道最多的系统服务之一。ActivityManagerService（以下简称AMS） 主要负责四大组件的启动、切换、调度以及应用进程的管理和调度工作。所有的APP应用都需要与AMS打交道Activity Manager的组成主要分为以下几个部分：
1. 服务代理：由ActivityManagerProxy实现，用于与Server端提供的系统服务进行进程间通信
2. 服务中枢：ActivityManagerNative继承自Binder并实现IActivityManager，它提供了服务接口和Binder接口的相互转化功能，并在内部存储服务代理对像，并提供了getDefault方法返回服务代理
3. Client：由ActivityManager封装一部分服务接口供Client调用。ActivityManager内部通过调用
ActivityManagerNative的getDefault方法，可以得到一个ActivityManagerProxy对像的引用，进而通过
该代理对像调用远程服务的方法
4. Server：由ActivityManagerService实现，提供Server端的系统服务

接下来将从AMS的启动流程、几个方面来学习AMS。

## AMS的启动流程

首先，在分析AMS的启动流程之前，我们先来回顾一下Android系统的启动流程。当我们按下开机键的时候，引导芯片代码从预定义的地方（固化在ROM中）加载到RAM中，这个就是我们所说的BootLoader引导程序，引导程序会进一步完成内核的设置，包括缓存、驱动、被保护存储器等。当内核设置完成好，会在文件系统中寻找init.c文件，然后启动init进程，并在init.main()方法中完成初始化设置。

init进程是Linux系统的第一个进程，它完成了文件目录的创建和挂载、初始化和启动属性服务，解析init.rc配置文件并启动zygote进程这三个重要的事情。zygote就是安卓系统的核心进程，然后又会fork出SystemService这一重要进程，我们所知道的ActivityManagerService、PowerManagerService、WindowManagerService都是由SystemService这一进程fork出来的，现在就来看一下它们是怎么启动起来的。

首先在Zygote启动的时候，会调用ZygoteInit.main()完成初始化工作。

```java
// ZygoteInit.main()
public static void main(String argv[]) {
    // ...省略
    try {
        boolean startSystemServer = false;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            }
        }
        // ...省略
        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

            if (r != null) {
                r.run();
                return;
            }
        }
    }
}
```
接下来看forkSystemServer()方法的实现：
```java
private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {

    String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                    + "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            "com.android.server.SystemServer",
    };

    int pid;

    try {
        parsedArgs = new ZygoteArguments(args);
        Zygote.applyDebuggerSystemProperty(parsedArgs);
        Zygote.applyInvokeWithSystemProperty(parsedArgs);

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
                parsedArgs.mUid, parsedArgs.mGid,
                parsedArgs.mGids,
                parsedArgs.mRuntimeFlags,
                null,
                parsedArgs.mPermittedCapabilities,
                parsedArgs.mEffectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}

static int forkSystemServer(int uid, int gid, int[] gids, int runtimeFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    ZygoteHooks.preFork();

    int pid = nativeForkSystemServer(
            uid, gid, gids, runtimeFlags, rlimits,
            permittedCapabilities, effectiveCapabilities);

    // Set the Java Language thread priority to the default value for new apps.
    Thread.currentThread().setPriority(Thread.NORM_PRIORITY);

    ZygoteHooks.postForkCommon();
    return pid;
}
```
在该方法中，首先初始化了相关参数args，然后调用Zygote.forkSystemServer()方法，最终调用的是nativeForkSystemServer方法，fork出了SystemServer进程，并返回进程号pid。如果返回的pid==0，就说明fork出的是子进程，则调用handleSystemServerProcess()，将之前整理的参数传入进行解析，最后返回得到的SystemServer进程。

接下来看SystemServer的启动，当zygote进程中开启了SystemServer进程后，就会从SystemServer.main()入口开启。：
```java
public final class SystemServer {

  //开始的入口
  public static void main(String[] args) {
          new SystemServer().run();
  }

  public SystemServer() {
      // Check for factory test mode.
      mFactoryTestMode = FactoryTest.getMode();
      // Remember if it's runtime restart(when sys.boot_completed is already set) or reboot
      mRuntimeRestart = "1".equals(SystemProperties.get("sys.boot_completed"));
  }

  private void run() {
      try {
          //开启Handler线程
          Looper.prepareMainLooper();

          System.loadLibrary("android_servers");

          performPendingShutdown();
          // 创建上下文对象
          createSystemContext();

          mSystemServiceManager = new SystemServiceManager(mSystemContext);
          LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
          // Prepare the thread pool for init tasks that can be parallelized
          SystemServerInitThreadPool.get();
      } finally {
          traceEnd();  // InitBeforeStartServices
      }

      // Start services.
      try {
          // 互相依赖的service会在这里被启动
          startBootstrapServices();
          //启动核心进程 DropBoxManagerService  BatteryService  UsageStatsService  WebViewUpdateService
          startCoreServices();
          startOtherServices();
          SystemServerInitThreadPool.shutdown();
      } catch (Throwable ex) {
          throw ex;
      } finally {
          traceEnd();
      }

      // 开启线程循环
      Looper.loop();
  }

}
```
在SystemServer的run()方法中，首先进行了一系列准备工作，包括开启Handler线程，加载"android_servers"的native Library，以及创建系统上下文对象，并初始化SystemServiceManager，最后将会启动不同的服务，其中ActivityManagerService是在startBootstrapServices()方法中启动的。

```java

private void startBootstrapServices() {

    // 开启ActivityManagerService 服务
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();


    // 开启电源管理服务，其他服务的开启都需要依赖它
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

    // 开启RecoverySystemService
    mSystemServiceManager.startService(RecoverySystemService.class);

    // LightsService

    mSystemServiceManager.startService(LightsService.class);

    // DisplayManagerService

    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    // 开启PackageManagerService

    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);

    // ...
}
```
经过上面这些步骤，AMS服务就已经被启动了，并且在`createSystemContext()`方法中，完成了ActivityThread的创建，并通过`attach()`方法将两者绑定。
