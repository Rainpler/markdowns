
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
        final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;




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
                // 意味着父类即LinearLayout是wrap_content，或者mode为UNSPECIFIED
                // 那么此时将当前子控件的高度置为wrap_content
                // 为何需要这么做，主要是因为当父类为wrap_content时，其大小实际上由子控件控制
                // 我们都知道，自定义控件的时候，通常我们会指定测量模式为wrap_content时的默认大小
                // 这里强制给定为wrap_content为的就是防止子控件高度为0.
                oldHeight = 0;
                lp.height = LayoutParams.WRAP_CONTENT;
            }

            // 这里可以看到，传入的height是跟weight有关。
            // 如果weight=0 ，则说明是正常测量，如果
            final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
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

        // 还记得我们变量里又说到过matchWidthLocally这个东东吗
        // 当父类（LinearLayout）不是match_parent或者精确值的时候，但子控件却是一个match_parent
        // 那么matchWidthLocally和matchWidth置为true
        // 意味着这个控件将会占据父类（水平方向）的所有空间
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
在代码中我注释了一部分，其中最值得注意的是`measureChildBeforeLayout()`方法。这个方法将会决定子控件可用的剩余分配空间。

`measureChildBeforeLayout()`最终调用的实际上是ViewGroup的`measureChildWithMargins()`，不同的是，在传入高度值的时候（垂直测量情况下），会对weight进行一下判定

假如当前子控件的weight加起来还是为0，则说明在当前子控件之前还没有遇到有weight的子控件，那么LinearLayout将会进行正常的测量，若之前遇到过有weight的子控件，那么LinearLayout传入0。

那么`measureChildWithMargins()`的最后一个参数，也就是LinearLayout在这里传入的这个高度值是用来干嘛的呢？

如果我们追溯下去，就会发现，这个函数最终其实是为了结合父类的MeasureSpec以及child自身的LayoutParams来对子控件测量。而最后传入的值，在子控件测量的时候被添加进去。
```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
在官方的注释中，我们可以看到这么一句：
>* @param heightUsed Extra space that has been used up by the parent vertically (possibly by other children of the parent)

事实上，我们在代码中也可以很清晰的看到，在`getChildMeasureSpec()`中，子控件需要把父控件的padding，自身的margin以及一个可调节的量三者一起测量出自身的大小。

那么假如在测量某个子控件之前，weight一直都是0，那么该控件在测量时，需要考虑在本控件之前的总高度，来根据剩余控件分配自身大小。而如果有weight，那么就不考虑已经被占用的控件，因为有了weight，子控件的高度将会在后面重新赋值。

##### 第二次测量
### onLayout()

在上面的代码中，LinearLayout做了针对没有weight的工作，在这里主要是确定自身的大小，然后再针对weight进行第二次测量来确定子控件的大小。

我们接着看下面的代码：
```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    //...接上面
    // 下面的这一段代码主要是为useLargestChild属性服务的，不在本文主要分析范围，略过
    if (mTotalLength > 0 && hasDividerBeforeChildAt(count)) {
        mTotalLength += mDividerHeight;
    }

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
}

```
上面这里是为weight情况做的预处理。

我们略过useLargestChild 的情况，主要看看if处理外的代码。在这里，我没有去掉官方的注释，而是保留了下来。

从中我们不难看出heightSize做了两次赋值，为何需要做两次赋值。

因为我们的布局除了子控件，还有自己本身的background，因此这里需要比较当前的子控件的总高度和背景的高度取大值。

接下来就是判定大小，我们都知道测量的MeasureSpec实际上是一个32位的int，高两位是测量模式，剩下的就是大小，因此```heightSize = heightSizeAndState & MEASURED_SIZE_MASK;```作用就是用来得到大小的精确值（不含测量模式）

接下来我们看这个方法里面第二占比最大的代码：

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
		//...接上面

		//算出剩余空间，假如之前是skipp的话，那么几乎可以肯定是有剩余空间（同时有weight）的
        int delta = heightSize - mTotalLength;
        if (skippedMeasure || delta != 0 && totalWeight > 0.0f) {
			// 限定weight总和范围，假如我们给过weighSum范围，那么子控件的weight总和受此影响
            float weightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

            mTotalLength = 0;

            for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);

                if (child.getVisibility() == View.GONE) {
                    continue;
                }

                LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();

                float childExtra = lp.weight;
                if (childExtra > 0) {
                    // 全篇最精华的一个地方。。。。拥有weight的时候计算方式,ps:执行到这里时，child依然还没进行自身的measure

					// 公式 = 剩余高度*（子控件的weight/weightSum），也就是子控件的weight占比*剩余高度
                    int share = (int) (childExtra * delta / weightSum);
					// weightSum计余
                    weightSum -= childExtra;
					// 剩余高度
                    delta -= share;

                    final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            mPaddingLeft + mPaddingRight +
                                    lp.leftMargin + lp.rightMargin, lp.width);

                    if ((lp.height != 0) || (heightMode != MeasureSpec.EXACTLY)) {
                        int childHeight = child.getMeasuredHeight() + share;
                        if (childHeight < 0) {
                            childHeight = 0;
                        }

                        child.measure(childWidthMeasureSpec,
                                MeasureSpec.makeMeasureSpec(childHeight, MeasureSpec.EXACTLY));
                    } else {

                        child.measure(childWidthMeasureSpec,
                                MeasureSpec.makeMeasureSpec(share > 0 ? share : 0,
                                        MeasureSpec.EXACTLY));
                    }


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


            mTotalLength += mPaddingTop + mPaddingBottom;

        }

		// 没有weight的情况下，只看useLargestChild参数，如果都无相关，那就走layout流程了，因此这里忽略
		else {
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                           weightedMaxWidth);

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
}
```
##### 2.2.2

**weight的两种情况**

这次我的注释比较少，主要是因为需要有一大段的文字来描述。

在weight计算方面，我们可以清晰的看到，weight为何是针对剩余空间进行分配的原理了。
我们打个比方，假如现在我们的LinearLayout的weightSum=10，总高度100，有两个子控件（他们的height=0dp），他们的weight分别为2：8。

那么在测量第一个子控件的时候，可用的剩余高度为100，第一个子控件的高度则是100*（2/10）=20，接下来可用的剩余高度为80

我们继续第二个控件的测量，此时它的高度实质上是80*（8/8）=80

到目前为止，看起来似乎都是正确的，但关于weight我们一直有一个疑问：**就是我们为子控件给定height=0dp和height=match_parent时我们就会发现我们的子控件的高度比是不同的，前者是2：8而后者是调转过来变成8：2	**

对于这个问题，我们不妨继续看看代码。

接下来我们会看到这么一个分支：
```java
if ((lp.height != 0) || (heightMode != MeasureSpec.EXACTLY))
{
    ...
}else {
    ...
}
```
首先我们不管heightMode，也就是父类的测量模式，剩下一个判定条件就是lp.height，也就是子类的高度。

既然有针对这个进行判定，那就是意味着肯定在此之前对child进行过measure，事实上，在这里我们一早就对这个地方进行过描述，这个方法正是`measureChildBeforeLayout()`。

还记得我们的`measureChildBeforeLayout()`执行的先行条件吗

YA，just as u see，正是不满足（LinearLayout的测量模式非EXACTLY/child.height==0/child.weight/child.weight>0）之中的child.height=0


因为除非我们指定height=0，否则match_parent是等于-1，wrap_content是等于-2.

在执行`measureChildBeforeLayout()`，由于我们的child的height=match_parent，因此此时可用空间实质上是整个LinearLayout，执行了`measureChildBeforeLayout()`后，此时的mTotalLength是整个LinearLayout的大小

回到我们的例子，假设我们的LinearLayout高度为100，两个child的高度都是match_parent，那么执行了`measureChildBeforeLayout()`后，我们两个子控件的高度都将会是这样：
>**child_1.height=100**

>**child_2.height=100**

>**mTotalLength=100+100=200**

在一系列的for之后，执行到我们剩余空间：
>int delta = heightSize - mTotalLength;

>**(delta=100[linearlayout的实际高度]-200=-100)**

没错，你看到的的确是一个负数。

接下来就是套用weight的计算公式：
>share=(int) (childExtra * delta / weightSum)

>即：**share=-100*(2/10)=-20;**

然后走到我们所说的if/else里面
```java
 if ((lp.height != 0) || (heightMode != MeasureSpec.EXACTLY)) {
                        // child was measured once already above...
                        // base new measurement on stored values
                        int childHeight = child.getMeasuredHeight() + share;
                        if (childHeight < 0) {
                            childHeight = 0;
                        }

                        child.measure(childWidthMeasureSpec,
                                MeasureSpec.makeMeasureSpec(childHeight, MeasureSpec.EXACTLY));
                    }
```

我们知道 **`child.getMeasuredHeight()=100`**

接着这里有一条 `int childHeight = child.getMeasuredHeight() + share;`

这意味着我们的 **`childHeight=100+(-20)=80;`**

接下来就是走child.measure，并把childHeight传进去，因此最终反馈到界面上，我们就会发现，在两个match_parent的子控件中，weight的比是反转的。

接下来没什么分析的，剩下的就是走layout流程了，对于layout方面，要讲的其实没什么东西，毕竟基本都是模板化的写法了。

## 总结
在这里，我们花费了大篇幅讲解`measureVertical()`的流程，事实上对于LinearLayout来说，其最大的特性也正是两个方向的排布以及weight的计算方式。

在这里我们不妨回过头看一下，其实我们会发现在测量过程中，设计者总是有意分开含有weight和不含有weight的测量方式，同时利用height跟0比较来更加的细分每一种情况。

可能初看的时候觉得代码太多，事实上一轮分析下来，方向还是很清晰的。毕竟有weight的地方前期都给个标志跳过，在测量完需要的数据（比如父控件的总高度什么的）后，再根据父控件的数据和weight再针对进行二次测量。

在文章的最后，我们小结一下对于测量这里的算法的不同情况下的区别以及原理：
  - 父控件是match_parent（或者精确值），子控件拥有weight，并且高度给定为0：
    + 子控件的高度比例将会跟我们分配的layout_weight一致，原因在于weight二次测量时走了else分支，传入的是计算出来的share值

  - 父控件是match_parent（或者精确值），子控件拥有weight，但高度给定为match_parent（或者精确值）：
    + 子控件高度比例将会跟我们分配的layout_weight相反，原因在于在此之前子控件测量过一次，同时子控件的测量高度为父控件的高度，在计算剩余空间的时候得出一个负值，加上自身的测量高度的时候反而更小

  - 父控件是wrap_content，子控件拥有weight：
    + 子控件的高度将会强行置为其wrap_content给的值并以wrap_content模式进行测量

  - 父控件是wrap_content，子控件没有weight：
    + 子控件的高度跟其他的viewgroup一致

**至此,LinearLayout针对measure的解析到此结束**
