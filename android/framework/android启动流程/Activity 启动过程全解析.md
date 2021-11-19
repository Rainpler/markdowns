


## 前言

*   一个 App 是怎么启动起来的？
*   App 的程序入口到底是哪里？
*   Launcher 到底是什么神奇的东西？
*   听说还有个 AMS 的东西，它是做什么的？
*   Binder 是什么？他是如何进行 IPC 通信的？
*   Activity 生命周期到底是什么时候调用的？被谁调用的？
*   等等...

你是不是还有很多类似的疑问一直没有解决？没关系，这篇文章将结合源码以及大量的优秀文章，站在巨人的肩膀上，更加通俗的来试着解释一些问题。但是毕竟源码繁多、经验有限，文中不免会出现一些纰漏甚至是错误，还恳请大家指出，互相学习。

## 学习目标

1.  了解从手机开机第一个 zygote 进程创建，到点击桌面上的图标，进入一个 App 的完整流程，并且从源码的角度了解到一个 Activity 的生命周期是怎么回事
2.  了解到 ActivityManagerServices(即 AMS)、ActivityStack、ActivityThread、Instrumentation 等 Android framework 中非常重要的基础类的作用，及相互间的关系
3.  了解 AMS 与 ActivityThread 之间利用 Binder 进行 IPC 通信的过程，了解 AMS 和 ActivityThread 在控制 Activity 生命周期起到的作用和相互之间的配合
4.  了解与 Activity 相关的 framework 层的其他琐碎问题

## 写作方式

这篇文章我决定采用一问一答的方式进行。

其实在这之前，我试过把每个流程的代码调用过程，用粘贴源代码的方式写在文章里，但是写完一部分之后，发现由于代码量太大，整篇文章和老太太的裹脚布一样——又臭又长，虽然每个重要的操作可以显示出详细调用过程，但是太关注于细节反而导致从整体上不能很好的把握。所以在原来的基础之上进行了修改，对关键的几个步骤进行重点介绍，力求语言简洁，重点突出，从而让大家在更高的层次上对 framework 层有个认识，然后结合后面我给出的参考资料，大家就可以更加快速，更加高效的了解这一块的整体架构。

## 主要对象功能介绍

我们下面的文章将围绕着这几个类进行介绍。可能你第一次看的时候，印象不深，不过没关系，当你跟随者我读完这篇文章的时候，我会在最后再次列出这些对象的功能，相信那时候你会对这些类更加的熟悉和深刻。

*   ActivityManagerServices，简称 AMS，服务端对象，负责系统中所有 Activity 的生命周期
*   ActivityThread，App 的真正入口。当开启 App 之后，会调用 main() 开始运行，开启消息循环队列，这就是传说中的 UI 线程或者叫主线程。与 ActivityManagerServices 配合，一起完成 Activity 的管理工作
*   ApplicationThread，用来实现 ActivityManagerService 与 ActivityThread 之间的交互。在 ActivityManagerService 需要管理相关 Application 中的 Activity 的生命周期时，通过 ApplicationThread 的代理对象与 ActivityThread 通讯。
*   ApplicationThreadProxy，是 ApplicationThread 在服务器端的代理，负责和客户端的 ApplicationThread 通讯。AMS 就是通过该代理与 ActivityThread 进行通信的。
*   Instrumentation，每一个应用程序只有一个 Instrumentation 对象，每个 Activity 内都有一个对该对象的引用。Instrumentation 可以理解为应用进程的管家，ActivityThread 要创建或暂停某个 Activity 时，都需要通过 Instrumentation 来进行具体的操作。
*   ActivityStack，Activity 在 AMS 的栈管理，用来记录已经启动的 Activity 的先后关系，状态信息等。通过 ActivityStack 决定是否需要启动新的进程。
*   ActivityRecord，ActivityStack 的管理对象，每个 Activity 在 AMS 对应一个 ActivityRecord，来记录 Activity 的状态以及其他的管理信息。其实就是服务器端的 Activity 对象的映像。
*   TaskRecord，AMS 抽象出来的一个 “任务” 的概念，是记录 ActivityRecord 的栈，一个 “Task” 包含若干个 ActivityRecord。AMS 用 TaskRecord 确保 Activity 启动和退出的顺序。如果你清楚 Activity 的 4 种 launchMode，那么对这个概念应该不陌生。

## 主要流程介绍


下面将按照 App 启动过程的先后顺序，一问一答，来解释一些事情。

让我们开始吧！

### zygote 是什么？有什么作用？


首先，你觉得这个单词眼熟不？当你的程序 Crash 的时候，打印的红色 log 下面通常带有这一个单词。

zygote 意为 “受精卵 “。Android 是基于 Linux 系统的，而在 Linux 中，所有的进程都是由 init 进程直接或者是间接 fork 出来的，zygote 进程也不例外。

在 Android 系统里面，zygote 是一个进程的名字。Android 是基于 Linux System 的，当你的手机开机的时候，Linux 的内核加载完成之后就会启动一个叫 “init“的进程。在 Linux System 里面，所有的进程都是由 init 进程 fork 出来的，我们的 zygote 进程也不例外。

我们都知道，每一个 App 其实都是

*   一个单独的 dalvik 虚拟机
*   一个单独的进程

所以当系统里面的第一个 zygote 进程运行之后，在这之后再开启 App，就相当于开启一个新的进程。而为了实现资源共用和更快的启动速度，Android 系统开启新进程的方式，是通过 fork 第一个 zygote 进程实现的。所以说，除了第一个 zygote 进程，其他应用所在的进程都是 zygote 的子进程，这下你明白为什么这个进程叫 “受精卵” 了吧？因为就像是一个受精卵一样，它能快速的分裂，并且产生遗传物质一样的细胞！

### SystemServer 是什么？有什么作用？它与 zygote 的关系是什么？


首先我要告诉你的是，SystemServer 也是一个进程，而且是由 zygote 进程 fork 出来的。

知道了 SystemServer 的本质，我们对它就不算太陌生了，这个进程是 Android Framework 里面两大非常重要的进程之一——另外一个进程就是上面的 zygote 进程。

为什么说 SystemServer 非常重要呢？因为系统里面重要的服务都是在这个进程里面开启的，比如
ActivityManagerService、PackageManagerService、WindowManagerService 等等，看着是不是都挺眼熟的？

那么这些系统服务是怎么开启起来的呢？

在 zygote 开启的时候，会调用 ZygoteInit.main() 进行初始化

```java
public static void main(String argv[]) {

     ...ignore some code...

    //在加载首个zygote的时候，会传入初始化参数，使得startSystemServer = true
     boolean startSystemServer = false;
     for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            ...ignore some code...

         //开始fork我们的SystemServer进程
     if (startSystemServer) {
                startSystemServer(abiList, socketName);
         }

     ...ignore some code...

}
```

我们看下 startSystemServer() 做了些什么

```java
/**留着这个注释，就是为了说明SystemServer确实是被fork出来的
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {

         ...ignore some code...

        //留着这段注释，就是为了说明上面ZygoteInit.main(String argv[])里面的argv就是通过这种方式传递进来的
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-",
            "com.android.server.SystemServer",
        };

        int pid;
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        //确实是fork出来的吧，我没骗你吧~不对，是fork出来的 -_-|||
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```


### ActivityManagerService 是什么？什么时候初始化的？有什么作用？


ActivityManagerService，简称 AMS，服务端对象，负责系统中所有 Activity 的生命周期。

ActivityManagerService 进行初始化的时机很明确，就是在 SystemServer 进程开启的时候，就会初始化 ActivityManagerService。从下面的代码中可以看到

```java
public final class SystemServer {

    //zygote的主入口
    public static void main(String[] args) {
        new SystemServer().run();
    }

    public SystemServer() {
        // Check for factory test mode.
        mFactoryTestMode = FactoryTest.getMode();
    }

    private void run() {

        ...ignore some code...

         //加载本地系统服务库，并进行初始化
        System.loadLibrary("android_servers");
        nativeInit();

        // 创建系统上下文
        createSystemContext();

        //初始化SystemServiceManager对象，下面的系统服务开启都需要调用SystemServiceManager.startService(Class<T>)，这个方法通过反射来启动对应的服务
        mSystemServiceManager = new SystemServiceManager(mSystemContext);

        //开启服务
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        ...ignore some code...

    }

    //初始化系统上下文对象mSystemContext，并设置默认的主题,mSystemContext实际上是一个ContextImpl对象。调用ActivityThread.systemMain()的时候，会调用ActivityThread.attach(true)，而在attach()里面，则创建了Application对象，并调用了Application.onCreate()。
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }

    //在这里开启了几个核心的服务，因为这些服务之间相互依赖，所以都放在了这个方法里面。
    private void startBootstrapServices() {

        ...ignore some code...

        //初始化ActivityManagerService
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);

        //初始化PowerManagerService，因为其他服务需要依赖这个Service，因此需要尽快的初始化
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // 现在电源管理已经开启，ActivityManagerService负责电源管理功能
        mActivityManagerService.initPowerManagement();

        // 初始化DisplayManagerService
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    //初始化PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, mInstaller,
       mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);

    ...ignore some code...

    }

}
```

经过上面这些步骤，我们的 ActivityManagerService 对象已经创建好了，并且完成了成员变量初始化。而且在这之前，调用 createSystemContext() 创建系统上下文的时候，也已经完成了 mSystemContext 和 ActivityThread 的创建。注意，这是系统进程开启时的流程，在这之后，会开启系统的 Launcher 程序，完成系统界面的加载与显示。

你是否会好奇，我为什么说 AMS 是服务端对象？下面我给你介绍下 Android 系统里面的服务器和客户端的概念。

其实服务器客户端的概念不仅仅存在于 Web 开发中，在 Android 的框架设计中，使用的也是这一种模式。服务器端指的就是所有 App 共用的系统服务，比如我们这里提到的 ActivityManagerService，和前面提到的 PackageManagerService、WindowManagerService 等等，这些基础的系统服务是被所有的 App 公用的，当某个 App 想实现某个操作的时候，要告诉这些系统服务，比如你想打开一个 App，那么我们知道了包名和 MainActivity 类名之后就可以打开

```java
Intent intent = new Intent(Intent.ACTION_MAIN);
intent.addCategory(Intent.CATEGORY_LAUNCHER);
ComponentName cn = new ComponentName(packageName, className);
intent.setComponent(cn);
startActivity(intent);
```

但是，我们的 App 通过调用 startActivity() 并不能直接打开另外一个 App，这个方法会通过一系列的调用，最后还是告诉 AMS 说：“我要打开这个 App，我知道他的住址和名字，你帮我打开吧！” 所以是 AMS 来通知 zygote 进程来 fork 一个新进程，来开启我们的目标 App 的。这就像是浏览器想要打开一个超链接一样，浏览器把网页地址发送给服务器，然后还是服务器把需要的资源文件发送给客户端的。

知道了 Android Framework 的客户端服务器架构之后，我们还需要了解一件事情，那就是我们的 App 和 AMS(SystemServer 进程) 还有 zygote 进程分属于三个独立的进程，他们之间如何通信呢？

App 与 AMS 通过 Binder 进行 IPC 通信，AMS(SystemServer 进程) 与 zygote 通过 Socket 进行 IPC 通信。

那么 AMS 有什么用呢？在前面我们知道了，如果想打开一个 App 的话，需要 AMS 去通知 zygote 进程，除此之外，其实所有的 Activity 的开启、暂停、关闭都需要 AMS 来控制，所以我们说，AMS 负责系统中所有 Activity 的生命周期。

在 Android 系统中，任何一个 Activity 的启动都是由 AMS 和应用程序进程（主要是 ActivityThread）相互配合来完成的。AMS 服务统一调度系统中所有进程的 Activity 启动，而每个 Activity 的启动过程则由其所属的进程具体来完成。

这样说你可能还是觉得比较抽象，没关系，下面有一部分是专门来介绍 AMS 与 ActivityThread 如何一起合作控制 Activity 的生命周期的。

### Launcher 是什么？什么时候启动的？


当我们点击手机桌面上的图标的时候，App 就由 Launcher 开始启动了。但是，你有没有思考过 Launcher 到底是一个什么东西？

Launcher 本质上也是一个应用程序，和我们的 App 一样，也是继承自 Activity

packages/apps/Launcher2/src/com/android/launcher2/Launcher.java

```java
public final class Launcher extends Activity
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks,
                   View.OnTouchListener {
                   }
```

Launcher 实现了点击、长按等回调接口，来接收用户的输入。既然是普通的 App，那么我们的开发经验在这里就仍然适用，比如，我们点击图标的时候，是怎么开启的应用呢？如果让你，你怎么做这个功能呢？捕捉图标点击事件，然后 startActivity() 发送对应的 Intent 请求呗！是的，Launcher 也是这么做的，就是这么 easy！

那么到底是处理的哪个对象的点击事件呢？既然 Launcher 是 App，并且有界面，那么肯定有布局文件呀，是的，我找到了布局文件 launcher.xml

```xml
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res/com.android.launcher"
    android:id="@+id/launcher">

    <com.android.launcher2.DragLayer
        android:id="@+id/drag_layer"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="true">

        <!-- Keep these behind the workspace so that they are not visible when
             we go into AllApps -->
        <include
            android:id="@+id/dock_divider"
            layout="@layout/workspace_divider"
            android:layout_marginBottom="@dimen/button_bar_height"
            android:layout_gravity="bottom" />

        <include
            android:id="@+id/paged_view_indicator"
            layout="@layout/scroll_indicator"
            android:layout_gravity="bottom"
            android:layout_marginBottom="@dimen/button_bar_height" />

        <!-- The workspace contains 5 screens of cells -->
        <com.android.launcher2.Workspace
            android:id="@+id/workspace"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:paddingStart="@dimen/workspace_left_padding"
            android:paddingEnd="@dimen/workspace_right_padding"
            android:paddingTop="@dimen/workspace_top_padding"
            android:paddingBottom="@dimen/workspace_bottom_padding"
            launcher:defaultScreen="2"
            launcher:cellCountX="@integer/cell_count_x"
            launcher:cellCountY="@integer/cell_count_y"
            launcher:pageSpacing="@dimen/workspace_page_spacing"
            launcher:scrollIndicatorPaddingLeft="@dimen/workspace_divider_padding_left"
            launcher:scrollIndicatorPaddingRight="@dimen/workspace_divider_padding_right">

            <include android:id="@+id/cell1" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell2" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell3" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell4" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell5" layout="@layout/workspace_screen" />
        </com.android.launcher2.Workspace>

    ...ignore some code...

    </com.android.launcher2.DragLayer>
</FrameLayout>
```

为了方便查看，我删除了很多代码，从上面这些我们应该可以看出一些东西来：Launcher 大量使用 <include/> 标签来实现界面的复用，而且定义了很多的自定义控件实现界面效果，dock_divider 从布局的参数声明上可以猜出，是底部操作栏和上面图标布局的分割线，而 paged_view_indicator 则是页面指示器，和 App 首次进入的引导页下面的界面引导是一样的道理。当然，我们最关心的是 Workspace 这个布局，因为注释里面说在这里面包含了 5 个屏幕的单元格，想必你也猜到了，这个就是在首页存放我们图标的那五个界面 (不同的 ROM 会做不同的 DIY，数量不固定)。

接下来，我们应该打开 workspace_screen 布局，看看里面有什么东东。

workspace_screen.xml

```xml
<com.android.launcher2.CellLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res/com.android.launcher"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:paddingStart="@dimen/cell_layout_left_padding"
    android:paddingEnd="@dimen/cell_layout_right_padding"
    android:paddingTop="@dimen/cell_layout_top_padding"
    android:paddingBottom="@dimen/cell_layout_bottom_padding"
    android:hapticFeedbackEnabled="false"
    launcher:cellWidth="@dimen/workspace_cell_width"
    launcher:cellHeight="@dimen/workspace_cell_height"
    launcher:widthGap="@dimen/workspace_width_gap"
    launcher:heightGap="@dimen/workspace_height_gap"
    launcher:maxGap="@dimen/workspace_max_gap" />
```

里面就一个 CellLayout，也是一个自定义布局，那么我们就可以猜到了，既然可以存放图标，那么这个自定义的布局很有可能是继承自 ViewGroup 或者是其子类，实际上，CellLayout 确实是继承自 ViewGroup。在 CellLayout 里面，只放了一个子 View，那就是 ShortcutAndWidgetContainer。从名字也可以看出来，ShortcutAndWidgetContainer 这个类就是用来存放**快捷图标**和__Widget 小部件__的，那么里面放的是什么对象呢？

在桌面上的图标，使用的是 BubbleTextView 对象，这个对象在 TextView 的基础之上，添加了一些特效，比如你长按移动图标的时候，图标位置会出现一个背景 (不同版本的效果不同)，所以我们找到 BubbleTextView 对象的点击事件，就可以找到 Launcher 如何开启一个 App 了。

除了在桌面上有图标之外，在程序列表中点击图标，也可以开启对应的程序。这里的图标使用的不是 BubbleTextView 对象，而是 PagedViewIcon 对象，我们如果找到它的点击事件，就也可以找到 Launcher 如何开启一个 App。

其实说这么多，和今天的主题隔着十万八千里，上面这些东西，你有兴趣就看，没兴趣就直接跳过，不知道不影响这篇文章阅读。

BubbleTextView 的点击事件在哪里呢？我来告诉你：在 Launcher.onClick(View v) 里面。

```java
/**
     * Launches the intent referred by the clicked shortcut
     */
    public void onClick(View v) {

          ...ignore some code...

         Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            // Open shortcut
            final Intent intent = ((ShortcutInfo) tag).intent;
            int[] pos = new int[2];
            v.getLocationOnScreen(pos);
            intent.setSourceBounds(new Rect(pos[0], pos[1],
                    pos[0] + v.getWidth(), pos[1] + v.getHeight()));
        //开始开启Activity咯~
            boolean success = startActivitySafely(v, intent, tag);

            if (success && v instanceof BubbleTextView) {
                mWaitingForResume = (BubbleTextView) v;
                mWaitingForResume.setStayPressed(true);
            }
        } else if (tag instanceof FolderInfo) {
            //如果点击的是图标文件夹，就打开文件夹
            if (v instanceof FolderIcon) {
                FolderIcon fi = (FolderIcon) v;
                handleFolderClick(fi);
            }
        } else if (v == mAllAppsButton) {
        ...ignore some code...
        }
    }
```

从上面的代码我们可以看到，在桌面上点击快捷图标的时候，会调用

```java
/**
 * The Apps/Customize page that displays all the applications, widgets, and shortcuts.
 */
public class AppsCustomizePagedView extends PagedViewWithDraggableItems implements
        View.OnClickListener, View.OnKeyListener, DragSource,
        PagedViewIcon.PressedCallback, PagedViewWidget.ShortPressListener,
        LauncherTransitionable {

     @Override
    public void onClick(View v) {

            ...ignore some code...

        if (v instanceof PagedViewIcon) {
            mLauncher.updateWallpaperVisibility(true);
            mLauncher.startActivitySafely(v, appInfo.intent, appInfo);
        } else if (v instanceof PagedViewWidget) {
                   ...ignore some code..
         }
     }
   }
```

那么从程序列表界面，点击图标的时候会发生什么呢？实际上，程序列表界面使用的是 AppsCustomizePagedView 对象，所以我在这个类里面找到了 onClick(View v)。

com.android.launcher2.AppsCustomizePagedView.java

```java
boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        try {
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        }
        return success;
    }
```

可以看到，调用的是

```java
boolean startActivity(View v, Intent intent, Object tag) {

        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);

            if (useLaunchAnimation) {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent, opts.toBundle());
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(),
                            opts.toBundle());
                }
            } else {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(), null);
                }
            }
            return true;
        } catch (SecurityException e) {
        ...
        }
        return false;
    }
```

和上面一样！这叫什么？这叫殊途同归！

所以咱们现在又明白了一件事情：不管从哪里点击图标，调用的都是 Launcher.startActivitySafely()。

*   下面我们就可以一步步的来看一下 Launcher.startActivitySafely() 到底做了什么事情。

```java
@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

调用了 startActivity(v, intent, tag)

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            ...ignore some code...
        } else {
            if (options != null) {
                 //当现在的Activity有父Activity的时候会调用，但是在startActivityFromChild()内部实际还是调用的mInstrumentation.execStartActivity()
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
         ...ignore some code...
    }
```

这里会调用 Activity.startActivity(intent, opts.toBundle())，这个方法熟悉吗？这就是我们经常用到的 Activity.startActivity(Intent) 的重载函数。而且由于设置了

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
              ...ignore some code...
      try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```

所以这个 Activity 会添加到一个新的 Task 栈中，而且，startActivity() 调用的其实是 startActivityForResult() 这个方法。

```java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

所以我们现在明确了，Launcher 中开启一个 App，其实和我们在 Activity 中直接 startActivity() 基本一样，都是调用了 Activity.startActivityForResult()。

### Instrumentation 是什么？和 ActivityThread 是什么关系？


还记得前面说过的 Instrumentation 对象吗？每个 Activity 都持有 Instrumentation 对象的一个引用，但是整个进程只会存在一个 Instrumentation 对象。当 startActivityForResult() 调用之后，实际上还是调用了 mInstrumentation.execStartActivity()

```java
final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
```

下面是 mInstrumentation.execStartActivity() 的实现

```java
ActivityManagerNative.getDefault()
                .startActivity
```

所以当我们在程序中调用 startActivity() 的 时候，实际上调用的是 Instrumentation 的相关的方法。

Instrumentation 意为 “仪器”，我们先看一下这个类里面包含哪些方法吧

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

我们可以看到，这个类里面的方法大多数和 Application 和 Activity 有关，是的，这个类就是完成对 Application 和 Activity 初始化和生命周期的工具类。比如说，我单独挑一个 callActivityOnCreate() 让你看看

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{

//从类声明上，我们可以看到ActivityManagerNative是Binder的一个子类，而且实现了IActivityManager接口
 static public IActivityManager getDefault() {
        return gDefault.get();
    }

 //通过单例模式获取一个IActivityManager对象，这个对象通过asInterface(b)获得
 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
}


//最终返回的还是一个ActivityManagerProxy对象
static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

     //这里面的Binder类型的obj参数会作为ActivityManagerProxy的成员变量保存为mRemote成员变量，负责进行IPC通信
        return new ActivityManagerProxy(obj);
    }


}
```

对 activity.performCreate(icicle); 这一行代码熟悉吗？这一行里面就调用了传说中的 Activity 的入口函数 onCreate()，不信？接着往下看

Activity.performCreate()

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

没骗你吧，onCreate 在这里调用了吧。但是有一件事情必须说清楚，那就是这个 Instrumentation 类这么重要，为啥我在开发的过程中，没有发现他的踪迹呢？

是的，Instrumentation 这个类很重要，对 Activity 生命周期方法的调用根本就离不开他，他可以说是一个大管家，但是，这个大管家比较害羞，是一个女的，管内不管外，是老板娘~

那么你可能要问了，老板是谁呀？
老板当然是大名鼎鼎的 ActivityThread 了！

ActivityThread 你都没听说过？那你肯定听说过传说中的 UI 线程吧？是的，这就是 UI 线程。我们前面说过，App 和 AMS 是通过 Binder 传递信息的，那么 ActivityThread 就是专门与 AMS 的外交工作的。

AMS 说：“ActivityThread，你给我暂停一个 Activity！”
ActivityThread 就说：“没问题！” 然后转身和 Instrumentation 说：“老婆，AMS 让暂停一个 Activity，我这里忙着呢，你快去帮我把这事办了把~”
于是，Instrumentation 就去把事儿搞定了。

所以说，AMS 是董事会，负责指挥和调度的，ActivityThread 是老板，虽然说家里的事自己说了算，但是需要听从 AMS 的指挥，而 Instrumentation 则是老板娘，负责家里的大事小事，但是一般不抛头露面，听一家之主 ActivityThread 的安排。

### 如何理解 AMS 和 ActivityThread 之间的 Binder 通信？


前面我们说到，在调用 startActivity() 的时候，实际上调用的是

```java
class ActivityManagerProxy implements IActivityManager{}

public final class ActivityManagerService extends ActivityManagerNative{}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{}
```

但是到这里还没完呢！里面又调用了下面的方法

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();

        ...ignore some code...

        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

这里的 ActivityManagerNative.getDefault 返回的就是 ActivityManagerService 的远程接口，即 ActivityManagerProxy。

怎么知道的呢？往下看

```java
private class ApplicationThread extends ApplicationThreadNative {}

  public abstract class ApplicationThreadNative extends Binder implements IApplicationThread{}

  class ApplicationThreadProxy implements IApplicationThread {}
```

再看 ActivityManagerProxy.startActivity()，在这里面做的事情就是 IPC 通信，利用 Binder 对象，调用 transact()，把所有需要的参数封装成 Parcel 对象，向 AMS 发送数据进行通信。

```java
@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }
```

Binder 本质上只是一种底层通信方式，和具体服务没有关系。为了提供具体服务，Server 必须提供一套接口函数以便 Client 通过远程访问使用各种服务。这时通常采用 Proxy 设计模式：将接口函数定义在一个抽象类中，Server 和 Client 都会以该抽象类为基类实现所有接口函数，所不同的是 Server 端是真正的功能实现，而 Client 端是对这些函数远程调用请求的包装。

为了更方便的说明客户端和服务器之间的 Binder 通信，下面以 ActivityManagerServices 和他在客户端的代理类 ActivityManagerProxy 为例。

ActivityManagerServices 和 ActivityManagerProxy 都实现了同一个接口——IActivityManager。

```java
@Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {

             ...ignore some code...

        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, userId, null, null);
    }
```

虽然都实现了同一个接口，但是代理对象 ActivityManagerProxy 并不会对这些方法进行真正地实现，ActivityManagerProxy 只是通过这种方式对方法的参数进行打包 (因为都实现了相同接口，所以可以保证同一个方法有相同的参数，即对要传输给服务器的数据进行打包)，真正实现的是 ActivityManagerService。

但是这个地方并不是直接由客户端传递给服务器，而是通过 Binder 驱动进行中转。其实我对 Binder 驱动并不熟悉，我们就把他当做一个中转站就 OK，客户端调用 ActivityManagerProxy 接口里面的方法，把数据传送给 Binder 驱动，然后 Binder 驱动就会把这些东西转发给服务器的 ActivityManagerServices，由 ActivityManagerServices 去真正的实施具体的操作。

但是 Binder 只能传递数据，并不知道是要调用 ActivityManagerServices 的哪个方法，所以在数据中会添加方法的唯一标识码，比如前面的 startActivity() 方法：

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, int userId, IActivityContainer iContainer, TaskRecord inTask) {

            ...ignore some code...

              int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options,
                    componentSpecified, null, container, inTask);

            ...ignore some code...

            }
```

上面的 START_ACTIVITY_TRANSACTION 就是方法标示，data 是要传输给 Binder 驱动的数据，reply 则接受操作的返回值。

即

客户端：ActivityManagerProxy =====>Binder 驱动 =====> ActivityManagerService：服务器

而且由于继承了同样的公共接口类，ActivityManagerProxy 提供了与 ActivityManagerService 一样的函数原型，使用户感觉不出 Server 是运行在本地还是远端，从而可以更加方便的调用这些重要的系统服务。

但是！这里 Binder 通信是单方向的，即从 ActivityManagerProxy 指向 ActivityManagerService 的，如果 AMS 想要通知 ActivityThread 做一些事情，应该咋办呢？

还是通过 Binder 通信，不过是换了另外一对，换成了 ApplicationThread 和 ApplicationThreadProxy。

客户端：ApplicationThread <=====Binder 驱动 <===== ApplicationThreadProxy：服务器

他们也都实现了相同的接口 IApplicationThread

```java
final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean componentSpecified, ActivityRecord[] outActivity, ActivityContainer container,
            TaskRecord inTask) {

              err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
              startFlags, true, options, inTask);
        if (err < 0) {
            notifyActivityDrawnForKeyguard();
        }
        return err;
    }
```

剩下的就不必多说了吧，和前面一样。

### AMS 接收到客户端的请求之后，会如何开启一个 Activity？


OK，至此，点击桌面图标调用 startActivity()，终于把数据和要开启 Activity 的请求发送到了 AMS 了。说了这么多，其实这些都在一瞬间完成了，下面咱们研究下 AMS 到底做了什么。

_注：前方有高能的方法调用链，如果你现在累了，请先喝杯咖啡或者是上趟厕所休息下_

AMS 收到 startActivity 的请求之后，会按照如下的方法链进行调用

调用 startActivity()

```java
final int startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {

            ...ignore some code...

            targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);

            ...ignore some code...

             return ActivityManager.START_SUCCESS;
            }
```

调用 startActivityAsUser()

```java
final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {

        //ActivityRecord中存储的TaskRecord信息
        TaskRecord rTask = r.task;

         ...ignore some code...

        //如果不是在新的ActivityTask(也就是TaskRecord)中的话，就找出要运行在的TaskRecord对象
     TaskRecord task = null;
        if (!newTask) {
            boolean startIt = true;
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    // task中的所有Activity都结束了
                    continue;
                }
                if (task == r.task) {
                    // 找到了
                    if (!startIt) {
                        task.addActivityToTop(r);
                        r.putInHistory();
                        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                                (r.info.flags & ActivityInfo.FLAG_SHOW_ON_LOCK_SCREEN) != 0,
                                r.userId, r.info.configChanges, task.voiceSession != null,
                                r.mLaunchTaskBehind);
                        if (VALIDATE_TOKENS) {
                            validateAppTokensLocked();
                        }
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }

      ...ignore some code...

        // Place a new activity at top of stack, so it is next to interact
        // with the user.
        task = r.task;
        task.addActivityToTop(r);
        task.setFrontOfTask();

        ...ignore some code...

         if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```

在这里又出现了一个新对象 ActivityStackSupervisor，通过这个类可以实现对 ActivityStack 的部分操作。

```java
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = getFocusedStack();
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

          ...ignore some code...

        return result;
    }
```

继续调用 startActivityLocked()

```java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            inResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            inResumeTopActivity = false;
        }
        return result;
    }
```

调用 startActivityUncheckedLocked(), 此时要启动的 Activity 已经通过检验，被认为是一个正当的启动请求。

终于，在这里调用到了 ActivityStack 的 startActivityLocked(ActivityRecord r, boolean newTask,boolean doResume, boolean keepCurTransition, Bundle options)。

ActivityRecord 代表的就是要开启的 Activity 对象，里面分装了很多信息，比如所在的 ActivityTask 等，如果这是首次打开应用，那么这个 Activity 会被放到 ActivityTask 的栈顶，

```java
final boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {

          ...ignore some code...
        //找出还没结束的首个ActivityRecord
       ActivityRecord next = topRunningActivityLocked(null);

      //如果一个没结束的Activity都没有，就开启Launcher程序
      if (next == null) {
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: No more activities go home");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            // Only resume home if on home display
            final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                    HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
            return isOnHomeDisplay() &&
                    mStackSupervisor.resumeHomeStackTask(returnTaskType, prev);
        }

        //先需要暂停当前的Activity。因为我们是在Lancher中启动mainActivity，所以当前mResumedActivity！=null，调用startPausingLocked()使得Launcher进入Pausing状态
          if (mResumedActivity != null) {
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
        }

  }
```

调用的是 ActivityStack.startActivityLocked()

```java
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        if (mPausingActivity != null) {
            completePauseLocked(false);
        }

       ...ignore some code...

        if (prev.app != null && prev.app.thread != null)
          try {
                mService.updateUsageStats(prev, false);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

      ...ignore some code...

 }
```

靠！这来回折腾什么呢！从 ActivityStackSupervisor 到 ActivityStack，又调回 ActivityStackSupervisor，这到底是在折腾什么玩意啊！！！

淡定... 淡定... 我知道你也在心里骂娘，世界如此美妙，你却如此暴躁，这样不好，不好...

来来来，咱们继续哈，刚才说到哪里了？哦，对，咱们一起看下 StackSupervisor.resumeTopActivitiesLocked(this, r, options)

```java
public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

我... 已无力吐槽了，又调回 ActivityStack 去了...

ActivityStack.resumeTopActivityLocked()

```java
public static void main(String[] args) {

          ...ignore some code...

       Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

          ...ignore some code...

 }
```

咱们坚持住，看一下 ActivityStack.resumeTopActivityInnerLocked() 到底进行了什么操作

```java
private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        //普通App进这里
        if (!system) {

            ...ignore some code...

            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
           } else {
             //这个分支在SystemServer加载的时候会进入，通过调用
             // private void createSystemContext() {
             //    ActivityThread activityThread = ActivityThread.systemMain()；
             //}

             // public static ActivityThread systemMain() {
        //        if (!ActivityManager.isHighEndGfx()) {
        //            HardwareRenderer.disable(true);
        //        } else {
        //            HardwareRenderer.enableForegroundTrimming();
        //        }
        //        ActivityThread thread = new ActivityThread();
        //        thread.attach(true);
        //        return thread;
        //    }
           }
    }
```

在这个方法里，prev.app 为记录启动 Lancher 进程的 ProcessRecord，prev.app.thread 为 Lancher 进程的远程调用接口 IApplicationThead，所以可以调用 prev.app.thread.schedulePauseActivity，到 Lancher 进程暂停指定 Activity。

```java
@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

在 Lancher 进程中消息传递，调用 ActivityThread.handlePauseActivity()，最终调用 ActivityThread.performPauseActivity() 暂停指定 Activity。接着通过前面所说的 Binder 通信，通知 AMS 已经完成暂停的操作。

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {


             thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
                    mCoreSettingsObserver.getCoreSettingsLocked());


            }
```

上面这些调用过程非常复杂，源码中各种条件判断让人眼花缭乱，所以说如果你没记住也没关系，你只要记住这个流程，理解了 Android 在控制 Activity 生命周期时是如何操作，以及是通过哪几个关键的类进行操作的就可以了，以后遇到相关的问题之道从哪块下手即可，这些过程我虽然也是撸了一遍，但还是记不清。

最后来一张高清无码大图，方便大家记忆：

[请戳这里 (图片 3.3M，请用电脑观看)](http://i11.tietuku.com/0582844414810f38.png)

### 送给你们的彩蛋


#### 不要使用 startActivityForResult(intent,RESULT_OK)


这是因为 startActivity() 是这样实现的

```java
private class ApplicationThread extends ApplicationThreadNative {

  public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

                 ...ignore some code...

             AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);

           }

}
```

而

```java
private class H extends Handler {

       ...ignore some code...

      public static final int BIND_APPLICATION        = 110;

       ...ignore some code...

      public void handleMessage(Message msg) {
          switch (msg.what) {
         ...ignore some code...
          case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
         ...ignore some code...
         }
 }
```

所以

```java
private void handleBindApplication(AppBindData data) {

 try {

           ...ignore some code...

            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

           ...ignore some code...

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
            }
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
 }
```

你不可能从 onActivityResult() 里面收到任何回调。而这个问题是相当难以被发现的，就是因为这个坑，我工作一年多来第一次加班到 9 点 (ˇˍˇ）

### 一个 App 的程序入口到底是什么？


是 ActivityThread.main()。

### 整个 App 的主线程的消息循环是在哪里创建的？


是在 ActivityThread 初始化的时候，就已经创建消息循环了，所以在主线程里面创建 Handler 不需要指定 Looper，而如果在其他线程使用 Handler，则需要单独使用 Looper.prepare() 和 Looper.loop() 创建消息循环。

```java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

    //传进来的是null，所以这里不会执行，onCreate在上一层执行
        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {

            }
        }
         ...ignore some code...

       }

        return app;
    }
```

### Application 是在什么时候创建的？onCreate() 什么时候调用的？


也是在 ActivityThread.main() 的时候，再具体点呢，就是在 thread.attach(false) 的时候。

看你的表情，不信是吧！凯子哥带你溜溜~

我们先看一下 ActivityThread.attach()

```java
static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);
        return app;
    }
```

这里需要关注的就是 mgr.attachApplication(mAppThread)，这个就会通过 Binder 调用到 AMS 里面对应的方法

```java
@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

然后就是

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
             thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            }
```

thread 是 IApplicationThread，实际上就是 ApplicationThread 在服务端的代理类 ApplicationThreadProxy，然后又通过 IPC 就会调用到 ApplicationThread 的对应方法

```java
private class ApplicationThread extends ApplicationThreadNative {

  public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

                 ...ignore some code...

             AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);

           }

}
```

我们需要关注的其实就是最后的 sendMessage()，里面有函数的编号 H.BIND_APPLICATION，然后这个 Messge 会被 H 这个 Handler 处理

```java
private class H extends Handler {

       ...ignore some code...

      public static final int BIND_APPLICATION        = 110;

       ...ignore some code...

      public void handleMessage(Message msg) {
          switch (msg.what) {
         ...ignore some code...
          case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
         ...ignore some code...
         }
 }
```

最后就在下面这个方法中，完成了实例化，拨那个企鹅通过 mInstrumentation.callApplicationOnCreate 实现了 onCreate() 的调用。

```java
private void handleBindApplication(AppBindData data) {

 try {

           ...ignore some code...

            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

           ...ignore some code...

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
            }
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
 }
```

data.info 是一个 LoadeApk 对象。
LoadeApk.data.info.makeApplication()

```java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
        Application app = null;
        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }
        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
    //传进来的是null，所以这里不会执行，onCreate在上一层执行
        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
            }
        }
         ...ignore some code...
       }
        return app;
    }
```

所以最后还是通过 Instrumentation.makeApplication() 实例化的，这个老板娘真的很厉害呀！

```java
static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);  
        return app;
    }
```

而且通过反射拿到 Application 对象之后，直接调用 attach()，所以 attach() 调用是在 onCreate() 之前的。

## 参考文章


下面的这些文章都是这方面比较精品的，希望你抽出时间研究，这可能需要花费很长时间，但是如果你想进阶为中高级开发者，这一步是必须的。

再次感谢下面这些文章的作者的分享精神。

### Binder


*   [Android Bander 设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)

### zygote


*   [Android 系统进程 Zygote 启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)
*   [Android 之 zygote 与进程创建](http://blog.csdn.net/xieqibao/article/details/6581975)
*   [Zygote 浅谈](http://www.th7.cn/Program/Android/201404/187670.shtml)

### ActivityThread、Instrumentation、AMS


*   [Android Activity.startActivity 流程简介](http://blog.csdn.net/myarrow/article/details/14224273)
*   [Android 应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696#comments)
*   [框架层理解 Activity 生命周期 (APP 启动过程)](http://laokaddk.blog.51cto.com/368606/1206840)
*   [Android 应用程序窗口设计框架介绍](http://blog.csdn.net/yangwen123/article/details/35987609)
*   [ActivityManagerService 分析一：AMS 的启动](http://www.xuebuyuan.com/2172927.html)
*   [Android 应用程序窗口设计框架介绍](http://blog.csdn.net/yangwen123/article/details/35987609)

### Launcher


*   [Android 4.0 Launcher 源码分析系列 (一)](http://mobile.51cto.com/hot-312129.htm)
*   [Android Launcher 分析和修改 9——Launcher 启动 APP 流程](http://www.cnblogs.com/mythou/p/3187881.html)

## 结语


OK，到这里，这篇文章算是告一段落了，我们再回头看看一开始的几个问题，你还困惑吗？

*   一个 App 是怎么启动起来的？
*   App 的程序入口到底是哪里？
*   Launcher 到底是什么神奇的东西？
*   听说还有个 AMS 的东西，它是做什么的？
*   Binder 是什么？他是如何进行 IPC 通信的？
*   Activity 生命周期到底是什么时候调用的？被谁调用的？

再回过头来看看这些类，你还迷惑吗？

*   ActivityManagerServices，简称 AMS，服务端对象，负责系统中所有 Activity 的生命周期
*   ActivityThread，App 的真正入口。当开启 App 之后，会调用 main() 开始运行，开启消息循环队列，这就是传说中的 UI 线程或者叫主线程。与 ActivityManagerServices 配合，一起完成 Activity 的管理工作
*   ApplicationThread，用来实现 ActivityManagerService 与 ActivityThread 之间的交互。在 ActivityManagerService 需要管理相关 Application 中的 Activity 的生命周期时，通过 ApplicationThread 的代理对象与 ActivityThread 通讯。
*   ApplicationThreadProxy，是 ApplicationThread 在服务器端的代理，负责和客户端的 ApplicationThread 通讯。AMS 就是通过该代理与 ActivityThread 进行通信的。
*   Instrumentation，每一个应用程序只有一个 Instrumentation 对象，每个 Activity 内都有一个对该对象的引用。Instrumentation 可以理解为应用进程的管家，ActivityThread 要创建或暂停某个 Activity 时，都需要通过 Instrumentation 来进行具体的操作。
*   ActivityStack，Activity 在 AMS 的栈管理，用来记录已经启动的 Activity 的先后关系，状态信息等。通过 ActivityStack 决定是否需要启动新的进程。
*   ActivityRecord，ActivityStack 的管理对象，每个 Activity 在 AMS 对应一个 ActivityRecord，来记录 Activity 的状态以及其他的管理信息。其实就是服务器端的 Activity 对象的映像。
*   TaskRecord，AMS 抽象出来的一个 “任务” 的概念，是记录 ActivityRecord 的栈，一个 “Task” 包含若干个 ActivityRecord。AMS 用 TaskRecord 确保 Activity 启动和退出的顺序。如果你清楚 Activity 的 4 种 launchMode，那么对这个概念应该不陌生。

如果你还感到迷惑的话，就把这篇文章多读几遍吧，信息量可能比较多，需要慢慢消化~
