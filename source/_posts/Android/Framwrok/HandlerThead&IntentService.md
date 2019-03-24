---
title: HandlerThread&IntentService
top: false
cover: false
categories: Android
tags:
  - IntentService
  - HandlerThread
date: 2019-03-24 17:17:36
img:
coverImg:
summary: 
---

在处理多个异步任务时, 有两种选择. 一种是并行, 这种情况可以使用线程池; 另一种是串行, 可以使用 HandlerThread来实现. 顾名思意, HandlerThread继承自Thread, 本质上也是一个线程. 

### HandlerThread源码
HandlerThread的代码不多, 刨去隐藏方法, 一次全贴出来了:
```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;
    // 构造方法要显示指定线程名称
    public HandlerThread(String name) {
        super(name);
        // 默认的线程优先级
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    // 指定名称和优先级
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    // 在Looper创建之后, 且开始循环消息之前被调用, 子类可以重写该方法, 做一些准备工作
    protected void onLooperPrepared() {}
    // run方法
    @Override
    public void run() {
        mTid = Process.myTid();
        // 创建Looper
        Looper.prepare();
        // 同步锁
        synchronized (this) {
            // 将Looper对象实例赋值给 mLooper 后, 唤醒锁
            mLooper = Looper.myLooper();
            notifyAll();
        }
        // 设置线程优先级
        Process.setThreadPriority(mPriority);
        // 回调方法, 可以在该方法中做一些准备工作
        onLooperPrepared();
        // 开始循环消息
        Looper.loop();
        mTid = -1;
    }
    // 给外部获取Looper对象实例
    public Looper getLooper() {
        // 线程未激活, 则返回 null
        if (!isAlive()) {
            return null;
        }
        // 同步锁.  因为 Looper是在run方法中异步创建的
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    // mLooper为空, 则等待 (直到被唤醒)
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
    // 退出 Looper 循环
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    // 安全的退出 Looper 循环
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
    public int getThreadId() {
        return mTid;
    }
}
```

可以发现HandlerThread其实就是帮我们完成了Looper相关的初始化和循环操作, 另外还提供了两个退出Looper的方法, 让我们在合适的实际可以用它们回收资源. 下面再看下 Looper 的 quit 和 quitSafely 方法:
```java
public void quit() {
    mQueue.quit(false);
}
public void quitSafely() {
    mQueue.quit(true);
}
```

实际都调用了 MessageQueue 的 quit 方法:
```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;
        if (safe) {
            // looper.quitSafely() 会走这里
            removeAllFutureMessagesLocked();
        } else {
            // looper.quit() 走这里
            removeAllMessagesLocked();
        }
        nativeWake(mPtr);
    }
}
```

继续贴代码:
```java
// 直接清空整个消息队列
private void removeAllMessagesLocked() {
    Message p = mMessages;
    // 循环将整个消息链表全部回收了
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
...
// 如果当前没有消息要处理了, 就直接清空整个消息队列; 否则要等这些消息执行完毕后, 再清空消息队列
private void removeAllFutureMessagesLocked() {
    final long now = SystemClock.uptimeMillis();
    Message p = mMessages;
    if (p != null) {
        if (p.when > now) {
            // 如果即将要处理的消息, 还没到时间, 则直接清空整个消息队列
            removeAllMessagesLocked();
        } else {
            // 走到这里, 表示当前有消息正在处理
            Message n;
            // 这个死循环的作用就是: 等待消息队列中所有 Message.when 已到处理事件的消息处理完毕之后, 再清空队列
            for (;;) {
                // 记录下一条消息
                n = p.next;
                // 没有下一条了, 直接退出
                if (n == null) {
                    return;
                }
                // 下一条消息还没到处理的时候, 跳出当前循环
                if (n.when > now) {
                    break;
                }
                // 走到这里, 表示下一条消息也开始处理了
                p = n;
            }
            p.next = null;
            // 循环将整个消息链表全部回收
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }
}
```

关于 HandlerThread 的代码就研究到这了. HandlerThread 适用于串行执行多个耗时操作的场景, 不过实际开发中, 我们也不会直接使用它, 因为Google在HandlerThread的基础上, 又封装了 IntentService.

### IntentService源码
```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message msg) {
            // 注意这里处于 mServiceLooper 所在的子线程
            // 收到消息后, 取出放在消息中的Intent, 然后回调 onHandleIntent
            onHandleIntent((Intent)msg.obj);
            // 消息处理完后, 关闭服务.  
            // msg.arg1 就是 onStart() 中的 startId. 可以通过 startId 来停止本次服务.
            // Service 每次被启动, 都会有一个 startId, 只有当所有这些 startId 全部停止后, Service 才会销毁
            // 所以才能在多次start同一个IntentService后, 会等所有任务依次执行完毕后, 服务才会销毁.
            stopSelf(msg.arg1);
        }
    }
    public IntentService(String name) {
        super();
        mName = name;
    }
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }
    @Override
    public void onCreate() {
        super.onCreate();
        // 创建 HandlerThread 并启动
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
        // 获取 HandlerThread 的 Looper
        mServiceLooper = thread.getLooper();
        // 使用该 Looper 创建 Handler
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
    // 服务被启动后, 将 Intent 封装到 Message里面, 然后通过 Handler 将消息发出
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        // 将 msg.arg1 设置为本次服务启动的 startId
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
    // 这里直接调用了 onStart 方法
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
    // 服务销毁时, 情况消息队列
    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }
    // 该方法返回了null, 所以不要通过 bindService() 启动该服务
    // 当然一定要这样做也可以, 只是没什么意义
    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }
    // 重写此方法, 在这里处理耗时操作
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

### IntentService使用
- 第一步: 继承IntentService, 在onHandleIntent方法中处理耗时任务
```java
public class TestIntentService extends IntentService {
    public TestIntentService() {
        super("test_thread");
    }
    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        // TODO
    }
}
```
- 第二步: 在 AndroidManifest.xml 中注册服务
```xml
<service android:name=".views.me.setting.TestIntentService"/>
```
- 第三步: 启动Service
```java
startService(new Intent(this, TestIntentService.class));
```