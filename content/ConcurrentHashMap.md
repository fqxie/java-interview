### ConCurrentHashMap 底层分析

#### 作用
虽然JDK8以后HashMap链表死循环(JDK7时会发生的严重并发问题)已经解决,但HashMap本质上仍旧是非线程安全的,其并没有对类方法进行同步,因此多线程下仍然会发生诸如一个线程读另一个线程扩容的数据操作问题.  
因此并发环境应当考虑线程安全的哈希表,而ConcurrentHashMap自然是首选.另外要知道HashTable已经是废弃的类了,其线程安全是基于所有的方法都加上synchronized,性能极差.而ConcurrentHashMap可以视为一个二级哈希表(JDK7),线程安全基于分段锁,进行读取操作并不会阻塞.  
![二级哈希表](https://raw.githubusercontent.com/MelloChan/java-interview/master/image/ConCurrentHashMap.png)    
如上是JDK7的ConcurrentHashMap的底层结构,其默认大小64,segment数组默认16,因此二级哈希数组的大小为64/16=4.但JDK8重新设计了ConcurrentHashMap,抛弃了segment,底层仍然是数组+链表+红黑树,内存占用和HashMap差不多,对写操作利用 CAS + synchronized(同步关键字的优化)解决并发问题,对读操作不加锁.
速度优于JDK7的ConcurrentHashMap.  

#### Node  

对val和next字段增添了volatile关键字,解决并发可见性问题.
```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
```

#### hash  

并发哈希表不允许键入null键值对,哈希操作放在了putVal方法  
```
int hash = spread(key.hashCode());
static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

#### initTable   

同HashMap,数组的初始化放在第一次put操作时.  
```
if (tab == null || (n = tab.length) == 0)
                tab = initTable();
                
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
             // sizeCtl 默认0 如果实例化时调用带参构造器则sizeCtl为2的幂次方 而大于等于零时线程会调用Unsafe的CAS方法修改sizeCtk为-1
             // 因此小于0证明其他线程在初始化中 当前线程让出CPU
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 替换sizeCtl为-1 仅有一个线程能操作成功  参数分别为:当前对象 内存值 期待值 更新值, CAS就是根据 内存值是否等于期待值来替换更新    
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    // 替换成功的线程开始尝试初始化 
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

#### put  

初始化完毕的哈希表开始计算索引然后插入数组中,不同的是JDK8采用 CAS+synchronized机制,而不是segment(分段锁,二级哈希)
```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 获取哈希值 与HashMap略为不同   (h ^ (h >>> 16)) & HASH_BITS;    HASH_BITS = 0x7fffffff;
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable(); // 初始化
            // JMM的缘故 直接从线程工作内存拿可能无法拿到最新值 volatile无法保证table里的引用具有可见性 因此直接从内存中拿取table索引位置的节点     
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  
                // 没有产生冲突 使用CAS插入节点
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
             // 命中的位置已经有节点了 直接采用同步关键字锁住节点     
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            // 循环遍历节点 两种可能 ①替换节点值 ②构建新节点插入尾插链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 若是树节点 则插入红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 若链表大于等于8 将链表改为红黑树
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    // 如果是替换旧值 则返回旧值    
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 添加节点数量 数量超过阈值时进行扩容 
        addCount(1L, binCount);
        return null;
    }
```

#### get  

get方法相对put方法较为简单,并发哈希表不会对读操作进行加锁,因此速度和哈希表差不多.
```
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        // 这里判断数组是否为null以及是否有元素 tabAt方法会直接从内存中取索引位置相应的节点 
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 取到的节点哈希值要与传入key的哈希值相等
            if ((eh = e.hash) == h) {
                // 节点命中 直接返回值
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 这里是红黑树的情况
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
             // 这里需要遍历链表   
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

#### 扩容与红黑树  

扩容时通过addCount方法的.  
```
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

