## Activity详细启动流程

startActivity都会跳转到startActivityForResult

ActivityResult对象通过mInstrumentation.execStartActivity()获取

```java
mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
```

### execStartActivity流程
* 遍历ActivityMonitor对象的数组，如果有匹配的Intent信息，判断ActivityMonitor对象是否运行活动启动，如果可以，判断requestcode，如果requestcode大于0，说明需要返回数据。
* Intent的migrateExtraStreamToClipData()，判断是否根据Intent跳转系统的页面
包括
```
1. ACTION_CHOOSER 多种打开方式的选择，例如选择用chrome还是qq浏览器APP打开一个h5页面
2. ACTION_SEND android系统本身的分享跳转
3. ACTION_SEND_MULTIPLE android本身的选择文件跳转
4. MediaStore.ACTION_IMAGE_CAPTURE,MediaStore.ACTION_IMAGE_CAPTURE_SECURE,MediaStore.ACTION_VIDEO_CAPTURE 手机图片和视频的选择跳转，注意ACTION_IMAGE_CAPTURE和ACTION_IMAGE_CAPTURE_SECURE，ACTION_IMAGE_CAPTURE_SECURE具有安全性，不会公开你个人的照片，例如图案锁锁住的
```
* intent.prepareToLeaveProcess(); 准备离开应用程序的进程，进入AMS进程，也意味着要进行进程间的通信了，这一步恰恰解释了为什么Intent传递数据需要序列号而不是直接传递对象，因为直接传递对象的办法在IPC的过程中就废了
* 调用AMS的startActivity方法启动Activity

### 进程间通信的过程
Instrumentation中调用
```java
ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mWho : null,
                        requestCode, 0, null, options);
```
如下
```java
static public IActivityManager getDefault() {
        return gDefault.get();
    }
```
gDefault返回的是asInterface(b),b是ServiceManager.getService("activity")，一个IBinder对象

asInterface返回ActivityManagerProxy对象,这个代理对象的startActivity方法里调用了
```java
mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
```
调用了server进程(ActivityManagerService)的startActivity方法

### ActivityManagerService启动Activity
调用ActivityStackSupervisor的startActivityMayWait方法

* resolveActivity方法，收集Activity的信息，返回一个表示了在AndroidManifest.xml中指定了属性的ActivityInfo对象
* 获取ActiviyContainer对象
* 获取pid和linux的uid
* 检查是否有另一个重的进程在运行中
  1. 如果是一个heavy-weight process,就从LRU缓存中取出（最近最常使用）进程对象
    获取存放Intent发送信息的IntentSender对象，根据这个对象更新ActiivityInfo对象并且启动选择Activity的页面，目的是处理2个一样的Activity可选的启动场景.
  2. 此时调用了startActivityLocked方法来获取返回值，如果需要改变配置，则调用ActivityManagerService的updateConfigurationLocked方法进行配置的更新。


##### 轻重进程的概念
轻量级进程，可以与父进程共享地址空间和系统资源
与普通进程的区别：轻量级进程只有一个最小的执行上下文和调度程序所需要的统计信息
```
* 与处理器竞争：因与特定内核线程关联，所以可以在系统范围内竞争处理器资源
* 使用资源：与父进程共享地址空间
* 调度：与普通进程一样
轻量级线程(LWP)是一种由内核支持的用户线程。它是基于内核线程的高级抽象，因此只有先支持内核线程，才能有LWP。每一个进程有一个或多个LWPs，每个LWP由一个内核线程支持。这种模型实际上就是恐龙书上所提到的一对一线程模型。在这种实现的操作系统中，LWP就是用户线程。
* 与用户线程的区别：LWP虽然本质上属于用户线程，但LWP线程库是建立在内核之上的，LWP的许多操作都要进行系统调用，因此效率不高。而这里的用户线程指的是完全建立在用户空间的线程库，用户线程的建立，同步，销毁，调度完全在用户空间完成，不需要内核的帮助。因此这种线程的操作是极其快速的且低消耗的。
```

#### 关于ActivityContainer
一个ActivityContainer对象包括
```
* Activity栈id
* Activity栈 历史Activity栈中的条目，每一个条目都代表了一个Activity
* ActivityRecord对象
* ActivityDisplay 与当前Activity栈关联的显示对象
```

