LruCache
===

# LRU算法原理

 * get操作：遍历队列中找到该元素，然后从原来位置删除，再插入到链表头部
 * put操作：如果元素已经在链表中，等同于get。如果没有：
 	*  缓存未满，插入到链表头
 	*  缓存已满，删除表尾节点，再插入到链表头部。
 * 问题： 假设缓存长度为3，已经有了1、2、3数据，之后调get(2)，插入4，再get(3)，此时队列怎么排列？

 
# 内部组成： [LinkedHashMap](https://blog.csdn.net/wangxilong1991/article/details/70172302)

LinkedHashMap 本质是HashMap，但是内部维护了双向链表，默认按插入顺序排列，也可修改accessOrder = true，按访问顺序排列。

```
public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }

```

当然，也需要重写removeEldestEntry，来决定当缓存满后就移除最不常用的数。
但是里面对于重新排放的顺序有很大不同：

```
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMapEntry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
 	
在get()里，如果按顺序访问会调afterNodeAccess，注意**move node to last**，**它是把最新元素查到末尾的，然后淘汰头部元素！**
 


