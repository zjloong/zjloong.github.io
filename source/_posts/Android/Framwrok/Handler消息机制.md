---
title: Handler消息机制
top: true
cover: true
categories: Android
tags:
  - Handler
date: 2019-03-23 23:25:59
img:
coverImg:
summary: 说起Android的消息处理机制, Handler肯定是无法被忽略的一个点. 通过Handler收发消息可以非常方便的实现线程间通信功能.
---

说起Android的消息处理机制, Handler肯定是无法被忽略的一个点. 通过Handler收发消息可以非常方便的实现线程间通信功能.

### Handler的基本使用方式
实际开发中, Handler的使用方式通常是这样的:
```java
Handler handler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        // 处理消息
    }
};
... 
// 各种姿势发送消息
handler.obtainMessage();
handler.sendMessage();
handler.sendEmptyMessage();	
```

或者是这样的:
```java
handler.post(() -> {
	// 执行任务
});
// 延迟 1 s 执行任务
handler.postDelayed(() -> {}, 1000);
```

实际上还有我们爱用的 runOnUiThread(), 里面也使用了 Handler.
```java
// Activity 公共方法
public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
        mHandler.post(action);
    } else {
        action.run();
    }
}
```

Handler可以说是无处不在. 下面通过简单的源码研究, 了解一下 Handler 的工作原理.

### Handler 的构造方法
```java
public Handler() {
    this(null, false);
}
public Handler(Callback callback) {
    this(callback, false);
}
public Handler(Looper looper) {
    this(looper, null, false);
}
public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}
```

可以发现, 上面这些方法, 最后都间接调用了下面这两个方法.
```java
public Handler(Callback callback, boolean async) {
  ... 省略 ...
  mLooper = Looper.myLooper();
  // Looper 为空, 就抛出异常
  if (mLooper == null) {
      throw new RuntimeException(
          "Can't create handler inside thread " + Thread.currentThread()
                  + " that has not called Looper.prepare()");
  }
  // 消息队列 - MessageQueue, 由 Looper维护.
  mQueue = mLooper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

可以发现两者的区别就是和 looper 有关的那一部分. 我们大概也知道, Looper 的作用, 就是帮助 Handler不断从消息队列中取出要处理的消息. 如果在子线程创建 Handler时, 一定要先调用 Looper.prepare(), 否则将抛出上面那段异常.
```java
new Thread(() -> {
    Looper.prepare();
    Handler handler = new Handler(){
		...
	};
	// 开始循环处理消息
    Looper.loop();
}).start();
```

而在主线程, 则不需要 Looper.prepare() 就可以直接创建 Handler. 因此应该是在某个地方已经帮我们创建好了 Looper. 我们知道Android程序的UI线程(主线程) 是个永不退出的消息循环, 程序的入口则在 ActivityThread 这个类里.
```java
public final class ActivityThread extends ClientTransactionHandler {
	public static void main(String[] args) {
		...
		Looper.prepareMainLooper();
		...
		Looper.loop();
	}
}
```
继续看 prepareMainLooper():
```java
public static void prepareMainLooper() {
	// 调用了 prepare
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```
再看看 prepare() 吧:
```java
public static void prepare() {
    prepare(true);
}
// quitAllowed 表示looper是否允许退出
private static void prepare(boolean quitAllowed) {
	// 一个 ThreadLocal, 只有一个 Looper
	// 解释下ThreadLocal: ThreadLocal是线程本地存储区, 每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能相互访问.
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
	// 创建 Looper, 并保存在 ThreadLocal 中
    sThreadLocal.set(new Looper(quitAllowed));
}
private Looper(boolean quitAllowed) {
	// 创建消息队列 MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

到这里, 我们已经知道, Handler需要依靠 Looper 来循环获取消息. 而在子线程创建Handler时, 需要我们自己创建Looper. Looper 保存在 ThreadLocal中, 且一个线程只会有一个 Looper, 而Looper中则维护着一个MessageQueue(消息队列). 因此在同一个线程中, 不管有多少个 Handler, 它们都共用一个 Looper和一个消息队列.

Handler的构造方法先看到这里, 下面在看看 Handler 发送消息相关的方法:

### 发送消息之 sendMessage
```java
public final boolean sendMessage(Message msg){
    return sendMessageDelayed(msg, 0);
}
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

还有其它的 sendXXX方法, 就不一一列出了, 反正最终都调用了 enqueueMessage()方法:
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	// 注意这里, 将 msg.target 设置为当前 Handler
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
	// 调用了 MessageQueue 的 enqueueMessage() 方法将消息放入消息队列
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

继续看看 MessageQueue.enqueueMessage() 是怎么添加消息的
```java
boolean enqueueMessage(Message msg, long when) {
	// 上面说了, msg.target 指向的是 Handler
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
		// 如果当前没有要处理的消息 或 新添加的消息要立刻处理 或 新添加的消息要优先于当前消息处理
        if (p == null || when == 0 || when < p.when) {
            // 将新添加消息的 next 指向 p   --- 可以发现, "消息队列" 其实是个单向链表结构
            msg.next = p;
			// 将新添加消息设置为当前要处理的消息
            mMessages = msg;
            needWake = mBlocked;
        } else {
			// 将消息按时间顺序插入到MessageQueue
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
				// p 设置为上一条消息
                prev = p;
				// 将 p 指向其原来的下一条消息
                p = p.next;
				// 如果到了结尾了, 则跳出循环
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
			// 将 新消息添加到链表中
            msg.next = p; 
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

看完 sendMessage(), 下面再看看发送消息的另外一种方式, post()

### 发送消息之 post
```java
public final boolean post(Runnable r){
   return  sendMessageDelayed(getPostMessage(r), 0);
}
public final boolean postDelayed(Runnable r, long delayMillis){
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
```

可以发现, 其实还是调用了 sendMessage 方法. 不过有一点不同是多了一个 getPostMessage, 用来从一个 Runnable 生成一个 Message.
```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
	// 注意这里将 runnable设置给了 message.callback
    m.callback = r;
    return m;
}
```

到这里, 关于将消息添加到消息队列的部分算是结束了. 最后在看下 Looper是如何循环获取消息的.

### 处理消息
```java
public static void loop() {
	// 从 ThreadLoacal中取出 Looper
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
	// 拿到 Looper 中的消息队列 MessageQueue
    final MessageQueue queue = me.mQueue;
    ... 省略 ...
	// 循环
    for (;;) {
		// 通过 messageQueue.next() 取出下一条要处理的消息
        Message msg = queue.next(); 
        if (msg == null) {
            // 没有要处理的消息, 结束循环
            return;
        }
		... 又省略了 ...
        try {
			// 将消息交给 Handler 处理.  还记得 msg.target 就是 Handler 吗
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            ...
        }
        ...
		// 消息回收. 为了提高效率, 避免频繁的创建 Message. 因此其维护了一个'消息池', 便于复用
        msg.recycleUnchecked();
    }
}
```

快要结束了, 再瞄一眼 MessageQueue 的 next() 方法.
```java
Message next() {
    ... 省略 ...
    for (;;) {
        ...
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
			// 记录当前要处理的 Message
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Handler 为空, 则向后查找
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 该消息还没到处理的时候
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 返回这条消息, 并设置好下一条要处理的消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 为空, 没有要处理的消息
                nextPollTimeoutMillis = -1;
            }
			...
        }
		...
    }
}
```

最后再看下 Handler 的 dispatchMessage() 方法:
```java
public void dispatchMessage(Message msg) {
	// 还记得 post 发送消息吗? msg.callback 就是 post方法中传过来的 Runnable
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
		// 如果创建 Handler 时, 传入了 Callback, 则交给 Callback 去处理 (一般好像都没传过)
		// public Handler(Callback callback)
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
		// 终于找到了 handleMessage
        handleMessage(msg);
    }
}
...
private static void handleCallback(Message message) {
	// 调用 Runnable 的 run 方法
    message.callback.run();
}
```

### 总结
- 整个消息机制中, 涉及到了 Handler、Message、Looper、ThreadLoacal、MessageQueue等对象
- Message就是需要传递的消息, 里面可以携带数据;
- Handler是一个面向开发者的辅助类, 平时我们就用它发送和处理消息;
- MessageQueue里面维护了一个消息队列(其实是一个单链表结构). 我们通过Handler发送消息时, 其实最终也是调用了 MessageQueue的方法将消息添加到队列中.
- Looper则有两个作用, 一是将它内部的 MessageQueue 提供给Handler, 而是不断循环从MessageQueue中去除消息交给Handler处理.
- 另外, 每个线程只有一个Looper, 其保存在ThreadLoacal中. 同一个线程中的所有Handler都共用此一个Looper, 以及它里面的一个MessageQueue.
- 最后, Handler的工作离不开Looper, 因此创建Handler前要通过 Looper.prepare() 创建 Looper, 同时要通过 Looper.loop()开启循环. 而主线程中之所以不用我们手动创建, 是因为ActivityThread已经帮我们创建好了.

### 拓展-Handler内存泄漏
通过前面的分析, 我们知道 message.target持有 Handler的引用, 如果又用常规的内部类的方式使用 Handler, 则 Handler又持有其外部类的引用(比如Activity), 因此可能导致内存泄漏. 因此可以Handler定义为单路的类, 或者使用静态内部类, 并通过弱引用的方式持有 Activity.
```java
private static class MyHandler extends Handler {
    private WeakReference<MyActivity> mActivity;
    public MyHandler(MyActivity activity) {
        this.mActivity = new WeakReference(activity);
    }
    @Override
    public void handleMessage(final Message msg) {
        MyActivity activity = mActivity.get();
        if (activity != null) {
            activity.handleMessage(msg);
        }
    }
}
// ... 别忘了再页面销毁时清除消息
protected void onDestroy() {
    mHandler.removeCallbacksAndMessages(null);
    super.onDestroy();
}
```