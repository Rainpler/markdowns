>自定义view是android的基础，一个效果理论上能够在android 上实现，你就应该具备实现他的能力。

作为一个安卓开发，掌握自定义View，开发高级UI的能力是进阶的基础，因此必须好好学习。

那么，自定义布局一般都包含哪些知识呢。以下是我们在自定义View开发中所涉及到的知识点。

布局：onLayout onMeasure  Layout  viewGroup
显示 ：onDraw  ：canvas paint matrix clip  rect  animation path（贝塞尔曲线）  line
交互 ：onTouchEvent    组合的viewGroup

关于自定义View通常分为两类
1. 自定义View：在没有现成的View，需要自己实现的时候，就使用自定义View，一般继承自View，SurfaceView或其他的View
2. 自定义ViewGroup：自定义ViewGroup一般是利用现有的组件根据特定的布局方式来组成新的组件，大多继承自ViewGroup或各种Layout


### 自定义View的绘制流程
![](https://www.hualigs.cn/image/609d2002f0869.jpg)
从流程图可以看到，主要有以下三个步骤
- onMeasure 测量view大小
- onLayout 确定子view布局
- onDraw 实际绘制内容
### 自定义流式布局
FlowLayout， 这个概念在移动端或者前端开发中很常见，特别是在多标签的展示中， 往往起到了关键的作用。然而Android 官方， 并没有为开发者提供这样一个布局。因此，我们来实现一个自定义的流式布局。
#### 构造函数
自定义布局，必须实现以下三个构造函数：
```java
 public FlowLayout(Context context) {
        super(context);
    }

    //反射
    public FlowLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    //主题style
    public FlowLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
```
一个Context参数的构造函数是我们最常见的，就像我们定义一个新的LinearLayout，就通常会使用``new LinearLayout(this)``来获得。

而添加了AttributeSet attrs参数的构造函数，可以让我们通过反射去修改自定义View的属性，再添加一个int defStyleAttr的属性，可以让我们使用主题style。

其实还有四个参数的非必要构造函数，可以使用自定义属性。
```java
public FlowLayout(Context context, @Nullable AttributeSet attrs, int defStyleAttr,int defStyleRes){
  super(context, attrs, defStyleAttr, defStyleRes);
}
```
#### onMeasure
我们先来看绘制流程的第一步，onMeasure。

顾名思义，这一步就是要测量view的大小，那么布局中有子view跟自身，我们该先测量谁呢？其实都可以，这是根据布局的实现决定的，先测量自己再测量孩子，通常只有ViewPager采用该方式实现，而绝大多数布局都是采用的先测量子view，再测量自身的大小。我们实现的自定义流式布局也是基于后者。

简单的实现流程如下：

```java

  //先测量孩子
  int childCount=getChildCount();
  for (int i = 0; i < childCount; i++) {
      View childView = getChildAt(i);
      childView.measure(childWidthMeasureSpec,childHeightMeasureSpec);
  }

  //再测量自身
  setMeasuredDimension(width,height);

```
对于子view来说，就是要确定其childWidthMeasureSpec和childHeightMeasureSpec两个参数，而对于自身来说，就是要确定width和height两个参数。

这里涉及到两个重要的类，**MeasureSpec**和**LayoutParams**

在Android中，xml布局文件实际上一种序列化的数据，其中以键值对的形式记录，比如

```java
android:layout_width="10dp"
android:layout_width="wrap_content"
android:layout_width="match_parent"
```
经过一系列加载过程，最后通过inflate函数，加载到ViewGroup中。

LayoutParams 是ViewGroup中的内部类，用于保存 View 的布局属性，ViewGroup 中定义了 ViewGroup.LayoutParams 作为布局参数的基类，定义了 width 和 height 属性，对应布局中的 layout_width 和 layout_height 属性。


```java
public static class LayoutParams {

        @SuppressWarnings({"UnusedDeclaration"})
        @Deprecated
        public static final int FILL_PARENT = -1;

        public static final int MATCH_PARENT = -1;

        public static final int WRAP_CONTENT = -2;

        @ViewDebug.ExportedProperty(category = "layout", mapping = {
            @ViewDebug.IntToString(from = MATCH_PARENT, to = "MATCH_PARENT"),
            @ViewDebug.IntToString(from = WRAP_CONTENT, to = "WRAP_CONTENT")
        })
        public int width;

        @ViewDebug.ExportedProperty(category = "layout", mapping = {
            @ViewDebug.IntToString(from = MATCH_PARENT, to = "MATCH_PARENT"),
            @ViewDebug.IntToString(from = WRAP_CONTENT, to = "WRAP_CONTENT")
        })
        public int height;
}
```

以 LayoutParams.width 为例，它可能是三种情况 LayoutParams.MATCH_PARENT，LayoutParams.WRAP_CONTENT 或者 一个确切的数值。

另外在 ViewGroup 中，也定义了 MarginLayoutParams 等子类，它继承自 ViewGroup.LayoutParams，对应布局中的 layout_margin 和layout_marginLeft，layout_marginRight，layout_marginTop，layout_marginRight。

MeasureSpec是View中的内部类，代表一个32位的int值，高2位代表SpecMode，低30位代表SpecSize，我们可以把MeasureSpec理解为测量规则，而这个测量规则是由测量模式和和该模式下的测量大小共同组成的。MeasureSpec 。MODE_SHIFT = 30的作用是移位，而mode有如下三种：
- UNSPECIFIED：不对View大小做限制，系统使用
- EXACTLY：确切的大小，如：100dp
- AT_MOST：大小不可超过某数值，如：matchParent, 最大不能超过你父类
```java
 public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /** @hide */
        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}


        public static final int UNSPECIFIED = 0 << MODE_SHIFT;


        public static final int EXACTLY     = 1 << MODE_SHIFT;


        public static final int AT_MOST     = 2 << MODE_SHIFT;


        @MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }

        /
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
```

而View的层级是呈树形结构，ViewGroup既可以作为容器，也能作为容器的子View。假如父ViewGroup设置了layout_width="warp_content"，那么要知道它的尺寸大小，必须就得知道子View的尺寸。因此测量的时候，是通过自上而下的递归实现的，父ViewGroup先测量子View的大小，而子View中的ViewGroup又继续测量其子View的大小，直至到达View树的叶子节点，保存自身测量的尺寸后，再依次向上反馈。

对于DecorView，其MeasureSpec由窗口尺寸和其自身的MeasureSpec来共同决定；对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定，MeasureSpec一旦确定后，onMeasure中就可以确定View的测量宽/高。

因为自身的MeasureSpec mode与子view的LayoutParam mode都有3种，确定子View的MeasureSpec组合起来便有9种。具体可见getChildMeasureSpec方法。
```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
于是，整个测量流程如下：
```java
//度量
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    clearMeasureParams();//内存 抖动
    //先度量孩子
    int childCount = getChildCount();
    int paddingLeft = getPaddingLeft();
    int paddingRight = getPaddingRight();
    int paddingTop = getPaddingTop();
    int paddingBottom = getPaddingBottom();

    int selfWidth = MeasureSpec.getSize(widthMeasureSpec);  //ViewGroup解析的父亲给我的宽度
    int selfHeight = MeasureSpec.getSize(heightMeasureSpec); // ViewGroup解析的父亲给我的高度

    List<View> lineViews = new ArrayList<>(); //保存一行中的所有的view
    int lineWidthUsed = 0; //记录这行已经使用了多宽的size
    int lineHeight = 0; // 一行的行高

    int parentNeededWidth = 0;  // measure过程中，子View要求的父ViewGroup的宽
    int parentNeededHeight = 0; // measure过程中，子View要求的父ViewGroup的高

    for (int i = 0; i < childCount; i++) {
        View childView = getChildAt(i);

        LayoutParams childLP = childView.getLayoutParams();
        if (childView.getVisibility() != View.GONE) {
            //将layoutParams转变成为 measureSpec
            int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec, paddingLeft + paddingRight, childLP.width);
            int childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec, paddingTop + paddingBottom, childLP.height);
            childView.measure(childWidthMeasureSpec, childHeightMeasureSpec);

            //获取子view的度量宽高
            int childMesauredWidth = childView.getMeasuredWidth();
            int childMeasuredHeight = childView.getMeasuredHeight();

            //如果需要换行
            if (childMesauredWidth + lineWidthUsed + mHorizontalSpacing > selfWidth) {

                //一旦换行，我们就可以判断当前行需要的宽和高了，所以此时要记录下来
                allLines.add(lineViews);
                lineHeights.add(lineHeight);

                parentNeededHeight = parentNeededHeight + lineHeight + mVerticalSpacing;
                parentNeededWidth = Math.max(parentNeededWidth, lineWidthUsed + mHorizontalSpacing);

                lineViews = new ArrayList<>();
                lineWidthUsed = 0;
                lineHeight = 0;
            }
            // view 是分行layout的，所以要记录每一行有哪些view，这样可以方便layout布局
            lineViews.add(childView);
            //每行都会有自己的宽和高
            lineWidthUsed = lineWidthUsed + childMesauredWidth + mHorizontalSpacing;
            lineHeight = Math.max(lineHeight, childMeasuredHeight);

            //处理最后一行数据
            if (i == childCount - 1) {
                allLines.add(lineViews);
                lineHeights.add(lineHeight);
                parentNeededHeight = parentNeededHeight + lineHeight + mVerticalSpacing;
                parentNeededWidth = Math.max(parentNeededWidth, lineWidthUsed + mHorizontalSpacing);
            }

        }
    }

    //再度量自己,保存
    //根据子View的度量结果，来重新度量自己ViewGroup
    // 作为一个ViewGroup，它自己也是一个View,它的大小也需要根据它的父亲给它提供的宽高来度量
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);

    int realWidth = (widthMode == MeasureSpec.EXACTLY) ? selfWidth: parentNeededWidth;
    int realHeight = (heightMode == MeasureSpec.EXACTLY) ?selfHeight: parentNeededHeight;
    setMeasuredDimension(realWidth, realHeight);
}
```

#### onLayout
测量完成后，就要执行onLayout函数，确定子view视图的位置。

layout()方法接收四个参数，分别代表着左、上、右、下的坐标，当然这个坐标是相对于当前视图的父视图而言的。可以看到，这里还把刚才测量出的宽度和高度传到了layout()方法中。
```java
 //布局
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int lineCount = allLines.size();

        int curL = getPaddingLeft();
        int curT = getPaddingTop();

        for (int i = 0; i < lineCount; i++){
            List<View> lineViews = allLines.get(i);

            int lineHeight = lineHeights.get(i);
            for (int j = 0; j < lineViews.size(); j++){
                View view = lineViews.get(j);
                int left = curL;
                int top =  curT;

//                int right = left + view.getWidth();
//                int bottom = top + view.getHeight();

                 int right = left + view.getMeasuredWidth();
                 int bottom = top + view.getMeasuredHeight();
                 view.layout(left,top,right,bottom);
                 curL = right + mHorizontalSpacing;
            }
            curT = curT + lineHeight + mVerticalSpacing;
            curL = getPaddingLeft();
        }

    }
```
在这里为什么不用getWidth()而是用getMeasuredWidth()呢，因为getWidth()必须在在layout()过程结束后才能获取到。而getMeasuredWidth()在在measure()过程结束后就可以获取到对应的值，通过setMeasuredDimension()方法来进行设置的。
