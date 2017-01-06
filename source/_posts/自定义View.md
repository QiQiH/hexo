---
title: 自定义View
date: 2016-11-20 20:47
tags: Android
categories: Android
---
花了几个小时的时间总算把自定义View给理清楚了，以前总是没时间去看，网上总是一大把写好的控件。![这里写图片描述](http://www.easyicon.net/api/resizeApi.php?id=1202904&size=32)
在学习自定义View之前还需要回顾一下关于View的绘图流程相关知识。View的绘图流程有三个onDraw，onLayout还有onDraw。可以这样简单地描述这三个过程：

> onMeasure:计算出视图的大小。 
> onLayout:确定试图在屏幕中的显示位置。  
> onDraw:将控件绘制出来。

此篇讲到的自定义View是从onDraw方法入手，通过自己的设计将想要的图形绘制出来。以一个圆形视图为例，如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161120211654317)
<!--more-->
首先，需要做的是让这个圆形视图继承自View，命名为CircleView:

```
public class CircleView extends View {
    private int color = Color.GREEN;

    public CircleView(Context context) {
        super(context);
    }

    public CircleView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onDraw(Canvas canvas) {
		//创建画笔
        Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(color);
        int width = getWidth();
        int height = getHeight();
        int radius = Math.min(width, height) / 2;
        //在画布上画出一个圆
        canvas.drawCircle(width /2, height / 2, radius, paint);
    }
}
```
 有一点需要注意，直接继承View或ViewGroup的控件，padding是默认无法生效的，即使添加了padding属性也不会有所变化。想要padding生效,需要在onDraw方法上适当的修改：
 

```
	@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(color);
        int paddingLeft = getPaddingLeft();
        int paddingRight = getPaddingRight();
        int paddingTop = getPaddingTop();
        int paddingBottom = getPaddingBottom();
        int width = getWidth() - paddingLeft - paddingRight;
        int height = getHeight() - paddingTop - paddingBottom;
        int radius = Math.min(width, height) /2;
        canvas.drawCircle(paddingLeft + width / 2, paddingTop + height / 2, radius, paint);
    }
```
 如果只是像上述那样自定义一个View，往往满足不了需求。假设我需要绘制其他颜色的圆形，这个属性就需要手动在XML中定义。这时候就需要添加自定义属性。
 首先，需要在value文件夹创建一个资源文件，（任意）命名为 circle_view.xml:
 

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="CircleView">
	    <!--添加circle_color属性-->
        <attr name="circle_color" format="color"/>
    </declare-styleable>
</resources>
```
 接着，在Xml文件中添加自定义的属性：
 

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="cc.qiblogs.custom.MainActivity">

    <cc.qiblogs.custom.CircleView
        android:layout_width="100dp"
        android:layout_height="100dp"
        app:circle_color="#000000"
        />
</RelativeLayout>

```

 然后，在自定义View中解析这个属性：
 

```
	public CircleView(Context context) {
        super(context);
    }

    public CircleView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context, attrs);
    }

    public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context, attrs);
    }

    private void init(Context context, AttributeSet attrs) {
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleView);
        color = a.getColor(R.styleable.CircleView_circle_color, Color.GREEN);
        a.recycle();
    }
```
 经测试，达到了理想的效果。
 ![这里写图片描述](http://img.blog.csdn.net/20161120211719927)