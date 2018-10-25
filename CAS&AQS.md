CAS&AQS
===

# CAS
可以进行原子性的访问和更新操作，并发时先对比再置换

```
	public final boolean compareAndSet(int expectedValue, int newValue)

```

 * 副作用：ABA问题
 * 这是通常只在 lock-free 算法下暴露的问题。CAS 是在更新时比较前值，如果对方只是恰好相同，例如期间发生了 A -> B -> A 的更新，仅仅判断数值是 A，可能导致不合理的修改操作。针对这种情况，Java 提供了 AtomicStampedReference 工具类，通过为引用建立类似版本号（stamp）的方式，来保证 CAS 的正确性。


# AQS
AQS 内部数据和方法，可以简单拆分为：

 * 一个 volatile 的整数成员表征状态，同时提供了 setState 和 getState 方法。
 * 一个先入先出（FIFO）的等待线程队列，以实现多线程间竞争和等待，这是 AQS 机制的核心之一。
 * 各种基于 CAS 的基础操作方法，以及各种期望具体同步结构去实现的 acquire/release 方法。
 * State的意思：大于0取消状态，小于0有效状态，表示等待状态四种cancelled，signal，condition，propagate

以 ReentrantLock 为例，它内部通过扩展 AQS 实现了 Sync 类型，以 AQS 的 state 来反映锁的持有情况。

```
	private final Sync sync;
	abstract static class Sync extends AbstractQueuedSynchronizer { …}

```

ReentrantLock 对应 acquire release 操作就调用了AQS 内部的tryAcquire 和 acquireQueued。

```
	public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
	}

```

在 ReentrantLock 中，tryAcquire 逻辑实现在 NonfairSync 和 FairSync 中，分别提供了进一步的非公平或公平性方法，而 AQS 内部 tryAcquire 仅仅是个接近未实现的方法（直接抛异常），这是留个实现者自己定义的操作。


以非公平的 tryAcquire 为例，其内部实现了如何配合状态与 CAS 获取锁，注意，对比公平版本的 tryAcquire，它在锁无人占有时，并不检查是否有其他等待者，这里体现了非公平的语义。

```
	final boolean nonfairTryAcquire(int acquires) {
	    final Thread current = Thread.currentThread();
	    int c = getState();// 获取当前 AQS 内部状态量
	    if (c == 0) { // 0 表示无人占有，则直接用 CAS 修改状态位，
	    	if (compareAndSetState(0, acquires)) {// 不检查排队情况，直接争抢
	        	setExclusiveOwnerThread(current);  // 并设置当前线程独占锁
	        	return true;
	    	}
	    } else if (current == getExclusiveOwnerThread()) { 
	    // 即使状态不是 0，也可能当前线程是锁持有者，因为这是再入锁
	    	int nextc = c + acquires;
	    	if (nextc < 0) // overflow
	        	throw new Error("Maximum lock count exceeded");
	    	setState(nextc);
	    	return true;
		}
		return false;
	}

```
接下来再来分析 acquireQueued，如果前面的 tryAcquire 失败，代表着锁争抢失败，进入排队竞争阶段。这里就是我们所说的，利用 FIFO 队列，实现线程间对锁的竞争的部分，算是是 AQS 的核心逻辑。

当前线程会被包装成为一个排他模式的节点（EXCLUSIVE），通过 addWaiter 方法添加到队列中。acquireQueued 的逻辑，简要来说，就是如果当前节点的前面是头节点，则试图获取锁，一切顺利则成为新的头节点；否则，有必要则等待。

```
	final boolean acquireQueued(final Node node, int arg) {
	      boolean interrupted = false;
	      try {
	    	for (;;) {// 循环
	        	final Node p = node.predecessor();// 获取前一个节点
	        	if (p == head && tryAcquire(arg)) { 
	        	// 如果前一个节点是头结点，表示当前节点合适去 tryAcquire
	            	setHead(node); // acquire 成功，则设置新的头节点
	            	p.next = null; // 将前面节点对当前节点的引用清空
	            	return interrupted;
	        	}
	        	if (shouldParkAfterFailedAcquire(p, node)) // 检查是否失败后需要 park
	            	interrupted |= parkAndCheckInterrupt();
	    	}
	       } catch (Throwable t) {
	    	cancelAcquire(node);// 出现异常，取消
	    	if (interrupted)
	        	    selfInterrupt();
	    	throw t;
	      }
	}

```

到这里线程试图获取锁的过程基本展现出来了，tryAcquire 是按照特定场景需要开发者去实现的部分，而线程间竞争则是 AQS 通过 Waiter 队列与 acquireQueued 提供的，在 release 方法中，同样会对队列进行对应操作。