---
layout: post
title:  "ConcurrentHashMap源码解析.md"
date:   2021-03-21
author: Dickie Yang
tags:
    - 源码
    - Java
---

## 构造函数

```
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   // 1.5倍初始容量 + 1，再往上去最近的2的整数次方。
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        // sizeCtl用于控制初始化或并发扩容时的线程数，初始值设置成cap容量大小。
        this.sizeCtl = cap;
    }
```



## 初始化容器

```
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        // table为空或长度为0时进行初始化。
        while ((tab = table) == null || tab.length == 0) {
        		// sizeCtl值在下一个分支设置为-1，表示有线程获取到了初始化的权限，其他线程执行到这里会自旋并且尝试						 // 放弃CPU的使用权。
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // CAS把sizeCtl设置为-1，控制并发初始化。
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                     	  // 初始化的大小为sizeCtl的值或默认容量，也就是最初赋值的cap的容量。
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // sizeCtl值更新成 0.75n，n为数组大小。
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



## put()方法

```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 处理一下key的hashCode分布不均匀的情况
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
            		// 在构造Map的时候并不会初始化，在put时并发初始化表。
                tab = initTable();
            // 如果hash对应的数组槽元素不存在，则CAS设置一个Node，结束循环。
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果hash对应的槽点元素为forwarding node，帮助其转移并加入节点。
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                // 以上都不满足，在槽点首节点进行加锁。
                synchronized (f) {
                		// 再次判断首节点是否与之前取到的一致。
                    if (tabAt(tab, i) == f) {
                    		// 首节点的hash值大于等于0，表示当前节点为普通链表节点。
                        if (fh >= 0) {
                            binCount = 1;
                            // 遍历链表，累加链表节点数量。
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果当前遍历到的节点hash值与传入的节点hash值相同，并且
                                // （如果key和传入key相同或key相等），结束循环。
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 遍历到最后一个节点，追加新节点。
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果首节点时TreeBin类型，则当前槽为红黑树。加入新节点到树。
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
                if (binCount != 0) {
                		// 当前槽节点数到达阈值8，则尝试转成红黑树。
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 增加元素个数
        addCount(1L, binCount);
        return null;
    }
```

## addCount()方法

```
    // x：这次需要在表中增加的元素个数，check：表示是否需要进行扩容检查，大于0都需要。
    // 传入1和链表长度，来增加Map中的元素个数。可能会触发扩容。
    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        // 通过CounterCells来统计元素个数，也就是分片统计。
        // 当CounterCells为空，则CAS原子增加baseCount的值。
        // 当CounterCells不为空或CAS失败，通过allAddCount()进行记录。
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            // 默认没有冲突。
            boolean uncontended = true;
            // 当CounterCells为空或随机选取的CounterCell为空或累加到CounterCell失败，
            // 直接fullAddCount()增加。
            // 当CounterCells不为空，随机选取一个CounterCell不为空，则CAS累加到当前CounterCell。
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            // check小于等于1，直接结束，不考虑扩容。
            if (check <= 1)
                return;
            // 统计Map中元素的数量。
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // 如果Map中元素个数达到sizeCtl的阈值，并且当前数组容量小于最大容量，开始扩容。
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 记录一个唯一固定的扩容戳。n的高位0的个数 ｜ （1 << 15）。
                int rs = resizeStamp(n);
                // sizeCtl小于0，表示有线程在扩容了。
                if (sc < 0) {
                		// 判断当前线程能否帮组扩容。
                		// sc >>> RESIZE_STAMP_SHIFT) != rs表示sizeCtl的高16位和rs是否相等，
                		// 此时的sizeCtl值由首个扩容线程设置为rs << 16 + 2，故此可以比较高16位和rs值。
                		// 若相等，还需要判断以下4点，符合其一则结束：
                		// sc == rs + 1表示扩容结束。
                		// sc == rs + MAX_RESIZERS表示达到可以帮组扩容的线程的最大值。
                		// nextTable == null表示扩容结束。
                		// transferIndex <= 0表示扩容任务被领完了。
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // sizeCtl加1，并帮组其扩容。
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                // 当前没有在扩容，将sizeCtl设置成一个负数，+2表示有一个线程在扩容了。
                // 高 16 位代表扩容的标记、低 16 位代表并行扩容的线程数。
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```



## treeifyBin()方法

```
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
        		// 如果当前数组容量小于64，则直接扩容数组，不转成红黑树。
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            // 数组的index槽首元素不为空，并且hash值不为负数时（为链表节点），对首元素加锁，转成红黑树。
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                		// double check。
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        // 遍历链表，构建红黑树。
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        // 把树的首节点设置到数组index槽的位置，已经加锁，可以直接设置。
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```



## tryPresize(n)方法

```
    private final void tryPresize(int size) {
				// size的1.5倍 + 1，再向上取最近的2的整数次方。
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        // sizeCtl >= 0 表示当前没有线程在扩容。
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            // 当前Map还未初始化，putAll()方法会进入这个分支。
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            // 计算的容量未超过阈值或到达最大容量，直接结束。
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
            		// 扩容过程见addCount方法分析的最后一个分支。
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
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
            }
        }
    }
```



## transfer()方法

> ConcurrentHashMap扩容的时机？
>
> - 每次涉及添加元素的时候，都会检查元素数量是否达到阈值（check标志>0）
> - 当某个槽内的元素个数达到8个以上，数组大小小于64时，会先进行扩容以避免链表转为红黑树
> - 在put元素的时候，发现当前槽点首节点时forwaringNode，会帮组其进行扩容

```
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
				// 计算步长stride。
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 初始化新的HashMap。
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                // 扩容为2倍。
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
            		// 重新设置sizeCtl，满足sc >>> RESIZE_STAMP_SHIFT) != rs，结束扩容。
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            // volatile类型，为扩容时nextTable要拆分的下标，设置为旧数组的长度。
            transferIndex = n;
        }
        int nextn = nextTab.length;
        // 创建一个ForwardingNode，表示一个正在被迁移的节点。
        // 之后所有迁移完毕的槽，都会设置指向这个节点。
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;// 等于true，则处理下一个下标。
        boolean finishing = false; // to ensure sweep before committing nextTab
        // 循环处理每个槽中的链表元素，i为槽的位置，bound为槽的边界。
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // advance表示i=transferIndex - 1到bound位置的过程中，是否继续。
            // 三个分支都是false，都不满足时才会一直执行，目的为了在对transferIndex执行CAS不成功时，
            // 需要自旋，以期拿到一个stride的迁移任务。
            while (advance) {
                int nextIndex, nextBound;
                // 线程成功拿到过一次任务就能满足，退出内循环。
                if (--i >= bound || finishing)
                    advance = false;
                // 任务已经被拿完了。
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // 对transferIndex进行CAS，每次递减一个stride。CAS成功，线程拿到一个stride的迁移任务。
                // CAS不成功，线程会自旋尝试。
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            // i已经越界，整个HashMap已经遍历完成。
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 若finishing=true表示整个HashMap扩容完成。
                if (finishing) {
                    nextTable = null;
                    table = nextTab;// 把新数组赋值给原来的数组。
                    // 重置sizeCtl为扩容后数组大小n的0.75倍。
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // CAS操作sizeCtl递减，表示自己完成了扩容任务。
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                		// 表示还有线程未完成迁移，直接结束，否则设置标志为true。
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            // 如果tab[i]是空的，表示迁移完毕（不用迁移），放置一个ForwardingNode
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {// 对tab[i]进行迁移操作，它可能是链表或红黑树。
                synchronized (f) {
                		// Double check
                    if (tabAt(tab, i) == f) {
                    		// ln表示低位，hn表示高位。
                        Node<K,V> ln, hn;
                        // 是一个链表
                        if (fh >= 0) {
                        		// 这里计算的时log(n)位&hashCode的值，要么为0、要么为n，
                        		// 比如n为16，就是hashCode&(00010000)
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            // 遍历链表。
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                            		// 对比后一位与前一位的hash值是否相等，
                            		// 当它们相等时lastRun不再更新，意味着后面的节点hash值都是一样的。
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            // 如果runBit为0，放在低位。
                            if (runBit == 0) {
                                ln = lastRun;
                                // lastRun后面的所有节点，直接链接到新的链表头部，前面的节点需要依次拷贝
                                hn = null;
                            }
                            // 否则，放在高位。
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            // 构造高位以及低位的链表。到lastRun为止，避免做多余的拷贝。
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0) // 把前面的ln作为后面的ln的next节点传入，构成链表
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // 分别把高、低位低链表设置到新的数组。
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            // 把原数组tab[i]设置为ForwardingNode。
                            setTabAt(tab, i, fwd);
                            advance = true;// 结束，处理下一个槽。
                        }
                        else if (f instanceof TreeBin) {// 红黑树
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

## helpTransfer()方法

```
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // 表示该槽正在扩容中
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            // 扩容还未完成。
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 扩容结束
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                // 低16位扩容线程数+1，帮助扩容。
                // 这里的帮助扩容是指是当前线程去参与分配stride进行扩容。
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

## get(key) 方法

> 为什么get方法完全没有加锁可以保证线程安全？
>
> 因为数组里面每个node的val和next指针都是volatile修饰的，保证了修改操作的可见性。

```
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        // 根据hash码找到元素在数组中的位置的下标
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 若槽内首元素hash码相等
            if ((eh = e.hash) == h) {
            		// key与首元素key相同或内容相等，返回对应的值
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 若首元素位不是正常的链表，会通过多态调用到实际的node实现的方法
            // forwarding node、
            // root of tree、
            // transient reservations 
            else if (eh < 0)
            		// e.find()是个虚方法
                return (p = e.find(h, key)) != null ? p.val : null;
            // 普通链表，直接遍历即可。
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

