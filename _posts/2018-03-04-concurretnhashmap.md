---
layout:     post
title:      Java并发包源码阅读（二）
subtitle:   ConcurrentHashMap
date:       2018-03-04
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Java并发
---

HashTable虽然是线程安全的，但是用synchronized保证线程安全效率非常低（线程1在使用put方法时，线程2进入阻塞，不能put也不能get）。JDK1.8开始ConcurrentHashMap的实现相较于之前几乎完全变了，取消了Segment分段锁的数据结构，取而代之的是数组+链表（红黑树）的结构。而对于锁的粒度，调整为对每个数组元素加锁（Node）。

# 回顾JDK1.8之前的ConcurrentHashMap

JDK1.8之前的ConcurrentHashMap采用分段锁的机制，实现并发的更新操作，底层由Segment数组和HashEntry数组组成。Segment继承ReentrantLock用来充当锁的角色，每个Segment对象守护每个散列映射表的若干个桶（相当于一个小的HashMap）。HashEntry用来封装映射表的键/值对，每个桶是由若干个HashEntry对象链接起来的链表。

# JDK1.8的ConcurrentHashMap实现

## 基本变量和内部类

```java
    //hash表初始化或扩容时的一个控制位标识量。负数代表正在进行初始化或扩容操作
    //-1代表正在初始化
    //-N 表示有N-1个线程正在进行扩容操作
    //正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小
    //值始终是当前ConcurrentHashMap容量的0.75倍
    private transient volatile int sizeCtl;

    //hash值是-1，表示这是一个forwardNode节点
    static final int MOVED     = -1; 
    //hash值是-2，表示这时一个TreeBin节点
    static final int TREEBIN   = -2; 

    //相当于hash方法，注意一般的对象得到的hash值都是正的，负值有特殊含义
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }

	//一个特殊的结点，hash值为-1，用于连接两个table
    static final class ForwardingNode<K,V> extends Node<K,V> {
        //这个表用来过渡
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            //构造hash值为-1的结点
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
      }

```

## get方法

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&

            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

get方法比较简单，大体上和HashMap的类似，注意get方法没有采用加锁操作,因为读/写都没有中间状态，且结点是volatile型，可以保证正确。

## put方法

源码如下所示：

```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //不允许key为空且不允许值为空
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //如果哈希表该位置为空，使用CAS方式添加一个结点
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   
            }
            //如果该位置不为空但是该位置的结点是forward结点，说明正在转移操作，先得到转移之后的表
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //针对首个结点进行加锁
                synchronized (f) {
                    //以线程安全方式判定一下加锁的结点是否真的是该位置第一个结点
                    if (tabAt(tab, i) == f) {
                        //如果是普通结点
                        if (fh >= 0) {
                            binCount = 1;
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
                        //如果是树结点
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
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //调用该方法来判断是否需要迁移表
        addCount(1L, binCount);
        return null;
    }
```
可以看到，put方法流程如下：
1. 键和值都不是空，否则抛出异常。
2. 自旋：
3. 如果表为空或者表长为0，需要先初始化。
3. 如果需要加入的位置如果为空，构造一个新结点加入，这个靠CAS实现线程安全。
4. 如果加入的位置不为空，但是是forward结点，说明在进行转移操作，先得到转移之后的表。
5. 否则就先对第一个结点加锁，接着判定加锁的结点是否真的是第一个结点（加锁前可能有其他结点插入了），如果是，接下来和HashMap的就没有太大区别了。

## transfer方法

```java
    //用来过渡的表
    private transient volatile Node<K,V>[] nextTable;  

    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }

    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {  
       int n = tab.length, stride;
       //stride表示分片的任务，最低16，只有1个核不用分片，否则n >>> 3 / NCPU  
       if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)  
           stride = MIN_TRANSFER_STRIDE;
       //如果nextTab为空，让其等于一个容量为原来两倍的数组
       if (nextTab == null) {           
           try {  
               @SuppressWarnings("unchecked")  
               Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
               nextTab = nt;  
           } catch (Throwable ex) {      
               sizeCtl = Integer.MAX_VALUE;  
               return;  
           }
           //赋给过渡数组，不需要加锁
           nextTable = nextTab;
           //transferIndex表示从transferIndex开始到后面所有的桶的迁移任务已经被分配出去了  
           transferIndex = n;  
       }  
       int nextn = nextTab.length;  
       ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);  
       boolean advance = true;
       boolean finishing = false;  
       for (int i = 0, bound = 0;;) {  
           Node<K,V> f; int fh;   
           while (advance) {  
               int nextIndex, nextBound;  
               if (--i >= bound || finishing)  
                   advance = false;
               //全部迁移任务已经完成  
               else if ((nextIndex = transferIndex) <= 0) {  
                   i = -1;  
                   advance = false;  
               }
               //获取本线程的分片范围，同时CAS更新transferindex值
               else if (U.compareAndSwapInt  
                        (this, TRANSFERINDEX, nextIndex,  
                         nextBound = (nextIndex > stride ?  
                                      nextIndex - stride : 0))) {  
                   bound = nextBound;  
                   i = nextIndex - 1;  
                   advance = false;  
               }  
           }  
           if (i < 0 || i >= n || i + n >= nextn) {  
               int sc;  
               if (finishing) {  
                   //如果所有的节点都已经完成复制工作 ，就把nextTable赋值给table，清空临时对象nextTable  
                   nextTable = null;  
                   table = nextTab;
                   //扩容阈值设置为原来容量的1.5倍  依然相当于现在容量的0.75倍  
                   sizeCtl = (n << 1) - (n >>> 1);
                   return;  
               }  
               //sizeCtl的初始值为负值(rs << RESIZE_STAMP_SHIFT) + 2
               //每有一个任务参与迁移操作，sizeCtl增加1 ，完成任务后退出要用CAS方式完成自减操作
               if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
               	   //判断任务是否已经全部完成，相等说明还有任务在迁移  
                   if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)  
                       return;  
                   finishing = advance = true;  
                   i = n;  
               }  
           }  
           //如果遍历到的节点为空 则CAS放入ForwardingNode指针，说明处理过了
           //如果put方法遇到这种情况，表示正在迁移  
           else if ((f = tabAt(tab, i)) == null)  
               advance = casTabAt(tab, i, null, fwd);  
           //如果遍历到ForwardingNode节点 ，说明这个点已经被处理过了，直接跳过
           else if ((fh = f.hash) == MOVED)  
               advance = true;  
           else {  
               //否则对第一个结点上锁
               synchronized (f) {  
                   if (tabAt(tab, i) == f) {  
                       Node<K,V> ln, hn;  
                       //普通node结点
                       if (fh >= 0) {  
                           int runBit = fh & n;   
                           Node<K,V> lastRun = f;  
                           for (Node<K,V> p = f.next; p != null; p = p.next) {  
                               int b = p.hash & n;  
                               if (b != runBit) {  
                                   runBit = b;  
                                   lastRun = p;  
                               }  
                           }  
                           if (runBit == 0) {  
                               ln = lastRun;  
                               hn = null;  
                           }  
                           else {  
                               hn = lastRun;  
                               ln = null;  
                           }  
                           for (Node<K,V> p = f; p != lastRun; p = p.next) {  
                               int ph = p.hash; K pk = p.key; V pv = p.val;  
                               if ((ph & n) == 0)  
                                   ln = new Node<K,V>(ph, pk, pv, ln);  
                               else  
                                   hn = new Node<K,V>(ph, pk, pv, hn);  
                           }  
                           setTabAt(nextTab, i, ln);  
                           setTabAt(nextTab, i + n, hn);  
                           //在table的i位置上插入forwardNode节点 ，表示已经处理过该节点  
                           setTabAt(tab, i, fwd);  
                           advance = true;  
                       }  
                       //红黑树结点
                       else if (f instanceof TreeBin) {  
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
                           //如果扩容后已经不再需要tree的结构，反向转换为链表结构  
                           ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :  
                               (hc != 0) ? new TreeBin<K,V>(lo) : t;  
                           hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :  
                               (lc != 0) ? new TreeBin<K,V>(hi) : t;  
                           setTabAt(nextTab, i, ln);  
                           setTabAt(nextTab, i + n, hn);  
                            //在table的i位置上插入forwardNode节点，表示已经处理过该节点  
                           setTabAt(tab, i, fwd);  
                           advance = true;  
                       }  
                   }  
               }  
           }  
       }  
   }  
```

可以看到扩容的流程是：
1. 如果过渡数组为空，新建一个为原数组大小的2倍的数组赋给它，这个过程不需要加锁，因为即使后面有线程也初始化nextTable，覆盖了也没什么问题。
+ 如果这个位置为空，就在原table中的i位置放入forward节点，表明已经处理过了。
+ 否则对第一个结点先加锁，如果这个位置是普通Node节点，就构造一个和原来顺序相同的链表，把他们分别放在nextTable的i和i+n的位置上，并把哈希表i位置上放一个forward结点。
+ 如果这个位置是红黑树节点，也构造一个链表，并且判断是否需要把树型结构转成链型，把处理的结果分别放在nextTable的i和i+n的位置上，并把哈希表i位置上放一个forward结点。
+ 遍历过所有的节点以后就完成了复制工作，这时让nextTable作为新的table，并且更新sizeCtl为新容量的0.75倍 ，完成扩容。
+ 如果遍历到的节点是forward节点，说明处理过了，就向后继续遍历。多线程遍历节点，处理了一个节点，就把对应点的值set为forward，另一个线程看到forward，就向后遍历。这样交叉就完成了复制工作。

# 参考文章

[jdk1.8的HashMap和ConcurrentHashMap](https://yq.aliyun.com/articles/68282)

[ConcurrentHashMap源码分析（JDK8版本）](http://blog.csdn.net/u010723709/article/details/48007881)
