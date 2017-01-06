---
title: 汇实现带右滑功能的ListView
date: 2016-08-01 19:06 
tags: Android
categories: Android
---
近来看到QQ的联系列表的右滑功能很不错，于是就想着自己也设计一下，看看怎样才能实现这种右滑的功能，参考了网上的一些资料，搞定了各种bug，终于实现了一个简单的带右滑功能的ListView。先附上效果图吧：
![这里写图片描述](http://img.blog.csdn.net/20160801190917119)
<!--more-->
以下是实现该功能的源码：
SwipeListView.java
```
package com.custom;

import android.content.Context;
import android.util.AttributeSet;
import android.util.DisplayMetrics;
import android.view.MotionEvent;
import android.view.ViewGroup;
import android.view.WindowManager;
import android.widget.LinearLayout;
import android.widget.ListView;

/**
 * Created by Administrator on 2016/7/30.
 */
public class SwipeListView extends ListView {
    /**
     * 屏幕宽度
     */
    private int mScreenWidth;
    /**
     * 按下的坐标
     */
    private int mDownX, mDownY;
    /**
     * 两个功能按钮的宽度
     */
    private int mDelBtnWidth, mTopBtnWidth;
    /**
     * 目标View
     */
    private ViewGroup mSelectedView;
    /**
     * 需要移动的布局
     */
    private LinearLayout.LayoutParams mLayoutParams;

    /**
     * 显示状态？
     */
    private boolean isBtnShow;

    public SwipeListView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public SwipeListView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        //获取屏幕宽度
        WindowManager wm = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        DisplayMetrics dm = new DisplayMetrics();
        wm.getDefaultDisplay().getMetrics(dm);
        mScreenWidth = dm.widthPixels;
    }

    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                performActionDown(ev);
                break;
            case MotionEvent.ACTION_UP:
                performActionUp();
                break;
            case MotionEvent.ACTION_MOVE:
                return performActionMove(ev);
        }

        return super.onTouchEvent(ev);
    }

    /**
     * 处理ActionDown事件
     * @param ev
     */
    private void performActionDown(MotionEvent ev) {
        if (isBtnShow) {
            recovery();
        }
        mDownX = (int) ev.getX();
        mDownY = (int) ev.getY();
        //获取选中的View
        mSelectedView = (ViewGroup) getChildAt(pointToPosition(mDownX, mDownY) - getFirstVisiblePosition());
        //获取按钮长度
        mDelBtnWidth = mSelectedView.getChildAt(2).getLayoutParams().width;
        mTopBtnWidth = mSelectedView.getChildAt(1).getLayoutParams().width;
        //获取需要移动的布局
        mLayoutParams = (LinearLayout.LayoutParams) mSelectedView.getChildAt(0).getLayoutParams();
        mLayoutParams.width = mScreenWidth;
        mSelectedView.getChildAt(0).setLayoutParams(mLayoutParams);
    }

    /**
     * 处理ActionMove事件
     * @param ev
     */
    private boolean performActionMove(MotionEvent ev) {
        int laterX = (int) ev.getX();
        int laterY = (int) ev.getY();
        //判断在X轴上移动的距离大于Y轴上移动的距离
        if(Math.abs(laterX - mDownX) > Math.abs(laterY - mDownY)) {
            //左移
            if(laterX < mDownX) {
                //滑动距离超过mTopBtnWidth就设置为mTopBtnWidth
                int scroll = laterX - mDownX;
                if(-scroll >= mTopBtnWidth) {
                    scroll = -mDelBtnWidth - mTopBtnWidth;
                }
                mLayoutParams.leftMargin = scroll;
                mSelectedView.getChildAt(0).setLayoutParams(mLayoutParams);
            }

            return true;
        }
        return super.onTouchEvent(ev);
    }

    /**
     * 处理ActionUp事件
     */
    private void performActionUp() {
        if (-mLayoutParams.leftMargin >= mTopBtnWidth) {
            mLayoutParams.leftMargin = -mDelBtnWidth - mTopBtnWidth;
            isBtnShow = true;
        } else {
            recovery();
        }
        mSelectedView.getChildAt(0).setLayoutParams(mLayoutParams);
    }

    /**
     * 恢复初始状态
     */
    public void recovery() {
        mLayoutParams.leftMargin = 0;
        mSelectedView.getChildAt(0).setLayoutParams(mLayoutParams);
        isBtnShow = false;
    }
}

```
接下来就是适配器了：

```
package com.custom;

import android.content.Context;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.BaseAdapter;
import android.widget.TextView;

import java.util.ArrayList;

/**
 * Created by Administrator on 2016/7/29.
 */
public class MyAdapter extends BaseAdapter {
    private LayoutInflater mInflater;
    private ArrayList<String> mList;
    private SwipeListView mListView;

    public MyAdapter(Context context, ArrayList<String> list, SwipeListView listView) {
        mInflater = LayoutInflater.from(context);
        mList = list;
        mListView = listView;
    }

    @Override
    public int getCount() {
        return mList.size();
    }

    @Override
    public Object getItem(int position) {
        return mList.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(final int position, View convertView, ViewGroup parent) {
        String s = mList.get(position);
        ViewHolder viewHolder = null;
        if (convertView == null) {
            viewHolder = new ViewHolder();
            convertView = mInflater.inflate(R.layout.item, null);
            viewHolder.tv = (TextView) convertView.findViewById(R.id.tv);
            viewHolder.top = (TextView) convertView.findViewById(R.id.top);
            viewHolder.del = (TextView) convertView.findViewById(R.id.delete);
            convertView.setTag(viewHolder);
        } else {
            viewHolder = (ViewHolder) convertView.getTag();
        }
        viewHolder.tv.setText(s);
        viewHolder.del.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
	            //删除
                mList.remove(position);
                notifyDataSetChanged();
                mListView.recovery();
            }
        });
        viewHolder.top.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
	            //置顶
                mList.add(0, mList.get(position));
                mList.remove(position + 1);
                notifyDataSetChanged();
                mListView.recovery();
            }
        });

        return convertView;
    }

    private class ViewHolder {
        TextView tv, top, del;
    }
}

```
附上XML文件：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal" >

    <TextView
        android:id="@+id/tv"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:paddingBottom="20dp"
        android:paddingLeft="10dp"
        android:paddingTop="20dp"
        android:background="@android:color/white"/>

    <TextView
        android:id="@+id/top"
        android:layout_width="70dp"
        android:layout_height="match_parent"
        android:background="#a7a6a6"
        android:gravity="center"
        android:paddingLeft="20dp"
        android:textColor="@android:color/white"
        android:paddingRight="20dp"
        android:text="置顶" />

    <TextView
        android:id="@+id/delete"
        android:layout_width="70dp"
        android:layout_height="match_parent"
        android:background="#FFFF0000"
        android:gravity="center"
        android:paddingLeft="20dp"
        android:textColor="@android:color/white"
        android:paddingRight="20dp"
        android:text="删除" />

</LinearLayout>
```

以上就是实现的代码，不足之处在于收回的时候有点生硬，需要动画过渡，才会展现得好一些。至于动画过渡，现在掌握也不太熟悉，就暂时先放下，有时间慢慢研究吧。
附上GitOS链接:http://git.oschina.net/QiHuangQi/SwipeListView