MVVM
===
个人认为：不一定非要用databinding，架构中有ViewModel就算MVVM

## 从响应式编程说起
* 观察者模式
	* 订阅发布模式是实现响应式的基础，这种模式我们都很熟悉了，主要是通过把观察者的回调注册进被观察者来实现二者的订阅关系，当被观察者notify的时候，则所有的观察就会自动响应。这种模式也实现了观察者和被观察者的解耦。
* 代表： Rxjava 、 EventBus


## 组合
> Jetpack : LiveData + Lifecycle + Room + Navigation

## 参考
* [使用 MVVM 开发 GitHub 客户端，以及对编程的一些思考](https://www.jianshu.com/p/b03710f19123)
* [响应式编程在Android 中的一些探索](https://juejin.im/post/5c026915f265da615876e42e)
