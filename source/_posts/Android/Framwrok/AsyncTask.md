---
title: AsyncTask
top: false
cover: false
categories: Android
tags:
  - AsyncTask
date: 2019-03-27 13:52:23
img:
coverImg:
summary: 
---

AsyncTask 是一个轻量级的异步任务类. AsyncTask内部封装了线程池和Handler, 让我们在执行异步任务时, 可以比较容易的将任务进度以及执行结果回调到UI线程.

### AsyncTask的使用方式
打开AsyncTask.java, 发现在文件头的注释中, 系统已经提供了一个AsyncTask的使用案例.
```java
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
    // 开始执行后台任务之前回调此方法, 处于主线程
    @Override
    protected void onPreExecute() {
        // 可以在此做一些初始化准备工作
    }
    /**
     * 后台执行异步任务
     * 参数类型由 AsyncTask 的第一个泛型决定, 如果不需要, 可定义泛型为 Void
     * 参数的值来自 execute() 方法
     */
    protected Long doInBackground(URL... urls) {
        int count = urls.length;
        long totalSize = 0;
        for (int i = 0; i < count; i++) {
            totalSize += Downloader.downloadFile(urls[i]);
            // 更新进度
            publishProgress((int) ((i / (float) count) * 100));
            // Escape early if cancel() is called
            if (isCancelled()) break;
        }
        return totalSize;
    }
    /**
     * 后台执行异步任务时, 如果通过调用 publishProgress() 方法更新进度, 则会在主线程回调此方法
     * 参数类型由 AsyncTask 的第二个泛型决定, 如果不需要, 可定义泛型为 Void
     * 参数的值来自 publishProgress()方法
     */
    protected void onProgressUpdate(Integer... progress) {
        setProgressPercent(progress[0]);
    }
    /**
     * 后台任务执行完毕后, 在主线程回调此方法
     * 参数类型由 AsyncTask 的第三个泛型决定, 如果不需要, 可定义泛型为 Void
     * 参数的值来自 doInBackground()方法的返回值
     */
    protected void onPostExecute(Long result) {
        showDialog("Downloaded " + result + " bytes");
    }
    // 任务取消时回调
    @Override
    protected void onCancelled(Long result) {}
}
...
new DownloadFilesTask().execute(url1, url2, url3);
```

AsyncTask的使用还是比较简单的, 下面就从 execute 方法开始, 了解一下它的工作原理.
### AsyncTask的execute方法
```java
// 在主线程调用
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
...
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {
    // 不是默认状态, 就抛出异常. 表示 AsyncTask只能执行一次
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
    // 将状态设置为 RUNNING
    mStatus = Status.RUNNING;
    // 回调 onPreExecute() 方法, 可以在这里做一些准备工作
    onPreExecute();
    // mWorker实际上是Callable的子类
    mWorker.mParams = params;
    // 执行任务.  exec就是上面传过来的 sDefaultExecutor
    exec.execute(mFuture);
    return this;
}
```

先看下 sDefaultExecutor 是什么:
### AsyncTask的任务执行器
```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
...
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
...
// 一个通过队列实现的串行执行器
private static class SerialExecutor implements Executor {
    // 队列
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    // 当前要执行的任务
    Runnable mActive;
    public synchronized void execute(final Runnable r) {
        // 向队列末尾添加一个 Runnable
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    // 当这个Runnable执行时, 调用参数Runnable的run方法, 这里的参数就是之前传入的 mFuture
                    r.run();
                } finally {
                    // 尝试从队列开头取出任务并执行
                    scheduleNext();
                }
            }
        });
        // 如果当前没有正在执行的任务, 就尝试从队列开头取出任务并执行
        if (mActive == null) {
            scheduleNext();
        }
    }
    protected synchronized void scheduleNext() {
        // 从队列开头取出任务, 如果不为空, 就通过线程池 THREAD_POOL_EXECUTOR 执行任务
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

THREAD_POOL_EXECUTOR是个线程池:
### AsyncTask的线程池
```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
// 线程池核心线程数
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
// 线程池最大线程数
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
// 非核心线程闲置时间
private static final int KEEP_ALIVE_SECONDS = 30;
// 线程池中的任务队列. 当有新任务加入时, 如果当前线程数小于核心线程数, 则创建线程, 否则将任务放入队列中等待
private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);
// 创建线程的工厂类, 这里的作用就是为线程池创建线程, 并命名
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);
        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
public static final Executor THREAD_POOL_EXECUTOR;
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

弄清楚 sDefaultExecutor 和 THREAD_POOL_EXECUTOR 各自的作用之后, 接下来看下AsyncTask具体是怎么执行异步任务的.
### AsyncTask的后台逻辑
```java
private static InternalHandler sHandler;
private final WorkerRunnable<Params, Result> mWorker;
private final FutureTask<Result> mFuture;
...
public AsyncTask(@Nullable Looper callbackLooper) {
    // 一般情况下, 参数 callbackLooper 都是空, 因此这里的 Handler 是使用的主线程的 Looper
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);
    // mWorker是 Callable的实例对象, 一般和FutureTask一起使用, 执行后台任务, 并可以获取任务结果
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            // 保存后台任务的执行结果
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                // 执行后台任务
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                // 发生异常, 标记为任务取消了
                mCancelled.set(true);
                throw tr;
            } finally {
                // 发送结果
                postResult(result);
            }
            return result;
        }
    };
    // 当执行器调用mFuture的run方法时, 里面会调用mWorker的call方法
    mFuture = new FutureTask<Result>(mWorker) {
        // 异步任务结束后, 回调此方法
        @Override
        protected void done() {
            try {
                // 如果前面的 postResult没有执行, 就会再调一次
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

至此, AsyncTask的工作流程差不多都清除了, 再看下它是如何将任务结果返回到主线程的.
### AsyncTask如何处理任务进度和结果
```java
// 通过 Handler 将进度发送到主线程的
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
...
// 还是通过 handler将结果发送到主线程
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
...
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // 处理任务结果
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                // 更新后台任务进度
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
...
private void finish(Result result) {
    if (isCancelled()) {
        // 任务取消了
        onCancelled(result);
    } else {
        // 任务执行完毕
        onPostExecute(result);
    }
    // 将状态设置为 FINISHED
    mStatus = Status.FINISHED;
}
```

### 补充和总结
其实如果只是想简单的在后台执行一下异步任务, 既不需要知道进度, 也不需要将结果通知到主线程, AsyncTask还提供了一个静态方法.
```java
@MainThread
public static void execute(Runnable runnable) {
    sDefaultExecutor.execute(runnable);
}
```

最后总结一下: AsyncTask类在加载时, 就会创建好任务执行器和线程池. 在初始化AsyncTask对象时, 会先后创建 Handler(用于线程通信)、Callable(用于处理后台任务和获取执行结果)、 FutureTask(用来执行Callable). 当我们调用AsyncTask的execute方法时, 会在执行器的队列末尾加入一个任务, 同时执行器会不断取出队列开头的任务, 然后交给线程池去执行.
