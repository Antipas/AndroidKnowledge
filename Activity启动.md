Activity启动和Window
===

![Activity启动](/Activity%E5%90%AF%E5%8A%A8.png)


# 问题:
* 一个进程启动后创建的第一个对象是什么？
* 一个Activity 在onCreate里调startActivity跳转后第二个Activity后，生命周期怎么回调？为什么？
* 一个Activity 在onCreate里 不调用setContentView()，直接调finish() 生命周期怎么回调？为什么？
* DecorView 包括哪些view？
* [启动文章](https://www.jianshu.com/p/72059201b10a)
* [setContentView文章](https://www.jianshu.com/p/687010ccad66)
 


# Activity的attach()
1. attachBaseContext
2. new PhoneWindow
3. setWindowManager

-
# Window
Window是一个抽象的概念，*每一个Window都对应着一个View和一个ViewRootImpl*，Window和View通过ViewRootImpl来建立联系，因此Window并不是不存在的，它是以View的形式存在。这点从ViewManager的定义也可以看得出来，它只提供三个接口方法：addView、updateViewLayout、removeView，这些方法都是针对View的，所以说Window是View的直接管理者。


-
# 回顾window.addview()
WmS 运行在单独的进程中。这里 IWindowSession 执行的 addtoDisplay 操作应该是 IPC 调用。接下来的Window添加过程，**我们会知道每个window创建时，最终都会创建一个 ViewRootImpl 对象。**

ViewRootImpl 是一很重要的类，类似 ApplicationThread 负责跟AmS通信一样，ViewRootImpl 的一个重要职责就是跟 WmS 通信，它通静态变量 sWindowSession（IWindowSession实例）与 WmS 进行通信。

**每个应用进程，仅有一个 sWindowSession 对象**，它对应了 WmS 中的 Session 子类，WmS 为每一个应用进程分配一个 Session 对象。WindowState 类有一个 IWindow mClient 参数，是由 Session 调用 addToDisplay 传递过来的，对应了 ViewRootImpl 中的 W 类的实例。

简单的总结一下，ViewRootImpl通过IWindowSession远程IPC通知WmS，并且由W类接收WmS的远程IPC通知。（这个W类和ActivityThread的H类同样精悍的命名，并且也是同样的工作职责！

-
# WindowManagerGlobal
一个进程就只有一个WindowManagerGlobal对象

* WindowManagerGlobal 存储的东西：

```
    //存储所有Window对应的View
    private final ArrayList<View> mViews = new ArrayList<View>();

    //存储所有Window对应的ViewRootImpl
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();

    //存储所有Window对应的布局参数
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();

    //存储正被删除的View对象（已经调用removeView但是还未完成删除操作的Window对象）     
    private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

* Token：在源码中token一般代表的是Binder对象，作用于IPC进程间数据通讯。并且它也包含着此次通讯所需要的信息，在ViewRootImpl里，token用来表示mWindow(W类，即IWindow)，并且在WmS中只有符合要求的token才能让Window正常显示。**这也是为什么dialog必须要传入Activity的Context，因为它有token。**
	* 应用窗口 : token表示的是activity的mToken(ActivityRecord)
	* 子窗口 : token表示的是父窗口的W对象，也就是mWindow(IWindow)

# WindowManagerGlobal.addview
1. 调整布局参数，并设置token
2. 找到父窗口的token(viewRootImpl的W类，也就是IWindow)
3. 创建ViewRootImpl，并且将view与之绑定。**此时 new W(this)，用来接收WmS消息**
4. 通过ViewRootImpl的setView方法，完成view的绘制流程(performTraversals的三部曲)，并添加到window上。


* [Activity WMS ViewRootImpl三者关系](https://blog.csdn.net/kc58236582/article/details/52088224)
* [Android Window 机制探索](https://blog.csdn.net/qian520ao/article/details/78555397)
* [解析Activity、Window、View三者关系](https://www.jianshu.com/p/aa1ffb414f43)

