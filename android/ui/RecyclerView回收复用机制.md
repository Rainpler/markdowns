我们都知道，RecyclerView具有回收复用的机制，那么我们对这个机制有足够了解了吗，在这里先提出几个问题：
- 回收什么？复用什么？
- 回收到哪里去？从哪里获得复用？
- 什么时候回收？什么时候复用？

既然是回收复用，那么静止中的RecyclerView就不存在这一操作，必然是在滑动的时候才会涉及。当我们在Adapter下面两个方法进行日志打印的时候，可以发现onCreateViewHolder方法打印到一定次数的时候，就不再打印了。那么是不是说明，回收复用的其实是ViewHolder对象呢？
```java
@Override
public CustomViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    Log.e(TAG, "onCreateViewHolder: " + getItemCount());
    return new CustomViewHolder(view);
}

@Override
public void onBindViewHolder(CustomViewHolder holder, int position) {
    Log.e(TAG, "onBindViewHolder: " + position);
}
```

那么我们该如何去探究RecyclerView的回收复用机制，既然当滑动的时候才会有回收复用的说法，我们可以从MOVE事件开始着手。查看onTouchEvent()的ACTION_MOVE事件，这里调用了RecyclerView的scrollByInternal方法：
```java
 boolean scrollByInternal(int x, int y, MotionEvent ev) {
        // ...
        if (mAdapter != null) {
            mReusableIntPair[0] = 0;
            mReusableIntPair[1] = 0;
            scrollStep(x, y, mReusableIntPair);
            consumedX = mReusableIntPair[0];
            consumedY = mReusableIntPair[1];
            unconsumedX = x - consumedX;
            unconsumedY = y - consumedY;
        }
        // 代码省略...

        return consumedNestedScroll || consumedX != 0 || consumedY != 0;
    }
```
在这里会根据scrollStep()的结果，得到x、y方向的消费值，最后返回事件消费的结果。我们来进一步看一下这个方法：
```java
void scrollStep(int dx, int dy, @Nullable int[] consumed) {
        //...

        int consumedX = 0;
        int consumedY = 0;
        if (dx != 0) {
            consumedX = mLayout.scrollHorizontallyBy(dx, mRecycler, mState);
        }
        if (dy != 0) {
            consumedY = mLayout.scrollVerticallyBy(dy, mRecycler, mState);
        }

       //...

        if (consumed != null) {
            consumed[0] = consumedX;
            consumed[1] = consumedY;
        }
    }
```
这里的关键
