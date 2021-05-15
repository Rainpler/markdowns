ActivityThread
performLaunchActivity()

Window window=null

Activity

attach()

mWindow = new PhoneWindow(this, window, activityConfigCallback);


setContentView 在PhoneWindow里面实现


installDecor();
mDecor = generateDecor(-1);
而DecorView 实际上是继承自FrameLayout

接下来通过

mContentParent = generateLayout(mDecor); 生成布局

我们深入进去 找到 // Inflate the window decor.注释
这里就是解析顶级布局资源的地方

mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

在这里通过inflater.inflate加载到root里去
final View root = inflater.inflate(layoutResource, null);

然后通过addView 将布局添加到DecorView中去

接下来，通过定位到了布局中的 com.android.internal.R.id.content
ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);

得到contentParent并返回。

我们回到PhoneWindow的setContentView方法，在完成installDecor()后，紧接着通过``mLayoutInflater.inflate(layoutResID, mContentParent)``将资源文件加载到mContentParent中

我们来关注这个inflate方法，inflate方法会一层层调用，最后调用到``inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)``方法中，在这里会涉及到一个attachToRoot的参数。如果该参数是false的话，就会在xml文件中获取到布局属性，如果传入为true的话，就需要在代码中手动去设置布局的属性。
```java
if (!attachToRoot) {
      // Set the layout params for temp if we are not
      // attaching. (If we are, we use addView, below)
      temp.setLayoutParams(params);
  }
```
那么它不仅仅是为了扩展性那么简单，我们可以从``final View temp = createViewFromTag(root, name, inflaterContext, attrs)``创建view的过程去理解为什么这样设计的。

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
