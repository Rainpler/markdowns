
## 前言
>声明.本项目源码基于Api 29

Android的常用布局里，LinearLayout属于使用频率很高的布局。RelativeLayout也是，但相比于RelativeLayout每个子控件都需要给上ID以供另一个相关控件摆放位置来说，LinearLayout两个方向上的排列规则在明显垂直/水平排列情况下使用更加方便。


相比于RelativeLayout无论如何都是两次测量的情况下，LinearLayout只有子控件设置了weight属性时，才会有二次测量，其余情况都是一次。

同时，出于性能上来说，一般而言功能越复杂的布局，性能也是越低的（不考虑嵌套的情况下）。


另外，LinearLayout的高级用法除了weight，还有divider，baselineAligned等用法，虽然用的不常见就是了。

**本篇主要针对LinearLayout垂直方向的测量、weight和divider进行分析，其余属性因为比较冷门，因此不会详说。**

##  源码分析
我们知道，自定义ViewGroup最重要的就是onMeasure()，onLayout()，onDraw()三个流程。而对于LinearLayout来说，同样离不开这几个方法。

由于在LinearLayout中onDraw()主要用于绘制divider分割线, 所以该方法只是根据横向还是纵向规则来绘制divider哦，别无其他的内容。因此不做分析。

所以我们重点来看onMeasure()和onLayout()的源码分析。

### onMeasure()


在LinearLayout的`onMeasure()`里面，会根据mOrientation这个int值来进行水平或者垂直的测量计算。
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```
我们都知道，int若没有初始化默认为0，因此如果我们不指定orientation，measure则会按照水平方向来测量。
```java
public static final int HORIZONTAL = 0;
public static final int VERTICAL = 1;
```
接下来我们主要通过分析`measureVertical()`方法，学习LinearLayout的测量过程。了解了垂直方向的测量之后，水平方向的也就不难理解了。

在LinearLayout的测量过程中，会根据子view的weight属性而测量不同的次数，整个过程大概如下：

1. 　第一次测量，会对大部分的子view进行测量, 并得出LinearLayout的高度。如果子view没有weight值且在xml中指定了高度，则可以得到具体的数值。否则子view如果是wrap的或者带有权重的测量的都不是最终的。
2. 第二次测量，补充测量前面没有测过的子view, 如果确定了LinearLayout的高度后，前面测量的子view并未有填充满linearlayout的高(因为带权重的view第一次测量的高度不是最终的)，这里会通过他们的权重比去计算出最终的各个带wrap, 或者带有权重的view的高度。
3. 最后设置LinearLayout的高度，并纠正那些子view的宽度是MATCH_PARENT的，重新测量这些子view的宽度。

整个方法我们分为三部分来学习：
 - 变量说明
 - 第一次测量
 - 第二次测量

##### 变量说明

这里对该方法涉及到的一些变量进行了说明，这样我们在阅读相关源码的也更方便理解。

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {

    //子view的高度和
    mTotalLength = 0;
    // maxWidth用来记录所有子控件中控件宽度最大的值。
    int maxWidth = 0;
    // 子控件的测量状态，会在遍历子控件测量的时候通过combineMeasuredStates来合并上一个子控件测量状态与当前遍历到的子控件的测量状态，采取的是按位相或
    int childState = 0;

    // 子控件中layout_weight<=0的View的最大宽度
    int alternativeMaxWidth = 0;
    // 子控件中layout_weight>0的View的最大宽度
    int weightedMaxWidth = 0;
    // 是否子控件全是match_parent的标志位，用于判断是否需要重新测量
    boolean allFillParent = true;
    // 所有子控件的weight之和
    float totalWeight = 0;

    // 直接子view的个数
    // 但实际上，直接返回getChildCount()
    final int count = getVirtualChildCount();

    // 得到测量模式
    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

    // 当子控件为match_parent的时候，该值为ture，同时判定的还有下面的matchWidthLocally，这个变量决定了子控件的测量是父控件干预还是填充父控件（剩余的空白位置）。
    boolean matchWidth = false;

    boolean skippedMeasure = false;

    //以LinearLayout中第几个子view的baseLine作为LinearLayout的基准线
    final int baselineChildIndex = mBaselineAlignedChildIndex;

    //是否使用最大高度的child属性
    final boolean useLargestChild = mUseLargestChild;

    int largestChildHeight = Integer.MIN_VALUE;

    //已经消费的剩余空间
    int consumedExcessSpace = 0;
    //没有被跳过的子view数
    int nonSkippedChildCount = 0;
}
```

##### 首次测量

接下来开始看测量的代码，这里会通过for循环遍历子控件，并进行初次测量，

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // ...接上
    for (int i = 0; i < count; ++i) {

        final View child = getVirtualChildAt(i);
        if (child == null) {
            // measureNullChild()方法返回的永远是0，可以扩展使用
            mTotalLength += measureNullChild(i);
            continue;
        }

        if (child.getVisibility() == GONE) {
           // getChildrenSkipCount()返回的也是0 ，可以扩展使用
            i += getChildrenSkipCount(child, i);
            continue;
        }

        nonSkippedChildCount++;

        // 根据showDivider的值（before/middle/end）来决定遍历到当前子控件时，高度是否需要加上divider的高度
        // 比如showDivider为before，那么只会在第0个子控件测量时加上divider高度，其余情况下都不加
        if (hasDividerBeforeChildAt(i)) {
            mTotalLength += mDividerWidth;
        }
        //获取到子view的LayoutParams
        final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                child.getLayoutParams();

        // 得到每个子控件的LayoutParams后，累加权重和，后面用于跟weightSum相比较
        totalWeight += lp.weight;

        //根据子view的height和weight属性决定是否要占据剩余空间
        //这个也跟我们布局的是一样，如果要使用权重的话，则heght=0，并填写weight值
        final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;

        // 我们都知道，测量模式有三种：
        // * UNSPECIFIED：父控件对子控件无约束
        // * Exactly：父控件对子控件强约束，子控件永远在父控件边界内，越界则裁剪。如果要记忆的话，可以记忆为有对应的具体数值或者是Match_parent
        // * AT_Most：子控件为wrap_content的时候，测量值为AT_MOST。
        if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
            // 这个if里面需要满足两个条件：
            // LinearLayout的测量模式为为MeasureSpec.EXACTLY
            // 说明它为match_parent(或者有具体值)
            // 测量到这里的时候，会给个标志位，稍后再处理。此时会计算总高度
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
            skippedMeasure = true;
        } else {
            // 到这个分支，则需要对不同的情况进行测量

            if (useExcessSpace) {
                // 意味着如果MesureSpec的宽度模式是UNSPECIFIED或者AT_MOST
                // 那么此时将当前子控件的高度置为wrap_content
                // 为何需要这么做，主要是因为当父类这时候的高度未定，其大小实际上由子控件控制
                // 我们都知道，自定义控件的时候，通常我们会指定测量模式为wrap_content时的默认大小
                // 这里强制给定为wrap_content为的就是防止子控件高度为0.
                oldHeight = 0;
                lp.height = LayoutParams.WRAP_CONTENT;
            }

            // 这里可以看到，传入的height是跟weight有关。
            final int usedHeight = totalWeight == 0 ? mTotalLength : 0;

            // 如果weight=0 ，则说明在当前子控件之前还没有遇到有weight的子控件，那么LinearLayout将会进行正常的测量，传入的userHeight就是已经占据的空间。
            // 否则传入userHeight为0
            measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                    heightMeasureSpec, usedHeight);


            final int childHeight = child.getMeasuredHeight();

            if (useExcessSpace) {
                // Restore the original height and record how much space
                // we've allocated to excess-only children so that we can
                // match the behavior of EXACTLY measurement.
                lp.height = 0;
                consumedExcessSpace += childHeight;
            }
            final int totalLength = mTotalLength;
            // getNextLocationOffset返回的永远是0，因此这里实际上是比较child测量前后的总高度，取大值。
            mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                   lp.bottomMargin + getNextLocationOffset(child));

            if (useLargestChild) {
                largestChildHeight = Math.max(childHeight, largestChildHeight);
            }
        }

        if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
           mBaselineChildTop = mTotalLength;
        }

        if (i < baselineChildIndex && lp.weight > 0) {
            throw new RuntimeException("A child of LinearLayout with index "
                    + "less than mBaselineAlignedChildIndex has weight > 0, which "
                    + "won't work.  Either remove the weight, or don't set "
                    + "mBaselineAlignedChildIndex.");
        }

        boolean matchWidthLocally = false;

        // 当父类（LinearLayout）不是match_parent或者精确值的时候，但子控件却是一个match_parent
        // 那么matchWidthLocally和matchWidth置为true
        // 意味着这个控件将会占据父类（水平方向）的所有空间
        // 后面将会对子view的宽度再处理
        if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
            matchWidth = true;
            matchWidthLocally = true;
        }

        final int margin = lp.leftMargin + lp.rightMargin;
        final int measuredWidth = child.getMeasuredWidth() + margin;
        maxWidth = Math.max(maxWidth, measuredWidth);
        childState = combineMeasuredStates(childState, child.getMeasuredState());

        allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

        if (lp.weight > 0) {
            weightedMaxWidth = Math.max(weightedMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);
        } else {
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);
        }

        i += getChildrenSkipCount(child, i);
    }
}
```
在这里，通过拿到子view的LayoutParams对象，然后根据其weight和height得到useExcessSpace值。如果父控件的heightMode== MeasureSpec.EXACTLY且useExcessSpace为true，则说明需要根据权重来分配剩余空间，那么就会跳过本次测量。

如果heightMode!=MeasureSpec.EXACTLY，则说明父控件的MesureSpec的宽度模式是UNSPECIFIED或者AT_MOST，这时候就会进入第一次测量。首先如果userExcessSpace，则会将子view的lp.height指定WARP_CONTENT。为何需要这么做，主要是因为当父类这时候的高度未定，其大小实际上由子控件控制 我们都知道，自定义控件的时候，通常我们会指定测量模式为wrap_content时的默认大小。这里强制给定为wrap_content为的就是防止子控件高度为0.

然后根据权重，决定接下来`measureChildBeforeLayout()`测量时传入的height值是mTotalLength还是0。

假如当前子控件的weight加起来还是为0，则说明在当前子控件之前还没有遇到有weight的子控件，那么LinearLayout将会进行正常的测量，若之前遇到过有weight的子控件，那么LinearLayout传入0，那么子view的高度会在后面重新进行测量。

##### 第二次测量

在第一个循环遍历里面，会对子view进行初次的测量。如果LinearLayout的heightMode为`MeasureSpec.EXACTLY`且子view的height等于0且weight>0，则说明其需要占据剩余空间，那么要等到其他子view确定完高度之后才能计算它的高度，这时候就会跳过测量。否则会进行第一次测量，但是由于weight的存在，第一次测量得到值并不是最终的结果，所以要进行第二次测量。

我们来看第二次的测量过程，我们将代码分为两部分。
```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // ....接上

    // 是否加上分界线的高度
    if (nonSkippedChildCount > 0 && hasDividerBeforeChildAt(count)) {
          mTotalLength += mDividerHeight;
    }

    // 如果useLargestChild为true
    // 该变量就是决定LineaerLayout高度是否与最大高度子view的有关
    // 假如最大子view高300，有三个子view，那么最终LinearLayout的高度将会是300*3=900
    if (useLargestChild &&
            (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
        mTotalLength = 0;

        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                mTotalLength += measureNullChild(i);
                continue;
            }

            if (child.getVisibility() == GONE) {
                i += getChildrenSkipCount(child, i);
                continue;
            }

            final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                    child.getLayoutParams();
            // Account for negative margins
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                    lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
        }
    }

    // Add in our padding
    mTotalLength += mPaddingTop + mPaddingBottom;

    int heightSize = mTotalLength;

    // Check against our minimum height
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

    // Reconcile our calculated size with the heightMeasureSpec
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK;

    //计算剩余的空间
    int remainingExcess = heightSize - mTotalLength
            + (mAllowInconsistentMeasurement ? 0 : consumedExcessSpace);

}
```
首先先对useChildLargest做了判断，然后heightSize做了两次赋值，为何需要做两次赋值？因为我们的布局除了子控件，还有自己本身的background，因此这里需要比较当前的子控件的总高度和背景的高度取大值。

接下来就是判定大小，我们都知道测量的MeasureSpec实际上是一个32位的int，高两位是测量模式，剩下的就是大小，因此```heightSize = heightSizeAndState & MEASURED_SIZE_MASK;```作用就是用来得到大小的精确值（不含测量模式）。

我们接着看下面的代码：

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // ....接上
    // 如果跳过了初次测量
    // 或者totalWeight>0且还有剩余空间
    // 那么就要进行第二次测量
    if (skippedMeasure
            || ((sRemeasureWeightedChildren || remainingExcess != 0) && totalWeight > 0.0f)) {
        float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

        mTotalLength = 0;

        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            if (child == null || child.getVisibility() == View.GONE) {
                continue;
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final float childWeight = lp.weight;
            if (childWeight > 0) {
                // 计算占据剩余空间的比重
                final int share = (int) (childWeight * remainingExcess / remainingWeightSum);

                // 计算剩余空间
                remainingExcess -= share;
                // 计算剩余权重
                remainingWeightSum -= childWeight;

                final int childHeight;
                if (mUseLargestChild && heightMode != MeasureSpec.EXACTLY) {
                    childHeight = largestChildHeight;
                } else if (lp.height == 0 && (!mAllowInconsistentMeasurement
                        || heightMode == MeasureSpec.EXACTLY)) {
                    // This child needs to be laid out from scratch using
                    // only its share of excess space.
                    childHeight = share;
                } else {
                    // This child had some intrinsic height to which we
                    // need to add its share of excess space.
                    childHeight = child.getMeasuredHeight() + share;
                }

                final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.max(0, childHeight), MeasureSpec.EXACTLY);
                final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                        lp.width);
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                // Child may now not fit in vertical dimension.
                childState = combineMeasuredStates(childState, child.getMeasuredState()
                        & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
            }

            final int margin =  lp.leftMargin + lp.rightMargin;
            final int measuredWidth = child.getMeasuredWidth() + margin;
            maxWidth = Math.max(maxWidth, measuredWidth);

            boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                    lp.width == LayoutParams.MATCH_PARENT;

            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);

            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                    lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
        }

        // Add in our padding
        mTotalLength += mPaddingTop + mPaddingBottom;
        // TODO: Should we recompute the heightSpec based on the new total length?
    } else {
        alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                       weightedMaxWidth);


        // We have no limit, so make all weighted views as tall as the largest child.
        // Children will have already been measured once.
        if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
            for (int i = 0; i < count; i++) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }

                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();

                float childExtra = lp.weight;
                if (childExtra > 0) {
                    child.measure(
                            MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                    MeasureSpec.EXACTLY),
                            MeasureSpec.makeMeasureSpec(largestChildHeight,
                                    MeasureSpec.EXACTLY));
                }
            }
        }
    }

    if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
        maxWidth = alternativeMaxWidth;
    }

    maxWidth += mPaddingLeft + mPaddingRight;

    // Check against our minimum width
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);

    if (matchWidth) {
        forceUniformWidth(count, heightMeasureSpec);
    }
}
```
我们看，只有跳过了首次测量，或者totalWeight>0且remainingExcess!=0，才会进入第二次测量的代码中。然后子view进行遍历，如果totalWeight!=0，说明还有根据权重来进行空间分配。然后通过子view权重的占比，计算出应该分配的空间share值。weight为何是针对剩余空间进行分配的原理了。

我们打个比方，假如现在我们的LinearLayout的weightSum=10，总高度100，有两个子控件（他们的height=0dp），他们的weight分别为2：8。那么在测量第一个子控件的时候，可用的剩余高度为100，第一个子控件的高度则是100\*（2/10）=20，接下来可用的剩余高度为80。我们继续第二个控件的测量，此时它的高度实质上是80\*（8/8）=80。

但是，关于weight还有一个疑惑点，假如两个子控件的height都是`MATCH_PARENT`，那么得到的高度比却成了8：2。这又是为什么呢。对于这个问题，我们来看看代码。

在通过权重计算完该分配空间之后，会进入到if语句的判断：
```java
if (mUseLargestChild && heightMode != MeasureSpec.EXACTLY) {
                    childHeight = largestChildHeight;
} else if (lp.height == 0 && (!mAllowInconsistentMeasurement
        || heightMode == MeasureSpec.EXACTLY)) {
    // This child needs to be laid out from scratch using
    // only its share of excess space.
    childHeight = share;
} else {
    // This child had some intrinsic height to which we
    // need to add its share of excess space.
    childHeight = child.getMeasuredHeight() + share;
}
```
首先，如果设定了measureWithLargestChild为true，且父类的测量模式不等于MeasureSpec.EXACTLY，则意味着子view的高度将会被设置为最大子view的高度。然后如果lp.height==0且heightMode == MeasureSpec.EXACTLY的话，则会将share值分配给子view，这也就是我们例子中的第一种情况。

如果仍然不满足呢？我们第二个例子中，将lp.height设置为了match_parent，那么就会走到最后的else中去，并将子view的高度设置为child.getMeasuredHeight()和share之和。


事实上，由于在第一次测量前，我们通过`lp.height==0&&weighte>0`该条件得到了useExcessSpace。而条件不满足之后，将会进行第一测量，这个方法正是`measureChildBeforeLayout()`。


在执行`measureChildBeforeLayout()`，由于我们的child的height=match_parent，因此此时可用空间实质上是整个LinearLayout，执行了`measureChildBeforeLayout()`后，此时的mTotalLength是整个LinearLayout的大小。

回到我们的例子，假设我们的LinearLayout高度为100，两个child的高度都是match_parent，那么执行了`measureChildBeforeLayout()`后，我们两个子控件的高度都将会是这样：
- child_1.height=100
- child_2.height=100
那么mTotalLength将会为两者之和200，然后得到的剩余空间将会是-100。

没错，你看到的的确是一个负数。接下来再来看share的计算公式：
```java
final int share = (int) (childWeight * remainingExcess / remainingWeightSum);
```
对于第一个子view来说，share=(2*(-100))/10=-20，那么走到
```java
childHeight = child.getMeasuredHeight() + share;
```
得到的子view高度将会是100-20=80。同理，得到第二个子view的高度将会是20。这就是为什么当子view的height为`MATCH_PARENT`的时候，高度比会相反。接下来就是走child.measure，并把childHeight传进去，从而指定子view的高度。

接下来就是将测量到的子view宽高进行累计，并通过`setMeasuredDimension()`方法保存到自己的测量维度中。如果matchWidth为true的话，则会进一步强行对子view的宽度进行一次调整。

接下来我们继续看onLayout()方法。

### onLayout()
`onLayout()`方法要完成的工作，主要是对view进行一个布局。在这里同样是对mOrientation进行了区分，我们也只分析`layoutVertical()`方法的实现即可。
```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```
#### 重力
在看该方法之前，我们先来温习android UI布局中的两个属性，`android:gravity` 和` android:layout_gravity` 两个属性可以设置控件的位置。
- **android:gravity** 组件的子组件在组件中的位置
- **android:layout_gravity** 组件自身在父组件中的位置

还有一种总结是：
- 名称**以layout_开头**的属性作用于组件的父组件，称这些属性为布局参数，它们会告知父布局如何在内部安排自己的子元素。
- 名称**不以layout_开头**的属性作用于组件本身，组件生成时，会调用某个方法按照属性及属性值进行自我配置。

#### layoutVertical()
而在LinearLayout 中，这两种属性的使用与布局属性中的`android:orientation`属性有关，我们先来看`layoutVertical()`的代码。

```java
void layoutVertical(int left, int top, int right, int bottom) {
    final int paddingLeft = mPaddingLeft;

    int childTop;
    int childLeft;

    // Where right end of child should go
    final int width = right - left;
    int childRight = width - mPaddingRight;

    // Space available for child
    int childSpace = width - paddingLeft - mPaddingRight;

    final int count = getVirtualChildCount();

    final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

    switch (majorGravity) {
       case Gravity.BOTTOM:
           // mTotalLength contains the padding already
           childTop = mPaddingTop + bottom - top - mTotalLength;
           break;

           // mTotalLength contains the padding already
       case Gravity.CENTER_VERTICAL:
           childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
           break;

       case Gravity.TOP:
       default:
           childTop = mPaddingTop;
           break;
    }

    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();

            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();

            int gravity = lp.gravity;
            if (gravity < 0) {
                gravity = minorGravity;
            }
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                            + lp.leftMargin - lp.rightMargin;
                    break;

                case Gravity.RIGHT:
                    childLeft = childRight - childWidth - lp.rightMargin;
                    break;

                case Gravity.LEFT:
                default:
                    childLeft = paddingLeft + lp.leftMargin;
                    break;
            }

            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            childTop += lp.topMargin;
            //实际上继续调用layout()方法
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
```

在`layoutVertical()`中，mGravity 表示LinearLayout自身的`android:gravity`属性，majorGravity和 minorGravity由 mGravity 与`Gravity.VERTICAL_GRAVITY_MASK`做位运算，得到的 和`Gravity.HORIZONTAL_GRAVITY_MASK`做位与运算而来，分别确定mGravity垂直方向上和水平方向上的属性值。

在布局过程中，先考虑majorGravity，它决定了子控件在垂直方向上的排布。然后遍历子控件，考虑每个子视图自身的`android:layout_gravity`属性值，即代码中的 lp.gravity。从代码中可以看出，如果lp.gravity 大于0， 就是说该子视图设置了`android:layout_gravity`属性，那么就使用它自己设置的，否则使用 minorGravity。

然后通过`getAbsoluteGravity()`方法得到绝对排布，在switch中在for循环中仅仅是判断了CENTER_HORIZONTAL、RIGHT、LEFT三中情况，即在orientation=vertical时，只有水平居中和居左和居右有效，因此这就是为什么我们在设置layout_gravity=center时，只有水平方向的设置有效果了。

在过setChildFrame()方法中，会继续调用child.layout()将子控件定位。


## 总结
在这里，我们花费了大篇幅讲解`measureVertical()`的流程，事实上对于LinearLayout来说，其最大的特性也正是两个方向的排布以及weight的计算方式。

在这里我们不妨回过头看一下，其实我们会发现在测量过程中，设计者总是有意分开含有weight和不含有weight的测量方式，同时利用height跟0比较来更加的细分每一种情况。

可能初看的时候觉得代码太多，事实上一轮分析下来，方向还是很清晰的。毕竟有weight的地方前期都给个标志跳过，在测量完需要的数据（比如父控件的总高度什么的）后，再根据父控件的数据和weight再针对进行二次测量。

在文章的最后，我们小结一下对于测量这里的算法的不同情况下的区别以及原理：

| 父控件      | 子控件        | 子view结果    |
| -------- | ------ | ----------------- |
| match_parent（或者精确值） | weight>0&&height!=0   | share值     |
| match_parent（或者精确值） | weight>0&&height==match_parent | 已经被测量过一次，  比例相反    |
| wrap_content        | weight>0        | 子控件的高度将会强行置为其wrap_content给的值并以wrap_content模式进行测量 |
| wrap_content      | weight==0      | 子控件的高度跟其他的ViewGroup一致    |

**到这里，LinearLayout针对measure的解析到此结束。**
