## 前言

ActivityManagerService是Android系统中一个特别重要的系统服务，也是我们上层APP打交道最多的系统服务之一。ActivityManagerService（以下简称AMS） 主要负责四大组件的启动、切换、调度以及应用进程的管理和调度工作。所有的APP应用都需要与AMS打交道Activity Manager的组成主要分为以下几个部分：
1. 服务代理：由ActivityManagerProxy实现，用于与Server端提供的系统服务进行进程间通信
2. 服务中枢：ActivityManagerNative继承自Binder并实现IActivityManager，它提供了服务接口和Binder接口的相互转化功能，并在内部存储服务代理对像，并提供了getDefault方法返回服务代理
3. Client：由ActivityManager封装一部分服务接口供Client调用。ActivityManager内部通过调用
ActivityManagerNative的getDefault方法，可以得到一个ActivityManagerProxy对像的引用，进而通过
该代理对像调用远程服务的方法
4. Server：由ActivityManagerService实现，提供Server端的系统服务

接下来将从AMS的启动流程、几个方面来学习AMS

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
接下来看forkSystemServer()方法的实现
```java

```
