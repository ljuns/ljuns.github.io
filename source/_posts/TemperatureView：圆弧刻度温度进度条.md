---
title: TemperatureView：圆弧刻度温度进度条
date: 2016-07-27 10:29:29
categories: 自定义 View
tags: 自定义 View

---
TemperatureView: 这是一个自定义 View ，用来显示温度的变化过程。前段时间公司要做一个实时监测机房温度的应用，现在把这个温度变化的 View 拿出来，自己对自定义 View 这方面的应用还不是很熟悉，这里也算是对自己工作的一个小总结。
二话不说，先上个效果图：<br/>
![](http://oaydqd1yy.bkt.clouddn.com/gif/temp.gif)
<!-- more -->
当然，正常室温变化是没有这么快的，再来张截图：<br/>
![](http://oaydqd1yy.bkt.clouddn.com/png/temp_1.png)
下面来说说这个过程，首先可以将它分为几个部分，分别为：
1、整个背景圆（可有可无）
2、进度弧（分为三段，颜色分别为绿黄红）
3、进度弧上的文字（正常，预警，警告）
4、刻度弧（紧靠着进度弧内侧的黑色弧）
5、刻度
6、中间的圆
7、指针
8、当前温度
那些自定义属性就不多说了，下面直接是绘制的过程（在此之前需要对绘制过程所用到的相关类有一定的了解）。
### 背景圆(可有可无，全凭个人喜欢)
``` Java
/**
 * 背景圆
 * @param canvas
 */
private void drawOutCircle(Canvas canvas) {
  // 已经将画布移到中心，所以圆心为（0,0）
  canvas.drawCircle(0, 0, mSize /2-dp2px(1), outCirclePaint);
  canvas.save();
}
```
mSize:是控件的大小，mSize /2-dp2px(1):是让这个背景圆和控件有点距离。
### 进度弧
``` Java
/**
 * 进度弧
 * @param canvas
 */
private void drawProgress(Canvas canvas) {
  // dp2px(10):留一点位置（可有可无）
  progressRadius = mSize /2 - dp2px(10);
  canvas.save();
  RectF rectF = new RectF(-progressRadius, -progressRadius, progressRadius, progressRadius);
  // 设置为圆角
  progressPaint.setStrokeCap(Paint.Cap.ROUND);
  progressPaint.setColor(Color.GREEN);
  // 从150度位置开始，经过120度
  canvas.drawArc(rectF, 150, 120, false, progressPaint);
  progressPaint.setColor(Color.RED);
  progressPaint.setStrokeCap(Paint.Cap.ROUND);
  canvas.drawArc(rectF, 330, 60, false, progressPaint);
  progressPaint.setColor(Color.YELLOW);
  progressPaint.setStrokeCap(Paint.Cap.BUTT);
  canvas.drawArc(rectF, 270, 60, false, progressPaint);
  canvas.restore();
}
```
整个进度弧是从150度的位置开始到30度为止，也就是说整个进度弧所占的角度为240度。将它分为40份（也就是最高40度的意思），所以刚好每份是6度。
给出的效果图中进度弧是__绿黄红__的，在这里的绘制顺序却是__绿红黄__，这是因为我想要将进度弧的两端（绿和红）设置为圆角，而黄色与绿色和红色的衔接处为直角。如果在这里的 Paint 只是设置为 __Paint.Cap.ROUND__ ，那么中间黄色部分和红绿的衔接处也是圆角的；如果按照绿黄红的顺序来绘制，那么黄色和红色衔接处是圆角的，而绿色和黄色衔接处是直角的。其实很简单，如果理解不了就动手试一下，保证瞬间明了。
### 进度弧上的文字（正常，预警，警告）
``` Java
/**
 * 进度弧上的文字
 * @param canvas
 */
private void drawProgressText(Canvas canvas) {
    canvas.save();
    String normal = "正常";
    String warn = "预警";
    String danger = "警告";
    // 因为文字在进度弧上，所以要旋转一定的角度
    canvas.rotate(-60, 0, 0);
    progressTextPaint.setTextSize(sp2px(12));
    canvas.drawText(normal, -dp2px(12), -scaleArcRadius - dp2px(4), progressTextPaint);
    canvas.rotate(90, 0, 0);
    canvas.drawText(warn, -dp2px(12), -scaleArcRadius - dp2px(4), progressTextPaint);
    canvas.rotate(60, 0, 0);
    canvas.drawText(danger, -dp2px(12), -scaleArcRadius - dp2px(4), progressTextPaint);
    canvas.rotate(-60, 0, 0);
    canvas.restore();
}
```
如果直接 drawText ，那么绘制的文字是水平的，想要把文字绘制在进度弧上，得先旋转一定的角度。拿绿色这段弧来说：它占了120度，所以中间位置是60度，那么就需要旋转60度了。
有一点要注意的是，绿色这段弧在左边，所以要逆时针旋转60度（旋转角度为负数是逆时针，正数是顺时针）。第二段弧上的文字为什么旋转了90度呢？它只需要旋转30度就可以了啊？那是因为第一段弧时旋转了－60度（逆时针），而第二段是从第一段的基础上旋转过来的，所以需要顺时针旋转90度。
### 刻度弧（紧靠着进度弧内侧的黑色弧）
``` Java
/**
 * 刻度弧
 * @param canvas
 */
private void drawScaleArc(Canvas canvas) {
    // 刻度弧紧靠进度弧
    scaleArcRadius = mSize/2 - (dp2px(15)+dp2px(PADDING)/4);
    canvas.save();
    // 画弧
    RectF rectF = new RectF(-scaleArcRadius, -scaleArcRadius,
            scaleArcRadius, scaleArcRadius);
    canvas.drawArc(rectF, 150, 240, false, scaleArcPaint);
}
```
这里只是简单的画条弧，只要注意半径的把握就可以了。
### 刻度
``` Java
    // 旋转的角度
    float mAngle = 240f / mTikeCount;
    // 画右半部分的刻度
    for (int i = 0; i <= mTikeCount/2; i++) {
        // 5的倍数就画长刻度，并标上刻度值
        if (i % 5 == 0) {
            scale = 20 + i + "";
            panelTextPaint.setTextSize(sp2px(15));
            float scaleWidth = panelTextPaint.measureText(scale);
            canvas.drawLine(0, -scaleArcRadius, 0, -scaleArcRadius+mLongTikeHeight, scalePaint);
            canvas.drawText(scale, -scaleWidth/2, -scaleArcRadius+mLongTikeHeight + dp2px(15), panelTextPaint);
        } else {
            canvas.drawLine(0, -scaleArcRadius, 0, -scaleArcRadius+ mShortTikeHeight, scalePaint);
        }
        canvas.rotate(mAngle, 0, 0);
    }
    // 画布回正
    canvas.rotate(-mAngle * mTikeCount/2 - 6, 0, 0);
    canvas.restore();
}
```
这里只给出了右半部分的刻度，左半部分类似。绘制时先是旋转角度（一共240度，分成40份，每次旋转6度），然后再画直线；画直线时还要判断如果是5的倍数，就画长一点，并且标上刻度值，就是那些0，5，10，15...
这里是右半部分，所以刻度值是从20开始，总共是20个刻度。最后要注意画布回正
__canvas.rotate(-mAngle * mTikeCount/2 - 6, 0, 0)__ 。
### 中间的圆
``` Java
/**
 * 中心圆
 * @param canvas
 */
private void drawInPoint(Canvas canvas) {
    canvas.save();
    canvas.drawCircle(0, 0, pointRadius, pointPaint);
    canvas.restore();
}
```
这个圆是为了凸显接下来的指针更好看一些，这个圆的半径就自行选择咯。
### 指针
``` Java
/**
 * 指针（这里分为左右部分是为了画出来的指针有立体感）
 * @param canvas
 */
private void drawPointer(Canvas canvas) {
    RectF rectF = new RectF(-pointRadius/2, -pointRadius/2, pointRadius/2, pointRadius/2);
    canvas.save();
    // 先将指针与刻度0位置对齐
    canvas.rotate(60, 0, 0);
    float angle = currentTemp * 6.0f;
    canvas.rotate(angle, 0, 0);
    // 指针左半部分
    Path leftPointerPath = new Path();
    leftPointerPath.moveTo(pointRadius/2, 0);
    leftPointerPath.addArc(rectF, 0, 360);
    leftPointerPath.lineTo(0, scaleArcRadius - mLongTikeHeight - dp2px(OFFSET) - dp2px(15));
    leftPointerPath.lineTo(-pointRadius/2, 0);
    leftPointerPath.close();
    // 指针右半部分
    Path rightPointerPath = new Path();
    rightPointerPath.moveTo(-pointRadius/2, 0);
    rightPointerPath.addArc(rectF, 0, -180);
    rightPointerPath.lineTo(0, scaleArcRadius - mLongTikeHeight - dp2px(OFFSET) - dp2px(15));
    rightPointerPath.lineTo(0, pointRadius/2);
    rightPointerPath.close();
    // 指针的圆
    Path circlePath = new Path();
    circlePath.addCircle(0, 0, pointRadius/4, Path.Direction.CW);

    canvas.drawPath(leftPointerPath, leftPointerPaint);
    canvas.drawPath(rightPointerPath, rightPointerPaint);
    canvas.drawPath(circlePath, pointerCirclePaint);
    canvas.restore();
}
```
其实指针部分可以一次性绘制完，但是为了看起来有点立体感，就将它分为左右两部分，但是都是类似的。
就拿左半部分来说：首先将画笔移动到中心圆的边缘（ __leftPointerPath.moveTo(pointRadius/2, 0)__ ），然后来个弧度是让指针的根部为弧形的，接下来是指针的长度（长度＝刻度弧的半径(scaleArcRadius) － 长刻度的长度(mLongTikeHeight) － 刻度值与刻度的间隔(dp2px(15)) － 尾部与刻度值的间隔(dp2px(OFFSET))），最后再将它们连接起来。
指针上的圆意思是指针根部灰色的小圆，这样看起来指针的根部是有个小洞的。
由于之前对这些 Path 的操作不怎么熟悉，所以在这里花的时间较多，而且指针这部分所占的篇幅也是蛮大的，因为难度也比较大嘛。
有兴趣的可以来这学习巩固下，这个系列都很给力:[安卓自定义 View 进阶 - Path 基础](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/%5B5%5DPath_Basic.md)

### 当前温度
``` Java
/**
 * 表盘上的文字
 * @param canvas
 */
private void drawPanelText(Canvas canvas) {
    canvas.save();
    String text = "当前温度";
    float length = panelTextPaint.measureText(text);
    panelTextPaint.setTextSize(sp2px(15));
    canvas.drawText(text, -length/2, scaleArcRadius/2 + dp2px(20), panelTextPaint);
    String temp = currentTemp + " ℃";
    panelTextPaint.setTextSize(sp2px(15));
    float tempTextLength = panelTextPaint.measureText(temp);
    canvas.drawText(temp, -tempTextLength/2, scaleArcRadius, panelTextPaint);
    canvas.restore();
}
```
这部分是表盘上显示的当前温度，首先是绘制__“当前温度”__ 这四个字,然后在下方绘制具体的温度值，注意好文字的位置就可以了，没什么难的。
到这里绘制就结束了，整个表盘也就出来了。其实自定义一个 View 最重要的是把它拆分为几个部分，然后再一部分一部分的绘制出来，至于其中的某些计算部分就得靠自己的细心了，但是多试几次还是可以出来的。
篇幅有限，只是拿出一些关键的代码，有兴趣可以来这看看源码，随便 Star 、Fork [Github:https://github.com/ljuns/TemperatureView](https://github.com/ljuns/TemperatureView)
