1. HashMap的基础数据结构：数组链表
    * HashMap的长度固定为2的n次方，[原因]
    * jdk1.8.* 的时候，HashMap的实现进行过升级，之前都是用链表的形式进行存放Entry，现在通过红黑树进行存放，提升了查找的效率的存放的效率。
    * 扩容：申请一个长度为两倍的新的数组，然后复制过去[待深入]
2. HashTable和ConcurrentHashMap
    * 与HashMap的区别：HashMap线程不安全，而这两个线程安全
    * 线程锁：
        * HashTable通过synchronized关键词修饰方法，将整块的方法上锁；
        * ConcurrentHashMap的Node对Value和next添加了volatile形容词进行线程锁
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
3. LinkedHashMap & TreeMap
    * 与HashMap的区别：都是有序的。
    * LinkedHashMap: 继承了HashMap，数据结构一样，不过链表是双向链表，Entry<K,V>中增加了Entry<K,V> before & Entry<K,V> after。同时，有个accessOrder属性控制排列顺序，true表示最近最少使用次序，false表示插入顺序。
        * accessOrder属性为true的最少使用次序的排序方法通过LRU算法实现，图片库glide保存图片就用的这种方式进行实现。
    * TreeMap: 通过一个红黑树实现对数据的存放和查找。
4. ArrayMap
    * Android自己优化的一个数据结构，通过两个数组存放数据，一个数组存放HashCode，一个数组以 {K1, V1, K2, V2, ..., Kn, Vn} 的方式存放数据。
    * ArrayMap是有序的，每次查找的时候通过二分法进行查找，效率高，但是插入速率由于要进行排序所以速率低。
