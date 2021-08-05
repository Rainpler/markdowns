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

RecyclerView继承自ViewGroup，但是并没有重写dispatchTouchEvent方法，所以我们来看它的onInterceptTouchEvent方法。
```java
 @Override
public boolean onInterceptTouchEvent(MotionEvent e) {
    // ...
    mInterceptingOnItemTouchListener = null;
    if (findInterceptingOnItemTouchListener(e)) {
        cancelScroll();
        return true;
    }
    // ...
  }
```
这个方法如果分发给OnItemTouchListener则返回true，则RecycleView将不向下执行直接拦截。而ItemTouchHelper正好实现了OnItemTouchListener接口。
```java
private boolean findInterceptingOnItemTouchListener(MotionEvent e) {
    int action = e.getAction();
    final int listenerCount = mOnItemTouchListeners.size();
    for (int i = 0; i < listenerCount; i++) {
        final OnItemTouchListener listener = mOnItemTouchListeners.get(i);
        if (listener.onInterceptTouchEvent(this, e) && action != MotionEvent.ACTION_CANCEL) {
            mInterceptingOnItemTouchListener = listener;
            return true;
        }
    }
    return false;
}
```
那么我们来看mOnItemTouchListener中onInterceptTouchEvent方法的具体实现。
```java
@Override
public boolean onInterceptTouchEvent(@NonNull RecyclerView recyclerView,@NonNull MotionEvent event) {
    mGestureDetector.onTouchEvent(event);
    final int action = event.getActionMasked();
    //ACTION_DOWN
    if (action == MotionEvent.ACTION_DOWN) {
        mActivePointerId = event.getPointerId(0);
        mInitialTouchX = event.getX();
        mInitialTouchY = event.getY();
        obtainVelocityTracker();
        if (mSelected == null) {
            final RecoverAnimation animation = findAnimation(event);
            if (animation != null) {
                mInitialTouchX -= animation.mX;
                mInitialTouchY -= animation.mY;
                endRecoverAnimation(animation.mViewHolder, true);
                // 结束动画时清除mPendingCleanup保存的itemView的集合
                if (mPendingCleanup.remove(animation.mViewHolder.itemView)) {
                    mCallback.clearView(mRecyclerView, animation.mViewHolder);
                }
                select(animation.mViewHolder, animation.mActionState);
                // 更新选中的itemView的距离
                updateDxDy(event, mSelectedFlags, 0);
            }
        }
    }
     // ACTION_CANCEL  ACTION_UP
    else if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
        mActivePointerId = ACTIVE_POINTER_ID_NONE;
        select(null, ACTION_STATE_IDLE);
    } else if (mActivePointerId != ACTIVE_POINTER_ID_NONE) {
        final int index = event.findPointerIndex(mActivePointerId);
        if (index >= 0) {
            // 检查是否可滑动
            checkSelectForSwipe(action, event, index);
        }
    }
    //保存事件到速度追踪器
    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(event);
    }
    return mSelected != null;
}
```
这个方法的代码有点多，当事件是MotionEvent.ACTION_DOWN的时候，记录初始触摸坐标，并初始化事件速度追踪类（VelocityTracker），查找集合中有没有和当前ItemView（触摸事件的x，y在itemView位置的内部）绑定的动画类RecoverAnimation ，如果有结束动画，重新确定手指选择的是哪个itemView。

手指抬起或发生意外取消的时候，将mActivePointerId设置为ACTIVE_POINTER_ID_NONE，并调用select(null, ACTION_STATE_IDLE)。当mActivePointerId != ACTIVE_POINTER_ID_NONE，也就是触摸的手指事件有效的情况下调用checkSelectForSwipe()，检查是否选择一个view来滑动。

在这里，最重要的就是select和checkSelectForSwipe这两个方法。第1次走拦截事件的时候，由于动画是null的，那么最后会走到checkSelectForSwipe()，接下来看看这个方法。
```java
void checkSelectForSwipe(int action, MotionEvent motionEvent, int pointerIndex) {
        // 当已经有选中的View时，或事件不等于滑动事件，或者mActionState=正在被拖动的状态，
        // 或者mCallback不支持滑动直接返回false
        if (mSelected != null || action != MotionEvent.ACTION_MOVE
            || mActionState == ACTION_STATE_DRAG || !mCallback.isItemViewSwipeEnabled()) {
            return;
        }
        // 如果当前mRecyclerView的状态是正在拖动的状态返回false
        if (mRecyclerView.getScrollState() == RecyclerView.SCROLL_STATE_DRAGGING) {
            return;
        }

        final ViewHolder vh = findSwipedView(motionEvent);
        if (vh == null) {
            return;
        }
        // 得到移动状态
        final int movementFlags = mCallback.getAbsoluteMovementFlags(mRecyclerView, vh);

        // 通过计算得到滑动多的状态参数值
        final int swipeFlags = (movementFlags & ACTION_MODE_SWIPE_MASK)
                >> (DIRECTION_FLAG_COUNT * ACTION_STATE_SWIPE);

        if (swipeFlags == 0) {
            return;
        }

        // mDx and mDy are only set in allowed directions. We use custom x/y here instead of
        // updateDxDy to avoid swiping if user moves more in the other direction
        final float x = motionEvent.getX(pointerIndex);
        final float y = motionEvent.getY(pointerIndex);

        // 计算移动距离
        final float dx = x - mInitialTouchX;
        final float dy = y - mInitialTouchY;
        // swipe target is chose w/o applying flags so it does not really check if swiping in that
        // direction is allowed. This why here, we use mDx mDy to check slope value again.
        final float absDx = Math.abs(dx);
        final float absDy = Math.abs(dy);

        if (absDx < mSlop && absDy < mSlop) {
            return;
        }
        if (absDx > absDy) {
          // dx小于0表示手指向左滑动,如果设置的swipeFlags不包括item的话，不做操作
            if (dx < 0 && (swipeFlags & LEFT) == 0) {
                return;
            }
            if (dx > 0 && (swipeFlags & RIGHT) == 0) {
                return;
            }
        } else {
            if (dy < 0 && (swipeFlags & UP) == 0) {
                return;
            }
            if (dy > 0 && (swipeFlags & DOWN) == 0) {
                return;
            }
        }
        // 当前选中itemView的偏移量归零
        mDx = mDy = 0f;
        mActivePointerId = motionEvent.getPointerId(0);
        // 满足滑动,设置滑动状态ACTION_STATE_SWIPE
        select(vh, ACTION_STATE_SWIPE);
    }
```
在该方法中，首先会先进行一系列的条件判断，选中的itemView不为空，事件不为滑动事件，不是拖拽事件，mCallback支持滑动。然后根据event的x,y判断触点在哪个itemview中，并获取得到ViewHolder，然后通过getAbsoluteMovementFlags方法获取支持的MOVE事件类型。

接下来计算滑动是那一个方向的，如果滑动不包括left，right，down，up的话直接返回false，以下判断都满足后清空偏移距离，并调用select(vh, ACTION_STATE_SWIPE)，此时传入的状态是滑动状态。接下来看看这个方法。
```java
void select(@Nullable ViewHolder selected, int actionState) {
    if (selected == mSelected && actionState == mActionState) {
        return;
    }
    mDragScrollStartTimeInMs = Long.MIN_VALUE;
    final int prevActionState = mActionState;
    // prevent duplicate animations
    endRecoverAnimation(selected, true);
    mActionState = actionState;
    if (actionState == ACTION_STATE_DRAG) {
        if (selected == null) {
            throw new IllegalArgumentException("Must pass a ViewHolder when dragging");
        }
        mOverdrawChild = selected.itemView;
        addChildDrawingOrderCallback();
    }
    int actionStateMask = (1 << (DIRECTION_FLAG_COUNT + DIRECTION_FLAG_COUNT * actionState)) - 1;
    boolean preventLayout = false;

    if (mSelected != null) {
        final ViewHolder prevSelected = mSelected;
        if (prevSelected.itemView.getParent() != null) {
            final int swipeDir = prevActionState == ACTION_STATE_DRAG ? 0
                    : swipeIfNecessary(prevSelected);
            releaseVelocityTracker();
            //计算targetXTranslateX/targetXTranslateY
            final float targetTranslateX, targetTranslateY;
            int animationType;
            switch (swipeDir) {
                case LEFT:
                case RIGHT:
                case START:
                case END:
                    targetTranslateY = 0;
                    targetTranslateX = Math.signum(mDx) * mRecyclerView.getWidth();
                    break;
                case UP:
                case DOWN:
                    targetTranslateX = 0;
                    targetTranslateY = Math.signum(mDy) * mRecyclerView.getHeight();
                    break;
                default:
                    targetTranslateX = 0;
                    targetTranslateY = 0;
            }
            // 标记动画状态为拖动
            if (prevActionState == ACTION_STATE_DRAG) {
                animationType = ANIMATION_TYPE_DRAG;
            } else if (swipeDir > 0) {
                animationType = ANIMATION_TYPE_SWIPE_SUCCESS;
            } else {
                animationType = ANIMATION_TYPE_SWIPE_CANCEL;
            }
            getSelectedDxDy(mTmpPosition);
            final float currentTranslateX = mTmpPosition[0];
            final float currentTranslateY = mTmpPosition[1];
            //定义恢复动画
            final RecoverAnimation rv = new RecoverAnimation(prevSelected, animationType,
                    prevActionState, currentTranslateX, currentTranslateY,
                    targetTranslateX, targetTranslateY) {
                @Override
                public void onAnimationEnd(Animator animation) {
                    super.onAnimationEnd(animation);
                    if (this.mOverridden) {
                        return;
                    }
                    //拖动或者滑动失败的方式
                    if (swipeDir <= 0) {
                        mCallback.clearView(mRecyclerView, prevSelected);
                    } else {
                        // 滑动动画结束后，将动画加入缓存mPendingCleanup
                        mPendingCleanup.add(prevSelected.itemView);
                        mIsPendingCleanup = true;
                        if (swipeDir > 0) {
                            // Animation might be ended by other animators during a layout.
                            // We defer callback to avoid editing adapter during a layout.
                            postDispatchSwipe(this, swipeDir);
                        }
                    }
                    // removed from the list after it is drawn for the last time
                    if (mOverdrawChild == prevSelected.itemView) {
                        removeChildDrawingOrderCallbackIfNecessary(prevSelected.itemView);
                    }
                }
            };
            //计算动画持续事件
            final long duration = mCallback.getAnimationDuration(mRecyclerView, animationType,
                    targetTranslateX - currentTranslateX, targetTranslateY - currentTranslateY);
            rv.setDuration(duration);
            mRecoverAnimations.add(rv);
            //开始执行动画
            rv.start();
            preventLayout = true;
        } else {
            removeChildDrawingOrderCallbackIfNecessary(prevSelected.itemView);
            mCallback.clearView(mRecyclerView, prevSelected);
        }
        mSelected = null;
    }
    if (selected != null) {
        mSelectedFlags =
                (mCallback.getAbsoluteMovementFlags(mRecyclerView, selected) & actionStateMask)
                        >> (mActionState * DIRECTION_FLAG_COUNT);
        mSelectedStartX = selected.itemView.getLeft();
        mSelectedStartY = selected.itemView.getTop();
        mSelected = selected;
        // 如果是拖动
        if (actionState == ACTION_STATE_DRAG) {
            // 回调长按的反馈
            mSelected.itemView.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
        }
    }
    final ViewParent rvParent = mRecyclerView.getParent();
    //通知Rv不拦截子view的事件
    if (rvParent != null) {
        rvParent.requestDisallowInterceptTouchEvent(mSelected != null);
    }
    if (!preventLayout) {
        mRecyclerView.getLayoutManager().requestSimpleAnimationsInNextLayout();
    }
    //通知回调
    mCallback.onSelectedChanged(mSelected, mActionState);
    //重新绘制
    mRecyclerView.invalidate();
}
```
这里的代码量也很多，这个方法的意思是，如果是滑动事件的话，计算获得位置的变化的信息，并将动画绑定到itemView上，最后回调mCallback.onSelectedChanged方法，并调用mRecyclerView的重绘。

也就是说onInterceptTouchEvent，主要实现了对选中itemView的各项赋值，只要存在满足条件的itemView，拦截事件成立，将执行onTouchEvent事件。
```java
@Override
public void onTouchEvent(@NonNull RecyclerView recyclerView, @NonNull MotionEvent event) {
    mGestureDetector.onTouchEvent(event);
    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(event);
    }
    if (mActivePointerId == ACTIVE_POINTER_ID_NONE) {
        return;
    }
    final int action = event.getActionMasked();
    final int activePointerIndex = event.findPointerIndex(mActivePointerId);
    if (activePointerIndex >= 0) {
        checkSelectForSwipe(action, event, activePointerIndex);
    }
    ViewHolder viewHolder = mSelected;
    if (viewHolder == null) {
        return;
    }
    switch (action) {
        case MotionEvent.ACTION_MOVE: {
            // Find the index of the active pointer and fetch its position
            if (activePointerIndex >= 0) {
                //更新mDx,mdy
                updateDxDy(event, mSelectedFlags, activePointerIndex);
                //处理拖拽事件
                moveIfNecessary(viewHolder);
                mRecyclerView.removeCallbacks(mScrollRunnable);
                mScrollRunnable.run();
                mRecyclerView.invalidate();
            }
            break;
        }
        case MotionEvent.ACTION_CANCEL:
            if (mVelocityTracker != null) {
                mVelocityTracker.clear();
            }
            // fall through
        case MotionEvent.ACTION_UP:
            select(null, ACTION_STATE_IDLE);
            mActivePointerId = ACTIVE_POINTER_ID_NONE;
            break;
        case MotionEvent.ACTION_POINTER_UP: {
            final int pointerIndex = event.getActionIndex();
            final int pointerId = event.getPointerId(pointerIndex);
            if (pointerId == mActivePointerId) {
                // This was our active pointer going up. Choose a new
                // active pointer and adjust accordingly.
                final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
                mActivePointerId = event.getPointerId(newPointerIndex);
                updateDxDy(event, mSelectedFlags, pointerIndex);
            }
            break;
        }
    }
}
```
在onInterceptTouchEvent方法中，已经确定了选中的itemview，因此在onTouchEvent方法中只需要负责对滑动事件的处理即可。我们来看代码，通过updateDxDy方法记录了水平偏移量mDx和竖直偏移量mDy，而moveIfNecessary方法中，如果是拖拽事件，则直接返回。因此我们来看RecyclerView的onDraw方法。

在RecyclerView之实现吸顶效果一文中，就已经介绍过ItemDecoration一类。回调RecyclerView的onDraw方法会依次调用ItemDecoration的onDraw和onDrawOver方法。而ItemTouchHelper又继承自ItemDecoration，因此我们来看一下它的onDraw方法。
```java
@Override
public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
    // we don't know if RV changed something so we should invalidate this index.
    mOverdrawChildPosition = -1;
    float dx = 0, dy = 0;
    if (mSelected != null) {
        getSelectedDxDy(mTmpPosition);
        dx = mTmpPosition[0];
        dy = mTmpPosition[1];
    }
    mCallback.onDraw(c, parent, mSelected,
            mRecoverAnimations, mActionState, dx, dy);
}
```
在该方法中，计算选中的itemView新的偏移量，然后回调mCallback.onDraw()方法。
```java
void onDraw(Canvas c, RecyclerView parent, ViewHolder selected,
          List<ItemTouchHelper.RecoverAnimation> recoverAnimationList,
          int actionState, float dX, float dY) {
      final int recoverAnimSize = recoverAnimationList.size();
      for (int i = 0; i < recoverAnimSize; i++) {
          final ItemTouchHelper.RecoverAnimation anim = recoverAnimationList.get(i);
          anim.update();
          final int count = c.save();
          onChildDraw(c, parent, anim.mViewHolder, anim.mX, anim.mY, anim.mActionState,
                  false);
          c.restoreToCount(count);
      }
      if (selected != null) {
          final int count = c.save();
          onChildDraw(c, parent, selected, dX, dY, actionState, true);
          c.restoreToCount(count);
      }
  }
```
每次重新选择itemView的时候，已经标记为选中的itemView绑定的anim会被移除，所以最后走slected!=null的onChildDraw方法。
