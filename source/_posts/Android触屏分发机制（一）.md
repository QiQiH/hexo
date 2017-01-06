---
title: Android触屏分发机制（一）
date: 2016-03-25 13:49
tags: Android 
categories: Android
---

面试学校工作室HCI时曾被要求写类似知乎的界面逻辑，其中遇到了一个问题就是layout的onTouch和button的onClick冲突，解决的方法便是了解触屏分发机制。这段时间查阅了许多资料，具体了解Android的触屏分发机制是怎么实现的。
按钮的点击事件相信大家闭着眼睛都能写得出来了：

```
btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i(TAG, "on click");
            }
        });
```
<!--more-->
如果说再给这个按钮添加一个onTouch事件，会输出怎样的结果呢？

```
btn.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        Log.i(TAG, "on touch action down");
                        break;
                    case MotionEvent.ACTION_UP:
                        Log.i(TAG, "on touch action up");
                        break;
                }
                return false;
            }
        });
```

以下是输出结果：
![这里写图片描述](http://img.blog.csdn.net/20160325130432197)
可以看出,onTouch时间是在onclick之前执行的。我们可能会发现onTouch的方法中有一个返回值false，如果把它改为true，结果会怎样？
![这里写图片描述](http://img.blog.csdn.net/20160325130854808)
此时Button的onClick事件并没有执行！我们可以这样理解，当onTouch时间返回true时，相当于把当前的点击事件消费（拦截）掉了，从而无法把点击事件继续分发下去。那为什么会因为一个返回值而导致结果不同呢？其中View有一个重要的方法dispatchTouchEvent()可以解释其中的原因，当按钮点击时，会首先执行dispatchTouchEvent()这个方法。从字面的意思上理解dispatch有调度，分发的意思，按钮点击后则在这个方法中分发事件。往上查找，我们会发现View中有这个方法，我们看看其方法的主要部分：

```
 public boolean dispatchTouchEvent(MotionEvent event) {
        //省略一部分
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
       //省略一部分
        return result;
    }
```
我们看看第二个if里的三个参数,
mListenerInfo:这个是我们在为Button设置onTouch事件时传递的参数，即new View.OnTouchListener(){//...}，这个值不为null;
第二个参数则是判断按钮是否可用。
第三个参数li.mOnTouchListener.onTouch(this, event)为onTouch事件的返回值。
明显，当第三个参数返回为true时，result = true,否则为false。接着看下一个if语句，意思为当result为false时则执行onTouchEvent(event)这个方法，并赋值result；可见我们的onTouch事件的返回值为false时会执行onTouchEvent(event)方法。那我们在看看onTouchEvent方法是如何实现的：

```
public boolean onTouchEvent(MotionEvent event) {
      //省略一部分
      if (!focusTaken) {
          if (mPerformClick == null) {
              mPerformClick = new PerformClick();
            }
          if (!post(mPerformClick)) {
            performClick();
            }
       }
	//省略一部分
    }
```

其中经过种种判断（由于代码过长，就没有贴出来了），会进入performClick()这个方法：

```
public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```
明显，我们可以看到在第6行执行了onClick事件，而mListenerInfo正是我们在为按钮添加onClick事件时传入的参数！
因此，我们可以总结一下点击按钮后事件是如何分发的：
![这里写图片描述](http://img.blog.csdn.net/20160325134709868)

> Android触屏分发机制（二）http://blog.csdn.net/u010429311/article/details/50979578