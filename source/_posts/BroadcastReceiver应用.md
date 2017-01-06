---
title: BroadcastReceiver应用
date: 2016-09-16 14:06
tags: Android
categories: Android
---
初学Android时对于BroadcastReceiver的认识很浅，基本上是知道有广播这个东西，而没有实际应用过。最近在实践中也感觉BroadcastReceiver的强大，所以需要重新对广播的知识进行一下梳理。
BroadcastReceiver意味“广播接受者”，可以用来接收用户定义的广播或是系统的广播。系统中也存在很多类似的广播机制，比如提醒用户低电量，当电量改变时，会发送一条广播，而接收到这个广播就能发送让用户充电的通知了。
在实际开发中，也会用到许多广播的例子，要创建广播，就需要继承BroadcastReceiver，实现onReciever()方法：

```
public class MyReceiver extends BroadcastReceiver {  
		  
	    private static final String TAG = "MyReceiver";    
	      
	    @Override  
	    public void onReceive(Context context, Intent intent) { 
			    //context为广播源,intent存储了广播的内容信息 
		    	Log.i(TAG, intent.getStringExtra("string"));
		}  
} 
```
但是仅仅通过上面的方法是没办法接收广播的，在此之前需要为广播注册一个广播地址，有了地址才能接受到广播的信号。而广播的注册又分为静态注册和动态注册。
<!--more-->
#### 静态注册 ####
就像intent的隐式调用一样，BroadcastReceiver的静态注册也需要在AndroidManifest.xml中声明：

```
<receiver android:name=".MyReciever">
            <intent-filter>
                <action android:name="android.intent.action.receiver"/>
            </intent-filter>
</receiver>
```
用上述的方法注册后，MyReceiver就可以接受来自android.intent.action.receiver的广播了。
#### 动态注册 ####
动态注册，就需要收到在代码中注册广播地址,除了手动注册，也还需要手动销毁：

```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MyReceiver";
    private MyReceiver myReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        myReceiver = new MyReceiver();
        //设置过滤
        IntentFilter filter = new IntentFilter();
        filter.addAction("android.intent.action.receiver");
        //注册广播地址
        registerReceiver(myReceiver, filter);
    }

    public class MyReceiver extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            Log.i(TAG, intent.getStringExtra("string"));
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        //需要手动解除注册，否则会抛出异常
        unregisterReceiver(myReceiver);
    }
}
```
静态注册与动态注册的区别在于动态注册跟随程序的生命周期，而静态注册则常驻在内存，即使程序退出了仍然可以接受广播。
广播注册了，就可以通过一下代码来发送广播：

```
Intent intent = new Intent();
intent.setAction("android.intent.action.receiver");
intent.putExtra("string", "MyReceiver");
sendBroadcast(intent);
```
![这里写图片描述](http://img.blog.csdn.net/20160916140506817)
留心会发现，在发送广播时，还有这么一个方法：

> sendOrderedBroadcast();


根据字面上意思，不难知道只是发送有顺序的广播，既然有顺序，那就应该有几个Receiver来接受广播，试着创建多个BroadcastReceiver 对象：

```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MyReceiver";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

    }
	//接收者0
    public static class MyReceiver extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            Log.i(TAG, intent.getStringExtra("string"));
        }
    }
	//接收者1
    public static class MyReceiver1 extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            Log.i(TAG, intent.getStringExtra("string"));
        }
    }
    //接收者2
    public static class MyReceiver2 extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            Log.i(TAG, intent.getStringExtra("string"));
        }
    }

}
```
然后需要在AndroidManifest.xml中说明接收者的优先级：

```
<receiver android:name=".MainActivity$MyReceiver">
        <intent-filter android:priority="100">
            <action android:name="android.intent.action.receiver"/>
	    </intent-filter>
</receiver>
<receiver android:name=".MainActivity$MyReceiver1">
        <intent-filter android:priority="99">
            <action android:name="android.intent.action.receiver"/>
        </intent-filter>
</receiver>
<receiver android:name=".MainActivity$MyReceiver2">
        <intent-filter android:priority="98">
            <action android:name="android.intent.action.receiver"/>
        </intent-filter>
</receiver>
```
接着发送一个有序广播：

```
Intent intent = new Intent("android.intent.action.receiver");
intent.putExtra("string", "MyReceiver");
//第二个参数为发送该广播的权限，如果需要权限需要手动在配置文件声明
sendOrderedBroadcast(intent, null);
```
结果：
![这里写图片描述](http://img.blog.csdn.net/20160916140526896)

另外，有序广播可以在任意一个环节对广播进行修改，如下所示：

```
//接收者1
public static class MyReceiver1 extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            Log.i(TAG, intent.getStringExtra("string"));
            Bundle bundle = new Bundle();
            bundle.putString("string", "MyReceiver1 has received");
            setResultExtras(bundle);
        }
    }
```
还需注意的是，接受者无法通过abortBroadcast()这个方法来终止广播。

BroadcastReceiver的简单应用
----------------------

检测网络状态

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

    }

    public static class MyReceiver extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            if (!NetWorkUtil.isNetworkConnected(context)) {
                Toast.makeText(context.getApplicationContext(), "网络不可用", Toast.LENGTH_SHORT).show();
            }
        }
    }

}
```

```
public class NetWorkUtil {
	//判断网络是否可用
    public static boolean isNetworkConnected(Context context) {
        if (context != null) {
            ConnectivityManager mConnectivityManager = (ConnectivityManager) context
                    .getSystemService(Context.CONNECTIVITY_SERVICE);
            NetworkInfo mNetworkInfo = mConnectivityManager.getActiveNetworkInfo();
            if (mNetworkInfo != null) {
                return mNetworkInfo.isAvailable();
            }
        }
        return false;
    }
}
```
在这里需要一个检测系统网络状态的权限和一接受网络变化广播的地址，需要在配置文件中注册：

```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

```
<receiver android:name=".MainActivity$MyReceiver">
        <intent-filter>
            <action android:name="android.net.conn.CONNECTIVITY_CHANGE"/>
        </intent-filter>
</receiver>

```
![这里写图片描述](http://img.blog.csdn.net/20160916140616506)
