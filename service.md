Service
===

## start 和 bind一起用的情况
1. *先bindService,再startService：*
在bind的Activity退出的时候,Service会执行unBind方法而不执行onDestory方法,因为有startService方法调用过,所以Activity与Service解除绑定后会有一个与调用者没有关连的Service存在
2. *先bindService,再startService：*再调用stopService,
Service的onDestory方法不会立刻执行,因为有一个与Service绑定的Activity,但是在Activity退出的时候,会执行onDestory,如果要立刻执行stopService,就得先解除绑定

3. *先startService，再bindService：*首先在主界面创建时，startService(intent)启动方式开启服务，保证服务长期后台运行；
然后调用服务时，bindService(intent, connection, BIND_AUTO_CREATE)绑定方式绑定服务，这样可以调用服务的方法；
调用服务功能结束后，unbindService(connection)解除绑定服务，置空中介对象；
最后不再需要服务时，stopService(intent)终止服务。

4. [文章](https://blog.csdn.net/jia635/article/details/77512607)


## bindSerive 原理

### *[为什么bindservice方式可以和Activity生命周期联动？？](https://blog.csdn.net/castledrv/article/details/80311502)*

因为bindService时LoadApk将ServiceConnection用ArrayMap保存了起来，当Activity被destroy时会执行**removeContextRegistrations**来清除 该context的相关注册。所以Activity退出时服务也被解绑。

--


MainActivity.bindService->ServerService.onCreate->ServerService.onBind->MainActivity.ServiceConnection.onServiceConnection-->IUmBrellaService.Stub.asInterface(service);

1. MainActivity.bindService 到ActivityManagerService 启动ServerService服务，并调用ServerService的onCreate函数
2. ActivityManagerService 继续调用ServerService的onBind函数，返回一个Binder对象给ActivityManagerService
3. ActivityManagerService拿到Binder对象后，作为参数传递到ServiceConnection对象的onServiceConnected函数
4. 通过onServiceConnected函数返回的IBinder，改造下IUmBrellaService.Stub.asInterface(service),就可以调用IUmBrellaService提供的接口了。