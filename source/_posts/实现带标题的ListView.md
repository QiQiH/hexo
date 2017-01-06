---
title: 实现带标题的ListView
date: 2016-04-25 21:06
tags: Android
categories: Android
---

在一些项目中，往往有要求为ListView里的内容分类，比如按日期分类，就要把相同日期的项目放在一起。可以看一些示例图，会清楚一些：
![这里写图片描述](http://img.blog.csdn.net/20160425205521795)![这里写图片描述](http://img.blog.csdn.net/20160425205543905)
以上根据标题来进行分类，实现代码如下：
<!--more-->
首先是数据项的模型：

```
public class Data {
    private String text1, text2, text3; //数据1 2 3

    public Data(String text1, String text2, String text3) {
        this.text1 = text1;
        this.text2 = text2;
        this.text3 = text3;
    }
    public String getText1() {
        return text1;
    }
    public String getText2() {
        return text2;
    }
    public String getText3() {
        return text3;
    }
}

```
因为要分类，所以需要一些类来存标题和数据Data集合：

```
public class Type {
    private String title;  //ListView头部显示的标题
    private List<Data> mList; //头部对应的内容集合

    public Type(String title) {
        this.title = title;
        mList = new ArrayList<>();
    }


    /**
     * 添加项目
     * @param data Data对象
     */
    public void addItem(Data data) {
        mList.add(data);
    }

    /**
     * 获取项目
     * @param position 如果position为1就返回标题
     * @return
     */
    public Object getItem(int position) {
        if (position == 0) {
            return title;
        } else {
            return mList.get(position - 1);
        }
    }

    /**
     * @return item数目，为集合大小+1
     */
    public int size() {
        return mList.size() + 1;
    }
}
```

紧接着实现适配器Adapter,主要的逻辑设计在此部分:

```
public class MyAdapter extends BaseAdapter {
    private static final int TYPE_HEADER = 0;  //代表标题
    private static final int TYPE_ITEM = 1;    //代表项目item


    private List<Type> mList;
    private LayoutInflater inflater;

    public MyAdapter(Context context, List<Type> list) {
        mList = list;
        inflater = LayoutInflater.from(context);
    }

    /**
     *
     * @return 所有项的总和
     */
    @Override
    public int getCount() {
        int count = 0;
        if (mList != null) {
            for (Type type : mList) {
                count += type.size();
            }
        }
        return count;
    }

    /**
     * 根据position的不同返回不同的值
     * @param position
     * @return
     */
    @Override
    public Object getItem(int position) {
        int head = 0;  //标题位置
        for (Type type : mList) {
            int size = type.size();
            int current = position - head;
            if (current < size) {
                //返回对应位置的值
                return type.getItem(current);
            }
            head += size;
        }

        return null;
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder viewHolder;
        switch (getItemViewType(position)) {
            //分为两种情况加载item
            case TYPE_HEADER: //加载标题布局
                if (convertView == null) {
                    viewHolder = new ViewHolder();
                    convertView = inflater.inflate(R.layout.header, parent, false);
                    viewHolder.title = (TextView) convertView.findViewById(R.id.date);
                    convertView.setTag(viewHolder);
                } else {
                    viewHolder = (ViewHolder) convertView.getTag();
                }
                viewHolder.title.setText((CharSequence) getItem(position));
                break;
            case TYPE_ITEM: //加载数据项目布局
                if (convertView == null) {
                    viewHolder = new ViewHolder();
                    convertView = inflater.inflate(R.layout.adapter, parent, false);
                    viewHolder.tv1 = (TextView) convertView.findViewById(R.id.text1);
                    viewHolder.tv2 = (TextView) convertView.findViewById(R.id.text2);
                    viewHolder.tv3 = (TextView) convertView.findViewById(R.id.text3);
                    convertView.setTag(viewHolder);
                } else {
                    viewHolder = (ViewHolder) convertView.getTag();
                }
                Data data = (Data) getItem(position);
                viewHolder.tv1.setText(data.getText1());
                viewHolder.tv2.setText(data.getText2());
                viewHolder.tv3.setText(data.getText3());
                break;
        }

        return convertView;
    }

    /**
     *
     * @return 返回item类型数目
     */
    @Override
    public int getViewTypeCount() {
        return 2;
    }

    /**
     * 获取当前item的类型
     * @param position
     * @return
     */
    @Override
    public int getItemViewType(int position) {
        int head = 0;
        for (Type type : mList) {
            int size = type.size();
            int current = position - head;
            if (current == 0) {
                return TYPE_HEADER;
            }
            head += size;
        }
        return TYPE_ITEM;
    }

    /**
     * 判断当前的item是否可以点击
     * @param position
     * @return
     */
    @Override
    public boolean isEnabled(int position) {
        return getItemViewType(position) != TYPE_HEADER;
    }
    @Override
    public boolean areAllItemsEnabled() {
        return false;
    }
    private class ViewHolder {
        TextView tv1, tv2, tv3, title;
    }
}
```
最后，简单写一个测试的Activity:

```
public class MainActivity extends AppCompatActivity {
    private ListView listView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        listView = (ListView) findViewById(R.id.list);
        List<Type> list = new ArrayList<>();
        for (int i = 1; i <= 5; i++) {
            Type type = new Type("标题" + i);
            for (int j = 0; j < Math.random() * 6; j++) {
                Data data = new Data("数据1", "数据2", "数据3");
                type.addItem(data);
            }
            list.add(type);
        }
        MyAdapter adapter = new MyAdapter(this, list);
        listView.setAdapter(adapter);
    }
}
```
附上两个所需的XML：
adapter.xml
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
		>

        <ImageView
            android:layout_width="70dp"
            android:layout_height="70dp"
            android:src="@android:drawable/btn_star_big_on"
            android:scaleType="centerCrop"
            />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="20dp"
            android:id="@+id/text1"
            android:text="text"
            android:textSize="17sp"
            android:layout_gravity="center_vertical"
            />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="20dp"
            android:id="@+id/text2"
            android:text="text"
            android:textSize="17sp"
            android:layout_gravity="center_vertical"
            />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="20dp"
            android:id="@+id/text3"
            android:text="text"
            android:textSize="17sp"
            android:layout_gravity="center_vertical"
            />

    </LinearLayout>

</LinearLayout>
```
header.xml
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="25sp"
        android:id="@+id/date"
        android:background="@android:color/holo_purple"
        />
</LinearLayout>
```

