Handler
===
1. 一个app进程启动后先进行初始化： Looper.prepareMainLooper()
2. new ActivityThread()
3. Looper.loop()


# prepare()

```
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

* quitAllowed 意思是 能不能退出loop循环，Looper.prepareMainLooper 是false也就是不能退出主线程的循环，而子线程调用prepare()是true 允许退出。
* 通过 sThreadLocal.set(new Looper(quitAllowed))，完成loop与thread的双向绑定
* 随后初始化MessageQueue


# loop()

* 在这里开启了无线循环
* MessageQueue的next方法从消息队列中取出一条消息，如果此时消息队列中有Message，那么next方法会立即返回该Message，如果此时消息队列中没有Message，那么next方法就会**阻塞式**地等待获取Message。 


# ThreadLocal
ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

* 底层数据结构是 ThreadLocalMap.Entry，注意是弱引用包起来的

```
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        .......
        private Entry[] table;
```
* 这个table数组的容量必须是2的N次方，和HashMap原理类似，为了保证哈希散列更均匀
* set()操作除了存储元素外，还有一个很重要的作用，就是replaceStaleEntry()和cleanSomeSlots()，这两个方法可以清除掉key == null 的实例，防止内存泄漏
* 每次使用完ThreadLocal，都调用它的remove()方法，清除数据
* 而ThreadLocalMap的set()采用开放定址法 解决哈希冲突
* [看这](https://www.cnblogs.com/molao-doing/articles/9097929.html)

# 如何在子线程启动Handler
```
class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```

[深入理解Handler](https://blog.csdn.net/iispring/article/details/47180325)

-


# view.post和handler.post
```
    /**
     * <p>Causes the Runnable to be added to the message queue.
     * The runnable will be run on the user interface thread.</p>
     *
     * @param action The Runnable that will be executed.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     *
     * @see #postDelayed
     * @see #removeCallbacks
     */
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        // Postpone the runnable until we know on which thread it needs to run.
        // Assume that the runnable will be successfully placed after attach.
        getRunQueue().post(action);
        return true;
    }
```
* 这是view.post的源码，可以看出关键在于AttachInfo。当我们在 Activity 的 onCreate() 里执行 View.post(Runnable) 时，因为这时候 View 还没有 attachedToWindow，所以这些 Runnable 操作其实并没有被执行，而是先通过 HandlerActionQueue 缓存起来。
* dispatchAttachedToWindow() 被调用的时机是在 ViewRootImol 的 performTraversals() 中，该方法会进行 View 树的测量、布局、绘制三大流程的操作。

# 继续看RunQueue
```
/**
 * Class used to enqueue pending work from Views when no Handler is attached.
 *
 * @hide Exposed for test framework only.
 */
public class HandlerActionQueue {
    private HandlerAction[] mActions;
    private int mCount;
```
注释说的很清楚：用于没被attach的情况，也就是说**在onCreate里使用handler.post，任务会在view绘制之前被执行，其他地方就被放到了MessageQueue**


# 问题
* 如果在onCreate里想要获得view的正确宽高应该用view.post还是handler.post ？
* [解析](https://www.cnblogs.com/dasusu/p/8047172.html)
* [非 UI 线程能invalidate么？](https://www.jianshu.com/p/753441fcbad2)


