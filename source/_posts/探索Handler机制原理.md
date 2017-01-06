---
title: 探索Handler机制原理
date: 2016-04-30 23:03
tags: Android
categories: Android
---

众所周知，Android是不能在主线程进行耗时操作，否则会抛出ANR（应用程序无响应）异常。于是，我们在程序开发中就会广泛地使用Handler来实现异步消息的处理，如读取网络数据并加载到程序的UI界面中。**Handler的运行需要MessegeQueue和Looper的支撑**，关于其三者的关系，接下来将通过源码来具体分析。
首先，我们来看一个使用Handler的情况：
```
public class MainActivity extends Activity {
    private Handler handler1, handler2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        handler1 = new Handler();  //第一个Handler对象在主线程创建

        new Thread(new Runnable() {
            @Override
            public void run() {
                handler2 = new Handler(); //第二个Handler在子线程创建
            }
        }).start();
    }
}
```
结果：
![这里写图片描述](http://img.blog.csdn.net/20160430205340698)
<!--more-->
我们可以看到，出现了一个异常：Can't create handler inside thread that has not called Looper.prepare()。该异常意思是不能在没有调用 Looper.prepare()的线程中创建Handler。**试着在handler2 = new Handler()前添加一句Looper.prepare()，再次运行，会发现错误消失了！**
既然，异常来自于Handler，我们可以看一下Handler的构造方法，看看是如何实现的：

```
//当调用无参构造函数时，会自动调用该方法，即this(null, false);
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper(); 
        if (mLooper == null) {  
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");//异常在此处抛出，mLooper为null
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
明显，mLooper为null，它是通过mLooper = Looper.myLooper()该语句来赋值的，那么我们有必要看看Looper中的myLooper()这个方法：

```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
}
```
代码比较简单，可以看出，mLooper是从ThreadLocal中取出来的（每个线程都保留独立的变量，关于ThreadLocal，可以参考一下http://blog.csdn.net/lufeng20/article/details/24314381）。那么sThreadLocal里的Looper对象在什么时候添加进去了呢？我们可以看看上面说到的Looper的prepare()方法：

```
public static void prepare() { 
        prepare(true);//调用下面重载的prepare()方法
}

private static void prepare(boolean quitAllowed) {
   if (sThreadLocal.get() != null) {
       throw new RuntimeException("Only one Looper may be created per thread");
   }
   sThreadLocal.set(new Looper(quitAllowed)); //为sThreadLocal添加一个Looper对象
}
```
看到这里，我们或许可以知道为什么文章开头那个代码片段会出错了。如果我们不使用Looper.prepare()方法的话，就无法为sThreadLocal添加Looper对象，从而在创建Handler对象时mLooper = null，抛出异常。进一步想，为什么handler1就不用调用Looper.prepare()这个方法呢？**原因在于ActivityThread中就已经调用了Looper.prepareMainLooper()这个方法，从而调用了Looper.prepare()方法。**具体可以阅读ActivityThread中的main函数源码可知。
那么，我们继续探索第二种情况，也是我们常见的情况：

```
 final Handler handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
            }
        };

new Thread(new Runnable() {
       @Override
        public void run() {
            Message message = new Message();
            Bundle b = new Bundle();
            b.putString("test", "test");
            message.setData(b);
            handler.sendMessage(message); //发送message
        }
}).start();
```
通过上述方式发送数据是再常见不过了。可是我们想想，既然发送了数据，那么数据被发送到哪里去了？那么，我们要再深入地看看sendMessage()方法是怎么实现的：

```
//最终会调用sendMessageAtTime这个方法
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;   //当前消息队列
        if (queue == null) {
	        //如果当前消息队列为空，抛出异常
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //队列不为空，调用enqueueMessage方法
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
从上面的代码可以看到，最后调用了enqueueMessage方法，即消息入队的意思，表明我们发送的消息是存储在消息队列MessageQueue 的对象中。至于上述代码中的mQueue，则在创建Looper对象时，一并创建了：

```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed); //创建消息队列，一个Looper对应一个消息队列
        mThread = Thread.currentThread();
}
```
紧接着，我们看一下enqueueMessage方法：

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;   //代表当前Handler对象
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis); //调用队列中的enqueueMessage方法
}
```
那么，再看看MessageQueue中的enqueueMessage方法：

```
boolean enqueueMessage(Message msg, long when) {
		//代码较长，使用简单的伪代码表示
		
        synchronized (this) {
            Message p = mMessages;  //设置当前消息
			
			//入队操作
            msg.next = p; // invariant: p == prev.next
            prev.next = msg; //指向下一消息
            }
        return true;
}
```
到这里，我们已经明白了发送的消息是如何处理的，那么既然入队了，肯定要知道它是如何处理队列里的消息。我们可以找到Looper中的loop()方法来解释：

```
//同样只是摘取重要的代码
 public static void loop() {
        final Looper me = myLooper(); //获取Handler对应的Looper对象
        final MessageQueue queue = me.mQueue; //获取当前消息队列

        for (;;) {               //不断循坏的处理消息
            Message msg = queue.next(); // 获取消息
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            msg.target.dispatchMessage(msg); //msg.target为Handler对象，回调dispatchMessage方法
            msg.recycleUnchecked();
        }
    }
```
那么Handler的最终实现方式应该就在dispatchMessage方法里了：

```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {  //如果设置了回调函数,则调用设置的回调函数
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg); //如果没有设置回调函数，则默认调用handleMessage方法
        }
    }
```
设置了回调函数的情况为：

```
final Handler handler = new Handler(new Handler.Callback() { //设置回调函数
            @Override
            public boolean handleMessage(Message msg) {
                return false;
            }
});
```
关于设置回调函数从而调用handleMessage方法和不设置来调用的区别，个人理解在于优先级，也类似于线程中实现Runable接口和继承Thread类。
好了，到这里，关于Handler，MessegeQueue和Looper三者之间的关系基本理清了，我们可以用这种标准的形式来实现并阐述Handler机制：

```
new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler = new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                    }
                };
                Looper.loop();
            }
        }).start();
```

最后画个示意图来总结一下：
![这里写图片描述](http://img.blog.csdn.net/20160430230204495)
