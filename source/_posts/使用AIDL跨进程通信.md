---
title: 使用AIDL跨进程通信
date: 2016-10-14 19:26
tags: Android
categories: Android
---
之前对IPC的研究有接触到AIDL，AIDL的作用实际上就是跨进程通信，因为进程间是各自维护着自己的一个内存，当前进程想要访问到其他进程的内存，就可以通过AIDL来实现。
假定现在进程1（**服务端**）中有一个学生信息的集合，现在要在进程2（**客户端**）通过学生ID获取到进程1中集合中某个学生的信息。下图为结构图：
![这里写图片描述](http://img.blog.csdn.net/20161014185515050)
<!--more-->
首先，需要创建Student类：

```
//必须序列化，不然无法传输二进制对象
public class Student implements Parcelable{
	//id，姓名，年龄
    private int id;
    private String name;
    private int age;

    public Student(int id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    protected Student(Parcel in) {
        id = in.readInt();
        name = in.readString();
        age = in.readInt();
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

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeInt(id);
        parcel.writeString(name);
        parcel.writeInt(age);
    }
}
```
接着，需要创建aidl文件，Student.aidl和IStuInterface.aidl:

```
// Student.aidl
package com.aidl;

// Declare any non-default types here with import statements
parcelable Student;
```
```
// IStuInterface.aidl
package com.aidl;

// 即使同一包内，也要导入
import com.aidl.Student;

interface IStuInterface {
	//返回学生对象
    Student getStudent(int id);
}
```
点击build project，会自动生成一个java文件
![这里写图片描述](http://img.blog.csdn.net/20161014190156310)
这个文件便是跨进程的具体实现，具体原理可参考：http://blog.csdn.net/u010429311/article/details/52300794
这个文件里面有一个抽象的方法：

```
public com.aidl.Student getStudent(int id) throws android.os.RemoteException;
```

因此，需要创建一个service来具体实现IStuInterface中的方法，来为进程2提供服务。

```
public class QueryService extends Service {
    public static final String TAG = "Service";

    private ArrayList<Student> students;

	//具体实现，返回对应id的Student对象
    private IStuInterface.Stub stub = new IStuInterface.Stub() {
        @Override
        public Student getStudent(int id) throws RemoteException {
            for (Student s : students) {
                if (s.getId() == id) {
                    return s;
                }
            }
            return null;
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return stub;
    }

	//onCreate首先会被执行
    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG, "onCreate");
        students = new ArrayList<>();
        for (int i = 0; i < 4; i++) {
            students.add(new Student(i, "student" + i, 20));
        }
    }

    @Override
    public boolean onUnbind(Intent intent) {
        Log.i(TAG, "onUnbind");
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i(TAG, "onDestroy");
    }
}

```
现在服务也做好了，就需要一个进程来使用这个服务，从而获取到Student对象：

```
public class StudentActivity extends AppCompatActivity {
    private EditText etId;
    private Button btnQuery;
    private TextView tvName, tvAge;
    private IStuInterface mService;

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            //将服务器端提供的IBinder对象转化为IStuInterface对象
            mService = IStuInterface.Stub.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mService = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_student);

        Intent i = new Intent();
        i.setPackage(getPackageName());
        i.setAction("com.aidl.QueryService");
        bindService(i, connection, Context.BIND_AUTO_CREATE);

        etId = (EditText) findViewById(R.id.et_id);
        btnQuery = (Button) findViewById(R.id.btn_query);
        tvName = (TextView) findViewById(R.id.tv_name);
        tvAge = (TextView) findViewById(R.id.tv_age);

        btnQuery.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                int id = Integer.parseInt(etId.getText().toString().trim());
                try {
                    Student student = mService.getStudent(id);
                    if (student != null) {
                        tvName.setText(student.getName());
                        tvAge.setText(student.getAge() + "");
                    } else {
                        tvName.setText("无记录");
                    }
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}

```
完成上面的操作基本上实现了AIDL的简单应用，但由于需要跨进程，StudentActivity 就需要运行在另外一个进程上，这就需要在AndroidManifest上配置 ：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.aidl">

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
        <!--通过android:process=":remote"来实现该Activity运行在新进程中-->
        <activity android:name=".StudentActivity"
            android:process=":remote"
            ></activity>
		<!--Service声明-->
        <service android:name=".QueryService"
            >
            <intent-filter>
                <action android:name="com.aidl.QueryService"/>
            </intent-filter>
        </service>
    </application>

</manifest>
```
当然，实现不同经常不局限于android:process=":remote"，也可以创建另外一个应用程序，同样也可以实现AIDL功能。
![结果](http://img.blog.csdn.net/20161014192351790)