---
title: 学习总结 -- View 事件分发机制和滑动冲突
date: 2017-03-10 12:52:18
categories: 学习笔记
tags: 事件分发, 滑动冲突

---

终于到了 View 这一关卡了，之前也有实践过自定义 View：[圆弧刻度温度进度条](https://ljuns.github.io/2016/07/27/TemperatureView%EF%BC%9A%E5%9C%86%E5%BC%A7%E5%88%BB%E5%BA%A6%E6%B8%A9%E5%BA%A6%E8%BF%9B%E5%BA%A6%E6%9D%A1/)，但是对于 View 底层的东西没什么了解，只是会用而已，抱着“知其然知其所以然”的心态，很多时候都会先去尝试使用，然后才来究其原因。这次会分两个部分来叙述本篇：事件分发机制、滑动冲突；自己本身对源码也不熟悉，所以本篇主要是理论概述，尽量不出现源码的东西。

<!-- more -->
## 事件分发机制

首先要声明这里用来分析的对象是 MotionEvent，即点击事件。

> 所谓点击事件的事件分发其实就是对 MotionEvent 事件的分发过程，即当一个 MotionEvent 产生了以后，系统需要把这个事件传递给一个具体的 View，而这个传递过程就是分发机制。

了解了分发机制后就来了解另一个概念，同一个事件序列：从手指接触屏幕的那一刻起到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件就叫同一个事件序列。这个事件序列以 down 事件开始，中间含有 n 个 move 事件，最终以 up 事件结束。

知道什么是同一个事件序列对后面的分析有很大的帮助，因为后续很多都是针对同一个事件序列来进行分析的。接下来看看点击事件的分发过程中三个很重要的方法：

**public boolean dispatchTouchEvent(MotionEvent event)**

**public boolean onInterceptTouchEvent(MotionEvent event)**

**public boolean onTouchEvent(MotionEvent event)**

这三个方法的关系就如下面的伪代码：

``` java
public boolean dispatchTouchEvent(MotionEvent event) {
  boolean consume = false;
  if (onInterceptTouchEvent(event)) {
    consume = onTouchEvent(event);
  } else {
    consume = child.dispatchTouchEvent(ev);
  }
  return consume;
}
```

通过这个伪代码可以很清晰的知道这三个方法之间的关系。它们的传递规则：对于一个根 ViewGroup 来说，点击事件产生后，首先会传递给 ViewGroup 本身，此时 ViewGroup 的 dispatchTouchEvent() 方法就会被调用，接着会调用 onInterceptTouchEvent() 方法，如果 onInterceptTouchEvent() 方法返回 true 就表示 ViewGroup 要拦截当前事件；接着再调用 onTouchEvent() 方法来处理该事件；但是如果 onInterceptTouchEvent() 方法返回 false 就表示 ViewGroup 不拦截当前事件，此时 ViewGroup 的 onTouchEvent() 方法就不会被调用，而是调用子 View 的 dispatchTouchEvent() 方法，如此反复直到事件最终被处理。

通常来说那三个方法的执行流程就如上所说的，但是还会有一些比较特殊的情况，比如设置 OnTouchListener、OnClickListener。

当一个 View 需要处理事件时，如果 View 设置了 OnTouchListener，那么 OnTouchListener 中的 onTouch() 方法就会被调用。如果 onTouch() 方法返回 false，则当前 View 的 onTouchEvent() 方法就会被调用；如果返回 true，当前 View 的 onTouchEvent() 方法将不会被调用。

在 onTouchEvent() 方法中，如果有设置 OnClickListener，那么 onClick() 方法是一定会被调用的。

曾经见过一道面试题，详细描述记不清了，大概意思是这样的：onTouch() 和 onTouchEvent() 谁先执行 ？有评论者说如果对 View 进行过比较深入的了解是接触不到这些的。

那么通过前面几段叙述，对事件分发机制应该有个比较清晰的理解了，还有以下几点需要注意：

1. 当一个点击事件产生后，该点击事件的传递遵循如下顺序：Activity -> Window -> View，即事件总是先传递给 Activity，Activity 再传递给 Window，最后 Window 再传递给顶级 View。顶级 View 接收到事件后就会按照事件分发机制去分发事件。
2. ViewGroup 默认不拦截任何事件，即 ViewGroup 的 onInterceptTouchEvent() 方法默认返回false。
3. View 的 onTouchEvent() 默认都会处理事件（返回 true）。
4. 子 View 可以通过 requestDisallowInterceptTouchEvent() 方法干预父 View 的事件分发过程，ACTION_DOWN 事件除外。

## 滑动冲突

在界面中只要内外两层同时可以滑动，这个时候就会产生滑动冲突。常见的滑动冲突场景可以简单的分为以下三种：

> 1. 外部滑动方向和内部滑动方向不一致；
> 2. 外部滑动方向和内部滑动方向一致；
> 3. 上面两种情况的嵌套。

**场景一：** 主要是 ViewPager 和 Fragment 配合使用所组成的页面滑动效果。在这种效果中可以通过 ViewPager 提供的左右滑动来切换页面，而每个页面内部往往又是一个 RecyclerView。按说这种效果是会出现滑动冲突的，但却不用我们去处理，原因是 ViewPager 内部已经处理了滑动冲突，所以我们使用的时候无须关注这个问题。

**场景二：** 如果在一个 ScrollView 中嵌套一个 RecyclerView，那么内外两层都在同一个方向可以滑动，当手指开始滑动的时候就会出现问题，因为系统不知道我们想要滑动的是哪一层。同理，如果内外两层都可以在左右方向滑动也会出现这种情况。

**场景三：** 场景三是场景一和场景二两种情况的嵌套，这种情况更为复杂。

比较常见的滑动冲突就是上面那三种，那么该怎么来解决呢？其实可以根据滑动类型是水平滑动还是竖直滑动来判断由谁来拦截事件。至于如何判断滑动类型就有比较多的方式了：可以依据滑动路径和水平方向所形成的夹角；也可以依据水平方向和竖直方向上的距离差来判断 … 。

滑动类型已经确定了，接下来就是确定滑动的接收者，究竟是谁来响应这个滑动类型？下面介绍两种具体的解决方法：

### 外部拦截

所谓外部拦截就是指点击事件都先经过父容器的拦截处理，如果父容器需要该事件就拦截，否则就不拦截而是交给子容器。外部拦截需要重写父容器的 onInterceptTouchEvent() 方法，在内部做响应的拦截即可。这种方法的伪代码如下：

``` java
public boolean oonInterceptTouchEvent(MotionEvent event) {
  boolean intercepted = false;
  int x = (int) event.getX();
  int y = (int) event.getY();
  switch (event.getAction()) {
    case MotionEvent.ACTION_DOWN:
      intercepted = false;
      break;
    case MotionEvent.ACTION_MOVE:
      if (父容器需要当前点击事件) {
        intercepted = true;
      } else {
        intercepted = false;
      }
      break;
    case MotionEvent.ACTION_UP:
      intercepted = false;
      break;
    default :
      break;
  }
  mLastXIntercept = x;
  mLastYIntercept = y;
  return intercepted;
}
```

在 onInterceptTouchEvent() 方法中，首先是 ACTION_DOWN 事件，父容器必须返回 false，即不拦截 ACTION_DOWN 事件，这是因为一旦父容器拦截了 ACTION_DOWN 事件，那么后续的 ACTION_MOVE 和 ACTION_UP 事件都会直接交由父容器处理，此时事件就不能传递给子元素了；其次是 ACTION_MOVE 事件，这个事件可以根据需要来决定是否拦截，如果父容器需要拦截就返回 true，否则就返回 false；最后是 ACTION_UP 事件，这里必须返回 false，因为 ACTION_UP 事件本身没有太多意义。

### 内部拦截

内部拦截是指父容器默认不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗处理掉，否则就返回给父容器进行处理。这种方法和外部拦截刚好相反，需要配合 requestDisallowInterceptTouchEvent() 方法才能正常工作，此时需要重写子元素的 dispatchTouchEvent() 方法。伪代码如下：

``` java
public boolean dispatchTouchEvent(MotionEvent event) {
  int x = (int) event.getX();
  int y = (int) event.getY();
  switch (event.getAction) {
    case MotionEvent.ACTION_DOWN:
      parent.requestDisallowInterceptTouchEvent(true);
      break;
    case MotinEvent.ACTION_MOVE:
      int deltaX = x - mLastX;
      int deltaY = y - mLastY;
      if (父容器需要此类点击事件) {
        parent.requestDisallowInterceptTouchEvent(false);
      }
      break;
    case MotionEvent.ACTION_UP:
      break;
    default :
      break;
  }
  mLastX = x;
  mLastY = y;
  return super.dispatchTouchEvent(event);
}
```

使用内部拦截除了需要对子元素进行修改以外，父元素也要修改为拦截除了 ACTION_DOWN 以外的其他事件。事件父元素所做的修改如下：

``` java
public boolean onInterceptTouchEvent(MotionEvent event) {
  int action = event.getAction();
  if (action == MotionEvent.ACTION_DOWN) {
    return false;
  } else {
    return true;
  }
}
```

为什么父元素不能拦截 ACTION_DOWN 事件呢？那是因为通过 requestDisallowInterceptTouchEvent() 方法会自动设置一个 FLAG_DISALLOW_INTERCEPT 标记，该标记会导致父元素无法拦截除了 ACTION_DOWN  以外的其他点击事件，而 ACTION_DOWN 事件会重置 FLAG_DISALLOW_INTERCEPT 标记，使之无效。

目前所理解的 View 事件分发机制和滑动冲突就这么多了，只是一些理论概述，接下来还需要好好的实践一番才能进一步掌握，同时也才能发现一些细节上的问题。