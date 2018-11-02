Glide
===

## Glide.With()

* 传入Application
	* 无特殊处理，自动就是和应用程序的生命周期同步 
* 传入Activity或Fragment
	* 向当前的Activity当中添加一个隐藏的Fragment，让加载过程与Activty生命周期同步，及时停止加载



## 获取内存缓存

```
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {

    private final MemoryCache cache;
    private final Map<Key, WeakReference<EngineResource<?>>> activeResources;
    ...

    private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> cached = getEngineResourceFromCache(key);
        if (cached != null) {
            cached.acquire();
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
        }
        return cached;
    }

    private EngineResource<?> getEngineResourceFromCache(Key key) {
        Resource<?> cached = cache.remove(key);
        final EngineResource result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            result = (EngineResource) cached;
        } else {
            result = new EngineResource(cached, true /*isCacheable*/);
        }
        return result;
    }

    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> active = null;
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
                active.acquire();
            } else {
                activeResources.remove(key);
            }
        }
        return active;
    }

    ...
}
```

* loadFromCache()
	* LRUCache
* loadFromActiveResources()
	* activeResources就是一个弱引用的HashMap，用来缓存正在使用中的图片 

loadFromActiveResources()方法就是从activeResources这个HashMap当中取值的。使用activeResources来缓存正在使用中的图片，可以保护这些图片不会被LruCache算法回收掉。

## ActiveCache、MemoryCache、BitmapPool
三者的缓存关系如下：

1. 在加载图片时，先去MemoryCache中查找图片，找到缓存后，将它移入ActiveCache，并将引用数量加1。

2. 当View复用的时候，如果原先的ImageView已经绑定了EngineResource，就需要调用EnginerResource的release方法，该方法会判断该Resource的引用数量是否为0，如果为0，就对资源进行释放。

3. 第2步Resource引用数量为0时，将Resource从ActiveResource中移除，并加入MemoryCache中，以供下次使用。

4. 当MemoryCache达到上限的时候，将里面最近未使用的Bitmap移除到BitmapPool中，用来做Bitmap复用，防止内存抖动。

5. 当ActiveCache或者MemoryCache中都没有指定图片时，就从磁盘缓存或者网络加载。

6. 内存缓存没有时，先从磁盘缓存加载，加载到原始数据后利用BitmapPool中可以复用的Bitmap进行图片复用，加载成功后回调ResourceCallback的onResourceReady进行回调，回调里面将原始资源组装成EngineResource，并将Resource加入ActiveResource。

7. 从网络获取的时候，会将网络数据缓存到本地，其余过程和6一样


## 硬盘缓存

* DiskCacheStrategy.NONE： 表示不缓存任何内容。
* DiskCacheStrategy.SOURCE： 表示只缓存原始图片。
* DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。
* DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。


当我们使用Glide去加载一张图片的时候，Glide默认并不会将原始图片展示出来，而是会对图片进行压缩和转换。总之就是经过种种一系列操作之后得到的图片，就叫转换过后的图片。而Glide默认情况下在硬盘缓存的就是转换过后的图片，我们通过调用diskCacheStrategy()方法则可以改变这一默认行为。**因此不同尺寸的同一图片也会被缓存多次**




## 参考：

 * [文章1](https://www.jianshu.com/p/c555a4fe5fd1)
 * [文章2](https://blog.csdn.net/guolin_blog/article/details/54895665)
