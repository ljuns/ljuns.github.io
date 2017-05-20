---
title: Android 多线程的各种实现方式解析
date: 2017-05-16 20:15:43
category: 疑难解析
tags: 多线程

---

线程在 Android 中是一个比较重要的概念，由于 Android 的特性，如果在主线程中执行耗时操作就会导致程序无法及时的响应，因此耗时操作必须放在子线程中去执行。

提到多线程必然会想起 Handler，前面写了一篇关于 Handler 的总结：[学习总结 -- Android 消息机制各个击破](http://ljuns.cn/2017/04/19/%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93-Android-%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E5%90%84%E4%B8%AA%E5%87%BB%E7%A0%B4/#more)，另外还有郭婶的 [Android异步消息处理机制完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/9991569)。

下面开始介绍 Android 多线程中一些比较重要的实现方式。

<!-- more --> 

## Handler

Android 消息机制主要是指 Handler 机制，关于 Handler 的分析可以看看上面那两篇文章，这里再简单补充一下 Handler 的 post() 方法：

``` java
    // post()
	public final boolean post(Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

	// getPostMessage()
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

首先 post() 方法需要传入一个 Runnable 对象作为参数，接着通过 getPostMessage() 方法把 Runnable 对象转变成一个 Message 并发送出去。前面一篇文章说到 Handler 的 post() 系列的方法最终都会调用 send() 系列的方法，在这也得到了证实。

getPostMessage() 方法的实现也很简单，首先构造一个 Message 对象，并且把 Runnable 对象存储在 Message 的 callback 中，然后把 Message 返回。

最后在 Handler 的 dispatchMessage() 方法中会判断 `msg.callback` 是否为空，而这个 `msg.callback` 就是 getPostMessage() 方法中的 `m.callback`。这样一来对 post() 方法的实现就更清晰了，具体的看这：[学习总结 -- Android 消息机制各个击破](http://ljuns.cn/2017/04/19/%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93-Android-%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E5%90%84%E4%B8%AA%E5%87%BB%E7%A0%B4/#more)。

## Activity.runOnUiThread()

假如在 Activity 中开启子线程进行了耗时操作，此时就可以很方便的使用 runOnUiThread() 方法来进行 UI 更新，为什么呢？来看看 runOnUiThread() 方法的实现：

``` java
    public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }
```

runOnUiThread() 方法也很简单，首先会判断当前线程是否为主线程(UI 线程)，如果不是则调用 Handler 的 post() 方法并把 Runnable 对象传进去；否则就直接调用 Runnable 的 run() 方法。原来 runOnUiThread() 方法最终是通过 Handler 的 post() 方法来实现的异步操作的，后面流程的都已经熟悉了。

## View.post()

View 中也有一个类似的方法：`View.post()`

``` java
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        ViewRootImpl.getRunQueue().post(action);
        return true;
    }
```

原来还是调用了 Handler 的 post() 方法，这个就不用解释了。

## AsyncTask

AsyncTask 是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程并在主线程中更新 UI。在 Android 3.0 版本前后 AsyncTask 的实现差别较大，这里的分析是基于 6.0 版本的。关于 AsyncTask 的使用也是比较简单的：

``` java
public class AsyncTaskActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_async_task);

        new AsyncTaskTest().execute("");
    }
    
    class AsyncTaskTest extends AsyncTask<String, Integer, String> {
        @Override
        protected void onPreExecute() {
            super.onPreExecute();
        }
        @Override
        protected String doInBackground(String... params) {
            return null;
        }
        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
        }
        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
        }
    }
}
```

用起来是很简便，底层实现呢？下面就来看看，根据上面的例子，首先需要创建一个 AsyncTask 对象，所以先来了解一下它的构造方法：

``` java
    public AsyncTask() {
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

构造方法倒是蛮长的，但也只是初始化了两个变量：mWorker、mFuture，并且在初始化 mFuture 的时候把 mWorker 传进去。mWorker 和 mFuture 又是什么东西呢？先来看看 mWorker：

``` java 
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
	Params[] mParams;
}
```

原来 WorkerRunnable 只是实现了 Callback 接口，并且创建了一个数组 mParams。下面看看 mFuture：

``` java
public class FutureTask<V> implements RunnableFuture<V> {
  	...
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
  ...
}
```

FutureTask 实现了 RunnableFuture 接口，并且将 mWorker 存储在 callable 中。其实 RunnableFuture 接口最终也是实现了 Runnable 和 Future 接口。

所以 mWorker 是一个 Callback 对象，而 mFuture 则是一个 Runnable 对象，这两个变量暂时用不到，先存储在内存中。创建 AsyncTask 对象后就调用它的 execute() 方法，这里才是入口：

``` java
	// execute()
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    // executeOnExecutor()
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
        mStatus = Status.RUNNING;
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
    }
```

execute() 方法其实很简单，直接调用了 executeOnExecutor() 方法。

在 executeOnExecutor() 方法中首先调用了 onPreExecute() 方法，此时程序还运行在主线程中，一般在 onPreExecute() 方法中做一些初始化的工作；然后把传进来的参数放在 mWorker 的 mParams 数组中；接着再调用 `exec.execute(mFuture)`。这个 exec 又是什么？查看 execute() 方法会发现原来 exec 是个 sDefaultExecutor。找到 sDefaultExecutor 定义的地方：

``` java
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

先是创建了一个静态常量 SERIAL_EXECUTOR，然后再把这个常量赋值给 sDefaultExecutor。所以 executeOnExecutor() 方法中执行的 `exec.execute(mFuture)`实际上是调用了 SerialExecutor 的 execute() 方法。

> SerialExecutor 是以常量的形式被使用的，因此在整个应用程序中的所有 AsyncTask 实例都会共用同一个SerialExecutor。

下面看看 SerialExecutor 这个类：

``` java
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

SerialExecutor 用一个无边界的队列 mTasks 来管理 Runnable，execute() 方法有个 Runnable 参数，这个参数当然就是刚刚传递进来的 mFuture 啦。

execute() 方法中首先创建一个 Runnable 对象并通过 offer() 方法添加到队列尾部，接着判断 mActive 是否为 null，如果为 null 就调用 scheduleNext() 方法。

而在 scheduleNext() 方法中会先通过 `mTasks.poll()` 从队列头部取出一个 Runnable 对象，然后再执行 THREAD_POOL_EXECUTOR 的 execute() 方法，并把取出来的 Runnable 对象传递进去。这个取出来的 Runnable 对象就是通过 offer() 方法添加进去的。

再来看看 THREAD_POOL_EXECUTOR 是什么：

``` java
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```

原来是个线程池对象，那么 THREAD_POOL_EXECUTOR 的 execute() 自然就是在线程池中执行的了。里面具体的执行涉及到了线程池的原理，由于各种原因就不往下走了，但是可以肯定的是最终会调用 mActive 的 run() 方法，所以 `r.run()` 方法是在线程池的子线程中执行的。

因为 r 对象是传递进来的 mFuture 对象，所以继续看 mFuture 的 run() 方法：

``` java
    public void run() {
        ...
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
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
  		...
        }
    }
```

可以看到里面调用了 callable 的 call() 方法，而这个 callable 其实就是在 AsyncTask 的构造方法中初始化 mFuture 时传递进去的 mWorker，所以这时就回到了 mWroker 的 call() 方法。这里把 AsyncTask 的构造方法中 mWorker 初始化的代码单独拿出来：

``` java
	// AsyncTask 的构造方法 
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

call() 方法中调用了 doInBackground() 方法，由于此时还是在线程池的子线程中运行，所以也就解释了 doInBackground() 中为什么可以执行那些耗时操作。call() 方法最后返回了 postResult() 方法：

``` java
    // postResult()
	private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

	// getHandler()
    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }
```

postResult() 里面是 Handler 异步消息处理，看到这就简单了。这里使用 getHandler() 方法拿到一个 InternalHandler 对象，并且用这个对象发出了一条消息，消息中携带了 MESSAGE_POST_RESULT 常量和一个任务执行结果的 Result 对象，那么消息最终肯定是在 InternalHandler 的 handleMessage() 方法中被处理的。来看看 handleMessage() 方法：

``` java
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

到了 handleMessage() 方法这里就已经把执行环境从线程池切换到主线程了，所以里面的逻辑都是在主线程中运行的。

handleMessage() 方法对消息的类型进行了判断，如果是 MESSAGE_POST_RESULT 消息，就会去执行 finish() 方法；而如果是 MESSAGE_POST_PROGRESS 消息，就会去执行 onProgressUpdate() 方法。finish() 方法如下：

``` java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

如果当前任务被取消了，就调用 onCancelled() 方法；否则就会调用 onPostExecute() 方法来更新 UI。

在 InternalHandler 的 handleMessage() 方法中，什么情况才是 MESSAGE_POST_PROGRESS 消息呢？当然就是调用 publishProgress() 方法的时候啦：

``` java
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```

执行到这就算是结束了，已经非常清晰了！其实说到底，AsyncTask 也是使用了异步消息处理机制，只是做了非常好的封装而已。来张时序图：

![](http://oh0xbzubu.bkt.clouddn.com/image/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2021.52.23.png?imageView2/0/q/75|watermark/2/text/5bCP5bGx55qE5oqA5pyv5bCP56qd/font/5a6L5L2T/fontsize/400/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

## IntentService

IntentService 是一种特殊的 Service，它继承了 Service 并且它是一个抽象类，因此必须创建它的子类才能使用 IntentService。IntentService 可用于执行后台耗时的任务，当任务执行后它会自动停止，同时由于 IntentService 是服务的原因，这导致它的优先级比单纯的线程要高很多，所以 IntentService 比较适合执行一些优先级较高的后台任务。看个小例子：

``` java
// Activity
public class IntentServiceActivity extends AppCompatActivity {
    Intent intent;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_intent_service);

         intent = new Intent(this, MyService.class);
        for (int i = 0; i < 10; i ++) {
            intent.putExtra("action", "cn.ljuns.action: " + i);
            startService(intent);
        }
    }
}

// IntentService
public class MyService extends IntentService {

    private static final String TAG = "MyService";

    public MyService() {
        super(TAG);
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        String action = intent.getStringExtra("action");
        Log.d(TAG, "onHandleIntent: " + action);
        SystemClock.sleep(3000);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d(TAG, " MyService onDestroy: ");
    }
}
```

在 Activity 中循环启动了一个 IntentService 来模拟 10 个后台任务，然后在 IntentService 的 onHandleIntent() 方法中睡眠 3 秒来模拟耗时操作。打印结果如下：

![](http://oh0xbzubu.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2013.54.43.png?imageView2/0/q/75|watermark/2/text/5bCP5bGx55qE5oqA5pyv5bCP56qd/font/5a6L5L2T/fontsize/400/fill/I0ZGRkZGRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

从打印结果可以看出，10 个后台任务是排队执行的，而且当任务执行完后会自动停止服务。这就是 IntentService 的神奇之处。接下来就来分析一下神奇的原因，当 IntentService 第一次被启动的时候会调用它的 onCrtate() 方法，所以先看看它的 onCreate() 方法：

``` java
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```

在 onCreate() 方法中先创建了一个 HandlerThread 对象，然后通过 getLooper() 方法拿到一个 Looper，接着再用这个 Looper 构造出一个 ServiceHandler 对象。这么看来 onCreate() 方法只是创建了一个 ServiceHandler 对象，但是 HandlerThread 是怎么拿到 Looper 对象的呢？那就得看看 HandlerThread 的实现：

``` java
public class HandlerThread extends Thread {
	...
    Looper mLooper;
	...
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    ...
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }    
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
	...
}
```

HandlerThread 的实现比较简单，这里只留下了一些比较重要的方法。下面简单介绍一下 HandlerThread：

HandlerThread 继承了 Thread，所以它也是个 Thread 对象；在 run() 方法中调用了 `Looper.prepare()` 来创建一个 Looper 和 MessageQueue，接着再调用 `Looper.loop()` 来开启消息循环，此时是运行在子线程中的，这点必须明白；而 getLooper() 方法自然就是获取刚刚创建 Looper 对象了；同时还提供了 quit() 和 quitSafely() 两个方法来退出消息循环。

这会再看看 IntentService 的 onCreate() 方法就清晰很多了：创建 HandlerThread 后调用它的 start() 方法来创建 Looper 和 MessageQueue，同时开启消息循环，然后再用 HandlerThread 的 Looper 来构造一个 ServiceHandler，所以 ServiceHandler 依附于 HandlerThread 的子线程。

每次启动 IntentService 时它的 onStartCommand() 方法都会被调用，看看 onStartCommand() 方法又做了什么：

``` java
    // onStartCommand()
	@Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

	// onStart()
    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```

onStartCommand() 方法调用了 onStart() 方法，并且把启动 IntentService 传过来的 intent 传递给了 onStart() 方法。看到 onStart() 方法突然很开心，因为又是 Handler 的消息机制，这个过程已经很熟悉了。

所以 onStart() 方法仅仅是通过 mServiceHandler 发送了一个消息，最终消息会在 ServiceHandler 的 handleMessage() 方法中被处理。是不是这样呢？继续看看 ServiceHandler 的实现：

``` java
	private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    @WorkerThread
    protected abstract void onHandleIntent(Intent intent);
```

在 handleMessage() 方法中调用了 onHandleIntent() 方法，onHandleIntent() 方法是个抽象方法，需要我们在子类中去实现，同时还调用了 stopSelf() 方法来停止服务。**由于 handleMessage() 方法是在 loop() 方法中几经波折后被调用的，而 loop() 方法又是在子线程中被调用的，所以最后在 onHandleIntent() 方法中做耗时操作也就是在子线程中做的耗时操作。**

一般来说，stopSelf(int startId) 在尝试停止服务之前会判断最近启动服务的次数是否和 startId 相等，如果相等就立刻停止服务，不相等则不停止服务。

Android 多线程的实现方式有多种，但是每种方式或多或少都和 Handler 消息机制有关，所以掌握 Handler 消息机制是掌握 Android 多线程的基础，也是核心。