Activity栈
===
启动一个app，系统就会为之创建一个task，来放置根Activity

# Affinity

拥有相同affinity的多个Activity理论同属于一个task，task自身的affinity决定于__根Activity__的affinity值。

 * 启动一个Activity过程中如果Intent使用了FLAG_ ACTIVITY_NEW _TASK标记，就会根据affinity查找或创建一个新的具有对应affinity的task。
 * 应用内的所有Activity都具有相同的affinity，都是从Application继承而来，而Application默认的affinity是<manifest>中的包名
 
 ```
<application   
		android:name="App"
		android:taskAffinity="com.hello.world" >
 ```


* FLAG_ ACTIVITY_NEW _TASK
* 
当Intent对象包含这个标记时，系统会寻找或创建一个新的task来放置目标Activity，寻找时依据目标Activity的taskAffinity属性进行匹配，如果找到一个task的taskAffinity与之相同，就将目标Activity压入此task中，如果查找无果，则创建一个新的task，并将该task的taskAffinity设置为目标Activity的taskActivity，将目标Activity放置于此task。注意，如果同一个应用中Activity的taskAffinity都使用默认值或都设置相同值时，应用内的Activity之间的跳转使用这个标记是没有意义的，因为当前应用task就是目标Activity最好的宿主


* FLAG_ ACTIVITY_ CLEAR_ TOP
* FLAG_ ACTIVITY_ CLEAR_ WHEN_ TASK_ RESET 和 FLAG_ ACTIVITY_ RESET_ TASK_ IF_ NEEDED 的配合使用：当app从后台恢复到前台时重置有该标记的Acitivity。