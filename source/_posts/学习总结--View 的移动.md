---
title: 学习总结-- View 的移动
date: 2017-01-22 17:04:26
categories: 学习笔记
tags: View 的移动

---
在自定义 View 的时候经常会用到有关 View 的移动，用的比较多的估计是动画，但是除了动画还有没有什么方法可以实现相同的效果呢？有，而且还有好多种方法，这里总结一下目前所了解到的有关 View 的移动方法。在开始之前先来看张图：</br>
<div align=center>
![](http://oaydqd1yy.bkt.clouddn.com/png/2017-01-23%2021.49.43.png)
</div>

<!-- more -->
这张图结合了 Android 坐标系和一些常用的 API（最大的矩形相当于手机屏幕；中间黄色的矩形是 ViewGroup，比如LinearLayout、RelativeLayout...；最中间的是具体的某个 View ，比如 Button、TextView...；小黑点表示触摸点），很直观的阐明了各个 API 的含义，根据这张图分别解释一下各个方法：

> 
* View 提供的获取坐标的方法：
  getLeft()：获取到的是 View 自身的左边到其父布局左边的距离；
  getTop()：获取到的是 View 自身的顶边到其父布局顶边的距离；
  getRight()：获取到的是 View 自身的右边到其父布局左边的距离；
  getBottom()：获取到的是 View 自身的底边到其父布局顶边的距离。
* MotionEvent 提供的获取坐标的方法：
  getX()：获取点击事件距离控件左边的距离，即视图坐标；
  getY()：获取点击事件距离控件顶边的距离，即视图坐标；
  getRawX()：获取点击事件距离整个屏幕左边的距离，即绝对坐标；
  getRawY()：获取点击事件距离整个屏幕顶边的距离，即绝对坐标。

有了上面这些知识点，对于接下来的计算就容易很多了。这次的总结分为以下方面：
1. layout()
2. offsetLeftAndRight() 与 offsetTopAndBottom()
3. setLayoutParams()
4. scrollTo() 与 scrollBy()
5. Scroller
6. 属性动画
7. ViewDragHelper

## layout()
如果对自定义 View 有一定的了解就会知道，在 View 的绘制过程中会调用 onLayout() 方法来设置 View 的位置。那么同样可以通过修改 View 的 left、top、right、bottom 四个属性来控制 View 的位置。下面来看看用 layout() 方法怎么实现 View 的移动：
1. 首先是自定义一个 View ，然后重写 onTouchEvent() 方法，在每次回调 onTouchEvent() 方法的时候获取触摸点的坐标：
``` Java
    // 相对位置
    int x = (int) event.getX();
    int y = (int) event.getY();
```
2. 在 ACTION_DOWN 事件中记录触摸点的坐标：
``` Java
    case MotionEvent.ACTION_DOWN:
        lastX = x;
        lastY = y;
        break;
```
3. 在 ACTION_MOVE 事件中计算偏移量，然后在 View 当前的 left、top、right、bottom 上加上偏移量，最后将相加的结果设置到 layout() 方法中：
``` Java
    case MotionEvent.ACTION_MOVE:
        // 计算偏移量
        int offSetX = x - lastX;
        int offSetY = y - lastY;

        // 在 View 当前的left、top、right、bottom 基础上加上偏移量
        layout(getLeft() + offSetX,
            getTop() + offSetY,
            getRight() + offSetX,
            getBottom() + offSetY);
        break;
```

这里有一点需要提示一下：**layout() 方法的参数顺序是 left、top、right、bottom**。经过上面的三个步骤，每次移动后 View 都会调用 layout() 方法对自己重新布局，从而达到移动 View 的效果。</br>
<div align=center>
![](https://user-gold-cdn.xitu.io/2017/1/23/38b76647713de1e1f99cbf186df93b52)
</div>

下面是 onTouchEvent() 方法完整的代码：

``` Java
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        // 相对位置
        int x = (int) event.getX();
        int y = (int) event.getY();

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = x;
                lastY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算偏移量
                int offSetX = x - lastX;
                int offSetY = y - lastY;

                // 在当前的left、top、right、bottom 基础上加上偏移量
                layout(getLeft() + offSetX,
                        getTop() + offSetY,
                        getRight() + offSetX,
                        getBottom() + offSetY);
                break;
            case MotionEvent.ACTION_UP:
                break;
        }
        return true;
    }
```
在上面的代码中使用的是 getX()、getY() 方法来获取触摸点的坐标值，即使用相对位置。自定义 View 的布局代码：
``` xml
    <cn.ljuns.androidgrowing.practice.DragView
        android:id="@+id/dragView"
        android:layout_width="100dp"
        android:layout_height="100dp">
    </cn.ljuns.androidgrowing.practice.DragView>
```
那可不可以使用 getRawX() 和 getRawY() 方法即绝对位置呢？答案是肯定的，只需要在前面的基础上修改一小部分代码就可以实现同样的效果：
``` Java
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        // 绝对位置
        int rawX = (int) event.getRawX();
        int rawY = (int) event.getRawY();

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = rawX;
                lastY = rawY;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算偏移量
                int offSetX = rawX - lastX;
                int offSetY = rawY - lastY;

                // 在当前的left、top、right、bottom 基础上加上偏移量
                layout(getLeft() + offSetX,
                        getTop() + offSetY,
                        getRight() + offSetX,
                        getBottom() + offSetY);

                // 重新设置坐标
                lastX = rawX;
                lastY = rawY;
                break;
            case MotionEvent.ACTION_UP:
                break;
        }
        return true;
    }
```
主要修改的是：1、在第1步获取触摸点坐标的时候用 getRawX()和 getRawY() 代替 getX()和 getY() ；2、在第3步 ACTION_MOVE 事件中重新设置初始坐标。至于为什么要在最后设置初始坐标，根据一开始给出的图自己比划比划就懂了。
## offsetLeftAndRight() 和 offsetTopAndBottom()
其实根据方法名字很容易猜到这两个方法的意思：左右的偏移、上下的偏移。那该怎么用呢？也很简单，不管是使用相对位置还是绝对位置来计算偏移量，前面的第1步和第2步不变，只需要把 layout() 方法替换为 offsetLeftAndRight() 和 offsetTopAndBottom() 就可以了，即：
``` Java
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        // 相对位置
        int x = (int) event.getX();
        int y = (int) event.getY();

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = x;
                lastY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算偏移量
                int offSetX = x - lastX;
                int offSetY = y - lastY;

                // 同时对 left 和 right 进行偏移
                offsetLeftAndRight(offSetX);
                // 同时对 top 和 bottom 进行偏移
                offsetTopAndBottom(offSetY);
                break;
            case MotionEvent.ACTION_UP:
                break;
        }
        return true;
    }
```
这里是用相对位置计算的偏移量，用绝对位置计算的偏移量也一样可以实现相同的效果，只要记住：**在最后需要重新设置初始坐标。**
## setLayoutParams()
首先我们要知道 LayoutParams 中保存了一个 View 的布局参数，通过改变 LayoutParams 来动态修改一个布局的位置参数也可以实现前面的效果。前面的第1、2步依然不变，只需修改第3步：
``` Java
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        // 相对位置
        int x = (int) event.getX();
        int y = (int) event.getY();

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = x;
                lastY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算偏移量
                int offSetX = x - lastX;
                int offSetY = y - lastY;

                /**
                 * LayoutParams 主要是通过修改 margin 来修改 view 的位置
                 */
                LinearLayout.LayoutParams params =
                        (LinearLayout.LayoutParams) getLayoutParams();
                params.leftMargin = getLeft() + offSetX;
                params.topMargin = getTop() + offSetY;
                setLayoutParams(params);
                break;
            case MotionEvent.ACTION_UP:
                break;
        }
        return true;
    }
```
## scrollTo() 和 scrollBy()
在一个 View 中，系统提供了 scrollTo() 和 scrollBy() 两种方法来改变一个 View 的位置。这两个方法的区别是：scrollTo(x, y) 表示移动到一个具体的坐标点（x, y）；scrollBy(x, y) 表示移动的偏移量为 x、y。与前面几种方式相同，只需修改第3步的关键方法就可以实现相同的效果：
``` Java
    @Override
    public boolean onTouchEvent(MotionEvent event) {

        // 相对位置
//        int x = (int) event.getX();
//        int y = (int) event.getY();

        // 绝对位置
        int rawX = (int) event.getRawX();
        int rawY = (int) event.getRawY();

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
//                lastX = x;
//                lastY = y;
                lastX = rawX;
                lastY = rawY;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算偏移量
//                int offSetX = x - lastX;
//                int offSetY = y - lastY;
                int offSetX = rawX - lastX;
                int offSetY = rawY - lastY;

               ((View)getParent()).scrollBy(-offSetX, -offSetY);
    //          ((View)getParent()).scrollTo(-offSetX, -offSetY);
                break;
            case MotionEvent.ACTION_UP:
                break;
        }
        return true;
    }
```
懵逼了吧？为毛是 `((View)getParent()).scrollBy(-offSetX, -offSetY)` 而不是 `scrollBy(offSetX, offSetY)` ？？为毛是 `(-offSetX, -offSetY)` ？？
第一个问题：因为 scrollTo() 和 scrollBy() 方法移动的是 View 的 content，即移动的是 View 的内容。例如 TextView 的 content 就是它的文本，所以如果要移动某个 View ，那么就要在 View 所在的 ViewGroup 中使用 scrollTo()、scrollBy() 方法。明白了第一个问题，第二个问题也就迎刃而解了：因为 scrollTo()、scrollBy() 方法作用在 ViewGroup 上，所以要往反方向移动才能实现我们需要的效果。
## Scroller
首先来看个效果图：</br>
<div align=center>
![](https://user-gold-cdn.xitu.io/2017/1/23/d9e1662485939d4cd7f9545f9e5690d6)
</div>

当我鼠标松开时，蓝色的矩形会平滑的移动，这是怎么做到的呢？
由于在 ACTION_MOVE 事件中不断获取手指移动的微小的偏移量，这样就将一段距离划分成了 N 个非常小的偏移量，在每个小的偏移量里面通过调用 scrollTo() 方法进行了移动。因为人眼的视觉暂留特性，使得在整体上是一个平滑移动的效果。
这就是 Scroller ，接下来看看代码是怎么实现的：
* 创建 Scroller 对象
``` Java
    // 初始化 Scroller
    mScroller = new Scroller(context);
```
* 重写 computeScroll() 方法
``` Java
    /**
     * 核心方法，该方法是个空方法,实质是通过调用 scrollTo 实现移动
     */
    @Override
    public void computeScroll() {
        super.computeScroll();
        // 判断 Scroller 是否执行完毕
        if (mScroller.computeScrollOffset()) {
            ((View)getParent()).scrollTo(
                    mScroller.getCurrX(),
                    mScroller.getCurrY());
            // 通过重绘来不断调用 computeScroll()
            invalidate();
        }
    }
```
computeScroll() 方法是使用Scroller 类的核心，系统在绘制 View 的时候会在 draw() 方法中调用该方法。Scroller 类提供了 computeScrollOffset() 方法来判断是否完成了整个滑动，同时也提供了 getCurrX()、getCurrY() 方法来获得当前的滑动坐标。
* 使用 startScroll() 开启平滑移动
``` Java
    View viewGroup = (View) getParent();
    // 启动
    mScroller.startScroll(viewGroup.getScrollX(),
            viewGroup.getScrollY(),
            -viewGroup.getScrollX(),
            -viewGroup.getScrollY(),
            3000);
    invalidate();
```
这里给它设置了一个时长：3000，是平滑移动的时长，当然也可以省略。最重要的是 computeScroll() 方法不会自动调用，只能通过 invalidate() -> draw() -> computeScroll() 来间接调用 computeScroll() ，所以一定要在最后调用 invalidate() 方法。
总的执行流程是这样的：startScroll() -> invalidate() -> draw() -> computeScroll() -> invalidate() -> draw() -> computeScroll()... 就这样一直循环下去直到结束。整个自定义 View 的完整代码：
``` Java
    public class DragView extends View {

        private int lastX;
        private int lastY;
        private Scroller mScroller;

        public DragView6(Context context) {
            this(context, null);
        }

        public DragView6(Context context, AttributeSet attrs) {
            super(context, attrs);
            // 设置背景色
            setBackgroundColor(Color.BLUE);
            mScroller = new Scroller(context);
        }

        /**
        * 核心方法，该方法是个空方法，实质是通过调用 scrollTo 实现移动
         */
        @Override
        public void computeScroll() {
            super.computeScroll();
            // 判断 Scroller 是否执行完毕
            if (mScroller.computeScrollOffset()) {
                ((View)getParent()).scrollTo(
                        mScroller.getCurrX(),
                        mScroller.getCurrY());
                // 通过重绘来不断调用 computeScroll
                postInvalidate();
            }
        }

        @Override
        public boolean onTouchEvent(MotionEvent event) {

            // 相对位置
            int x = (int) event.getX();
            int y = (int) event.getY();

            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    lastX = x;
                    lastY = y;
                    break;
                case MotionEvent.ACTION_MOVE:
                    // 计算偏移量
                    int offSetX = x - lastX;
                    int offSetY = y - lastY;

                    ((View)getParent()).scrollBy(-offSetX, -offSetY);
                    break;
                case MotionEvent.ACTION_UP:
                    View viewGroup = (View) getParent();
                    // 启动
                    mScroller.startScroll(viewGroup.getScrollX(),
                            viewGroup.getScrollY(),
                            -viewGroup.getScrollX(),
                            -viewGroup.getScrollY(),
                            3000);
                    invalidate();
                    break;
            }
            return true;
        }
    }
```

## 属性动画
之前写了一篇属性动画的总结：[学习总结--属性动画](http://blog.csdn.net/ljuns/article/details/54426420)，用属性动画来实现 View 的移动会更简单，先获取到需要移动的 View ，然后给它设置动画：
``` Java
    //属性动画
    dragView = (DragView) findViewById(R.id.dragView);
    ObjectAnimator animator1 = ObjectAnimator.ofFloat(dragView,
                "translationX", 0, 200);
    ObjectAnimator animator2 = ObjectAnimator.ofFloat(dragView,
                "translationY", 0, 200);
    AnimatorSet set = new AnimatorSet();
    set.playTogether(animator1, animator2);
    set.setDuration(3000);
    set.start();
```
这里是属性动画组合，View 会从坐标 (0, 0) 平滑移动到坐标 (200, 200)，持续时长是3000ms(也就是3秒)。

## ViewDragHelper
在 support 库中有 DrawerLayout 和 SlidingPaneLayout 两个布局可以实现侧边栏的滑动，而它们的核心就是 ViewDragHelper 类，通过 ViewDragHelper 基本可以实现各种不同的滑动、拖放的需求。依然是前面的效果：
* 初始化 ViewDragHelper
``` Java
    // 初始化
    mHelper = ViewDragHelper.create(this, callback);
```
第一个参数是要监听的 View，通常是一个 ViewGroup ；第二个参数是一个 Callback 回调。
* 拦截事件
  重写 onInterceptTouchEvent() 和 onTouchEvent() 方法。如果不了解这两个方法，得先去了解下 Android 的事件拦截机制。
``` Java
    /**
     * 事件拦截
     */
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return mHelper.shouldInterceptTouchEvent(ev);
    }

    /**
     * 事件处理
     */
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // 将触摸事件传递给 ViewDragHelper
        mHelper.processTouchEvent(event);
        return true;
    }
```
* 重写 computeScroll()
``` Java
    @Override
    public void computeScroll() {
        if (mHelper.continueSettling(true)) {
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }
```
computeScroll() 方法在前面 Scroller 类的时候有提到过，ViewGroupHelper 内部也是通过 Scroller 来实现平滑移动的。
* Callback 回调
``` Java
    private class HelperCallback extends ViewDragHelper.Callback{

        /**
         * 检测触摸事件
         */
        @Override
        public boolean tryCaptureView(View child, int pointerId) {
            // 如果当前触摸的 View 是 mView 就开始检测触摸事件
            return mView == child;
        }

        @Override
        public int clampViewPositionVertical(View child,
                                int top, int dy) {
            return top;
        }

        @Override
        public int clampViewPositionHorizontal(View child,
                                int left, int dx) {
            return left;
        }
    }
```
通过 tryCaptureView() 方法可以指定哪一个子 View 可以移动；clampViewPositionVertical() 和 clampViewPositionHorizontal() 分别对应垂直和水平方向上的滑动。它们的默认返回值是0，即不滑动。
* 加载布局
``` Java
    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mView = getChildAt(0);
    }
```
通过 getChildAt() 方法按顺序来加载子 View。

其实 ViewDragHelper 还有更多更复杂的用法，可以实现更炫的效果，感兴趣的可以自己去搜索一下相关的文章。
上面一共总结了7种方法可以实现 View 的移动，这篇学习总结也就到这了。这是第二篇学习总结，接下来会继续学习继续总结。
