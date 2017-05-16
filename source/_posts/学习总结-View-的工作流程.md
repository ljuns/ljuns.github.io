---
title: 学习总结-- View 的工作流程
date: 2017-04-06 12:49:41
categories: 学习笔记
tags: View 的工作流程
---

对于新手来说，自定义 View 无疑是一重点难点(自己也还是菜鸟)，之前也尝试过一些自定义 View，虽然效果大概是那么回事，但是还是有很大的局限性，特别是现在回过头去看看，有些细节方面处理得并不够。因为一开始对 View 的认识并不够，缺乏根本上的认识。所以这次的总结分为 4 个点，都是在自定义 View 时需要注意并且自定义一个好的 View 不可或缺的知识点。 

<!-- more -->

> View 的工作流程主要是指 measure、layout、draw 这三大流程，即测量、布局和绘制，其中 measure 确定 View  的测量宽高，layout 确定 View 的最终宽高和四个顶点的位置，而 draw 则将 View 绘制到屏幕上。

在说 View 的三大流程前先来个知识储备：MeasureSpec。MeasureSpec 是什么？它是一个短小精悍的类，通过它可以帮助我们进行测量 View。确切的说 MeasureSpec 在很大程度上决定了一个 View 的尺寸规格，但并不是绝对的，因为 View  的 MeasureSpec 的创建过程还受父容器的影响。

## MeasureSpec

MeasureSpec 是一个 32 位的值，其中高 2 位为测量模式(即 specMode)，低 30 位为测量的大小(即 specSize)，在计算中使用位运算是为了提高和优化效率。

specMode 分为以下三种：

* UNSPECIFIED

  这种情况时父容器不对 View 有任何限制，要多大给多大，一般用于系统内部。

* EXACTLY

  精确值模式，当控件的 layout_width 属性和 layout_height 属性指定为确切的值或是 match_parent 的时候对应这种情况。

* AT_MOST

  最大值模式，当控件的 layout_width 属性和 layout_height 属性指定为 wrap_content 时对应这种情况。

这里主要是了解三种测量模式分别对应哪些情况(其实是为了说明 EXACTLY 和 AT_MOST 对应的情况)，为接下来的讲述做准备。

## measure

measure 过程要分为单纯的 View 和 ViewGroup 两种情况来说，因为他们的内部实现不太一样。

### 1. View 的 measure 过程

View 的 measure() 方法是一个 final 类型的方法，子类不能重写此方法，而在 measure() 方法内部会调用 View 的 onMeasure() 方法，所以一般情况下自定义一个 View 都是重写 onMeasure() 方法。先看一下源码中 onMeasure() 方法的实现：

``` java
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {							setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                  getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
  	}
```

onMeasure() 方法还是比较简洁的：setMeasureDimension() 方法是设置 View 宽\高的测量值，接下来再看看源码中 getDefaultSize() 方法的实现：

``` java
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                result = size;
                break;
            case MeasureSpec.AT_MOST: // wrap_content
            case MeasureSpec.EXACTLY: // math_parent、确切的数值
                result = specSize;
                break;
        }
        return result;
    }
```

UNSPECIFIED 一般用于系统内部的测量过程，这里就不展开说了。所以 getDefaultSize() 方法可以简单的理解为：它的返回值就是 measureSpec 中的 specSize。

从 getDefaultSize() 方法也可以看出当自定义控件设置为 wrap_content 和 match_parent 时效果是一样的，所以直接继承 View 的自定义控件需要重写 onMeasure() 方法，并且设置 wrap_content 时自身的大小。怎么解决呢？这里提供一个方案，在重写 onMeasure() 方法的时候处理：

``` java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int width = startMeasure(widthMeasureSpec);
        int height = startMeasure(heightMeasureSpec);
        
        setMeasuredDimension(width, height);
    }

    private int startMeasure(int spec) {
        int result = 0;
        int specSize = MeasureSpec.getSize(spec);
        int specMode = MeasureSpec.getMode(spec);

        if (specMode == MeasureSpec.EXACTLY) {
            result = specSize;
        } else {
            result = 200;
            if (specMode == MeasureSpec.AT_MOST) {
                result = Math.min(result, specSize);
            }
        }
        return result;
    }
```

上述 startMeasure() 方法中的 200 这个数值不是固定的，随意给就好了。总的来说，startMeasure() 方法对 specMode 的三种模式都做了处理，这样在自定义控件中也可以使用 wrap_content 这个属性值了。

### ViewGroup 的 measure 过程

> 对于 ViewGroup 来说，除了完成自己的 measure 过程以外，还会遍历调用所有子元素的 measure() 方法，各个子元素再递归去执行这个过程。

ViewGroup 是一个抽象类，没有重写 View 的 onMeasure() 方法，而是提供了一个 measureChildren() 方法，源码中的 measureChildren() 如下：

``` java
	protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```

measureChildren() 方法中会遍历所有子元素，并对子元素进行 measure (上述代码中的 measureChild() 方法)。再来看看 measureChild() 方法的实现：

``` java
	protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

这个方法也还是比较好理解的，首先是取出子元素的 LayoutParams，然后再通过 getChildMeasureSpec() 方法来创建子元素的 MeasureSpec，最后将 MeasureSpec 直接传递给 View 的 measure() 方法进行测量，那么就是上面说过的 View 的 measure 过程了。

在某些情况下，系统可能需要多次 measure 才能确定最终的测量宽高，所以可以在 onLayout() 方法中通过 getMeasureWidth() \ getMeasureHeight() 去获取 View 测量后的宽高，此时获取到的值是准确的。

## layout

layout 的作用是 ViewGroup 用来确定子元素的位置，当 ViewGroup 的位置被确定后，ViewGroup 在 onLayout() 方法中会遍历所有的子元素并调用子元素的 layout() 方法，在子元素的 layout() 方法中子元素的 onLayout() 方法又会被调用。

ViewGroup 的位置是怎么确定的呢？其实在 ViewGroup 的 layout() 方法中会通过 super.layout() 来调用父类 View 的 layout() 方法，所以 ViewGroup 位置确定的具体实现是在 View 的 layout() 中，先来看看 View 的 layout() 方法：

``` java
    @SuppressWarnings({"unchecked"})
    public void layout(int l, int t, int r, int b) {
        ...
          boolean changed = isLayoutModeOptical(mParent) ?
                  setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

          if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
              onLayout(changed, l, t, r, b);
              mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

              ListenerInfo li = mListenerInfo;
              if (li != null && li.mOnLayoutChangeListeners != null) {
                  ArrayList<OnLayoutChangeListener> listenersCopy =
                  (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                  int numListeners = listenersCopy.size();
                  for (int i = 0; i < numListeners; ++i) {
                      listenersCopy.get(i)
                        .onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                  }
              }
          }
  		...       
    }
```

上述方法省略了一些无关代码，在该方法中出现了 setOpticalFrame() 和 setFrame() 这两个方法，它们又是干什么的呢？其实 setOpticalFrame() 内部也是调用了 setFrame() 方法的，那就看看 setFrame() 方法：

``` java
    protected boolean setFrame(int left, int top, int right, int bottom) {
        ...
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
		...
    }
```

这里 setFrame() 方法只给出了主要代码，它的主要作用也就是初始化 mLeft、mTop、mRight、mBottom 这四个值。

所以 View 的 layout() 方法可以简单理解为：首先是通过 setFrame() 方法来设定 View ( ViewGroup ) 的四个顶点的位置，也就是在此时确定了 ViewGroup 的位置；然后再调用 onLayout() 方法，这个方法是用来确定子元素的位置。

由于 View 的 onLayout() 方法是个空方法，所以找 Veiw 的子类 LinearLayout ( RelativeLayout 也可以) 的 onLayout() 方法来看看：

``` java
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```

这个 onLayout() 方法很简洁，接下来选择 layoutVertical() 方法继续往下走：

``` java
    void layoutVertical(int left, int top, int right, int bottom) {
        ...
        final int count = getVirtualChildCount();
		...
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();           
                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                ...
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
              	...
            }
        }
    }
```

layoutVertical() 只给出了主要代码，它的主要作用就是遍历所有子元素并通过 getMeasuredWidth() 和 getMeasuredHeight() 方法拿到子元素的测量宽高，最后再调用 setChildFrame() 来为子元素指定位置。setChildFrame() 方法：

``` java
    private void setChildFrame(View child, int left, int top, int width, int height) {        
        child.layout(left, top, left + width, top + height);
    }
```

setChildFrame() 方法仅仅是调用子元素的 layout() 方法而已，这样父元素在 layout() 方法中确定自己的位置后就通过 onLayout() 方法去调用子元素的 layout() 方法，子元素又会通过 layout() 方法确定自己的位置。这样一层一层地传递下去就完成了整个 View 树的 layout 过程。 

## draw

相对来说，draw 的过程就比较简单了，它的作用是将 View 绘制到屏幕上。先来看看 draw() 方法的实现：

``` java
    @CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = 
          (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE && 
          (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // we're done...
            return;
        }
      ...
    }
```

通过 View 的 draw() 方法可以明显的看出它有 6 个步骤，其中步骤 2 和步骤 5 可能会被跳过，所以主要看剩下的 4 个：

1. 绘制背景：drawBackground(canvas)；
2. 绘制自己：onDraw(canvas);
3. 绘制 children：dispatchDraw(canvas);
4. 绘制装饰器，比如 scrollBar：onDrawForeground(canvas);

View 绘制流程的传递是通过 dispatchDraw() 方法来实现的，dispatchDraw() 方法会遍历调用所有子元素的 draw() 方法，这样 draw 过程就一层一层地传递下去了。

到这里，View 的整个工作流程就差不多了，从 measure 到 layout 再到 draw，每个过程都会去遍历调用子元素相对应的方法来完成工作，从而完成一整个 View 的工作。