---
title: Toast源码分析
top: false
cover: true
categories: Android
tags:
  - Toast
  - BadTokenException
date: 2019-03-26 12:39:36
img:
coverImg:
summary: 
---

Toast作为向用户展示提示信息的一种方式, 既不会像Dialog一样打断用户的操作体验, 也不会响应任何点击事件, 并且会在短暂的显示之后自动消失. 因此常常会用来显示一些不是非常重要的提示.

### Toast的用法
最简单的用法, 只需一行代码就能展示一段提示信息:
```java
Toast.makeText(this, "toast", Toast.LENGTH_SHORT).show();
```

不过这种方式不能对默认的样式进行修改, 并且在不同的ROM上, 可能会表现出不一样的结果. 所以Toast也提供了一些方法可以让我们做一些自定义的设置.
```java
// 设置要显示的 view
public void setView(View view) 
// 设置左右边距
public void setMargin(float horizontalMargin, float verticalMargin) 
// 设置对齐方式, 以及 水平 和 垂直方向的偏移
public void setGravity(int gravity, int xOffset, int yOffset) 
```

如果觉得以上这些方法还不够用的话, 甚至可以拿到它的布局属性LayoutParams, 想怎么改就怎么改.(不过是隐藏方法, 需要反射获取)
```java
/**
 * Gets the LayoutParams for the Toast window.
 * @hide
 */
public WindowManager.LayoutParams getWindowParams()
```

Toast的用法非常简单, 下面通过简单的源码分析了解下它的工作原理

### Toast的makeText方法
```java
public static Toast makeText(Context context, @StringRes int resId, @Duration int duration)throws Resources.NotFoundException {
    return makeText(context, context.getResources().getText(resId), duration);
}
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
    return makeText(context, null, text, duration);
}
 /** 
  * @param looper         用来循环处理消息, 后面有用到
  * @param text           提示的文案
  * @param duration       显示时间
  * @hide 隐藏方法
  */
public static Toast makeText(@NonNull Context context, @Nullable Looper looper, @NonNull CharSequence text, @Duration int duration) {
    // 通过构造方法 new 了一个 Toast 实例
    Toast result = new Toast(context, looper);
    // 初始化布局
    LayoutInflater inflate = (LayoutInflater)context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    // 设置文案
    tv.setText(text);
    // 设置显示的view
    result.mNextView = v;
    // 设置时间
    result.mDuration = duration;
    return result;
}
```

可以看出 makeText 方法中只是创建了一个 Toast, 然后设置了要显示的view, 还有时间. 下面看看 Toast 的构造方法

### Toast的构造方法
```java
public Toast(Context context) {
    this(context, null);
}
public Toast(@NonNull Context context, @Nullable Looper looper) {
    mContext = context;
    // new 了一个 TN
    mTN = new TN(context.getPackageName(), looper);
    // 设置垂直偏移
    mTN.mY = context.getResources().getDimensionPixelSize(com.android.internal.R.dimen.toast_y_offset);
    // 设置对齐方式
    mTN.mGravity = context.getResources().getInteger(com.android.internal.R.integer.config_toastDefaultGravity);
}
```

TN是什么? 这个暂时先放一放, 继续看完 Toast的 show 方法再说:

### Toast的show方法
```java
public void show() {
    if (mNextView == null) {
        throw new RuntimeException("setView must have been called");
    }
    // ???
    INotificationManager service = getService();
    // 获取包名
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    // 将要显示的view也设置给了 TN
    tn.mNextView = mNextView;
    try {
        // 调用 service 的 enqueueToast方法, 应该就是用来显示Toast
        // 传入了包名, TN对象的实例,  时长
        service.enqueueToast(pkg, tn, mDuration);
    } catch (RemoteException e) {
        // Empty
    }
}
```

继续看看 getService 获得的是个什么鬼?
```java
static private INotificationManager getService() {
    if (sService != null) {
        return sService;
    }
    // 典型的aidl写法: 通过 ServiceManager 找到一个名字叫 "notification" 的Binder, 然后创建一个本地的代理对象
    // 关于Binder的原理这里就不展开了, 不太了解的可以查查资料
    sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
    return sService;
}
```

既然看到了这里, 下面肯定要找到这个Binder才能继续跟下去了. 根据系统一惯的命名风格, 这个aidl接口的名字叫 INotificationManager. 那我们就全局搜一下 NotificationManagerService, 发现果然有; 然后再搜一下 enqueueToast 方法, 好巧又搜到了; 看看这个方法归属的类:
```java
private final IBinder mService = new INotificationManager.Stub() {
    ...
    @Override
    public void enqueueToast(String pkg, ITransientNotification callback, int duration){
        ...
    }
    ...
}
```

果然实现了 INotificationManager.Stub, 不用怀疑, 就是这里了. 在研究enqueueToast这个方法之前, 我们先回去搞清楚 TN到底是个什么东西.
```java
private static class TN extends ITransientNotification.Stub {
    ...
    TN(String packageName, @Nullable Looper looper) {
         final WindowManager.LayoutParams params = mParams;
         ...
         // 类型是 TYPE_TOAST
         params.type = WindowManager.LayoutParams.TYPE_TOAST;
         // 保持屏幕常亮, 不获得焦点, 不处理触摸事件
         params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        ... (设置一些布局属性)省略 ...
        if (looper == null) {
            looper = Looper.myLooper();
            if (looper == null) {
                // 需要 Looper 来处理消息
                throw new RuntimeException("Can't toast on a thread that has not called Looper.prepare()");
            }
        }
        // Handler将在Looper所在的线程处理消息
        mHandler = new Handler(looper, null) {
            ... 
        };
    }
}
```

是不是很熟悉, 原来TN也是一个Binder. 其实TN的作用, 就类似于接口回调的功能. 我们在show一个Toast时, 会通过Binder机制访问处于系统进程的NotificationManagerService, 同时也在app进程向 SystemService注册了一个名字叫 "android.app.ITransientNotification" 的Binder. NotificationManagerService在处理完相关逻辑后, 会通过这个Binder回调到app进程, 让Toast处理后续逻辑.

弄清楚这些类的关系和各自的作用后, 再继续研究enqueueToast方法.

### NotificationManagerService里面的 enqueueToast
```java
public void enqueueToast(String pkg, ITransientNotification callback, int duration){
    ... 
    // 包名和TN不能为空
    if (pkg == null || callback == null) {
        Slog.e(TAG, "Not doing toast. pkg=" + pkg + " callback=" + callback);
        return ;
    }
    // 是否系统Toast
    final boolean isSystemToast = isCallerSystemOrPhone() || ("android".equals(pkg));
    // 用户是否禁用了 Toast权限
    final boolean isPackageSuspended = isPackageSuspendedForUser(pkg, Binder.getCallingUid());
    // 如果不是系统应用, 且用户禁用了Toast, 则不处理后续逻辑
    if (ENABLE_BLOCKED_TOASTS && !isSystemToast && (!areNotificationsEnabledForPackage(pkg, Binder.getCallingUid()) || isPackageSuspended)) {
        Slog.e(TAG, "Suppressing toast from package " + pkg + (isPackageSuspended
                        ? " due to package suspended by administrator."
                        : " by user request."));
        return;
    }
    // 同步锁
    synchronized (mToastQueue) {
        int callingPid = Binder.getCallingPid();
        long callingId = Binder.clearCallingIdentity();
        try {
            ToastRecord record;
            int index;
            // 是否已经处于 Toast队列中了
            if (!isSystemToast) {
                index = indexOfToastPackageLocked(pkg);
            } else {
                index = indexOfToastLocked(pkg, callback);
            }
            if (index >= 0) {
                // 如果已经处于队列中, 则更新下时间还和TN
                record = mToastQueue.get(index);
                record.update(duration);
                record.update(callback);
            } else {
                // 创建一个新的ToastRecord, 添加到序列中. (注意设置了一个 token, 为什么要用 Binder ?)
                Binder token = new Binder();
                mWindowManagerInternal.addWindowToken(token, TYPE_TOAST, DEFAULT_DISPLAY);
                record = new ToastRecord(callingPid, pkg, callback, duration, token);
                mToastQueue.add(record);
                index = mToastQueue.size() - 1;
            }
            keepProcessAliveIfNeededLocked(callingPid);
            // 如果处于Toast队列的最前端, 则直接展示
            if (index == 0) {
                showNextToastLocked();
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
}
```

继续看 showNextToastLocked 方法:
```java
void showNextToastLocked() {
    // 取出队列最前面的 ToastRecord
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        try {
            // 回调到 TN, 并传入了 token  (因为Binder可以跨进程通信)
            record.callback.show(record.token);
            // 延迟一段时间后, 隐藏 Toast
            scheduleTimeoutLocked(record);
            return;
        } catch (RemoteException e) {
            // remove it from the list and let the process die
            int index = mToastQueue.indexOf(record);
            if (index >= 0) {
                mToastQueue.remove(index);
            }
            keepProcessAliveIfNeededLocked(record.pid);
            if (mToastQueue.size() > 0) {
                record = mToastQueue.get(0);
            } else {
                record = null;
            }
        }
    }
}
```

发现 showNextToastLocked 中先是回调到了Toast.TN去显示Toast, 紧接着又开始处理Toast的隐藏逻辑. 关于Toast.TN的部分等下再看, 先看下 scheduleTimeoutLocked:
```java
private void scheduleTimeoutLocked(ToastRecord r){
    mHandler.removeCallbacksAndMessages(r);
    Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
    long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
    // 通过Handler发送一个编码为 MESSAGE_TIMEOUT 的延迟消息, 实现Toast的自动隐藏隐藏
    mHandler.sendMessageDelayed(m, delay);
}
```

搜索关键字 MESSAGE_TIMEOUT, 发现mHandler在处理该消息时, 调用了下面的方法:
```java
private void handleTimeout(ToastRecord record){
    synchronized (mToastQueue) {
        // 找出 ToastRecord 在队列中的索引
        int index = indexOfToastLocked(record.pkg, record.callback);
        if (index >= 0) {
            // 真正的隐藏逻辑在这里
            cancelToastLocked(index);
        }
    }
}
```

关于 NotificationManagerService 部分的快结束了, 最后看下 cancelToastLocked 方法:
```java
void cancelToastLocked(int index) {
    ToastRecord record = mToastQueue.get(index);
    try {
        // 回调 Toast.TN, 隐藏 toast
        record.callback.hide();
    } catch (RemoteException e) {}
    // 移除 ToastRecord
    ToastRecord lastToast = mToastQueue.remove(index);
    // 移除 token
    mWindowManagerInternal.removeWindowToken(lastToast.token, true, DEFAULT_DISPLAY);
    keepProcessAliveIfNeededLocked(record.pid);
    // 继续显示下一个Toast
    if (mToastQueue.size() > 0) {
        showNextToastLocked();
    }
}
```

回到app进程, 看看TN最后是怎么处理Toast的.

### TN的show方法和hide方法
```java
public void show(IBinder windowToken) {
    mHandler.obtainMessage(SHOW, windowToken).sendToTarget();
}
...
public void hide() {
    mHandler.obtainMessage(HIDE).sendToTarget();
}
```

去Handler中转一圈, 发现最后分别调用了下面的两个方法.
```java
// 显示toast
public void handleShow(IBinder windowToken) {
    // 如果此时发现又调用了隐藏或取消, 则直接不显示
    if (mHandler.hasMessages(CANCEL) || mHandler.hasMessages(HIDE)) {
        return;
    }
    if (mView != mNextView) {
        // remove the old view if necessary
        handleHide();
        mView = mNextView;
        Context context = mView.getContext().getApplicationContext();
        String packageName = mView.getContext().getOpPackageName();
        if (context == null) {
            context = mView.getContext();
        }
        mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        ... 省略的都是设置一些属性的步骤 ...
        // 设置token
        mParams.token = windowToken;
        if (mView.getParent() != null) {
            mWM.removeView(mView);
        }
        try {
            // 显示toast. 通过 WindowManager 添加 view 实现
            mWM.addView(mView, mParams);
            trySendAccessibilityEvent();
        } catch (WindowManager.BadTokenException e) {
            // 在主线程卡顿时, 可能由于消息不能及时处理, 导致在向窗口addView时, NotificationManagerService已经移除了token, 此时会出现此异常
        }
    }
}
...
// 隐藏toast
public void handleHide() {
    if (mView != null) {
        if (mView.getParent() != null) {
            // 移除 view
            mWM.removeViewImmediate(mView);
        }
        mView = null;
    }
}
```

关于Toast的分析就到这了, 下面补充一个BadTokenException的异常问题. 

### BadTokenException
通过上面的分析, 我们已经知道了产生该异常的原因, 并且发现源码中已经对异常进行了捕获, 但是在Android7.1上,还是需要我们自己手动解决一下. 直接捕获Toast的show方法是没用的, 因为其内部经过两次IPC通信之后, 真正显示Toast的逻辑并不在show方法中. 我们需要通过反射拿到mTN里面的mHandler对象, 直接在其处理消息的过程中进行捕获.
```java
if (isSdk25()) {
    Field tnField = Toast.class.getDeclaredField("mTN");
    tnField.setAccessible(true);
    Object mTn = tnField.get(mToast);
    Field handlerField = mTn.getClass().getDeclaredField("mHandler");
    handlerField.setAccessible(true);
    Handler handlerOfTn = (Handler) handlerField.get(mTn);
    handlerField.set(mTn, new SafeHandler(handlerOfTn));
}
...
class SafeHandler extends Handler {
    // 用来保存Tn原有handler
    private Handler mNestedHandler;
    public SafeHandler(Handler nestedHandler) {
        // 构造方法里将Tn原有Handler传入
        mNestedHandler = nestedHandler;
    }
    // 在这里捕获
    @Override
    public void dispatchMessage(Message msg) {
        try {
            super.dispatchMessage(msg);
        } catch (WindowManager.BadTokenException e) {}
    }
    @Override
    public void handleMessage(Message msg) {
        // 交由原有Handler处理
        mNestedHandler.handleMessage(msg);
}
```