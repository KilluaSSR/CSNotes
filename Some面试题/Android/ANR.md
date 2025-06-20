> **在不同地方的onCreate/onReceive加入 `Thread.sleep()`，都有谁会产生ANR？**

## 广播接收器 BroadcastReceiver

- onReceive方法：它在主线程执行。在onReceive方法加入`Thread.sleep()`会导致ANR。这是因为广播接收器有时间限制。对于前台广播是10s，后台广播是60s。如果广播接收器的onReceive方法执行超过指定时间，就会出现`Broadcast Queue Timeout`的ANR。
```JAVA
// frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
private final void deliverToRegisteredReceiverLocked(BroadcastRecord r, BroadcastFilter filter) {
    // 设置ANR定时器
    setBroadcastTimeoutLocked(timeoutTime);
    
    try {
        // 调用BroadcastReceiver.onReceive()
        performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                new Intent(r.intent), r.resultCode, r.resultData, r.resultExtras,
                r.ordered, r.initialSticky, r.userId);
    } finally {
        // 处理完成后的清理工作
    }
}

void setBroadcastTimeoutLocked(long timeoutTime) {
    if (!mPendingBroadcastTimeoutMessage) {
        Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
        mHandler.sendMessageAtTime(msg, timeoutTime);
        mPendingBroadcastTimeoutMessage = true;
    }
}
```
## 服务 Service

- onCreate方法：它在主线程执行。在onCreate方法加入`Thread.sleep()`会导致ANR。前台服务20s，后台服务200s，如果没有及时完成onCreate方法，就会产生 `Service Timeout `。
```JAVA
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app) {
    // 设置ANR定时器
    bumpServiceExecutingLocked(r, execInFg, "create");
    
    try {
        // 调用Service.onCreate()
        app.thread.scheduleCreateService(r, r.serviceInfo, 
            mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
            app.getReportedProcState());
    } catch (Exception e) {
        // 出现异常，移除定时器
        serviceDoneExecutingLocked(r, inDestroying, inDestroying);
        throw e;
    }
}

private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    // 发送延时消息
    scheduleServiceTimeoutLocked(r.app);
}

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    Message msg = mAm.mHandler.obtainMessage(ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    // 前台服务20s，后台服务200s
    mAm.mHandler.sendMessageDelayed(msg, proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

## 活动Activity

- 不会产生ANR。根本原因是`ActivityThread.java`的`performLaunchActivity`方法没设置ANR定时器。它只在输入事件处理的时候发生。Input事件分发超时5s会ANR，但是与onCreate无关

## 内容提供器 ContentProvider

- 不会产生ANR。ContentProvider的ANR只在publish过程中产生。

```java
// ActivityManagerService.java
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller, String name, ...) {
    // 只在这里设置ANR定时器
    final long timeout = SystemClock.uptimeMillis() + CONTENT_PROVIDER_PUBLISH_TIMEOUT;
    
    // 等待ContentProvider发布，不是等待onCreate()完成
    synchronized (cpr) {
        while (cpr.provider == null) {
            if (SystemClock.uptimeMillis() >= timeout) {
                // 触发ANR
            }
            cpr.wait(CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }
    }
}
```