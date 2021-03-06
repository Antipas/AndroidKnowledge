1. HashMap的基础数据结构：数组链表
    * HashMap的长度固定为2的n次方，是为了在计算桶位索引值h & (len-1）低位是1。[原因](https://www.cnblogs.com/chengxiao/p/6059914.html#t3)
    * jdk1.8.* 的时候，HashMap的实现进行过升级，之前都是用链表的形式解决哈希冲突，新版会判断当链表长度超过一个阈值之后把链表内容进行树化（红黑树）
    * 扩容：申请一个长度为两倍的新的数组，然后重新计算桶位索引值，再复制过去。相对于ArrayMap的数组拷贝这里更小号内存[待深入]
2. HashTable和ConcurrentHashMap
    * 与HashMap的区别：HashMap线程不安全，而这两个线程安全
    * 线程锁：
        * HashTable通过synchronized关键词修饰方法，将整块的方法上锁；
        * ConcurrentHashMap的Node对Value和next添加了volatile形容词进行线程锁
        * 
            ``` java
            static class Node<K,V> implements Map.Entry<K,V> {
                final int hash;
                final K key;
                volatile V val;
                volatile Node<K,V> next;
                ...
            }

            ```
        * ConcurrentHashMap的 ``` Segment<K,V> extends ReentrantLock ```，[待深入]。
        * [补充+问题]ConcurrentHashMap的数据结构分为三层，但是源码中没有明确表达Segment和Node的关系，只在一个方法中出现过
            ```
            graph TB
                A[ConcurrenthashMap] --> B[Segment]
                B --> C[HashEntry]
            ```
            ``` java
            private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException {
                // For serialization compatibility
                // Emulate segment calculation from previous version of this class
                int sshift = 0;
                int ssize = 1;
                while (ssize < DEFAULT_CONCURRENCY_LEVEL) {
                    ++sshift;
                    ssize <<= 1;
                }
                int segmentShift = 32 - sshift;
                int segmentMask = ssize - 1;
                @SuppressWarnings("unchecked")
                Segment<K,V>[] segments = (Segment<K,V>[])
                    new Segment<?,?>[DEFAULT_CONCURRENCY_LEVEL];
                for (int i = 0; i < segments.length; ++i)
                    segments[i] = new Segment<K,V>(LOAD_FACTOR);
                java.io.ObjectOutputStream.PutField streamFields = s.putFields();
                streamFields.put("segments", segments);
                streamFields.put("segmentShift", segmentShift);
                streamFields.put("segmentMask", segmentMask);
                s.writeFields();

                Node<K,V>[] t;
                if ((t = table) != null) {
                    Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
                    for (Node<K,V> p; (p = it.advance()) != null; ) {
                        s.writeObject(p.key);
                        s.writeObject(p.val);
                    }
                }
                s.writeObject(null);
                s.writeObject(null);
            }
            ```
    * ConcurrentHashMap在JDK1.8后的改动：使用CAS操作替代分段锁。如果该桶位里有元素（有冲突元素）则使用synchronized在链表头加锁然后继续操作。
    * 分段锁的劣势：如果需要加锁整个容器的话，需要获得所有的分段锁，比如扩容时，需要重新计算散列值分散到更大的桶。
3. LinkedHashMap & TreeMap
    * 与HashMap的区别：都是有序的。
    * LinkedHashMap: 继承了HashMap，数据结构一样，不过链表是双向链表，Entry<K,V>中增加了Entry<K,V> before & Entry<K,V> after。同时，有个accessOrder属性控制排列顺序，true表示最近最少使用次序，false表示插入顺序。
        * accessOrder属性为true的最少使用次序的排序方法通过LRU算法实现，图片库glide保存图片就用的这种方式进行实现。
    * TreeMap: 通过一个红黑树实现对数据的存放和查找。
4. ArrayMap
    * Android自己优化的一个数据结构，通过两个数组存放数据，一个数组存放HashCode，一个数组以 {K1, V1, K2, V2, ..., Kn, Vn} 的方式存放数据。
    * 牺牲时间换空间。
    * ArrayMap是有序的，每次查找的时候通过二分法进行查找，效率高，但是插入速率由于要进行排序所以速率低。
    * 处理哈希冲突：以该key为中心点，分别上下展开，逐个去对比查找。
    * ArrayMap对于hashes.length为4和8的两种情况会进行缓存。在分配的时候，如果需要分配这两种大小的数组，就可以直接从缓存中取得，否则，就直接new两个数组。
    * 有自动收缩数组空间的功能。
    * 当数据量少的时候选用ArrayMap更合适因为内存消耗少，数据量大时用HashMap合适。
    * [源码分析](https://www.jianshu.com/p/1fb660978b14)

    
[补充](https://www.cnblogs.com/beatIteWeNerverGiveUp/p/5709841.html)

[这个挺全面](https://www.jianshu.com/p/939b8a672070)
