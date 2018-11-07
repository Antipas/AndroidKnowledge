Binder
===

# Linux进程通信方式
 * 管道
 * Socket
 * 信号
 * 消息队列
 * Mmap

在Android系统中，负责各个用户进程通过Binder通信的内核模块叫做Binder驱动，它的原理是mmap：mmap将一个文件或者其它对象映射进内存。文件被映射到多个页上，如果文件的大小不是所有页的大小之和，最后一个页不被使用的空间将会清零。mmap执行相反的操作，删除特定地址区域的对象映射。
当使用mmap映射文件到进程后,就可以直接操作这段虚拟地址进行文件的读写等操作,不必再调用read,write等系统调用.但需注意,直接对该段内存写时不会写入超过当前文件大小的内容。

但是mmap需要在两个进程之间进行两次拷贝，但Binder只需一次。[这里说的好](https://blog.csdn.net/appdsn/article/details/79311455)

# 为什么用Binder
主要有两点，性能和安全。在移动设备上，广泛地使用跨进程通信肯定对通信机制本身提出了严格的要求；Binder相对出传统的Socket方式，更加高效；另外，传统的进程通信方式对于通信双方的身份并没有做出严格的验证，只有在上层协议上进行架设；比如Socket通信ip地址是客户端手动填入的，都可以进行伪造；而**Binder机制从协议本身就支持对通信双方做身份校检**，因而大大提升了安全性。这个也是Android权限模型的基础。


# 跨进程原理

**CS架构**

[看这个图](https://blog.csdn.net/appdsn/article/details/79311290)

1. 注册成为ServiceManager。比如AMS、WMS，在系统里一开始就初始化到了一个Hashmap的缓存里，保存自己的名字和Binder。单例模式。

2. Client向SM查询。Service查询到之后返回的是**Binder代理对象。**Binder跨进程传输并不是真的把一个对象传输到了另外一个进程；传输过程好像是Binder跨进程穿越的时候，它在一个进程留下了一个真身，在另外一个进程幻化出一个影子（这个影子可以很多个）；Client进程的操作其实是对于影子的操作，影子利用Binder驱动最终让真身完成操作。代理模式。

* IBinder是一个接口，它代表了一种跨进程传输的能力；只要实现了这个接口，就能将这个对象进行跨进程传递；这是驱动底层支持的；在跨进程数据流经驱动的时候，驱动会识别IBinder类型的数据，从而自动完成不同进程Binder本地对象以及Binder代理对象的转换。


* IBinder负责数据传输，那么client与server端的调用契约（这里不用接口避免混淆）呢？这里的IInterface代表的就是远程server对象具有什么能力。具体来说，就是aidl里面的接口。

* Java层的Binder类，代表的其实就是Binder本地对象。BinderProxy类是Binder类的一个内部类，它代表远程进程的Binder对象的本地代理；这两个类都继承自IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。
* 在使用AIDL的时候，编译工具会给我们生成一个Stub的静态内部类；这个类继承了Binder, 说明它是一个Binder本地对象，它实现了IInterface接口，表明它具有远程Server承诺给Client的能力；Stub是一个抽象类，具体的IInterface的相关实现需要我们手动完成，这里使用了策略模式。
* aidl工具根据aidl文件自动生成的java接口的解析：首先，它声明了几个接口方法，同时还声明了几个整型的id用于标识这些方法，id用于标识在transact过程中客户端所请求的到底是哪个方法；接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，**当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub内部的代理类Proxy来完成。**
	1. asInterface(android.os.IBinder obj)：用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，**如果客户端和服务端是在同一个进程中，那么这个方法返回的是服务端的Stub对象本身，否则返回的是系统封装的Stub.Proxy对象。**
	2. asBinder：返回当前Binder对象。
	3. onTransact：**这个方法运行在服务端中的Binder线程池中**，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。
* 两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以
* 不同的进程会有不同的虚拟机、Application、内存空间，比如：
	1. **静态成员和单例模式完全失效；**即使两个进程访问同一个类的静态变量值都有可能不一样，从fork出新进程开始，一生二，从此分道扬镳
	2. 线程同步机制完全失效：无论锁对象还是锁全局对象都无法保证线程同步；同上
	3. SharedPreferences的可靠性下降：SharedPreferences不支持并发读写；
	4. Application会多次创建

* Binder死亡通知：
	1. linkToDeath和unlinkToDeath
	2. 使用Service建立的onServiceDisconnected
	3. 注意二者区别：一个是UI线程，一个是客户端线程池

```
	binder.linkToDeath(new IBinder.DeathRecipient(){
		@Override
		public void binderDied(){
			//	运行在客户端的Binder线程池
			manager.asBinder().unlinkToDeath(this, 0)
		}
	} ,0)

```
* AIDL权限验证
	* 写入AndroidManifest中的 < permission ...... />， 然后checkCallingOrSelfPermission("permission.....")
	* 可以在 onBind验证，这样验证失败的客户端无法绑定服务, return null
	* 也可以在 onTransact中验证，return false


# AMS
再去翻阅系统的ActivityManagerServer的源码，就知道哪一个类是什么角色了：IActivityManager是一个IInterface，它代表远程Service具有什么能力，ActivityManagerNative指的是Binder本地对象（类似AIDL工具生成的Stub类），这个类是抽象类，它的实现是ActivityManagerService；因此对于AMS的最终操作都会进入ActivityManagerService这个真正实现；同时如果仔细观察，ActivityManagerNative.java里面有一个非公开类ActivityManagerProxy, 它代表的就是Binder代理对象；是不是跟AIDL模型一模一样呢？那么ActivityManager是什么？他不过是一个管理类而已，可以看到真正的操作都是转发给ActivityManagerNative进而交给他的实现ActivityManagerService 完成的。

