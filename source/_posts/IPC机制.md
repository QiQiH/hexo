---
title: IPC机制
date: 2016-08-24 15:23
tags: Android
categories: Android
---
最近在了解Service时接触到Android IPC，IPC全称是Inter-Process Communication，意思是进程间通信，当然也包括了跨进程通信。要了解IPC，首先需要知道的是Android中开启多进程模式的方式。

开启多进程
-----

想要在一个应用程序中开启多个进程，在Android中有这一种方法：即在AndroidMenifest.xml中给四大组件指定一个android:process的属性，以下为示例：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.ipc">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity
            android:name=".SecondActivity"
            android:label="@string/app_name"
            android:process=":remote"
            >
        </activity>
    </application>

</manifest>
```
程序运行后打开DDMS会发现出现了两个进程，如图：
![这里写图片描述](http://img.blog.csdn.net/20160824133717267)
<!--more-->
一个是com.ipc即包名，另外一个则是添加了上述配置文件 :remote后缀，当然 android:process这个属性也可以自己命名，如 android:process="com.ipc.remote"。其中的区别是前者是私有进程，其他应用不可以和它跑在同一个线程，后者为全局进程，功能相反。
## 多进程的问题 ##
经过上面的测试，会发现原来在一个程序中开启多进程那么简单。其实，看似简单，却有着许多进程间数据共享的问题。举个例子，我们再创建一个类Manager，里面存在一个全局变量：

```
public class Manager {
    public static int mId = 1;
}
```

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.i("TAG", "Id = " + Manager.mId + "");
        //在进程1中修改mId
        Manager.mId = 2;
        startActivity(new Intent(this, SecondActivity.class));
    }
}
```

```
public class SecondActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        Log.i("TAG", "Id = " + Manager.mId + "");
    }
}
```
这时候我们会发现即使在MainActivity中修改了mId的值，但在另一个进程中mId的值仍然是1。到这里不禁疑惑，为什么会出现这样的问题？原因在于SecondActivity运行在另一个进程，Android会为每个进程分配一个虚拟机，而每个虚拟机分配了不同的地址空间。上述情况，虽然只有一个Manager类，但事实上由于在不同进程中，SecondActivity也会拥有一个Manager的副本，因此导致了数据的不一致。如此看来，在单例模式以及线程同步上也会出现这样的问题。
## IPC基础 ##
既然出现了上述的问题，就应该有解决的方法，解决该问题就应该先了解一下IPC基础
 
 - Parcelable，Serializable
 一提到序列化马上就能想到两个接口：Parcelable和Serializable。Serializable在Java开发中比较常见，Parcelable是在Android中的序列化方式。具体的信息可以看看这篇文章：http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0204/2410.html

 - Binder
 其实一谈到Binder，就会觉得Binder是个很复杂的东西，看了许多资料都觉得迷迷糊糊。引用一个大神对于Binder的理解：
 

> Binder是Android的一个类，它实现了IBinder接口。Binder是Android中一种跨进程通信方式，是ServiceManager连接各种Manager和相应ManagerService的桥梁。同时它也是客户端和服务器进行通信的媒介，当调用bindService时，服务器就会返回包含服务端业务的调用的Binder对象，通过这个对象，客户端就可以获取服务端提供的数据，这里的服务包括普通服务和AIDL。

在Android开发中，Binder主要用在Service，而Service中会涉及到一个知识点，那就是AIDL。
 AIDL:Android Interface Definition Language,即Android接口定义语言。由于Android在不同进程中不能共享内存，则需要寻找一种方式来解决，AIDL则是解决问题的方法。通过AIDL也可以更好的理解Binder。
 首先，需要新建一个AIDL的例子。以Android Studio为例，新建三个文件Student.java，Student.aidl，IStuManager.adil。其中AIDL的创建方式如下图：
 ![这里写图片描述](http://img.blog.csdn.net/20160824143007228)

```
/**
 * Student类作为跨进程通信用到的数据，需要序列化
 */
public class Student implements Parcelable {
    public int sId;
    public String name;

    protected Student(Parcel in) {
        sId = in.readInt();
        name = in.readString();
    }

    public static final Creator<Student> CREATOR = new Creator<Student>() {
        @Override
        public Student createFromParcel(Parcel in) {
            return new Student(in);
        }

        @Override
        public Student[] newArray(int size) {
            return new Student[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(sId);
        dest.writeString(name);
    }
}
```

```
// Student.aidl
package com.ipc;
parcelable Student;
```

```
// IStuManager.aidl
package com.ipc;
//使用ADIL，即使在同一个包，也许导入
import com.ipc.Student;
interface IStuManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
    /**
    *自定义的两个接口
    */
    List<Student> getStudentList();
    void addStudent(in Student stu);
}
```
点一下Make Project，会发现自动生成了一个文件：
![这里写图片描述](http://img.blog.csdn.net/20160824144247264)
```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 */
package com.ipc;

public interface IStuManager extends android.os.IInterface {
	//Stub是一个Binder
    public static abstract class Stub extends android.os.Binder implements com.ipc.IStuManager {
	    //Binder标识
        private static final java.lang.String DESCRIPTOR = "com.ipc.IStuManager";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        /**
        * 将服务器端的Binder对象转换为客户端所需的AIDL对象。如果服务器和客户端同一进程，则返回服务端本身，否则返回Stu.proxy对象
        */
        public static com.ipc.IStuManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.ipc.IStuManager))) {
                return ((com.ipc.IStuManager) iin);
            }
            return new com.ipc.IStuManager.Stub.Proxy(obj);
        }

		//返回当前的Binder对象
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }	
		/**
		*如果服务端和客户端不同进程，会调用这个方法，通过Stub的内部代理类Proxy完成
		*@params code 确定调用的方法 data 调用方法的参数 reply 如果目标方法有返回值，则存进reply
		*/
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_basicTypes: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    long _arg1;
                    _arg1 = data.readLong();
                    boolean _arg2;
                    _arg2 = (0 != data.readInt());
                    float _arg3;
                    _arg3 = data.readFloat();
                    double _arg4;
                    _arg4 = data.readDouble();
                    java.lang.String _arg5;
                    _arg5 = data.readString();
                    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                    reply.writeNoException();
                    return true;
                }
                //调用获取学生列表的方法
                case TRANSACTION_getStudentList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.ipc.Student> _result = this.getStudentList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                //调用添加学生的方法
                case TRANSACTION_addStudent: {
                    data.enforceInterface(DESCRIPTOR);
                    com.ipc.Student _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.ipc.Student.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addStudent(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

		//内部代理类，两个方法在此类内部实现
        private static class Proxy implements com.ipc.IStuManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(anInt);
                    _data.writeLong(aLong);
                    _data.writeInt(((aBoolean) ? (1) : (0)));
                    _data.writeFloat(aFloat);
                    _data.writeDouble(aDouble);
                    _data.writeString(aString);
                    mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

			//获取列表
            @Override
            public java.util.List<com.ipc.Student> getStudentList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.ipc.Student> _result;
                try {
	                //将参数写入data，用于调用目标方法，并挂起当前线程
                    _data.writeInterfaceToken(DESCRIPTOR);
                    //跨进程，调用transact方法
                    mRemote.transact(Stub.TRANSACTION_getStudentList, _data, _reply, 0);
                    _reply.readException();
                    //取出结果，作为返回值，继续当前线程
                    _result = _reply.createTypedArrayList(com.ipc.Student.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

			//添加学生，原理同上
            @Override
            public void addStudent(com.ipc.Student stu) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((stu != null)) {
                        _data.writeInt(1);
                        stu.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addStudent, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

		//标识目标方法的固定数值
        static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getStudentList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_addStudent = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    }

    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;

    public java.util.List<com.ipc.Student> getStudentList() throws android.os.RemoteException;

    public void addStudent(com.ipc.Student stu) throws android.os.RemoteException;
}

```
（上述代码中涉及到代理模式，可以看看http://blog.csdn.net/u010429311/article/details/52303091）

需要注意，从服务器调用transact方法到返回结果，是一个耗时操作，因此同样也存在线程安全问题需要注意。通过以上的叙述，对于Binder的理解还是有一点帮助。通过一个来总结一下Binder机制：
![这里写图片描述](http://img.blog.csdn.net/20160824152300295)
以上是对IPC的初步理解，之后有时间继续学习如何使用多进程通信机制吧。