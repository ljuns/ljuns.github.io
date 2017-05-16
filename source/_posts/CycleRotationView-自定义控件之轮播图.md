---
title: 'CycleRotationView：自定义控件之轮播图'
date: 2016-11-21 16:10:31
categories: 自定义 View
tags: 自定义控件

---
CycleRotationView：自定义控件，主要功能是实现类似与各种商城首页的广告轮播图。其实像这种比较常见的自定义控件早就满大街了，虽然说“不要重复发明轮子”，但是不代表不用关心轮子是怎么造的，本着“知其然知其所以然”的态度，完成了这个控件(总不能需要用到时才来学习吧)。老规矩，先来个效果图(没有效果图的文章都是耍流氓)：<br/>
<div align=center>
![](http://oaydqd1yy.bkt.clouddn.com/gif/cycle.gif)
</div>
<!-- more -->
对于轮播图的图片切换，大家立马会想到 ViewPager 这个控件，因为它已经拥有左右滑动的功能，用来实现轮播图效果会简单一些，本篇也是基于 ViewPager 这个控件的实现，将其分成下面四个部分：
1、图片轮播
2、指示器
3、图片指示器联动
4、定时轮播
下面开始实现(有点啰嗦，勿喷...)：
### 图片轮播
#### 布局文件
``` xml
<RelativeLayout
       android:layout_width="match_parent"
       android:layout_height="120dp">
       <android.support.v4.view.ViewPager
           android:id="@+id/viewPager"
           android:layout_width="match_parent"
           android:layout_height="120dp" />
       <LinearLayout
           android:layout_width="match_parent"
           android:layout_height="18dp"
           android:layout_alignParentBottom="true"
           android:background="#66000000"
           android:orientation="horizontal">
           <TextView
               android:id="@+id/title"
               android:layout_width="0dp"
               android:layout_height="wrap_content"
               android:layout_marginLeft="15dp"
               android:layout_weight="1"
               android:textColor="#FFFFFF" />
           <LinearLayout
               android:id="@+id/pointGroup"
               android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:layout_gravity="center"
               android:layout_marginRight="15dp"
               android:orientation="horizontal" />
       </LinearLayout>
   </RelativeLayout>
   ```
   先来看看布局文件，最外层的 RelativeLayout 代表整个控件，包含 ViewPager 和一个 LinearLayout；在 LinearLayout 中又包含 TextView 和 LinearLayout。这里说明一下：第一个 LinearLayout 包含标题(id=title)和指示器(id=pointGroup)，第二个 LinearLayout 用来包裹指示器(在代码里完成)，给张理解图(这里我并没有设置标题)：
   <div align=center>
![](http://oaydqd1yy.bkt.clouddn.com/png/cycle_show.png)
</div>
   #### 代码实现
``` Java
    /**
     * 初始化布局
     * @param mContext
     */
    private void initView(Context mContext) {
        View view = LayoutInflater.from(mContext)
                .inflate(R.layout.cycle_rotation_layout, this, true);
        mViewPager = (ViewPager) view.findViewById(R.id.viewPager);
        mPointGroup = (LinearLayout)view.findViewById(R.id.pointGroup);
    }
```
首先是加载刚才的布局文件，把 ViewPager 和 pointGroup 找出来；
``` Java
 	for (int i = 0; i < images.length; i++) {
        // 创建 ImageView，并设置图片
        ImageView img = new ImageView(mContext);
        // 把图片扩大到 ImageView 的大小
        img.setScaleType(ImageView.ScaleType.FIT_XY);
        img.setImageResource(images[i]);
        mList.add(img);
        }
```
根据用户传递过来的图片的数量进行遍历创建对应数量的 ImageView ，然后给创建的 ImageView 设置图片，再用一个 List 集合把 ImageView 装起来待用。在 Fragement + ViewPager + TabLayout 的场景里 ViewPager 里填充的是 Fragement，而这里的 ViewPager 填充的是 ImageView；
``` Java
	/**
    * 适配器
    */
    class CycleAdapter extends PagerAdapter {
    
        @Override
        public int getCount() {
            return Integer.MAX_VALUE;
        }
        
        @Override
        public Object instantiateItem(ViewGroup container, int position) {
            position = position % mList.size();
            // 判断该 item 是否已经有 parent
            final View child = mList.get(position);
            if (child.getParent() != null) {
                container.removeView(child);
            }
            container.addView(mList.get(position));
            return mList.get(position);
        }
        
        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {}
        
        @Override
        public boolean isViewFromObject(View view, Object object) {
            return view == object;
        }
    }
```
适配器上场了，在 getCount() 方法里返回 Integer.MAX_VALUE 是因为 ViewPager 不会循环滑动，给这么一个数值(据说 Integer.MAX_VALUE 的值是 10亿，我没试过)可以让它一直滑下去。问题来了：这么大的数值，会不会出现性能问题？这个我也好奇，看谁更有兴趣咯。
还要说一下 instantiateItem() 方法，为什么要判断 parent 的状态？原因是如果不这样处理，在向左滑动 ViewPager 的时候会报错：<font color=#FF0000> The specified child already has a parent. You must call removeView </font>。
<font color=#FF0000>__注意：__</font>instantiateItem() 进行了判断，destroyItem() 就不用再 removeView() 了，否则会出现空白页。
### 指示器
``` Java
 	/**
    * 创建指示器
    */
    for(int i = 0; i < images.length; i ++) {
        // 创建指示器，实质也是 ImageView
        ImageView point = new ImageView(mContext);
        point.setImageResource(R.drawable.shape_point_selector);
        // 指示器布局参数
        LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(pointSize, pointSize);
        // 第2个起才设置左边距
        if (i > 0) {
            params.leftMargin = pointMargin;
            point.setSelected(false); // 默认选中第1个
        } else {
            point.setSelected(true); // 默认选中第1个
        }
        point.setLayoutParams(params); // 给指示器设置参数
        mPointGroup.addView(point); // 把指示器添加到容器
    }
```
创建指示器的时候为了和 ImageView 的数量对应，所以同样要遍历用户设置的图片数量。下面是指示器的相关布局文件：
shape_point_selector:
```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_selected="true" android:drawable="@drawable/shape_point_selected" />
    <item android:state_selected="false" android:drawable="@drawable/shape_point_normal" />
</selector>
```
shape_point_selected:
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
    <solid android:color="#FFFFFF"/>
</shape>
```
shape_point_normal:
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
    <solid android:color="#66000000"/>
</shape>
```
ViewPager 和 pointGroup 都创建好了，接下来是他们俩的联动：
### 联动
``` Java
 mViewPager.setAdapter(new CycleAdapter());

        // ViewPager 的监听事件
        mViewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {}

            int lastPosition;
            @Override
            public void onPageSelected(int position) {
                position = position % mList.size();
                // 设置当前圆点选中
                mPointGroup.getChildAt(position).setSelected(true);
                // 设置前一个圆点不选中
                mPointGroup.getChildAt(lastPosition).setSelected(false);
                lastPosition = position;
            }

            @Override
            public void onPageScrollStateChanged(int state) {}
        });
```
给 ViewPager 添加监听事件，在选中某个页面的同时设置相应位置指示器也被选中。position % mList.size()：Adapter 的 getCount() 中返回了很大的值，这里求余可以保证不会出现空指针异常。
### 定时轮播
``` Java
 mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                // 当前选中的 item
                int currentItem = mViewPager.getCurrentItem();
                // 判断是否是最后一个 item
                if (currentItem == mViewPager.getAdapter().getCount() - 1) {
                    mViewPager.setCurrentItem(1);
                } else {
                    mViewPager.setCurrentItem(currentItem + 1);
                }
                // 不断给自己发消息
                mHandler.postDelayed(this, 3000);
            }
        }, 3000);
```
定时轮播也很简单，用 Handler 来延时给自己发消息。在 ViewPager 滑动到最后一个时给它设置 mViewPager.setCurrentItem(1) 来实现循环轮播。
基本的轮播功能到这里就完成了，但是还有很多的拓展功能，比如点击事件，用户设置图片链接，这些我都已经实现了，这里就不一一介绍了，有兴趣的可以看看源码：[cycleRotation](https://github.com/ljuns/AndroidGrowing)

