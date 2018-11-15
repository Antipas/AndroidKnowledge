# EventBus原理

## 观察者模式

发布者即发送消息的一方（即调用EventBus.getDefault().post(event)的一方），订阅者即接收消息的一方（即调用EventBus.getDefault().register()的一方）

## register()
1. 找到订阅者的方法。找出传进来的订阅者的所有订阅方法，然后遍历订阅者的方法。
2. 通过反射来获取订阅者中所有的方法，并根据方法的类型，参数和注解找到订阅方法。
3. 订阅者的注册

```
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // 通过反射来获取订阅者中所有的方法
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // 

```

## 粘性事件
普通事件是先注册，然后发送事件才能收到；而粘性事件，在发送事件之后再订阅该事件也能收到。此外，粘性事件会保存在内存中，每次进入都会去内存中查找获取最新的粘性事件，除非你手动解除注册。

调用HandlerPoster.enqueue入队，（HandlerPoster extends Handler）最终的数据结构是PendingPost，单链表实现的队列

```
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }
```


在这个Handler里开启无限循环接受事件：

```
    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
```

可以看出handleMessage对事件链表的处理是不断循环获取链表中的PendingPost，当链表中的事件处理完毕后退出while循环。或者是在规定的时间内maxMillisInsideHandleMessage没有将事件处理完，则继续调用sendMessage。继而重新回调handleMessage方法，事件会继续执行。**这是为了避免队列过长或者事件耗时较长时，while循环阻塞主线程造成卡顿。**
