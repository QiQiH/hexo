---
title: 浅析View工作原理
date: 2016-08-12 13:15
tags: Android
categories: Android
---

View是android os里一个重要的组成部分，在开发过程中我们经常会用到的一些View组件就有TextView，Button，ListView等等。有些组件经过设计后会展示出更好的视觉效果。那么我们会感到疑惑，一个组件肯定不会凭空产生，那么我们在代码使用了这些组件后它们又经历了一个怎样的过程才使得它们能够在屏幕上展示出来呢？
首先，需要知道的是**View的绘制流程是从ViewRootImpl这个类里的 performTraversals方法开始，**过程中经历了measure, layout, draw三个流程，最终使得组件在屏幕上展现出来。其中，有三个主要的方法依次被执行：**onMeasure, onLayout以及onDraw方法**。
## 一、onMeasure() ##
<!--more-->
看到这个方法名，大概就会知道这个方法有什么作用了，它主要负责视图大小的测量。既然View的绘制流程是从ViewRootImpl这个类里的 performTraversals方法开始，那我们可以看看这个方法的代码，不难发现会有这三条语句：

```
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
//...
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```
从上面可以看到performTraversals会调用performMeasure这个方法，而这个方法的参数来自于之前的getRootMeasureSpec方法，这两个参数这分别用于确定视图的宽度和高度的规格和大小。至于如何确定，则需要先了解一下MeasureSpec这个变量。
MeasureSpec是一个32位的int值，高2位代表SpecMode（测量模式），低30位代表SpecSize（测量规格）。其中SpecMode有三种类型：

 1. UNSPECIFIED
父容器会View不会有任何限制，设置多大即为多大，一般用不上。
 2. EXACTLY
 父容器已经检测出View的精确大小，并且该大小即为SpecSize。它对应于LayoutParams里的macth_parent和精确数值两种模式。
 3. AT_MOST
父容器指定一个可用大小即SpecSize，View的大小不能超过这个数值。它对应于LayoutParams里的wrap_content模式。

那接下里就需要了解一下childWidthMeasureSpec 与 childHeightMeasureSpec 是怎么得来的：

```
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
	        //精确模式，大小即为窗口大小
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
	        //最大模式，大小不确定，且不能超过窗口大小
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
	        //固定数值
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```
对MeasureSpec有一定了解后，再继续看看接下来的performMeasure这个方法：

```
private void performMeasure(int childWidthMeasureSpec, int 	 childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
这里可以看到调用了View里的measure方法，同样也是上述两个参数：

```
 public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	      /**
		  *  省略部分代码
	      */
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
		 /**
		  *  省略部分代码
	      */          
    }
```
注意到measure方法被final修饰了，因此我们并不能重写measure这个方法。可以看到这时候又调用了onMeasure()这个方法：

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```
此处才是真正开始测量View的地方，它会调用getDefaultSize这个方法来获取视图的大小：

```
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        //从MeasureSpec中取出specMode与specSize
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
	        //如果是AT_MOST和EXACTLY，都返回测量值
            result = specSize;
            break;
        }
        return result;
    }
```
通过getDefaultSize来获取视图的大小，然后通过setMeasuredDimension这个方法来设置大小，就完成了一个View的测量。但我们知道一个程序中往往不止一两个View，因此需要遍历的测量每个视图的大小，而这个方法在ViewGroup中就可以找到了：

```
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        //view数组，子视图
        final View[] children = mChildren;
        //遍历子视图，每个子视图执行measure方法，完成整个ViewGroup的测量
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }


protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();
		//获取MeasureSpec值
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);
		//测量child View
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
## 二、onLayout（） ##
 测量过程完成，紧接着就到了布局的过程了。这时候还是要回到ViewRootImpl里的 performTraversals方法中，很容易会发现在performMeasure方法执行过后又执行了performLayout这个方法，以开始布局视图。而与measure过程类似，在执行performLayout时又调用了View的layout方法：

```
 host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
```
继续看看layout方法：

```
/**
* @params l, t, r, b 对应四个顶点位置
*/
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
		
		//设置新位置
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
	        //执行onLayout方法
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
```
可以看到与measure过程如出一辙，又接着调用了onLayout方法，所以接着看看onLayout是怎么实现的：

```
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
```
点进去后却发现里面什么都没有！为什么？原因在于子视图的位置是基于父视图的，即由父视图来执行子视图的布局。父视图那便是ViewGroup，而ViewGroup却是一个抽象方法，显然，具体的实现方法会在继承它的子类中实现，那么有哪些子类呢？这显然不陌生，我们常用的就有LinearLayout, RelativeLayout等等。那就挑个LinearLayout来看看它的onLayout方法：

```
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        //通过以下两个方法可以实现视图的垂直布局与横向布局
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```
由于每个布局都有自己独特的布局方式，对于具体的布局方法不做具体研究。
至此layout的过程也执行完毕。
## 三、onDraw（） ##
通过performMeasure与performLayout这两个过程，视图的测量与布局就已经完成了，剩下最后一步，就是把确定好的视图按照规定的布局画出来，从而展示给用户。那么，performTraversals里也能发现有performDraw这个方法。毫无疑问，performDraw方法里又和其他两个过程一样调用了draw方法：

```
public void draw(Canvas canvas) {  
    if (ViewDebug.TRACE_HIERARCHY) {  
        ViewDebug.trace(this, ViewDebug.HierarchyTraceType.DRAW);  
    }  
    final int privateFlags = mPrivateFlags;  
    final boolean dirtyOpaque = (privateFlags & DIRTY_MASK) == DIRTY_OPAQUE &&  
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);  
    mPrivateFlags = (privateFlags & ~DIRTY_MASK) | DRAWN;  
    // Step 1, draw the background, if needed  
    int saveCount;  
    if (!dirtyOpaque) {  
	    //设置背景
        final Drawable background = mBGDrawable;  
        if (background != null) {  
            final int scrollX = mScrollX;  
            final int scrollY = mScrollY;  
            if (mBackgroundSizeChanged) {  
                background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);  
                mBackgroundSizeChanged = false;  
            }  
            if ((scrollX | scrollY) == 0) {  
		        //绘制背景
                background.draw(canvas);  
            } else {  
                canvas.translate(scrollX, scrollY);  
                background.draw(canvas);  
                canvas.translate(-scrollX, -scrollY);  
            }  
        }  
    }  
    final int viewFlags = mViewFlags;  
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;  
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;  
    if (!verticalEdges && !horizontalEdges) {  
        // Step 3, draw the content  绘制视图内容
        if (!dirtyOpaque) onDraw(canvas);  
        // Step 4, draw the children  绘制子视图
        dispatchDraw(canvas);  
        // Step 6, draw decorations (scrollbars)  绘制装饰 如滚动条
        onDrawScrollBars(canvas);  
        // we're done...  
        return;  
    }  
} 
```
在上面的代码中，我们也会注意到有两个方法：onDraw和dispatchDraw，点进去发现这两个方法都是空方法，显然都要在ViewGroup去寻找这些方法。而对于每个不同的布局，其实现方式的也会有所不同。

经过上述的三个过程，一个视图也就能成功地显示出来了！最后，可以画个图来总结一下来描述一下View的工作原理：
![这里写图片描述](http://img.blog.csdn.net/20160812131449596)

