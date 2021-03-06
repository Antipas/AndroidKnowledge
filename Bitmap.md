Bitmap
===

# 占用内存计算

* 同个app，加载同一张图片，在不同手机中占内存一样么？
* 同个app，加载同一张图片的不同格式(png, jpg)，在同一手机中占内存一样么？
* 同个app，加载res和磁盘的图片，在同一手机中占内存一样么？
* 同个app，加载res/drawable和res/drawable-hdpi 的图片，在同一手机中占内存一样么？
* 同个app，加载同一处图片到不同尺寸的imageview，在同一手机中占内存一样么？


#[分析](https://www.jianshu.com/p/3c5ac5fdb62a)

###结论：

* 一张图片占用的内存大小的计算公式：分辨率 * 像素点大小；但分辨率不一定是原图的分辨率，需要结合一些场景来讨论，像素点大小就几种情况：ARGB_8888(4B)、RGB_565(2B) 等等。

* 如果不对图片进行优化处理，如压缩、裁剪之类的操作，那么 Android 系统会根据图片的不同来源决定是否需要对原图的分辨率进行转换后再加载进内存。

* 图片来源是 res 内的不同资源目录时，系统会根据设备当前的 dpi 值以及资源目录所对应的 dpi 值，做一次分辨率转换，规则如下：新分辨率 = 原图横向分辨率 * (设备的 dpi / 目录对应的 dpi ) * 原图纵向分辨率 * (设备的 dpi / 目录对应的 dpi )。

* 其他图片的来源，如磁盘，文件，流等，均按照原图的分辨率来进行计算图片的内存大小。

* jpg、png 只是图片的容器，图片文件本身的大小与它所占用的内存大小没有什么关系，当然它们的压缩算法并不一样，在解码时所耗的内存与效率此时就会有些区别。

基于以上理论，以下场景的出现是合理的：

* 同个 app，在不同 dpi 设备中，同个界面的相同图片所占的内存大小有可能不一样。

* 同个 app，同一张图片，但图片放于不同的 res 内的资源目录里时，所占的内存大小有可能不一样。

以上场景之所说有可能，是因为，一旦使用某个热门的图片开源库，那么，以上理论基本就不适用了。

因为系统支持对图片进行优化处理，允许先将图片压缩，降低分辨率后再加载进内存，以达到降低占用内存大小的目的

而热门的开源图片库，内部基本都会有一些图片的优化处理操作：

当使用 fresco 时，不管图片来源是哪里，即使是 res，图片占用的内存大小仍旧以原图的分辨率计算。

当使用 Glide 时，如果有设置图片显示的控件，那么会自动按照控件的大小，降低图片的分辨率加载。图片来源是 res 的分辨率转换规则对它也无效。


# 优化

###BitmapFactory.Options.inSampleSize

* 一个像素占用多大内存？
* Bitmap如何复用？复用的限制
* Bitmap.compress()。**通过它压缩的图片文件大小会变小，但是解码成bitmap后占得内存是不变的。**
* Bitmap.recycle() 回收了哪部分的内存？
* [文章](https://www.jianshu.com/p/e49ec7d053b3)
* `decodeResource` `decodeStream` `decodeByteArray` 速度分析、使用场景分析
* [探索Bitmap使用姿势](https://lizhaoxuan.github.io/2017/07/11/%E6%8E%A2%E7%B4%A2Bitmap%E4%BD%BF%E7%94%A8%E5%A7%BF%E5%8A%BF)


# 大图展示
使用BitmapRegionDecoder进行局部加载
