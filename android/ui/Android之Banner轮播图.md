我们在

### ViewPager 缓存页面
#### setOffscreenPageLimit
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
同时，在这里还涉及了adapter的几个基本方法。当开始更新的时候，先调用mAdapter.startUpdate(this)
```java

但是预加载也会带来相应的问题，当预加载了P3，P4时，虽然它们还未可见，但是仍会请求服务器大量数据导致卡顿。我们可以使用懒加载解决预加载带来的问题
### ViewPager懒加载机制
### ViewPager1与ViewPager2的差异化
