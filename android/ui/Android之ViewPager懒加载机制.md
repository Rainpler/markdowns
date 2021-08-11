我们在

### ViewPager 页面缓存

在使用ViewPager的时候，我们通常会使用setOffscreenPageLimit(int limit)方法来缓存界面，通过传入limit变量，控制缓存的页面数目。

实际上，传入的limit值并不是说缓存页面数目为1，而是以当前选中Fragment为中心，前后各缓存的页面数目。因此，setOffscreenPageLimit(1)实际缓存的最大页面数目为2个，limit=2 的话，实际缓存的最大页面数目为4个。如果当前Fragment为第一个或者最后一个，则只会缓存左边或者右边的页面。我们来看一下ViewPager是如何实现页面的缓存的。
```java
public void setOffscreenPageLimit(int limit) {
    if (limit < DEFAULT_OFFSCREEN_PAGES) {
        limit = DEFAULT_OFFSCREEN_PAGES;
    }
    if (limit != mOffscreenPageLimit) {
        mOffscreenPageLimit = limit;
        populate();
    }
}
```
从该方法可以看出，limit传入的有效值必须是≥1的，否则也会被设置为默认值1，如果limit不等于上次赋值，则赋值给mOffscreenPageLimit，并进一步调用populate()方法。
```java
void populate() {
    populate(mCurItem);
}

void populate(int newCurrentItem) {
    //更新mCurItem为当前选中位置
    if (mCurItem != newCurrentItem) {
        oldCurInfo = infoForPosition(mCurItem);
        mCurItem = newCurrentItem;
    }
    //略 ...

    //开始更新
    mAdapter.startUpdate(this);

    //0.确定前后缓存位置
    //即[startPos,endPos] = [mCurItem - pageLimit, mCurItem + pageLimit]
    final int pageLimit = mOffscreenPageLimit;
    final int startPos = Math.max(0, mCurItem - pageLimit);
    final int N = mAdapter.getCount();
    final int endPos = Math.min(N - 1, mCurItem + pageLimit);


    //1. 找到第一个等于mCurItem的位置
    int curIndex = -1;
    ItemInfo curItem = null;
    for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
        final ItemInfo ii = mItems.get(curIndex);
        if (ii.position >= mCurItem) {

            if (ii.position == mCurItem) curItem = ii;
            break;
        }
    }

    // mItems 中可能会找不到 curItem，需要 addNewItem
    // addNewItem 函数中会调用mAdapter.instantiateItem(this, position)创建新的视图
    if (curItem == null && N > 0) {
        curItem = addNewItem(mCurItem, curIndex);
    }


    //2.决定保留的页面范围，即[startPos，endPos]
    if (curItem != null) {
        //确定左边界
        float extraWidthLeft = 0.f;
        int itemIndex = curIndex - 1;
        ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
        //获取子View的可用宽大小，即viewPager测量宽度-内边距
        final int clientWidth = getClientWidth();
       // 计算左侧预加载视图宽度
        final float leftWidthNeeded = clientWidth <= 0 ? 0 :
                2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
        //逆序遍历
        for (int pos = mCurItem - 1; pos >= 0; pos--) {
            //如果在边界之外
            if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
                if (ii == null) {
                    break;
                }
                //移除该页面元素,销毁视图
                if (pos == ii.position && !ii.scrolling) {
                    mItems.remove(itemIndex);
                    mAdapter.destroyItem(this, pos, ii.object);

                    itemIndex--;
                    curIndex--;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                }
            } else if (ii != null && pos == ii.position) {
              // 若该左侧元素在内存中，则更新记录
                extraWidthLeft += ii.widthFactor;
                itemIndex--;
                ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            } else {
                // 若该左侧元素不在内存中，则重新添加，再一次来到了addNewItem
                ii = addNewItem(pos, itemIndex + 1);
                extraWidthLeft += ii.widthFactor;
                curIndex++;
                ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            }
        }

        //确定右边界
        //...与确定左边界大体一致


        //计算页面偏移
        calculatePageOffsets(curItem, curIndex, oldCurInfo);

        //3.设置当前选中item
        mAdapter.setPrimaryItem(this, mCurItem, curItem.object);
    }


    //4. 结束更新
    mAdapter.finishUpdate(this);

    // 下面两部分分别是 LayoutParams 和 Focus 处理
    // 遍历子视图，若宽度不合法则重绘
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View child = getChildAt(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        lp.childIndex = i;
        if (!lp.isDecor && lp.widthFactor == 0.f) {
            // 0 means requery the adapter for this, it doesn't have a valid width.
            //没有有效的宽度，则获取内存中保存的信息给子视图的LayoutParams
            final ItemInfo ii = infoForChild(child);
            if (ii != null) {
                lp.widthFactor = ii.widthFactor;
                lp.position = ii.position;
            }
        }
    }
    //重新将子绘图顺序排序
    sortChildDrawingOrder();

    //如果焦点在ViewPager上
    if (hasFocus()) {
        //找到焦点View，如果存在则获取它的信息
        View currentFocused = findFocus();
        ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
        //如果没找到，或者找到的焦点view不是当前位置，则遍历元素，如果找到对应元素则请求焦点
        if (ii == null || ii.position != mCurItem) {
            for (int i = 0; i < getChildCount(); i++) {
                View child = getChildAt(i);
                ii = infoForChild(child);
                if (ii != null && ii.position == mCurItem) {
                    //找到view，请求焦点
                    if (child.requestFocus(View.FOCUS_FORWARD)) {
                        break;
                    }
                }
            }
        }
    }
}
```
从代码来看，整个populate()有几个重要的步骤，首先先通过pageLimit确定了前后缓存的范围，然后决定要保留的页面，并将范围外的页面移除，最后设置当前选中的item，并结束更新事务。

ItemInfo，后面会用到它，它是ViewPager的一个内部类，包含了一个页面的基本信息，在调用Adapter的instantiateItem方法时，在ViewPager内部就会创建这个对象，但它不包含view，结构如下：
```java
static class ItemInfo {
    Object object;      // 为adapter中instantiateItem方法返回的对象
    int position;       // 页面position
    boolean scrolling;  // 是否正在滑动
    float widthFactor;  // 当前页面宽度和ViewPager宽度的比例
    float offset;       // 当前页面在所有已加载的页面中的索引
}
```
同时，在这里还涉及了adapter的几个基本方法，几乎是整个adapter的生命周期。
- **startUpdate()**  　开始更新事务
- **instantiateItem()**　 创建页面视图
- **destroyItem()**  　销毁页面视图
- **setPrimaryItem()**　设置当前选中页面
- **finishUpdate()** 　结束更新事务

![](../../res/adapter示意图.jpg)

但是预加载也会带来相应的问题，例如当预加载了P3，P4时，虽然它们还未可见，但是仍会请求服务器大量数据导致卡顿。为了解决这个问题，我们可以使用懒加载机制。

### ViewPager懒加载机制
懒加载，其实也就是延迟加载,就是等到该页面的UI展示给用户时，再加载该页面的数据(从网络、 数据库等)，而不是依靠ViewPager预加载机制提前加载两三个， 甚至更多页面的数据。这样可以提高所属Activity的初始化速度，也可以为用户节省流量。而这种懒加载的方式也已经/正在被诸多APP所采用。

为了学习ViewPager的懒加载机制，首先我们要了解ViewPager的源码。

#### ViewPager的适配器原理

>ViewPager要展示数据，必须绑定Adapter适配器，通过适配器完成数据的准备、展示和销毁工作。

Android为我们提供了PagerAdapter的基本抽象类，由此衍生出FragmentPagerAdapter与FragmentStatePagerAdapter两个实现类。我们主要来看
```java
public abstract class PagerAdapter {
    private final DataSetObservable mObservable = new DataSetObservable();
    private DataSetObserver mViewPagerObserver;

    public static final int POSITION_UNCHANGED = -1;
    public static final int POSITION_NONE = -2;


    public abstract int getCount();

    //准备适配
    public void startUpdate(@NonNull ViewGroup container) {
        startUpdate((View) container);
    }

    //准备适配的数据
    @NonNull
    public Object instantiateItem(@NonNull ViewGroup container, int position) {
        return instantiateItem((View) container, position);
    }

    //销毁适配的item数据
    public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        destroyItem((View) container, position, object);
    }
    //设置当前显示的item数据
    public void setPrimaryItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        setPrimaryItem((View) container, position, object);
    }

    //结束适配
    public void finishUpdate(@NonNull ViewGroup container) {
        finishUpdate((View) container);
    }

}
```
我们在分析populate函数的时候，就会调用到adapter的几个重要方法。我们来FragmentPagerAdapter中的实现。
```java
@Override
public void startUpdate(@NonNull ViewGroup container) {
    if (container.getId() == View.NO_ID) {
        throw new IllegalStateException("ViewPager with adapter " + this
                + " requires a view id");
    }
}
```
开始适配阶段，实现中只做了有效性的判断，并没有做其他处理。接下来是instantiateItem函数的实现。
```java
public Object instantiateItem(@NonNull ViewGroup container, int position) {
    //初始化mCurTransaction
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }


    final long itemId = getItemId(position);

    // Do we already have this fragment?
    String name = makeFragmentName(container.getId(), itemId);
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
            mCurTransaction.setMaxLifecycle(fragment, Lifecycle.State.STARTED);
        } else {
            fragment.setUserVisibleHint(false);
        }
    }
    return fragment;
}
```



### ViewPager1与ViewPager2的差异化
