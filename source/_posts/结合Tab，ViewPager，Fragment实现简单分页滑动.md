---
title: 结合Tab，ViewPager，Fragment实现简单分页滑动
date: 2016-04-06 11:13
tags: Android
categories: Android
---

在APP设计当中，使用ViewPager和Fragment来实现分页滑动并不少见，该设计可以利用少量的空间来实现多内容的展示。效果图如下：
![这里写图片描述](http://img.blog.csdn.net/20160406103353922)
![这里写图片描述](http://img.blog.csdn.net/20160406103403406)

<!--more-->
以下是实现该功能的代码：

MainActivity
```
public class MainActivity extends AppCompatActivity {
    private ViewPager viewPager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
    }


	//tab点击事件
    private ActionBar.TabListener mListener = new ActionBar.TabListener() {
        @Override
        public void onTabSelected(ActionBar.Tab tab, FragmentTransaction ft) {
            if (viewPager != null) {
                viewPager.setCurrentItem(tab.getPosition()); //设置当前显示的fragment
            }
        }

        @Override
        public void onTabUnselected(ActionBar.Tab tab, FragmentTransaction ft) {

        }

        @Override
        public void onTabReselected(ActionBar.Tab tab, FragmentTransaction ft) {

        }
    };

    private void initView() {
        final ActionBar actionBar = getSupportActionBar();
        ActionBar.Tab tab1 = actionBar.newTab(); //新建tab
        ActionBar.Tab tab2 = actionBar.newTab();
        tab1.setText("Tab1");
        tab2.setText("Tab2");
        tab1.setTabListener(mListener);  //设置tab监听事件，必须在actionBar.addTab语句之前，否则报错
        tab2.setTabListener(mListener);
        actionBar.addTab(tab1);
        actionBar.addTab(tab2);
        actionBar.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);//设置模式

        viewPager = (ViewPager) findViewById(R.id.viewPager);
        ArrayList<Fragment> fragments = new ArrayList<>(2);  //添加两个fragment或者更多
        MyFragment fragment1 = new MyFragment(R.layout.fragment_tab1);
        MyFragment fragment2 = new MyFragment(R.layout.fragment_tab2);
        fragments.add(fragment1);
        fragments.add(fragment2);
        viewPager.setAdapter(new FragmentAdapter(getSupportFragmentManager(), fragments)); //设置适配器
        viewPager.setOnPageChangeListener(new ViewPager.OnPageChangeListener() { //页面滑动后tab跟着移动
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {

            }

            @Override
            public void onPageSelected(int position) {
                actionBar.setSelectedNavigationItem(position); //设置当前tab选中的位置
            }

            @Override
            public void onPageScrollStateChanged(int state) {

            }
        });
    }
}
```
MyFragement

```
public class MyFragment extends Fragment {
    private int resource;                //需要解析的布局资源

    public MyFragment(int resource) {
        this.resource = resource;
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View v = inflater.inflate(resource, container, false); //解析布局，此处第三个参数应为false,否则会返回父视图的布局，导致OOM
        return v;
    }
}
```
FragmentAdapter

```
//适配器，与ListView等所需适配器的设置类似
public class FragmentAdapter extends FragmentPagerAdapter {
    private ArrayList<Fragment> fragments;

    public FragmentAdapter(FragmentManager fm, ArrayList<Fragment> fragments) {
        super(fm);
        this.fragments = fragments;
    }

    @Override
    public Fragment getItem(int position) {
        return fragments.get(position);
    }

    @Override
    public int getCount() {
        return fragments.size();
    }
}
```
至于XML的实现相对简单：
activity_main.xml

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.viewpager.MainActivity">

    <android.support.v4.view.ViewPager
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:id="@+id/viewPager"
        >

    </android.support.v4.view.ViewPager>
</RelativeLayout>
```