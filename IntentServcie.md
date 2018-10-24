IntentService
===
* IntentService本质 = Handler + HandlerThread
* 按顺序、在后台执行
* 若启动IntentService 多次，那么 每个耗时操作 则 以队列的方式 在 IntentService的 onHandleIntent回调方法中依次执行，执行完自动结束。但onStartCommand 也会调用多次
* 为什么不能bindService去执行任务？？因为并不会调用onStart() 或 onStartcommand()，故不会将消息发送到消息队列，那么onHandleIntent()将不会回调，即无法实现多线程的操作，失去了它本身的意义
* [总结](https://www.jianshu.com/p/8a3c44a9173a)


```
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

```
可以看到，start之后把intent包装在message里入队

-

```
    /**
     * Sets intent redelivery preferences.  Usually called from the constructor
     * with your preferred semantics.
     *
     * <p>If enabled is true,
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_REDELIVER_INTENT}, so if this process dies before
     * {@link #onHandleIntent(Intent)} returns, the process will be restarted
     * and the intent redelivered.  If multiple Intents have been sent, only
     * the most recent one is guaranteed to be redelivered.
     *
     * <p>If enabled is false (the default),
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_NOT_STICKY}, and if the process dies, the Intent
     * dies along with it.
     */
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

```
还有个redelivery功能：如果任务执行完成之前Service挂了会自动restart，然后处理最近的一个intent


JobIntentService
===
Android O 之后会自动用JobScheduler，之前用的
* 由于Android O的后台限制，创建后台服务需要使用JobScheduler来由系统进行调度任务的执行，而使用JobService的方式比较繁琐，8.0及以上提供了JobIntentService帮助开发者更方便的将任务交给JobScheduler调度，其本质是Service后台任务在他的OnhandleWork()中进行，子类重写该方法即可


```
static WorkEnqueuer getWorkEnqueuer(Context context, ComponentName cn, boolean hasJobId,
        int jobId) {
    WorkEnqueuer we = sClassWorkEnqueuer.get(cn);
    if (we == null) {
        if (Build.VERSION.SDK_INT >= 26) {
            if (!hasJobId) {
                throw new IllegalArgumentException("Can't be here without a job id");
            }
            we = new JobWorkEnqueuer(context, cn, jobId);
        } else {
            we = new CompatWorkEnqueuer(context, cn);
        }
        sClassWorkEnqueuer.put(cn, we);
    }
    return we;
}
```

这里区分了任务队列的不同，看看Android O之前的处理——CompatWorkEnqueuer 里的入队方法：

```
@Override
void enqueueWork(Intent work) {
    Intent intent = new Intent(work);
    intent.setComponent(mComponentName);
    if (DEBUG) Log.d(TAG, "Starting service for work: " + work);
    if (mContext.startService(intent) != null) {
        synchronized (this) {
            if (!mLaunchingService) {
                mLaunchingService = true;
                if (!mServiceProcessing) {
                    // If the service is not already holding the wake lock for
                    // itself, acquire it now to keep the system running until
                    // we get this work dispatched.  We use a timeout here to
                    // protect against whatever problem may cause it to not get
                    // the work.
                    mLaunchWakeLock.acquire(60 * 1000);
                }
            }
        }
    }
}
```
注意这里使用了PowerManager.WakeLock的PARTIAL_ WAKE_LOCK

 * 有四种flag对应不同的level：
 	* PARTIAL_ WAKE_LOCK：保持CPU 运转，屏幕和键盘灯有可能是关闭的。
 	* SCREEN_ DIM_WAKE_LOCK：保持CPU 运转，允许保持屏幕显示但有可能是灰的，允许关闭键盘灯
	* SCREEN_ BRIGHT_WAKE_LOCK：保持CPU 运转，允许保持屏幕高亮显示，允许关闭键盘灯
	* FULL_ WAKE_LOCK：保持CPU 运转，保持屏幕高亮显示，键盘灯也保持亮度
	* ACQUIRE_ CAUSES_WAKEUP：正常唤醒锁实际上并不打开照明。相反，一旦打开他们会一直仍然保持(例如来世user的activity)。当获得wakelock，这个标志会使屏幕或/和键盘立即打开。一个典型的使用就是可以立即看到那些对用户重要的通知。
	* ON_ AFTER_RELEASE：设置了这个标志，当wakelock释放时用户activity计时器会被重置，导致照明持续一段时间。如果你在wacklock条件中循环，这个可以用来减少闪烁 
 
 * 因此必须加个权限：
```
<uses-permission android:name="android.permission.WAKE_LOCK"></uses-permission>
```

-

```
@Override
public void onCreate() {
    super.onCreate();
    if (DEBUG) Log.d(TAG, "CREATING: " + this);
    if (Build.VERSION.SDK_INT >= 26) {
        mJobImpl = new JobServiceEngineImpl(this);
        mCompatWorkEnqueuer = null;
    } else {
        mJobImpl = null;
        ComponentName cn = new ComponentName(this, this.getClass());
        mCompatWorkEnqueuer = getWorkEnqueuer(this, cn, false, 0);
    }
}
```
这种区分在onCreate里也有体现，这里看看Android O之后的实现——JobServiceEngineImpl

```
/**
 * This is a task to dequeue and process work in the background.
 */
final class CommandProcessor extends AsyncTask<Void, Void, Void> {
    @Override
    protected Void doInBackground(Void... params) {
        GenericWorkItem work;

        if (DEBUG) Log.d(TAG, "Starting to dequeue work...");

        while ((work = dequeueWork()) != null) {
            if (DEBUG) Log.d(TAG, "Processing next work: " + work);
            onHandleWork(work.getIntent());
            if (DEBUG) Log.d(TAG, "Completing work: " + work);
            work.complete();
        }

        if (DEBUG) Log.d(TAG, "Done processing work!");

        return null;
    }

    @Override
    protected void onCancelled(Void aVoid) {
        processorFinished();
    }

    @Override
    protected void onPostExecute(Void aVoid) {
        processorFinished();
    }
}
```
由此可见，最终还是通过了AsyncTask执行任务，而且是在任务线程里回调onHandleWork

```
@Override
void enqueueWork(Intent work) {
    if (DEBUG) Log.d(TAG, "Enqueueing work: " + work);
    mJobScheduler.enqueue(mJobInfo, new JobWorkItem(work));
}
```
最终，这个任务还是通过JobScheduler来调度的

[总结](https://blog.csdn.net/houson_c/article/details/78461751)


# 联想进程保活

别再用守护进程、锁屏一像素、无声音乐的方式了，现在的方式是：

 * [JobScheduler](https://www.aliyun.com/jiaocheng/15370.html) 即每个需要后台的业务处理为一个job,通过系统管理job,来提高资源的利用率,从而提高性能,节省电源。这样又能满足APP开发商的要求,又能满足系统性能的要求。

 * [WorkManager](https://blog.csdn.net/guiying712/article/details/80386338) 可以轻松地让异步任务延迟执行以及何时运行它们，这些API可让我们创建任务并将其交给WorkManager，以便立即或在适当的时间运行。例如，应用程序可能需要不时从网络下载新资源，我们可以使用WorkManager API设置一个任务，然后选择适合它运行的环境（例如“仅在设备充电和联网时”），并在符合条件时将其交给 WorkManager 运行，即使该应用程序被强制退出或者设备重新启动，这个任务仍然可以保证运行。