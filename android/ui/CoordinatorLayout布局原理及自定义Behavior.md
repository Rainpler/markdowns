CoordinatorLayout

### Behavior

1. 处理子控件之间依赖下的交互
2. 处理子控件之间的嵌套滑动
3. 处理子控件的测量与布局
4. 处理子控件的事件拦截与响应
5.
以上四个功能， 都建立于CoordainatorLayout中提供的一个叫做Behavior的“ 插件” 之上。Behavior内部也提供了相应方法来对应这四个不同的功能。

为什么？解耦

#### 处理子控件之间依赖交互
当CoordainatorLayout中子控件depandency的位置、大小等发生改变的时候，那么在CoordainatorLayout内部会通知所有依赖depandency的控件，并调用对应声明的Behavior，告知其依赖的depandency发生改变。那么如何判断依赖(layoutDependsOn)， 接受到通知后如何处理(onDependentViewChanged/onDependentViewRemoved)。 这些都交由Behavior来处理。


#### 处理子控件之间嵌套滑动
CoordinatorLayout实现了NestedScrollingParent2接口。那么当事件（scroll或fling)产生后， 内部实现了NestedScrollingChild接口的子控件会将事件分发给CoordinatorLayout，CoordinatorLayout又会将事件传递给所有的Behavior。然后在Behavior中实现子控件的嵌套滑动。

1. 产生事件(scroll或fling)的控件必须是CoordinatorLayout的直接子View吗？
2. 响应Behavior的控件必须是 CoordinatorLayout的直接子View吗？

#### 处理子控件的测量与布局
CoordainatorLayout主要负责的是子控件之间的交互，内部控件的测量与布局，都非常简单。在特殊的情况下，如子控件需要处理宽高和布局的时候，那么交由Behavior内部的onMeasureChild与onLayoutChild方法来进行处理。

#### 处理子控件的事件拦截与响应
同理对于事件的拦截与处理，如果子控件需要拦截并消耗事件，那么交由给Behavior内部的onInterceptTouchEvent与onTouchEvent方法进行处理。

### 源码
从onAttachedToWindow  开始
```java
@Override
public void onAttachedToWindow() {
    super.onAttachedToWindow();
    resetTouchBehaviors(false);
    if (mNeedsPreDrawListener) {
        if (mOnPreDrawListener == null) {
            mOnPreDrawListener = new OnPreDrawListener();
        }
        final ViewTreeObserver vto = getViewTreeObserver();
        vto.addOnPreDrawListener(mOnPreDrawListener);
    }
    if (mLastInsets == null && ViewCompat.getFitsSystemWindows(this)) {
        // We're set to fitSystemWindows but we haven't had any insets yet...
        // We should request a new dispatch of window insets
        ViewCompat.requestApplyInsets(this);
    }
    mIsAttachedToWindow = true;
}
```
在这里，关键的方法是通过getViewTreeObserver()获取到视图树，然后为其添加onPreDrawListener。

**ViewTreeObserver**通过注册一个观察者来监听视图树，当视图树的布局、视图树的焦点、视图树将要绘制、视图树滚动等发生改变时，ViewTreeObserver都会收到通知， ViewTreeObserver不能被实例化，可以用`View.getViewTreeObserver()`来获得View的生命周期。

| 内部接口类                   | 备注                                                     |
| ---------------------------- | -------------------------------------------------------- |
| onPreDrawListener            | 当视图树将要被绘制时                                     |
| onGlobalLayoutListener       | 当视图树的布局发生改变或者View在视图树的可变状态发生改变 |
| onGlobalFocusChangerListener | 当一个视图树的焦点发生变化                               |
| onScrollChangeListener       | 当视图树的一些组件发生滚动时                             |
| onTouchModeChangeListener    | 当视图树的触摸模式发生改变时                             |

当绘制将要开始的时候，会调用ViewTreeObserver.dispatchOnPreDraw()方法，通知观察者绘制即将开始，然后进一步调用OnPreDrawListener.onPreDraw()。

```java
class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
    @Override
    public boolean onPreDraw() {
        onChildViewsChanged(EVENT_PRE_DRAW);
        return true;
    }
}
```
然后调用onChildViewsChanged()，并传入事件类型为EVENT_PRE_DRAW，除此之外还有EVENT_NESTED_SCROLL标记滑动事件，EVENT_VIEW_REMOVED标记移除view事件。
```java
final void onChildViewsChanged(@DispatchChangeEvent final int type) {
    final int layoutDirection = ViewCompat.getLayoutDirection(this);
    final int childCount = mDependencySortedChildren.size();
    final Rect inset = acquireTempRect();
    final Rect drawRect = acquireTempRect();
    final Rect lastDrawRect = acquireTempRect();

    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        if (type == EVENT_PRE_DRAW && child.getVisibility() == View.GONE) {
            // Do not try to update GONE child views in pre draw updates.
            continue;
        }

        // 在定位之前检查子视图
        for (int j = 0; j < i; j++) {
            final View checkChild = mDependencySortedChildren.get(j);
            if (lp.mAnchorDirectChild == checkChild) {
                offsetChildToAnchor(child, layoutDirection);
            }
        }

        // 获取当前child的Rect
        getChildRect(child, true, drawRect);

        // 累计插入size
        if (lp.insetEdge != Gravity.NO_GRAVITY && !drawRect.isEmpty()) {
            final int absInsetEdge = GravityCompat.getAbsoluteGravity(
                    lp.insetEdge, layoutDirection);
            switch (absInsetEdge & Gravity.VERTICAL_GRAVITY_MASK) {
                case Gravity.TOP:
                    inset.top = Math.max(inset.top, drawRect.bottom);
                    break;
                case Gravity.BOTTOM:
                    inset.bottom = Math.max(inset.bottom, getHeight() - drawRect.top);
                    break;
            }
            switch (absInsetEdge & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.LEFT:
                    inset.left = Math.max(inset.left, drawRect.right);
                    break;
                case Gravity.RIGHT:
                    inset.right = Math.max(inset.right, getWidth() - drawRect.left);
                    break;
            }
        }

        // 如过有必要，减少插入的inset值
        if (lp.dodgeInsetEdges != Gravity.NO_GRAVITY && child.getVisibility() == View.VISIBLE) {
            offsetChildByInset(child, inset, layoutDirection);
        }

        if (type != EVENT_VIEW_REMOVED) {
            // Did it change? if not continue
            getLastChildRect(child, lastDrawRect);
            if (lastDrawRect.equals(drawRect)) {
                continue;
            }
            recordLastChildRect(child, drawRect);
        }

        // 如果有变化，更新与child有依赖交互的view
        for (int j = i + 1; j < childCount; j++) {
            final View checkChild = mDependencySortedChildren.get(j);
            final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
            final Behavior b = checkLp.getBehavior();

            if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                    // If this is from a pre-draw and we have already been changed
                    // from a nested scroll, skip the dispatch and reset the flag
                    checkLp.resetChangedAfterNestedScroll();
                    continue;
                }

                final boolean handled;
                switch (type) {
                    case EVENT_VIEW_REMOVED:
                        // EVENT_VIEW_REMOVED means that we need to dispatch
                        // onDependentViewRemoved() instead
                        b.onDependentViewRemoved(this, checkChild, child);
                        handled = true;
                        break;
                    default:
                        // Otherwise we dispatch onDependentViewChanged()
                        handled = b.onDependentViewChanged(this, checkChild, child);
                        break;
                }

                if (type == EVENT_NESTED_SCROLL) {
                    // If this is from a nested scroll, set the flag so that we may skip
                    // any resulting onPreDraw dispatch (if needed)
                    checkLp.setChangedAfterNestedScroll(handled);
                }
            }
        }
    }

    releaseTempRect(inset);
    releaseTempRect(drawRect);
    releaseTempRect(lastDrawRect);
}
```

 super.setOnHierarchyChangeListener(new HierarchyChangeListener());


 ```java
  public interface OnHierarchyChangeListener {

    void onChildViewAdded(View parent, View child);
    void onChildViewRemoved(View parent, View child);
}
 ```

 mDependencySortedChildren 多此一举
 getChildAt()

图的数据结构
private final DirectedAcyclicGraph<View> mChildDag = new DirectedAcyclicGraph<>();
保存childview 与dependency的依赖关系，然后保存到mDependencySortedChildren中去。

邻接表 有向无环图
初始化

Measure

prepareChildren
```java
private void prepareChildren() {
    mDependencySortedChildren.clear();
    mChildDag.clear();

    for (int i = 0, count = getChildCount(); i < count; i++) {
        final View view = getChildAt(i);

        final LayoutParams lp = getResolvedLayoutParams(view);
        lp.findAnchorView(this, view);

        mChildDag.addNode(view);

        // Now iterate again over the other children, adding any dependencies to the graph
        for (int j = 0; j < count; j++) {
            if (j == i) {
                continue;
            }
            final View other = getChildAt(j);
            if (lp.dependsOn(this, view, other)) {
                if (!mChildDag.contains(other)) {
                    // Make sure that the other node is added
                    mChildDag.addNode(other);
                }
                // Now add the dependency to the graph
                mChildDag.addEdge(other, view);
            }
        }
    }

    // Finally add the sorted graph list to our list
    mDependencySortedChildren.addAll(mChildDag.getSortedList());
    // We also need to reverse the result since we want the start of the list to contain
    // Views which have no dependencies, then dependent views after that
    Collections.reverse(mDependencySortedChildren);
}
```
### 自定义Behavior
