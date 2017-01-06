---
title: 深入探索AsyncTask
date: 2016-05-01 14:24
tags: Android
categories: Android
---

关于AsyncTask，相信大家都不陌生。由于Android UI线程的不安全性，所以如果想要在子线程中更新UI，就必须使用异步消息处理的方式。之前研究了Handler的处理机制（http://blog.csdn.net/u010429311/article/details/51288479），对于AsyncTask的理解还是挺有帮助的。在这里，我们继续深入的理解AsnycTask的工作原理。在此之前，我们可以先写一个简单的例子，然后循序渐进地去理解它。

<!--more-->
```
public class MainActivity extends Activity {
    private Button btnExecute, btnCancel;
    private ProgressBar bar;
    private MyTask task;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btnExecute = (Button) findViewById(R.id.start);
        btnCancel = (Button) findViewById(R.id.cancel);
        bar = (ProgressBar) findViewById(R.id.progressBar);

        btnExecute.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                task = new MyTask(); //创建任务
                task.execute();  //执行任务
            }
        });
        btnCancel.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (!task.isCancelled())
                    task.cancel(true);  //中断任务
            }
        });

    }

    class MyTask extends AsyncTask<Void, Integer, Void> {
        @Override
        protected void onPreExecute() { //在任务开始之前在执行，可进行一些UI的显示处理
            bar.setProgress(0);
            btnExecute.setEnabled(false);
            btnCancel.setEnabled(true);
        }

		@Override
        protected Void doInBackground(Void... params) { //任务开始执行
            for (int i = 1; i <= 100; i++) {
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    task.cancel(true);
                }
                publishProgress(i); //不断地更新ProgressBar的数据
            }
            return null;
	        }
		}
        @Override
        protected void onPostExecute(Void aVoid) { //任务结束后执行
            btnExecute.setEnabled(true);
            btnCancel.setEnabled(false);
        }

        @Override
        protected void onProgressUpdate(Integer... values) { //调用publishProgress方法后执行
            bar.setProgress(values[0]);   //更新数据
        }

        @Override
        protected void onCancelled(Void aVoid) {  //任务被中断后执行
            bar.setProgress(0);
            btnExecute.setEnabled(true);
            btnCancel.setEnabled(false);
        }
}
```
结果：
![这里写图片描述](http://img.blog.csdn.net/20160501142855660)
上面实现了简单的操作，既然要了解AsyncTask的工作原理，我们首先要看看第一步创建MyTask对象时发生了哪些事情，因此，就要去找AsyncTask的构造函数了：

```
 public AsyncTask() {
        //创建一个WorkerRunable对象
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
        
        //创建FutureTask对象
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
在构造函数中，只创建了两个对象，而且不知道他们具体的作用是什么。细心留意，我们会发现doInBackground(mParams)这条语句。说明mWorker 的call()方法会在某一时刻被执行，我们就暂且放着不管。继续研究下一个重要的方法execute()。当执行task.execute()时，到底执行了什么操作，我们看看源码是怎么实现的：

```
 public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params); //执行了executeOnExecutor方法
    }
```

```
 public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) { //非挂起状态
            switch (mStatus) {
                case RUNNING:   //当前任务在运行状态
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:  //当前任务已结束
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING; //设置当前状态为运行状态
        onPreExecute();   //从这里可以看到最先执行的是onPreExecute()方法
        mWorker.mParams = params; //设置mWorker的参数
        exec.execute(mFuture);  //在线程池中执行mFuture任务
        return this;
    }
```
我们发现在execute方法中调用executeOnExecutor方法并传入了sDefaultExecutor参数，用来执行exec.execute(mFuture)。那么sDefaultExecutor这个线程池在哪里定义了？我们可以在AsyncTask的数据域中找到：

```
 public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
 private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
可以看到，sDefaultExecutor 是一个SerialExecutor对象，我们具体看看SerialExecutor类以及它的execute方法：

```
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();  //任务队列
        Runnable mActive;  //将要执行的任务

        public synchronized void execute(final Runnable r) {  //传入的参数为mFuture，即FutureTask对象
            mTasks.offer(new Runnable() {  //任务的线程入队，此处把任务放在了子线程处理
                public void run() {
                    try {
                        r.run();   //调用MFuture的run()方法
                    } finally {
                        scheduleNext();  //无论发生什么，都会执行下一任务
                    }
                }
            });
            if (mActive == null) {  //第一次运行为null
                scheduleNext();
            }
        }

		//使用线程池THREAD_POOL_EXECUTOR来执行下一任务mActive
        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);  //执行任务，会调用任务中的run方法,即r.run()
            }
        }
    }
```
看到这里，我们会发现 已经出现了两个线程池，所以先区别一下：

>THREAD_POOL_EXECUTOR     用来真正地去执行任务
>sDefaultExecutor                     用来保存当前地任务，以实现任务的入队

从上述代码可以看出，每当执行任务时，都会调用run()方法，那么run()方法在FutureTask这个类中是如何实现的？之前我们创建了一个mFuture对象：

```
 public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable; //此处的callable正是在创建AsyncTask时所创建的mWorker对象
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();  //调用了call()方法
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
在run()方法中，我们看到调用了mWorker的call()方法，之前AsyncTask的构造函数中的mWorker对象在这里终于用上了，**此时调用了doInBackground()方法，并且我们会发现这个时候任务处于子线程中！**所以，这个时候，我们就可以用doInBackground()方法去处理一些耗时的操作。

```
//创建一个WorkerRunable对象
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
```
在完成doInBackground()方法后紧接着就执行了postResult()方法：

```
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        //MESSAGE_POST_RESULT代表任务即将结束，发送结果
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT, 
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
看到这里，就已经进入了熟悉的画面了，这里显然是Handler的处理机制，既然发送了消息，那么在Handler的实例中一定重写了handleMessage方法，我们通过 getHandler()方法中可以看到返回的是一个sHandler，**sHandler是AsyncTask类的一个静态的成员变量，由于最终要切换到主线程进行UI操作，所以sHandler必须在类初始化时就定义，因此决定了AsyncTask的实例必须在主线程创建，否则会报错。**它是InternalHandler的实例：

```
private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());  //对应一个Looper
        }
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;  //获取结果
            switch (msg.what) {
                case MESSAGE_POST_RESULT:   //表示当前任务即将结束
                    // There is only one result
                    result.mTask.finish(result.mData[0]);  //结束任务
                    break;
                case MESSAGE_POST_PROGRESS:  //表示当前需要更新进度
                    result.mTask.onProgressUpdate(result.mData); //调用我们实现的onProgressUpdate()方法，实现进度的更新
                    break;
            }
        }
    }
```
我们发现MESSAGE_POST_PROGRESS这个值并没有出现过，它执行的操作是onProgressUpdate()，这让我们很容易就联想到了publishProgress()方法，在源码中寻找这个方法：

```
protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
	        //MESSAGE_POST_PROGRESS 代表了更新数据，所以执行了publishProgress()方法就可以更新当前的进度了
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```

上面在结束时还调用finish()方法：

```
private void finish(Result result) {
        //如果取消了任务，如task.cancel(true)，就会执行onCancelled()方法，否则执行onPostExecute()方法
        if (isCancelled()) {  
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;  //无论哪种形式，当前的任务都在此结束了
    }
```
看到这里，AsyncTask的工作原理就非常清晰了，真的是一波三折，让我不得不感叹工程师们的逻辑思维是多么的强大(╯﹏╰)