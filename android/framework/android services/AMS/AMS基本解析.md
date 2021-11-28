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

          // 检查最近一次关机是否失败
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

          // 安装系统核心provider、WindowManagerSerivce,Settings Observer
          // startSystemUi  SystemReady
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
    mActivityManagerService.setSystemProcess();
}
```
在这里，我们注意到在startService方法中创建ActivityManagerService的时候，传入的是ActivityManagerService.Lifecycle，并通过反射调用构造函数，得到ActivityManagerService.Lifecycle的实例，最后返回作为ActivityManagerService的对象。
```java
public <T extends SystemService> T startService(Class<T> serviceClass) {
   try {
        final String name = serviceClass.getName();
        final T service;
        try {
          Constructor<T> constructor = serviceClass.getConstructor(Context.class);
          service = constructor.newInstance(mContext);
          startService(service);
          return service;
       }
    }
```
接下来我们来看一下这个ActivityManagerService.Lifecycle是一个什么样的类。
```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;
    private static ActivityTaskManagerService sAtm;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context, sAtm);
    }

    public static ActivityManagerService startService(
            SystemServiceManager ssm, ActivityTaskManagerService atm) {
        sAtm = atm;
        return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
    }

    @Override
    public void onStart() {
        mService.start();
    }

    @Override
    public void onBootPhase(int phase) {
        mService.mBootPhase = phase;
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            mService.mBatteryStatsService.systemServicesReady();
            mService.mServices.systemServicesReady();
        } else if (phase == PHASE_ACTIVITY_MANAGER_READY) {
            mService.startBroadcastObservers();
        } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
            mService.mPackageWatchdog.onPackagesReady();
        }
    }

    @Override
    public void onCleanupUser(int userId) {
        mService.mBatteryStatsService.onCleanupUser(userId);
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```
Lifecycle继承自SystemService，在构造方法中实例化了ActivityManagerService实例，并覆写了SystemService的`onStart()`和`onBootPhase()`方法。从这里来看，ActivityManagerService.Lifecycle持有ActivityManagerService的引用，SystemService通过Lifecycle来对它进行管理，在不同的生命周期，调用ActivityManagerService相应的方法。


我们一开始就说了，AMS是一个特别重要的系统服务，主要负责四大组件的管理和调度工作，所以再来看看AMS的初始化做了哪些工作：
```java
public ActivityManagerService(Context systemContext) {
    LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
    mInjector = new Injector();
    mContext = systemContext;//赋值mContext
    mFactoryTest = FactoryTest.getMode();
    //获取当前的ActivityThread
    mSystemThread = ActivityThread.currentActivityThread();
    //赋值mUiContext
    mUiContext = mSystemThread.getSystemUiContext();
    Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());
    mPermissionReviewRequired = mContext.getResources().getBoolean(
    com.android.internal.R.bool.config_permissionReviewRequired);

    //创建Handler线程，用来处理handler消息
    mHandlerThread = new ServiceThread(TAG, THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();

    mHandler = new MainHandler(mHandlerThread.getLooper());
    mUiHandler = mInjector.getUiHandler(this);//处理ui相关msg的Handler
    mProcStartHandlerThread = new ServiceThread(TAG + ":procStart", THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
    mProcStartHandlerThread.start();
    mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

    //管理AMS的一些常量，厂商定制系统就可能修改此处
    mConstants = new ActivityManagerConstants(this, mHandler);
    /* static; one-time init here */
    if (sKillHandler == null) {
        sKillThread = new ServiceThread(TAG + ":kill",
        THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
        sKillThread.start();
        sKillHandler = new KillHandler(sKillThread.getLooper());
    }
    //初始化管理前台、后台广播的队列， 系统会优先遍历发送前台广播
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler, "foreground", BROADCAST_FG_TIMEOUT, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler, "background", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;
    //初始化管理Service的 ActiveServices对象
    mServices = new ActiveServices(this);
    //初始化Provider的管理者
    mProviderMap = new ProviderMap(this);
    //初始化APP错误日志的打印器
    mAppErrors = new AppErrors(mUiContext, this);
    //创建电池统计服务， 并输出到指定目录
    File dataDir = Environment.getDataDirectory();
    File systemDir = new File(dataDir, "system");
    systemDir.mkdirs();
    mAppWarnings = new AppWarnings(this, mUiContext, mHandler, mUiHandler, systemDir);
    // TODO: Move creation of battery stats service outside of activity manager service.
    mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, mHandler);
    mBatteryStatsService.getActiveStatistics().readLocked();
    mBatteryStatsService.scheduleWriteToDisk();
    mOnBattery = DEBUG_POWER ? true :
          mBatteryStatsService.getActiveStatistics().getIsOnBattery();
    //创建进程统计分析服务，追踪统计哪些进程有滥用或不良行为
    mBatteryStatsService.getActiveStatistics().setCallback(this);
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
    mAppOpsService = mInjector.getAppOpsService(new File(systemDir,"appops.xml"), mHandler);
    //加载Uri的授权文件
    mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"), "urigrants");
    //负责管理多用户
    mUserController = new UserController(this);
    //vr功能的控制器
    mVrController = new VrController(this);
    //初始化OpenGL版本号
    GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version", ConfigurationInfo.GL_ES_VERSION_UNDEFINED);
    if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
        mUseFifoUiScheduling = true;
    }
    mTrackingAssociations = "1".equals(SystemProperties.get("debug.trackassociations"));
    mTempConfig.setToDefaults();
    mTempConfig.setLocales(LocaleList.getDefault());
    mConfigurationSeq = mTempConfig.seq = 1;

    //管理ActivityStack的重要类，这里面记录着activity状态信息，是AMS中的核心类
    mStackSupervisor = createStackSupervisor();
    mStackSupervisor.onConfigurationChanged(mTempConfig);
    //根据当前可见的Activity类型，控制Keyguard遮挡，关闭和转换。 Keyguard就是我们的锁屏相关页面
    mKeyguardController = mStackSupervisor.getKeyguardController();
    // 管理APK的兼容性配置
    // 解析/data/system/packages-compat.xml文件，该文件用于存储那些需要考虑屏幕尺寸的APK信息，
    mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
    //Intent防火墙，Google定义了一组规则，来过滤intent，如果触发了，则intent会被系统丢弃，且不会告知发送者
    mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
    mTaskChangeNotificationController =
    new TaskChangeNotificationController(this, mStackSupervisor, mHandler);
    //这是activity启动的处理类，这里管理者activity启动中用到的intent信息和flag标识，也和stack和task有重要的联系
    mActivityStartController = new ActivityStartController(this);
    mRecentTasks = createRecentTasks();
    mStackSupervisor.setRecentTasks(mRecentTasks);
    mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mHandler);
    // app activity生命周期相关
    mLifecycleManager = new ClientLifecycleManager();
    //启动一个线程专门跟进cpu当前状态信息，AMS对当前cpu状态了如指掌，可以更加高效的安排其他工作
    mProcessCpuThread = new Thread("CpuTracker") {
        @Override
        public void run() {
            synchronized (mProcessCpuTracker) {
                mProcessCpuInitLatch.countDown();
                mProcessCpuTracker.init();
            }
            while (true) {
                try {
                    try {
                        synchronized(this) {
                            final long now = SystemClock.uptimeMillis();
                            long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                            long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME) - now;
                            //Slog.i(TAG, "Cpu delay=" + nextCpuDelay

                            // + ", write delay=" + nextWriteDelay);
                            if (nextWriteDelay < nextCpuDelay) {
                                nextCpuDelay = nextWriteDelay;
                            }
                            if (nextCpuDelay > 0) {
                                mProcessCpuMutexFree.set(true);
                                this.wait(nextCpuDelay);
                            }
                      }
                  } catch (InterruptedException e) {
                  }
                  updateCpuStatsNow();
              } catch (Exception e) {
                  Slog.e(TAG, "Unexpected exception collecting process
                  stats", e);
              }
          }
        }
    };
    mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);
    //看门狗，监听进程。这个类每分钟调用一次监视器。 如果进程没有任何返回就杀掉
    Watchdog.getInstance().addMonitor(this);
    Watchdog.getInstance().addThread(mHandler);
    // bind background thread to little cores
    // this is expected to fail inside of framework tests because apps can't touch cpusets directly
    // make sure we've already adjusted system_server's internal view of itself first
    updateOomAdjLocked();
    try {
        Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
        Process.THREAD_GROUP_BG_NONINTERACTIVE);
    } catch (Exception e) {
        Slog.w(TAG, "Setting background thread cpuset failed");
    }
}

```

最后，当开启完一系列服务后，会调用`setSystemProcess()`方法
```java
public void setSystemProcess() {
    try {
        //讲AMS注册到SM中
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        // 进程统计
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        //内存
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
        //图像信息
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        //数据库
        ServiceManager.addService("dbinfo", new DbBinder(this));
        //cpu
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
        //权限
        ServiceManager.addService("permission", new PermissionController(this));
        //进程服务
        ServiceManager.addService("processinfo", new ProcessInfoService(this));

        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

        synchronized (this) {
            ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
                    false,
                    0,
                    new HostingRecord("system"));
            app.setPersistent(true);
            app.pid = MY_PID;
            app.getWindowProcessController().setPid(MY_PID);
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            mPidsSelfLocked.put(app);
            mProcessList.updateLruProcessLocked(app, false, null);
            // 手机杀进程有关
            updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
                "Unable to find android system package", e);
    }

    // Start watching app ops after we and the package manager are up and running.
    mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
            new IAppOpsCallback.Stub() {
                @Override public void opChanged(int op, int uid, String packageName) {
                    if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                        if (mAppOpsService.checkOperation(op, uid, packageName)
                                != AppOpsManager.MODE_ALLOWED) {
                            runInBackgroundDisabled(uid);
                        }
                    }
                }
            });
}
```

- 注册服务。首先将ActivityManagerService注册到ServiceManager中，其次将几个与系统性能调试相关的服务注册到ServiceManager。这里还涉及到两个重要操作：

- 查询并处理ApplicationInfo。首先调用PackageManagerService的接口，查询包名为android的应用程序的ApplicationInfo信息，对应于**framework-res.apk**。然后以该信息为参数调用`ActivityThread上的installSystemApplicationInfo()`方法。

- 创建并处理ProcessRecord。调用ActivityManagerService上的newProcessRecordLocked，创建一个ProcessRecord类型的对象，并保存该对象的信息

经过上面这些步骤，ActivityManagerService的启动流程就走完了，且在`createSystemContext()`方法中完成了ActivityThread的创建，并通过`attach()`方法将两者绑定。
