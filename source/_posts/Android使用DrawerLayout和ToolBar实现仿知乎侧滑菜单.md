---
title: Android使用DrawerLayout和ToolBar实现仿知乎侧滑菜单
date: 2016-01-23 21:44
tags: Android
categories: Android
---

侧滑菜单现在在很多app上都可以看到，以下文章主要讲如何实现实现Android的侧滑菜单。可以先看一个简单的侧滑菜单设计。
示例图：
![这里写图片描述](http://img.blog.csdn.net/20160123211550000)
从屏幕左端向右滑动或点击左上角按钮可打开侧滑栏菜单。

<!--more-->
具体实现
----

activity_main.xml
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.widget.DrawerLayout
    android:layout_height="fill_parent"
    android:layout_width="fill_parent"
    android:id="@+id/myDrawer"
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    >

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:id="@+id/hideLayout"
        >

		<!-- 顶部栏 -->
        <android.support.v7.widget.Toolbar
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="@color/colorPrimary"
            android:minHeight="?attr/actionBarSize"
            android:fitsSystemWindows="true"
            android:id="@+id/mToolBar"
            android:popupTheme="@style/Widget.AppCompat.ActionBar"
            app:theme="@style/Widget.AppCompat.Light.ActionBar"
            >

        </android.support.v7.widget.Toolbar>

		<!-- 用于加载导航栏的容器 -->
        <FrameLayout
            android:layout_width="match_parent"
            android:id="@+id/frame"
            android:layout_height="match_parent">
        </FrameLayout>

    </LinearLayout>

	<!-- 导航栏的设置 此处可设置导航栏头部布局及菜单布局-->
    <android.support.design.widget.NavigationView
        android:id="@+id/nav"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:background="#ffffff"
        app:headerLayout="@layout/header" 
        app:menu="@menu/nav_menu"
        android:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        app:theme="@style/ThemeOverlay.AppCompat.ActionBar"
        >


    </android.support.design.widget.NavigationView>

</android.support.v4.widget.DrawerLayout>

```

紧接着要添加菜单，在menu文件夹下新建xml文件
nav_menu.xml

```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <group android:checkableBehavior="single">
        <item android:id="@+id/item1"
            android:icon="@drawable/smile"
            android:title="item1"
            >
        </item>
        <item android:id="@+id/item2"
            android:icon="@drawable/sorry"
            android:title="item2"
            >
        </item>
        <item android:id="@+id/item3"
            android:icon="@drawable/oh_boy"
            android:title="item3"
            >
        </item>
        <item android:id="@+id/item4"
            android:icon="@drawable/joy"
            android:title="item4"
            >
        </item>
        <item android:id="@+id/item5"
            android:icon="@drawable/now_what"
            android:title="item5"
            >
        </item>
    </group>
</menu>
```
还有头部的布局，即上图哆啦A梦处
header.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:background="#c40606"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="10dp"


        >

        <ImageView
            android:layout_width="150dp"
            android:layout_height="150dp"
            android:src="@drawable/doraemon"
            android:layout_marginBottom="20dp"
            />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp"
            android:layout_paddingLeft="20dp"
            android:text="乐观的蓝胖子"
            android:textSize="20sp"
            />
    </LinearLayout>

</LinearLayout>
```
此外需要注意的是，使用toolbar时需要把系统的ActionBar去掉，方法是修改style.xml文件中的主题设置
style.xml
```
<resources>
	<!-- 改为没有ActionBar的主题 Theme.AppCompat.Light.NoActionBar -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <!-- 这里设置左上角可点击展开的小图标 -->
        <item name="drawerArrowStyle">@style/AppTheme.DrawerArrowToggle</item>
    </style>

    <style name="AppTheme.DrawerArrowToggle" parent="Base.Widget.AppCompat.DrawerArrowToggle">
        <item name="color">@android:color/white</item>
    </style>

    <style name="AppTheme.AppBarOverlay" parent="ThemeOverlay.AppCompat.Dark.ActionBar" />

    <style name="AppTheme.PopupOverlay" parent="ThemeOverlay.AppCompat.Light" />
</resources>

```
完成上述步骤后即可设置其操作，在MainActivity中写入如下代码：

```
		mToolBar = (Toolbar) findViewById(R.id.mToolBar); //ToolBar
        mDrawer = (DrawerLayout) findViewById(R.id.myDrawer); //DrawerLayout
        navigationView = (NavigationView) findViewById(R.id.nav); //NavigationView导航栏
        mToolBar.setTitle("首页");
        mToolBar.setTitleTextColor(Color.parseColor("#FFFFFF"));
        setSupportActionBar(mToolBar);
        
		//设置左上角的图标响应
        getSupportActionBar().setHomeButtonEnabled(true); 
        getSupportActionBar().setDisplayHomeAsUpEnabled(true);

        mDrawerToggle = new ActionBarDrawerToggle(this, mDrawer, mToolBar, 0, 0) {
            @Override
            public void onDrawerClosed(View drawerView) {
                super.onDrawerClosed(drawerView);
            }

            @Override
            public void onDrawerOpened(View drawerView) {
                super.onDrawerOpened(drawerView);
            }
        };
        mDrawerToggle.syncState();
        mDrawer.setDrawerListener(mDrawerToggle); //设置侧滑监听
```

问题总结
----
1.为什么会出现侧滑菜单无法覆盖ToolBar?
当需要侧滑菜单覆盖ToolBar时，必须把DrawerLayout作为根布局。而不是LinearLayout或其他。
2.为什么没有在编写导航栏时没有android.support.design.widget.NavigationView？
右键项目==>open module setting==>Dependencies==>添加库com.android.support:design:23.1.1