---
title: Android中GridView实现长按多选功能
date: 2016-02-24 22:56
tags: Android
categories: Android
---

前言：GridView可用于展示多行多列的统一格式数据，但本身没有多选操作。现通过一系列代码实现GridView的长按多选操作，可以先看一个示例图。
![这里写图片描述](http://img.blog.csdn.net/20160224225227364)
<!--more-->
以下是实现该功能的主要代码：
MainActivity.java
```
package com.mygridview;

import android.app.AlertDialog;
import android.content.DialogInterface;
import android.support.v7.app.ActionBar;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.KeyEvent;
import android.view.View;
import android.widget.AdapterView;
import android.widget.Button;
import android.widget.CheckBox;
import android.widget.GridView;
import android.widget.RelativeLayout;
import android.widget.TextView;
import android.widget.Toast;

import java.util.ArrayList;
import java.util.HashMap;

public class MainActivity extends AppCompatActivity implements View.OnClickListener, AdapterView.OnItemClickListener, AdapterView.OnItemLongClickListener {
    private ActionBar actionBar;
    private Button btnCancel, btnDel, btnSelectAll, btnClear;
    private RelativeLayout headerLayout;
    private TextView numText;
    private GridView gridView;
    private ArrayList<HashMap<String, Object>> items; //每一个item
    private ArrayList<Boolean> selectItems; //用于存储已选中项目的位置
    private MyAdapter adapter;
    private boolean isState;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initView();
    }

    private void initView() {
        actionBar = getSupportActionBar();
        actionBar.setTitle("GridView");

        numText = (TextView) findViewById(R.id.number);

        btnCancel = (Button) findViewById(R.id.btn_cancel);
        btnDel = (Button) findViewById(R.id.btn_del);
        btnSelectAll = (Button) findViewById(R.id.btn_select_all);
        btnClear = (Button) findViewById(R.id.btn_clear);
        btnCancel.setOnClickListener(this);
        btnDel.setOnClickListener(this);
        btnSelectAll.setOnClickListener(this);
        btnClear.setOnClickListener(this);

        headerLayout = (RelativeLayout) findViewById(R.id.header);
        gridView = (GridView) findViewById(R.id.grid_view);
        items = new ArrayList<>();
        selectItems = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            HashMap<String, Object> item = new HashMap<>();
            item.put("image", R.drawable.image);
            items.add(item);
        }
        adapter = new MyAdapter(this, items, R.layout.image, new String[]{"image"}, new int[]{R.id.my_image});
        gridView.setAdapter(adapter);
        gridView.setOnItemClickListener(this);
        gridView.setOnItemLongClickListener(this);

    }

    public ArrayList<Boolean> getSelectItems() {
        return selectItems;
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_cancel :
                selectItems.clear();
                numText.setText("已选择1项");
                adapter.setIsState(false);
                setState(false);
                break;
            case R.id.btn_clear :
                setSelectAll(false);
                break;
            case R.id.btn_select_all:
                setSelectAll(true);
                break;
            case R.id.btn_del:
                delSelections();
                break;
        }
    }

    @Override
    public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
        if (isState) {
            CheckBox checkBox = (CheckBox) view.findViewById(R.id.check_box);
            if (checkBox.isChecked()) {
                checkBox.setChecked(false);
                selectItems.set(position, false);
            } else {
                checkBox.setChecked(true);
                selectItems.set(position, true);
            }
            adapter.notifyDataSetChanged();
            setSelectNum();
        }
    }

    @Override
    public boolean onItemLongClick(AdapterView<?> parent, View view, int position, long id) {
        if (!isState) {
            selectItems = new ArrayList<>();
            for (int i = 0; i < items.size(); i++) {
                selectItems.add(false);
            }
            CheckBox box = (CheckBox) view.findViewById(R.id.check_box);
            box.setChecked(true);
            selectItems.set(position, true);
            setState(true);
            adapter.setIsState(true);
            setSelectNum();
        }
        return false;
    }

    //设置当前状态 是否在多选模式
    private void setState(boolean b) {
        isState = b;
        if (b) {
            headerLayout.setVisibility(View.VISIBLE);
            btnDel.setVisibility(View.VISIBLE);
            actionBar.hide();
        } else {
            headerLayout.setVisibility(View.GONE);
            btnDel.setVisibility(View.GONE);
            actionBar.show();
        }
    }

    //显示已选项数目
    private void setSelectNum() {
        int num = 0;
        for (Boolean b : selectItems) {
            if (b)
                num ++;
        }
        numText.setText("已选择" + num + "项");
    }

    //全选
    private void setSelectAll(boolean b) {
       for (int i = 0; i < selectItems.size(); i++) {
           selectItems.set(i, b);
           adapter.notifyDataSetChanged();
           setSelectNum();
       }
        btnSelectAll.setVisibility(b? View.GONE : View.VISIBLE);
        btnClear.setVisibility(b ? View.VISIBLE : View.GONE);
    }

    //删除
    private void delSelections() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        if (!selectItems.contains(true)) {
            builder.setTitle("提示").setMessage("当前未选中项目").setPositiveButton("确认", null).create().show();
        } else {
            builder.setTitle("提示");
            builder.setMessage("确认删除所选项目？");
            builder.setPositiveButton("确认", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    for (int i = 0; i < items.size(); i++) {
                        if (selectItems.get(i)) {
                            items.set(i, null);
                        }
                    }
                    while (items.contains(null)) {
                        items.remove(null);
                    }
                    selectItems = new ArrayList<>();
                    for (int i = 0; i < items.size(); i++) {
                        selectItems.add(false);
                    }

                    adapter.setData(items);
                    adapter.notifyDataSetChanged();
                    if (items.size() == 0) {
                        adapter.setIsState(false);
                        setState(false);
                        return;
                    }
                }
            });
            builder.setNegativeButton("取消", null);
            builder.create().show();
        }

    }

    private long mExitTime;
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_BACK) {
            if (isState) {
                selectItems.clear();
                numText.setText("已选择1项");
                adapter.setIsState(false);
                setState(false);
                return true;
            } else {
                if ((System.currentTimeMillis() - mExitTime) > 2000) {
                    Toast.makeText(this, "再按一次退出程序", Toast.LENGTH_SHORT).show();
                    mExitTime = System.currentTimeMillis();
                }
                else finish();
                return true;
            }

        }

        return super.onKeyDown(keyCode, event);
    }

}

```
设配器Adapter

```
package com.mygridview;

import android.content.Context;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.CheckBox;
import android.widget.SimpleAdapter;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;


public class MyAdapter extends SimpleAdapter {
    private List<? extends Map<String, ?>> data;
    private int resourse;
    private Context context;
    private LayoutInflater inflater;
    private boolean isState = false;


    public MyAdapter(Context context, List<? extends Map<String, ?>> data, int resource, String[] from, int[] to) {
        super(context, data, resource, from, to);
        inflater = LayoutInflater.from(context);
        this.resourse = resource;
        this.data = data;
        this.context = context;
    }

    public void setData(List<? extends Map<String, ?>> data) {
        this.data = data;
    }

    public void setIsState(boolean isState) {
        this.isState = isState;
        notifyDataSetChanged();
    }

    @Override
    public int getCount() {
        return data.size();
    }

    @Override
    public Object getItem(int position) {
        return data.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ArrayList<Boolean> list = ((MainActivity) context).getSelectItems();
        convertView = inflater.inflate(resourse ,null);
        CheckBox checkBox = (CheckBox) convertView.findViewById(R.id.check_box);
        checkBox.setVisibility(isState? View.VISIBLE : View.GONE);
        if (list.size() != 0)
            checkBox.setChecked(list.get(position));
        return super.getView(position, convertView, parent);
    }
}
```
主页面布局 activity_main.xml
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.mygridview.MainActivity">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentTop="true"
        android:id="@+id/header"
        android:visibility="gone"
        android:background="@color/colorPrimary"
        >
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/btn_cancel"
            android:background="@android:color/transparent"
            android:text="取消"
            />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="已选择1项"
            android:layout_centerVertical="true"
            android:layout_centerInParent="true"
            android:id="@+id/number"
            />
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="全选"
            android:background="@android:color/transparent"
            android:id="@+id/btn_select_all"
            android:layout_alignParentEnd="true"
            />
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="清除"
            android:background="@android:color/transparent"
            android:id="@+id/btn_clear"
            android:visibility="gone"
            android:layout_alignParentEnd="true"
            />

    </RelativeLayout>


    <GridView
        android:layout_below="@+id/header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:numColumns="3"
        android:id="@+id/grid_view"
        android:verticalSpacing="10dp"
        android:horizontalSpacing="10dp"
        android:layout_above="@+id/btn_del"
        >

    </GridView>

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="删除"
        android:id="@+id/btn_del"
        android:visibility="gone"
        android:layout_alignParentBottom="true"
        android:layout_centerHorizontal="true" />


</RelativeLayout>

```
每个item的布局 image.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:descendantFocusability="blocksDescendants"
    >

    <RelativeLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">

        <ImageView
            android:layout_width="120dp"
            android:layout_height="150dp"
            android:src="@drawable/image"
            android:scaleType="centerCrop"
            android:id="@+id/my_image"
            />
        <CheckBox
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/check_box"
            android:visibility="gone"
            android:layout_alignParentTop="true"
            android:layout_alignRight="@+id/my_image"
            android:layout_alignEnd="@+id/my_image" />

    </RelativeLayout>
</LinearLayout>
```
问题总结：
1.当进入多选模式时点击图片CheckBox不会选中。
答：由于图片和多选框组成的view的点击事件与多选框的点击事件冲突，所以无法选中，可以在xml的根布局中加入android:descendantFocusability="blocksDescendants"即可解决。

