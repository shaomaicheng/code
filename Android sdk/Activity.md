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
* resolveActivity方法，收集Activity的信息，返回一个表示了在AndroidManifest.xml中指定了属性的ActivityInfo对象


