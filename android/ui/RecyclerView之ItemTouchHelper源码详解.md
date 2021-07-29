ItemTouchHelper是官方提供的一个RecyclerView的帮助类，利用它可以用少量的代码完成itemview的拖拽和滑动。

我们要使用ItemTouchHelper，需要先创建一个ItemTouchHelper.CallBack，该类是ItemTouchHelper的内部类，通过覆写其方法可以实现一些自定义效果。然后创建ItemTouchHelper实例，并与RecyclerView绑定。
```java
ItemTouchHelper.Callback callback = new MyItemTouchHelperCallback( adapter);
ItemTouchHelper itemTouchHelper = new ItemTouchHelper(callback);
itemTouchHelper.attachToRecyclerView(rv);
```
接下来我们从attachToRecyclerView出发，开始查看源码
```java
public void attachToRecyclerView(@Nullable RecyclerView recyclerView) {
    if (mRecyclerView == recyclerView) {
        return; // nothing to do
    }
    if (mRecyclerView != null) {
        destroyCallbacks();
    }
    mRecyclerView = recyclerView;
    if (mRecyclerView != null) {
        final Resources resources = recyclerView.getResources();
        mSwipeEscapeVelocity = resources
                .getDimension(R.dimen.item_touch_helper_swipe_escape_velocity);
        mMaxSwipeVelocity = resources
                .getDimension(R.dimen.item_touch_helper_swipe_escape_max_velocity);
        setupCallbacks();
    }
}
```
首先判断持有的mRecyclerView是否一样，避免重复绑定。接下来先销毁原来的CallBacks，然后通过setupCallBacks重新设置。
```java
private void setupCallbacks() {
    ViewConfiguration vc = ViewConfiguration.get(mRecyclerView.getContext());
    mSlop = vc.getScaledTouchSlop();
    mRecyclerView.addItemDecoration(this);
    mRecyclerView.addOnItemTouchListener(mOnItemTouchListener);
    mRecyclerView.addOnChildAttachStateChangeListener(this);
    initGestureDetector();
}
```
在该方法中，为mRecyclerView添加ItemDecoration、ItemTouchListener、OnChildAttachStateChangeListener三个对象。

ItemTouchHelper自身就继承自ItemDecoration类的，通过覆写onDraw跟onDrawOver方法来实现item的装饰效果。而mOnItemTouchListener是ItemDecoration中的一个final类实例，在声明阶段即初始化。
```java
private final OnItemTouchListener mOnItemTouchListener = new OnItemTouchListener() {
        @Override
        public boolean onInterceptTouchEvent(RecyclerView recyclerView, MotionEvent event) {
            mGestureDetector.onTouchEvent(event);
            ...
        }

        @Override
        public void onTouchEvent(RecyclerView recyclerView, MotionEvent event) {
            ...
        }

        @Override
        public void onRequestDisallowInterceptTouchEvent(boolean disallowIntercept) {
            ...
        }
    };
```
它通过覆写OnItemTouchListener的三个方法，对触控事件进行拦截处理。那么RecyclerView是如何将事件分发到ItemTouchHelper处呢，我们来看一下。
