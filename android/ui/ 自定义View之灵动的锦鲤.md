今天来实现自定义View之灵动的锦鲤，首先来看效果。

![](../../res/锦鲤游动.gif)

我们要实现锦鲤的绘制，有以下三个步骤：
1. 实现小鱼的绘制
2. 实现小鱼的原地摆动
3. 实现小鱼点击滑动
## 锦鲤的绘制

首先先来看锦鲤的绘制，在这里锦鲤是由简单的图形组合而成，我们可以看分解图，将锦鲤共分成**头**、**鱼鳍**、**身体**、**节肢**、**尾巴**几个部分组成。

![](../../res/锦鲤分解图.jpg)

从分解图上看，涉及到了圆、贝塞尔曲线、梯形、三角形等图形的绘制，为此我们先来了解以下绘制相关的知识。

### 绘制基础
#### Drawable
Drawable是一种可以在Canvas上进行绘制的抽象的概念，颜色、图片等都可以是一个Drawable。Drawable可以通过XML定义，或者通过代码创建，在Android中Drawable是一个抽象类，每个具体的Drawable都是其子类。在这里我们用ImageView来显示基于Drawable绘制的FishDrawable对象。

使用Drawable具有以下优点：
1. 使用简单，比自定义View成本低
2. 非图片类的Drawable所占空间小，能减小apk大小

自定义Drawable需要重写setAlpha、setColorFilter、getOpacity、draw四个方法。
```java
public class FishDrawable extends Drawable {
    @Override
    public void draw(@NonNull Canvas canvas) {

    }

    @Override
    public void setAlpha(int alpha) {

    }

    @Override
    public void setColorFilter(@Nullable ColorFilter colorFilter) {

    }

    @Override
    public int getOpacity() {
        return PixelFormat.OPAQUE;
    }
}

```

#### Path
Path封装了由直线和曲线(二次，三次贝塞尔曲线)构成的几何路径。你能用Canvas中的drawPath来把这条路径画出来(同样支持Paint的不同绘制模式)，也可以用于剪裁画布和根据路径绘制文字。

![](../../res/贝塞尔曲线.jpg)

注意：用drawPath绘制了后，Path的路径还是存在的，所以如果需要绘制新的路径，需要先调用Path的reset方法。

#### Canvas
Canvas 在一般的情况下可以看作是一张画布，所有的绘图操作如drawBitmap, drawCircle都发生在这张画布上，这张画板还定义了一些属性比如Matrix，颜色等等。但是如果需要实现一些相对复杂的绘图操作，比如多层动画，地图（地图可以有多个地图层叠加而成，比如：政区层，道路层，兴趣点层）。Canvas提供了图层（Layer）支持，缺省情况可以看作是只有一个图层Layer。如果需要按层次来绘图，Android的Canvas可以使用saveLayerXXX, restore 来创建一些中间层，对于这些Layer是按照“栈结构“来管理的：

![](../../res/canvas的layer.jpg)

创建一个新的Layer到“栈”中，可以使用saveLayer, savaLayerAlpha, 从“栈”中推出一个Layer，可以使用restore,restoreToCount。当Layer入栈时，后续的DrawXXX操作都发生在这个 Layer上，而Layer退栈时，就会把本层绘制的图像“绘制”到上层或是Canvas上

### 绘制锦鲤
为了方便绘制，我们通过xy轴坐标系来确定绘制锦鲤时的坐标点。

![](../../res/锦鲤绘制坐标系.jpg)

另外由于我们的锦鲤是可以旋转的，所以需要确定一个旋转的中心，在旋转的时候，通过三角函数去计算新的坐标点。

![](../../res/三角函数计算图.jpg)

由此我们给出以下方法，通过传入起始坐标点，也就是旋转中心，半径的长度，已经旋转角度，就可以用过三角函数方法得到对应点的x、y值。
```java
/**
     * @param startPoint 起始点坐标
     * @param length     要求的点到起始点的直线距离 -- 线长
     * @param angle      鱼当前的朝向角度
     * @return
     */
    public PointF calculatePoint(PointF startPoint, float length, float angle) {
        // x坐标
        float deltaX = (float) (Math.cos(Math.toRadians(angle)) * length);
        // y坐标
        float deltaY = (float) (Math.sin(Math.toRadians(angle - 180)) * length);

        return new PointF(startPoint.x + deltaX, startPoint.y + deltaY);
    }
```

对于锦鲤的绘制，我们设定以下初始参数:
```java
//透明度
private int OTHER_ALPHA = 110;
private int BODY_ALPHA = 160;
// 鱼的主要朝向角度
private float fishMainAngle = 0;
// 绘制鱼头的半径
private float HEAD_RADIUS = 100;
// 鱼的重心
private PointF middlePoint =new PointF(4.19f * HEAD_RADIUS, 4.19f * HEAD_RADIUS);
// 鱼身长度
private float BODY_LENGTH = HEAD_RADIUS * 3.2f;
// 寻找鱼鳍起始点坐标的线长
private float FIND_FINS_LENGTH = 0.9f * HEAD_RADIUS;
// 鱼鳍的长度
private float FINS_LENGTH = 1.3f * HEAD_RADIUS;
// 大圆的半径
private float BIG_CIRCLE_RADIUS = 0.7f * HEAD_RADIUS;
// 中圆的半径
private float MIDDLE_CIRCLE_RADIUS = 0.6f * BIG_CIRCLE_RADIUS;
// 小圆半径
private float SMALL_CIRCLE_RADIUS = 0.4f * MIDDLE_CIRCLE_RADIUS;
// --寻找尾部中圆圆心的线长
private final float FIND_MIDDLE_CIRCLE_LENGTH = BIG_CIRCLE_RADIUS * (0.6f + 1);
// --寻找尾部小圆圆心的线长
private final float FIND_SMALL_CIRCLE_LENGTH = MIDDLE_CIRCLE_RADIUS * (0.4f + 2.7f);
// --寻找大三角形底边中心点的线长
private final float FIND_TRIANGLE_LENGTH = MIDDLE_CIRCLE_RADIUS * 2.7f;
```

为了便于绘制，我们的鱼的初始角度为0°，绘制时候的坐标系原点为鱼的重心。
#### 锦鲤的头部
由分解图得，鱼头就是一个圆形，我们首先确定鱼头的圆心位置，然后通过canvas.drawCircle()方法绘制。
```java

@Override
public void draw(@NonNull Canvas canvas) {
    PointF headPoint = calculatePoint(middlePoint, BODY_LENGTH / 2, fishAngle);
    canvas.drawCircle(headPoint.x, headPoint.y, HEAD_RADIUS, mPaint);
    ...
}
```
#### 锦鲤的鱼鳍
由分解图得，鱼鳍为一个二阶贝塞尔曲线，因此需要确定起始点与终点，与控制点。
```java
@Override
public void draw(@NonNull Canvas canvas) {
    ...
    // 画右鱼鳍
    PointF rightFinsPoint = calculatePoint(headPoint, FIND_FINS_LENGTH, fishAngle - 110);
    makeFins(canvas, rightFinsPoint, fishAngle, true);
    // 画左鱼鳍
    PointF leftFinsPoint = calculatePoint(headPoint, FIND_FINS_LENGTH, fishAngle + 110);
    makeFins(canvas, leftFinsPoint, fishAngle, false);
    ...
}

private void makeFins(Canvas canvas, PointF startPoint, float fishAngle, boolean isRight) {
      float controlAngle = 115;

      // 鱼鳍的终点 --- 二阶贝塞尔曲线的终点
      PointF endPoint = calculatePoint(startPoint, FINS_LENGTH, fishAngle - 180);
      // 控制点
      PointF controlPoint = calculatePoint(startPoint, FINS_LENGTH * 1.8f,
              isRight ? fishAngle - controlAngle : fishAngle + controlAngle);
      // 绘制
      mPath.reset();
      // 将画笔移动到起始点
      mPath.moveTo(startPoint.x, startPoint.y);
      // 二阶贝塞尔曲线
      mPath.quadTo(controlPoint.x, controlPoint.y, endPoint.x, endPoint.y);
      canvas.drawPath(mPath, mPaint);
  }
```
#### 锦鲤的身体
由分解图，锦鲤的身体是由两段贝塞尔曲线形成的，因此需要确定身体的四个点与两个控制点的坐标。
```java
PointF bodyBottomCenterPoint = calculatePoint(headPoint, BODY_LENGTH, fishAngle - 180);

private void makeBody(Canvas canvas, PointF headPoint, PointF bodyBottomCenterPoint, float fishAngle) {
        // 身体的四个点求出来
    PointF topLeftPoint = calculatePoint(headPoint, HEAD_RADIUS, fishAngle + 90);
    PointF topRightPoint = calculatePoint(headPoint, HEAD_RADIUS, fishAngle - 90);
    PointF bottomLeftPoint = calculatePoint(bodyBottomCenterPoint, BIG_CIRCLE_RADIUS,
            fishAngle + 90);
    PointF bottomRightPoint = calculatePoint(bodyBottomCenterPoint, BIG_CIRCLE_RADIUS,
            fishAngle - 90);

    // 二阶贝塞尔曲线的控制点 --- 决定鱼的胖瘦
    PointF controlLeft = calculatePoint(headPoint, BODY_LENGTH * 0.56f,
            fishAngle + 130);
    PointF controlRight = calculatePoint(headPoint, BODY_LENGTH * 0.56f,
            fishAngle - 130);

    // 绘制
    mPath.reset();
    mPath.moveTo(topLeftPoint.x, topLeftPoint.y);
    mPath.quadTo(controlLeft.x, controlLeft.y, bottomLeftPoint.x, bottomLeftPoint.y);
    mPath.lineTo(bottomRightPoint.x, bottomRightPoint.y);
    mPath.quadTo(controlRight.x, controlRight.y, topRightPoint.x, topRightPoint.y);
    mPaint.setAlpha(BODY_ALPHA);
    canvas.drawPath(mPath, mPaint);
}
```
#### 锦鲤的节肢
由分解图，锦鲤共有两段节肢，第一段节肢由两个圆与一个梯形组成。

第二段节肢由一个圆与一个梯形组成。
```java
private PointF makeSegment(Canvas canvas, PointF bottomCenterPoint, float bigRadius, float smallRadius,
                               float findSmallCircleLength, float fishAngle, boolean hasBigCircle) {

    // 梯形上底圆的圆心
    PointF upperCenterPoint = calculatePoint(bottomCenterPoint, findSmallCircleLength,
            fishAngle - 180);
    // 梯形的四个点
    PointF bottomLeftPoint = calculatePoint(bottomCenterPoint, bigRadius, fishAngle + 90);
    PointF bottomRightPoint = calculatePoint(bottomCenterPoint, bigRadius, fishAngle - 90);
    PointF upperLeftPoint = calculatePoint(upperCenterPoint, smallRadius, fishAngle + 90);
    PointF upperRightPoint = calculatePoint(upperCenterPoint, smallRadius, fishAngle - 90);

    if (hasBigCircle) {
        // 画大圆 --- 只在节肢1 上才绘画
        canvas.drawCircle(bottomCenterPoint.x, bottomCenterPoint.y, bigRadius, mPaint);
    }
    // 画小圆
    canvas.drawCircle(upperCenterPoint.x, upperCenterPoint.y, smallRadius, mPaint);

    // 画梯形
    mPath.reset();
    mPath.moveTo(upperLeftPoint.x, upperLeftPoint.y);
    mPath.lineTo(upperRightPoint.x, upperRightPoint.y);
    mPath.lineTo(bottomRightPoint.x, bottomRightPoint.y);
    mPath.lineTo(bottomLeftPoint.x, bottomLeftPoint.y);
    canvas.drawPath(mPath, mPaint);

    return upperCenterPoint;
}
```
#### 锦鲤的尾巴
由分解图，锦鲤的尾巴由大小不同的两个等边三角形组成。
```java
private void makeTriangle(Canvas canvas, PointF startPoint, float findCenterLength,
                              float findEdgeLength, float fishAngle) {
    // 三角形底边的中心坐标
    PointF centerPoint = calculatePoint(startPoint, findCenterLength, fishAngle - 180);
    // 三角形底边两点
    PointF leftPoint = calculatePoint(centerPoint, findEdgeLength, fishAngle + 90);
    PointF rightPoint = calculatePoint(centerPoint, findEdgeLength, fishAngle - 90);

    mPath.reset();
    mPath.moveTo(startPoint.x, startPoint.y);
    mPath.lineTo(leftPoint.x, leftPoint.y);
    mPath.lineTo(rightPoint.x, rightPoint.y);
    canvas.drawPath(mPath, mPaint);
}
```

到这里，锦鲤的绘制就已经完成了，接下来我们来看锦鲤游动的动画该如何实现。
### 锦鲤的原地游动
### 点击时的锦鲤游动
