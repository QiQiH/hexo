---
title: 自定义ViewGroup
date: 2016-11-20 21:05
tags: Android
categories: Android
---
上一篇了解了如何去实现自定义View，这篇就了解下如何自定义一个ViewGroup。其实，自定义View和ViewGroup都需要从View的绘制机制来入手，可见了解绘图机制对于自定义View和ViewGroup的作用还是很大的。自定义ViewGroup主要在onMeasure和onLayout两个过程下手。通过onMeasure来确定好视图的大小，通过onLayout来确定每个视图的位置。
 这次以设计一个类似LinearLayout的垂直布局为例，主视图里的每个子视图在自定义的ViewGroup中都是垂直排列的。
 依旧要创建一个MyViewGroup让它继承自ViewGroup， 然后重写onMeasure和onLayout两个过程：

<!--more-->
```
package cc.qiblogs.custom;

import android.content.Context;
import android.util.AttributeSet;
import android.view.View;
import android.view.ViewGroup;


public class MyViewGroup extends ViewGroup {
    public MyViewGroup(Context context) {
        super(context);
    }

    public MyViewGroup(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public MyViewGroup(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int wSize = MeasureSpec.getSize(widthMeasureSpec); //宽度
        int hSize = MeasureSpec.getSize(heightMeasureSpec); //水平方向布局的模式
        int wMode = MeasureSpec.getMode(widthMeasureSpec); //高度
        int hMode = MeasureSpec.getMode(heightMeasureSpec); //垂直方向布局的模式

		//如果是wrap_content则使用width和height作为最终的宽高
        int width = 0, height = 0;
        //子视图数量
        int count = getChildCount();
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            //测量子视图
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
            //获取子视图的边距参数
            MarginLayoutParams layoutParams = (MarginLayoutParams) child.getLayoutParams();
            int cWidth = child.getMeasuredWidth() +
                    layoutParams.leftMargin + layoutParams.rightMargin;
            int cHeight = child.getMeasuredHeight() +
                    layoutParams.topMargin + layoutParams.bottomMargin;
            //wrap_content下宽度取最大
            width = Math.max(cWidth, width);
            //父视图的总高度
            height += cHeight;
        }
		//设置测量宽高
        setMeasuredDimension(wMode == MeasureSpec.EXACTLY? wSize : width,
                hMode == MeasureSpec.EXACTLY? hSize : height);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int count = getChildCount();
        //子视图高度之和，用来确定顶点坐标
        int height = 0;
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            MarginLayoutParams layoutParams = (MarginLayoutParams) child.getLayoutParams();
            //边距
            int topMargin = layoutParams.topMargin;
            int leftMargin = layoutParams.leftMargin;
            //子视图宽高
            int cWidth = child.getMeasuredWidth();
            int cHeight = child.getMeasuredHeight();
            //为每个子视图设置四个顶点坐标
            child.layout(leftMargin, topMargin + height, leftMargin + cWidth, topMargin + cHeight + height);
            height += topMargin + cHeight;
        }
    }

	/**
	* 使子视图的layout_margin生效
	*/
    @Override
    protected LayoutParams generateLayoutParams(ViewGroup.LayoutParams p) {
        return new LayoutParams(p);
    }

    @Override
    protected LayoutParams generateDefaultLayoutParams() {
        return new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.WRAP_CONTENT);
    }

    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new LayoutParams(getContext(), attrs);
    }


    public static class LayoutParams extends MarginLayoutParams {
        public LayoutParams(Context c, AttributeSet attrs) {
            super(c, attrs);
        }

        public LayoutParams(int width, int height) {
            super(width, height);
        }

        public LayoutParams(ViewGroup.LayoutParams source) {
            super(source);
        }

        public LayoutParams(ViewGroup.MarginLayoutParams source) {
            super(source);
        }
    }
}

```

  编写XML做个简单的测试：
  

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="cc.qiblogs.custom.MainActivity">
    
    <cc.qiblogs.custom.MyViewGroup
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="#f96363"
        >

        <Button
            android:layout_width="100dp"
            android:layout_height="wrap_content"
            android:text="btn1"
            android:layout_marginLeft="20dp"
            />

        <Button
            android:layout_width="200dp"
            android:layout_height="wrap_content"
            android:text="btn2"
            android:layout_marginRight="30dp"
            />

        <Button
            android:layout_width="70dp"
            android:layout_height="wrap_content"
            android:text="btn3"
            />

    </cc.qiblogs.custom.MyViewGroup>
</RelativeLayout>

```
 结果如图：
![这里写图片描述](http://img.blog.csdn.net/20161120211831413)