Rxjava 
===


## 线程切换原理

* 多次用 subscribeOn 指定上游线程为什么只有第一次有效？
* 上游事件是怎么跑到子线程里执行的？
* 子线程是谁如何创建的
* 异步取消任务原理


```
final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }

```

可以看出：把订阅任务放到了一个Runnable里执行

--

```
public abstract static class Worker implements Disposable {
  
        @NonNull
        public Disposable schedule(@NonNull Runnable run) {
            return schedule(run, 0L, TimeUnit.NANOSECONDS);
        }

  
        @NonNull
        public abstract Disposable schedule(@NonNull Runnable run, long delay, @NonNull TimeUnit unit);        
    }


```
之后会把任务封装为Worker，通过Worker的schedule方法开始执行这个task。


## Schedulers.newThread()
**在NewThreadWorker类里：**


```
public static ScheduledExecutorService create(ThreadFactory factory) {
        final ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, factory);
        if (PURGE_ENABLED && exec instanceof ScheduledThreadPoolExecutor) {
            ScheduledThreadPoolExecutor e = (ScheduledThreadPoolExecutor) exec;
            POOLS.put(e, exec);
        }
        return exec;
    }

```

可以看出：在NewThreadScheduler的createWorker()方法中，通过其构建好的线程工厂，在Worker实现类的构造函数中创建了一个ScheduledExecutorService的实例，**核心线程为1**，是通过SchedulerPoolFactory创建的。

```
    public ScheduledAction scheduleActual(final Action0 action, long delayTime, TimeUnit unit) {
        Action0 decoratedAction = schedulersHook.onSchedule(action);
        ScheduledAction run = new ScheduledAction(decoratedAction);
        Future<?> f;
        if (delayTime <= 0) {
            f = executor.submit(run);
        } else {
            f = executor.schedule(run, delayTime, unit);
        }
        run.add(f);

        return run;
    }
```

## subscribeOn问题： 
因为上游Observable只有一个任务，就是subscribe(准确的来说是subscribeActual()），而subscribeOn 要做的事情就是把上游任务切换到一个指定线程里，那么一旦被切换到了某个指定的线程里，后面的切换不就是没有意义了

## 异步取消任务
当递归执行subscription cancel的时候，总会跑到某个subscriber里面，我们称呼它为A，它的worker正在执行，虽然执行代码的片段不一定是A的run或者是onNext(有可能是它下游的某个subscriber) 但是一定会运行在A的worker里面，所以执行worker.dipose停掉这个worker就可以取消异步任务(所有的runnable都会被封装成为FutureTask 从而可以被取消)


**重要结论：调用subscribe时的从下往上是subscribeOn切换线程,之后调用onNext传递数据时的从上往下是ObserveOn切换线程.**


## AndroidSchedulers.mainThread

参考Worker实现类HandlerWorker的schedule()

```
public Disposable schedule(Runnable run, long delay, TimeUnit unit) {
    /**忽略一些代码**/
    run = RxJavaPlugins.onSchedule(run);

    ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);

    Message message = Message.obtain(handler, scheduled);
    message.obj = this; // Used as token for batch disposal of this worker's runnables.

    handler.sendMessageDelayed(message, unit.toMillis(delay));

    if (disposed) {
          handler.removeCallbacks(scheduled);
          return Disposables.disposed();
    }
    return scheduled;
}

```

这里传入了主线程的handler去执行

## 参考
[文章1](https://www.jianshu.com/p/3dd582bb10cc)

[文章2](https://www.jianshu.com/p/7ab1049c3ea6)
