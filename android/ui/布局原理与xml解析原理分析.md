## 布局原理
我们要学习布局原理，就是要明白UI线程是怎么工作的，而在Android中，ActivityThread就是我们常说的主线程或UI线程，ActivityThread的main方法是整个APP的入口。因为我们学习的只是布局的原理，所以我们的代码只关注跟UI相关的代码。
#### ActivityThread
首先来看``performLaunchActivity()``方法，这是activity启动的核心方法。
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ......
    Window window = null;
    if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
        window = r.mPendingRemoveWindow;
        r.mPendingRemoveWindow = null;
        r.mPendingRemoveWindowManager = null;
    }
    appContext.setOuterContext(activity);
    activity.attach(appContext, this, getInstrumentation(), r.token,
            r.ident, app, r.intent, r.activityInfo, title, r.parent,
            r.embeddedID, r.lastNonConfigurationInstances, config,
            r.referrer, r.voiceInteractor, window, r.configCallback);
    ......
}
```
找到window的变量，这是Android视窗的顶层窗口，初始值为null，然后进入``activity.attach()``方法找到
```java
mWindow = new PhoneWindow(this, window, activityConfigCallback);
```
在这里，对mWindow进行了实例化，通过查看Window类的注释可以知道，PhoneWindow是Window类的唯一抽象实现。Activity的setContentView()方法最终实现是调用PhoneWindow中的setContentView()方法的。
```java
@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```
如果mContentParent为空，则会进入``installDecor()``方法。Decor是装饰的意思，在这个方法里，通过
```java
mDecor = generateDecor(-1);
```
生成一个继承自FrameLayout的DecorView，DecorView是整个Window界面最顶层的View。接下来通过
```java
mContentParent = generateLayout(mDecor);
```
生成布局，我们深入进去找到 ``// Inflate the window decor``注释，这里就是解析顶级布局资源的地方，找到
```java
mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
```
在这里通过``inflater.inflate()``加载到root里去
```java
final View root = inflater.inflate(layoutResource, null);
```
然后通过addView将布局添加到DecorView中,去接下来通过定位到了布局中的 ``com.android.internal.R.id.content``，得到contentParent并返回。
```java
ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
```

我们回到PhoneWindow的setContentView方法，在完成``installDecor()``后，紧接着通过
```java
mLayoutInflater.inflate(layoutResID, mContentParent)
```
将资源文件加载到mContentParent中，我们来关注这个inflate方法，inflate方法会一层层调用，最后调用到
```java
inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)
```
在这里会涉及到一个attachToRoot的参数。如果该参数是false的话，就会在xml文件中获取到布局属性，如果传入为true的话，就需要在代码中手动去设置布局的属性。
```java
if (!attachToRoot) {
      // Set the layout params for temp if we are not
      // attaching. (If we are, we use addView, below)
      temp.setLayoutParams(params);
  }
```
我们来看``final View temp = createViewFromTag(root, name, inflaterContext, attrs)``创建view的过程。

在初始化的时候，如果factory不为空，则view的创建会被拦截，调用factory.onCreateView()方法。否则view的创建会走下面代码段：
```java
if (view == null) {
    final Object lastContext = mConstructorArgs[0];
    mConstructorArgs[0] = context;
    try {
        if (-1 == name.indexOf('.')) {
            view = onCreateView(parent, name, attrs);
        } else {
            view = createView(name, null, attrs);
        }
    } finally {
        mConstructorArgs[0] = lastContext;
    }
}
```
如果是xml的则会直接走createView()，如果是view的话，则会进入onCreateView()，最终也会进入createView()。深入该函数继续看，在这里会有一个sConstructorMap的变量，用于存放所有的构造方法，首先会检查是否存在name的构造方法缓存，如果不存在的话，通过反射获得其构造方法并存入sConstructorMap中。
```java
 if (constructor == null) {
      // Class not found in the cache, see if it's real, and try to add it
      clazz = mContext.getClassLoader().loadClass(
              prefix != null ? (prefix + name) : name).asSubclass(View.class);

      if (mFilter != null && clazz != null) {
          boolean allowed = mFilter.onLoadClass(clazz);
          if (!allowed) {
              failNotAllowed(name, prefix, attrs);
          }
      }
      constructor = clazz.getConstructor(mConstructorSignature);
      constructor.setAccessible(true);
      sConstructorMap.put(name, constructor);
  }
```
然后调用``constructor.newInstance()``实例化view。
```java
final View view = constructor.newInstance(args);
return view;
```
前面说们说了，如果工厂类不为空，则会拦截View的创建过程，那么现在我们来看一下这些工厂类到底是什么。
```java
 public interface Factory {

    public View onCreateView(String name, Context context, AttributeSet attrs);
}

public interface Factory2 extends Factory {

    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}
```
两者都是要onCreateView函数，而Factory2比Factory多了一个父亲的参数。阅读到这里我们可以发现，如果我们要干预view的创建过程，一个是重写LayoutInflater方法，另一个可以实现自己的工厂类，拦截view的创建。

## xml解析原理
我们将编译得到的apk包拖至AS中，可以看到apk中的文件，其中有一个resources.arsc二进制文件，它是应用包中的资源映射 。

还是回到ActivityThread.java，其中有一个``handleBindApplication()``的方法，注释有说
Register the UI Thread as a sensitive thread to the runtime，意思就是将ui线程注册为运行时的敏感线程。

首先，需要先创建一个Instrumentation仪表类实例，查看其提供的方法，比如
- callActivityOnCreate
- callApplicationOnCreate
- newActivity
-  callActivityOnNewIntent

等基本上在application和activity的所有生命周期调用中，都会先调用instrumentation的相应方法。并且针对应用内的所有activity都生效。为程序员提供了一个强大的能力，有更多的可能性进入android app框架执行流程。

```java
if (data.instrumentationName !=null) {
...
    java.lang.ClassLoader cl = instrContext.getClassLoader();
    mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
...
} else {
    mInstrumentation =newInstrumentation();
}
```

然后会调用data.info.makeApplication()方法创建一个Application对象，info是LoadedApk类的一个实例，包含了当前加载的apk的本地状态。

```java
app = data.info.makeApplication(data.restrictedBackupMode, null);
mInitialApplication = app;
```
在方法中，先通过createAppContext()创建上下文，然后调用mInstrumentation的newApplication()方法得到application的实例对象。
```java
ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
app = mActivityThread.mInstrumentation.newApplication(
      cl, appClass, appContext);
appContext.setOuterContext(app);
```
然后创建ContextImpl实例，接下来从apk中获取资源并设置资源。
```java
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
            null);
    context.setResources(packageInfo.getResources());
    return context;
}
```
在getResources()方法中，通过获取ResourcesManager的实例，并调用它的getResources()方法。
```java
public Resources getResources() {
        if (mResources == null) {
            final String[] splitPaths;
            try {
                splitPaths = getSplitPaths(null);
            } catch (NameNotFoundException e) {
                // This should never fail.
                throw new AssertionError("null split not found");
            }

            mResources = ResourcesManager.getInstance().getResources(null, mResDir,
                    splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,
                    Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),
                    getClassLoader());
        }
        return mResources;
    }
```
来看ResourcesManager的getResources()实现:
```java
try {
    Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "ResourcesManager#getResources");
    final ResourcesKey key = new ResourcesKey(
            resDir,
            splitResDirs,
            overlayDirs,
            libDirs,
            displayId,
            overrideConfig != null ? new Configuration(overrideConfig) : null, // Copy
            compatInfo);
    classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();
    return getOrCreateResources(activityToken, key, classLoader);
} finally {
    Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
}
```
新建一个ResourcesKey，然后将传入的classLoader一起，传给getOrcCreateResources()方法，传入的activityToken是从上一个方法传入的null。

获取资源实际是通过ResourcesImpl来完成，来看getOrcCreateResources()方法中，由于传入的activityToken为null，因此会进入else，首先通过findResourcesImplForKeyLocked()查找缓存，是否存在所需的ResourcesImpl。
```java
private @Nullable Resources getOrCreateResources(@Nullable IBinder activityToken,
            @NonNull ResourcesKey key, @NonNull ClassLoader classLoader) {
        synchronized (this) {
            ...
            if (activityToken != null) {
                ...

            } else {
                ArrayUtils.unstableRemoveIf(mResourceReferences, sEmptyReferencePredicate);

                ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
                if (resourcesImpl != null) {
                    return getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
                }

            }

            ResourcesImpl resourcesImpl = createResourcesImpl(key);
            if (resourcesImpl == null) {
                return null;
            }

            mResourceImpls.put(key, new WeakReference<>(resourcesImpl));

            final Resources resources;
            if (activityToken != null) {
                resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                        resourcesImpl, key.mCompatInfo);
            } else {
                resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
            }
            return resources;
        }
```
第一次由于ResourcesImpl没有初始化，所以缓存不存在，所以会通过createResourcesImpl()去创建。
```java
private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) {
    final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
    daj.setCompatibilityInfo(key.mCompatInfo);

    final AssetManager assets = createAssetManager(key);
    if (assets == null) {
        return null;
    }

    final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);
    final Configuration config = generateConfig(key, dm);
    final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);

    if (DEBUG) {
        Slog.d(TAG, "- creating impl=" + impl + " with key: " + key);
    }
    return impl;
}
```
在方法内，通过createAssetManager(key)创建一个AssetManager的实例去加载资源，而它的实现中又是通过调用``assets.addAssetPath(key.mResDir)``该方法内最后调用native方法完成资源加载。

当ResourcesImpl创建完成后，又会接着调用getOrCreateResourcesLocked()去初始化Resources对象实例。最终将包装好的resources作为资源类返回，资源的信息都被存储在Resources中的ResourcesImpl中的Asset对象中。

所以实际上，Resource和ResourceImpl都是包装的壳，最终资源的读取都是通过assets来进行的。

此外，还有这些重要的API需要我们了解：
```java
/*package*/ native final int getResourceIdentifier(String name,String defType,String defPackage);
/*package*/ native final String getResourceName(int resid);
/*package*/ native final String getResourcePackageName(int resid);
/*package*/ native final String getResourceTypeName(int resid);
/*package*/ native final String getResourceEntryName(int resid);
```
  mInstrumentation.callApplicationOnCreate(app);
