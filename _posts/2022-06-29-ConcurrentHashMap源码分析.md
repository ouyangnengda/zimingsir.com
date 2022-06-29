---
layout: default
title: "ConcurrentHashMap源码分析"
data: 2022-06-29 00:34:00 -0000
category: java
---

本文专门解析 JDK 1.8 ConcurrentHashMap，文中你可以找到大多数关于 ConcurrentHashMap 问题的答案，正因为本文仅专注于 ConcurrentHashMap，因此就没有与 HashMap 等纵向比较的内容。
> 为了便于说明下文的 chm 与 ConcurrentHashMap 统一。

[TOC]

我们来看一下 chm 持有数据的结构
```java
// chm 通过一个 Node<K, V> 数组 table 来持有全部数据，每一个 Node 表示一个元素。而每个 Node 的 next 属性则指向链表中的下一个元素，当链表长度大于等于 8 时链表可能会被转化为红黑树以降低搜索复杂度。
transient volatile Node<K,V>[] table;

// nextTable 是一个临时的 Node<K, V> 数组，当数据需要迁移的时候，chm 会把数据都迁移到 nextTable 上，待数据迁移完成再 table = nextTable。
private transient volatile Node<K,V>[] nextTable;
```
![ConcurrentHashMap——数组结构图 _1_.png](https://i.loli.net/2020/04/14/tZyOGpDwiuY9gVf.png)

先要说明一下不同 hash 值代表的意思，以及 sizeCtl 这个复杂变量的各种含义。
第一件事，不同hash值所表示的含义

| 数据类型 | 变量名 | hash值 | 含义 |
| ---- | --- | -- | --|
| int | MOVED | -1 | 该位置的桶已被转移到新数组 |
| int | TREEBIN | -2 | 该节点为红黑树 root 节点 |
| int | RESERVED | -3 | `不清楚` |
| int | HASH_BITS | 0x7fffffff | 使得到的 Hash 值为正数 |

第二件事，是关于sizeCtl各种取值的含义：

| 数据类型 | 数值 | 含义 |
| ---- |-- | --|
| int | 正整数 | 初始化之前表示初始化容量                                   |
| int | 正整数 | 初始化之后表示扩容阈值，值为 0.75*`table.length` |
| int | 0 | 表示初始化容量为 0 |
| int | -1 | 表示数组正在进行初始化 |
| int | -（1+N） | 表示有 N 个线程正在迁移数组，-2 表示有一个线程正在迁移数组 |

### ConcurrentHashMap(int initialCapacity)
将传进来的数值乘个 1.5，加个 1，再往上取个最近的二次方的数。

容量为 10 运算之后得到 16 向上取一个最近的 2 的次方得到的就是16
容量为 11 运算之后得到 17 向上取一个最近的 2 的次方得到的就是32

```java
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        /**
         * int MAXIMUM_CAPACITY = 1 << 30;
         * 如果容量大于等于 chm 最大容量的一半，那就直接把最大的容量赋值给 cap；
         * 否则，将 （1.5*容量+1） 向上取一个最近的 2 的次方。
         */
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                MAXIMUM_CAPACITY :
                tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
}
````

### V get(Object key)
我们首先来看一下 `get(Object key)` 方法。

&emsp;&emsp;我建议这时候自己先想一想：如果叫你实现这个 get 方法你的步骤是怎样的？

```java
public V get(Object key) {
        Node<K,V>[] tab;
        Node<K,V> e, p;
        int n, eh;
        K ek;
        // 使得 hash 数的分布更加均匀，同时保证返回正数,详情看下面 spread 函数的解析
        int h = spread(key.hashCode());
        // 只有数组存在，长度大于0，对应元素存在才执行下面的代码，否则直接跳到最后面返回一个 null
        if ((tab = table) != null && (n = tab.length) > 0 && (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            /**
             * 表明该节点正在扩容或该节点为红黑树节点
             * 因为扩容时节点的hash会被谁值为值为MOVED = -1
             * 红黑树节点的hash值全为TREEBIN = -2
             * 红黑树和ForwardNode 分别对 find 方法有重载
             */
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 搜索链表
            while ((e = e.next) != null) {
                if (e.hash == h &&
                        ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

#### int spread(int h)
这个方法做了两件事：
1. 让 h 的高 16 位与低 16 位进行异或，使得 hash 值的分布更加均匀
2. 将高低位异或后的结果与 HASH_BITS 进行与运算，使得到的值为正数。

二进制中最高位为 1 表示负数，最高位为 0 表正数 。又因为 0x7fffffff 是一个最高位为 0 其余位为 1 的数，因此与 0x7fffffff 相与得到的总会是一个正数。

```java
int HASH_BITS = 0x7fffffff;
// 0x7fffffff二进制表示：0111 1111 1111 1111   1111 1111 1111 1111;
int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
}
```

### V putVal(K key, V value, boolean onlyIfAbsent)

onlyIfAbsent 是什么意思？

> onlyIfAbsent 的意思是：如果要 put 的值已经存在，那我不会覆盖它。  onlyIfAbsent 为 false 表示可以覆盖。
> 举例：当 chm 存在一个键值对（K = 1，V = 1）这时我 put 一个键值对（K = 1，V = 2），因为put方法的 onlyIfAbsent 为  false，所以 chm 中的键值对成了（K = 1，V = 2），返回旧值 V = 1。

ConcurrentHashMap 和 HashTable 为什么不允许 K 和 V 的值为 null ？
> 因为这两个数据结构都用于并发操作，当你取得一个 null 的时候，我们不能确定这个 null 是你取到的 V 值，还是因为没找到而返回 null。
> 如果再用 conatinsKey 去确定的话，在这两个方法执行的间隔中，数值会可能会变化，因为这是一个支持高并发的数据结构，所以干脆禁止 K V 为 null。

```java
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        
        if (key == null || value == null)
            throw new NullPointerException();
        int hash = spread(key.hashCode());
        // 存储一个桶拥有的节点数，当节点大于等于8时，需进行树化
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f;
            int n, i, fh;
            // 初始化table
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 对应位置为 null，则尝试 cas 插入
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 开头写明白了，hash == MOVED 说明这个桶已经被迁移走了
            else if ((fh = f.hash) == MOVED)
                // 帮助完成数据迁移的工作
                tab = helpTransfer(tab, f);
            // 下面就是正常状态下的查找操作，暂时不展开，先把上面的理解好
            else {...}
        }
        
        //标记新元素的加入
        addCount(1L, binCount);
        return null;
    }
```
下面我用一个流程图说明了putVal方法的大致逻辑，应该算是蛮清楚的。
![ConcurrentHashMap—putVal流程图 _1_.png](https://i.loli.net/2020/12/16/XkjyLaRdvDgcnps.png)

现在我再展开正常状态下的查找操作的那一段代码，到了这里就比较简单了。

```java
else {
    V oldVal = null;
    // 锁住头结点
    synchronized (f) {
        // 双重检查，判断节点上的元素是否遭到过修改，如果失败则进行下一次 for 循环
        if (tabAt(tab, i) == f) {
            // 头结点的 hash 值大于零说明是链表
            if (fh >= 0) {
                binCount = 1;
                // binCount 等于链表的节点数
                for (Node<K,V> e = f;; ++binCount) {
                    K ek;
                    if (e.hash == hash &&
                        ((ek = e.key) == key ||
                         (ek != null && key.equals(ek)))) {
                        oldVal = e.val;
                        // 如果onlyIfAbsent为false，那就使用新值覆盖旧值
                        if (!onlyIfAbsent)
                            e.val = value;
                        break;
                    }
                    Node<K,V> pred = e;
                    // 尾插法，建议画图理解
                    if ((e = e.next) == null) {
                        pred.next = new Node<K,V>(hash, key,
                                                  value, null);
                        break;
                    }
                }
            }
            // TreeBin为红黑树的root节点，它持有一整树
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
        // 链表树化的值：int TREEIFY_THRESHOLD = 8;
        if (binCount >= TREEIFY_THRESHOLD)
            // 即使到了这里，也不一定会树化，如果数组长度小于64则进行数组的扩容
            treeifyBin(tab, i);
        if (oldVal != null)
            return oldVal;
        break;
    }
}
```

#### void treeifyBin(Node<K,V>[] tab, int index)
上面这一段代码的结尾处刚说到，如果一个链表的节点数大于等于 8 ，则会进入`treeifyBin(tab, i)`。
能够进入到这个方法那么 i 位置上的节点数必然是大于等于 8 的，即使达到了红黑树的树化阈值那也不一定会树化。
* 如果现在数组长度小于 64，那么进行数组的扩容，将这些碰撞的元素分散到一个更大的空间中去
* 如果长度大于等于 64，那么就树化该节点上的链表，并用一个TreeBin结构——这是树的root节点——来持有树。

```java
private final void treeifyBin(ConcurrentHashMap.Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            // 数组长度小于64时进行的是数组的扩容
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                // 注意，在这里n就已经加倍了
                tryPresize(n << 1);
            // hash > 0 表示该节点是一个处在正常状态下的节点，没有在迁移，也不是红黑树节点
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    // 双重检查
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
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
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```
&emsp;&emsp;上面这一段代码应该还是蛮清楚的，就两个分支：扩容和树化，区别这两个分支的因素是数组容量，容量小于 64 扩容，否则树化。

#### void tryPresize(int size)

迁移任务中的任务包概念是什么？

> 在多线程环境下，旧数组的迁移任务可以由多个线程同时进行，chm 会将一整个迁移任务分成很多个任务包，每个任务包的大小是 stride（步长）。
> 每一个线程要加入迁移任务首先要将 rs + 1，表示有一个新的线程加入了迁移任务。一个线程完成了自己的迁移任务之后需要系统重新分配任务包，再进行迁移。

```java
    // 这一段代码主要是进行迁移前的准备工作
    private final void tryPresize(int size) {

        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
                tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        // sizeCtl大于 0，表示sizeCtl现在存的就是数组容量的初始值
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            /**
             * 下面这个一段 if 语句，用来初始化数组
             *
             * 我是从putVal -> treeifyBin -> tryPresize 一路看来的
             * 我很疑惑明明 putVal 里面已经保证了数组的初始化，为什么这里还要判断数组是否为 null？
             * 理由很简单，不仅仅是 treeifyBin，还有别的方法调用了 tryPresize，而那个方法没有保证数组的初始化
             * ，我找了一下发现，putAll 方法也调用了tryPresize。
             */
            if (tab == null || (n = tab.length) == 0) {
                // 取较大值来创建数组
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
                        // 将sizeCtl设置为新数组容量的四分之三，表示为扩容阈值。
                        sizeCtl = sc;
                    }
                }
            }
            /**
             * int MAX_VALUE = 0x7fffffff
             * 0x7fffffff == 0111 1111 1111 1111 1111 1111 1111 1111
             * MAXIMUM_CAPACITY = 1 << 30;
             * 如果使用 sizeCtl 来初始化 || 传进来的size大于等于 MAXIMUM_CAPACITY 表示在迁移的时候发生了OOM，在transfer方法的初始化代码中可能会发生OOM
             */
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                // 先去看resizeStamp方法的解析，看完后接着往下走
                int rs = resizeStamp(n);
                // sc为-1表示正在进行初始化，sc < -1 表示正在进行迁移
                if (sc < 0) {
                    Node<K,V>[] nt;
                    /**
                     * (sc >>> RESIZE_STAMP_SHIFT) != rs ==>不清楚原因
                     * sc == rs + 1             ==>不清楚
                     * sc == rs + MAX_RESIZERS  ==>同时参与的线程数已达最大值
                     * (nt = nextTable) == null ==>整个数组的迁移已经完成
                     * transferIndex <= 0       ==>迁移已经完成
                     */
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                            sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                            transferIndex <= 0)
                        break;
                    /**
                     * 将自己重新加入迁移任务，迁移线程 +1
                     */
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                /**
                 * sc >= 0 创建新数组,将当前线程作为迁移的第一个线程。详情看resizeStamp方法解析
                 */
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                        (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```

#### int resizeStamp(int n)
&emsp;&emsp;这应该算是 chm 的难点之一，但我保证能让你理解好这个方法。

&emsp;&emsp;传进来的这个n表示的是旧数组的容量，我们的目的是通过resizeStamp方法将数组容量n和sizeCtl能够用一个32位的数字表示。技巧就是：这个二进制从左往右数第一位为 1，表示这个二进制是一个负数，紧接着的 15 位表示数组容量，剩下来 16 位表示sizeCtl的值。

```java
// 数组容量的最大值
private static final int MAXIMUM_CAPACITY = 1 << 30;

private static int RESIZE_STAMP_BITS = 16;

// 既然上面是 16，那 32 - 16 得到的自然是 15，记住它的值 15，等下要用到。
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```
#### Integer.numberOfLeadingZeros(int i)

这个方法的作用是取得传入数字，从高位起遇到第一个 1 之前的 0 的个数。OK!

看一下我举的例子，形象理解一下
```
Integer.numberOfLeadingZeros(n) 
将这个数作为一个二进制数从左开始数 0 的个数，遇到第一个一时就停下，返回得到的值。

n = 8；表示数组容量为 8 
32 位下二进制：0000 0000 0000 0000 0000 0000 0000 1000
那Integer.numberOfLeadingZeros(8)得到的结果为28。
建议你到 IDEA 输出一下看看。
System.out.println(Integer.numberOfLeadingZeros(8));

n = 1 << 30;这是 chm数组的最大容量
32 位下二进制：0100 0000 0000 0000 0000 0000 0000 0000
那Integer.numberOfLeadingZeros(8)得到的结果为1。
System.out.println(Integer.numberOfLeadingZeros(1 << 30));
```
&emsp;&emsp;因为 chm 的容量总是 2 的次方，如果把容量转换为二进制，那这个二进制中只有一个 1，其余全为 0。我们只需 6 位二进制就能表示 chm 的所有容量情况，之所以能做到这种事，是因为 chm 的容量总是 2 的次方，它的二进制值中有且只有一个 1。

你可能会想 resizeStamp 方法的左边我知道干什么的了，右边呢？
我们来看看右边

```java
// 
private static int RESIZE_STAMP_BITS = 16;

(1 << (RESIZE_STAMP_BITS - 1))
--> (1 << (16 - 1))
--> (1 << 15)
--> 0000 0000 0000 0000 1000 0000 0000 0000
得到一个只有第 16 位为1，其余位全为 0 的数字。
```

接下来就剩将上面两个操作进行一下或运算了，我认为或运算就是将对应位置的1留下来，只要这个位置有 1，无论这个 1  是在上面或是下面，那这个 1 都能被留到结果中。

```java
下面就是四种0和1相或的情况了。
0 | 0 = 0；
0 | 1 = 1；
1 | 0 = 1;
1 | 1 = 1;
```
```java
// 这是原来的 resizeStamp 方法
static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
// 两个例子来加深理解

// 方法的右边
(1 << (RESIZE_STAMP_BITS - 1))结果如下：
0000 0000 0000 0000 1000 0000 0000 0000
    
// 第一个例子：
Integer.numberOfLeadingZeros(8)结果如下：
28 == 0000 0000 0000 0000 0000 0000 0001 1100

方法右边与 28 进行 | 运算，得到的结果如下：
0000 0000 0000 0000 1000 0000 0001 1100

// 第二个例子：
Integer.numberOfLeadingZeros(MAXIMUM_CAPACITY)结果如下：
1 == 0000 0000 0000 0000 0000 0000 0000 0001

方法右边与 MAXIMUM_CAPACITY 进行 | 运算，得到的结果如下：
0000 0000 0000 0000 1000 0000 0000 0001
```
你要明白，第 16 位这个 1 右边表示的就是 chm 的容量


####　(rs << RESIZE_STAMP_SHIFT) + 2)

rs 经过 resizeStamp 之后，它的二进制的第 16 位为1，第 16 位的右手边存储着数组的容量。

结果 rs << RESIZE_STAMP_SHIFT 之后，第 16 位这个 1 到了第 32 位，第 31 位到第 17位存储着数组的容量。移位之后首位为 1，它表示一个负数。

建议你在草稿纸上写一写，体会一下移位的过程。

假设 rs 移位之后为 `1000 0000 0000 1000 0000 0000 0000 0000`
移位后 +2 的结果   `1000 0000 0000 1000 0000 0000 0000 0010`

**很明显这个负数变得更小了。**

### void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)

这个方法里面包装了两倍扩容建立新数组的过程，但主要的内容还是迁移。

这个方法的代码有 160 行，我不会直接把这 160 行都贴出来，一行一行讲解。因为要一下理解这 160 行实在是难，我的做法是：我把主要的逻辑贴出来，把部分逻辑的代码折叠。我希望这样做能够让你把这个方法的大体思路理解好，之后我再把这 160 行全都贴出来，一行行讲解。

{ . . . }是我折叠代码的标志
```java
// transferIndex 的大小是未完成迁移任务的元素数量 + 1
private transient volatile int transferIndex;

private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 如果是多核处理器那么步长为 n / (8*NCPU)，如果得到的步长小于16，则将步长设置为16。如果为单核那步长就直接被设置为n了。
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE;
        // 创建一个两倍长度的新数组并赋值给nextTable，将n赋值给transferIndex。这是我们迁移任务的目的地。
        if (nextTab == null) {...}
        int nextn = nextTab.length;
        /**
         * 这个构造方法做了两件事：
         * 1. new ForwardIngNode<K,V>(hash = MOVED, key = null, value = null, next = null);
         * 2. nextTable = nextTab;
         * 而 nextTable 是 chm 的属性之一，文章最开头我就说过。
         */
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // 单个桶的迁移任务
        boolean advance = true;
        // 整个数组的迁移任务
        boolean finishing = false;
        /**
         * 插一句嘴，finishing表示的是整个数组的迁移任务，advance表示的是一个桶的迁移任务。
         * 你们有没有好奇为什么没有一个变量表示任务包的迁移呢？
         * 任务包的迁移由下面的 i 和 bound 负责。i 表示任务包的左边界，bound 表示任务包的右边界。
         */
        
        // i 指向这个任务包最右边的一个节点，bound 指向任务包最左一个节点。
        // 迁移任务是由右往左的，i逐渐减少。当 i == bound 代表没有需要迁移的元素了
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 设置任务包范围
            while (advance) {...}
            // 任务包或者全部迁移是否完成的判断
            if (i < 0 || i >= n || i + n >= nextn) {...}
            // 老元素为 null，就直接进行 cas 把 fwd 赋到那个位置
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 我不理解为什么要判断一下 hash == MOVED
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            // 这里是真正对一个元素执行迁移任务
            else {...}
        }
    }
```

结合下面的流程图理解

![ConcurrentHashMap-transfer流程图 _1_.png](https://i.loli.net/2020/04/14/76fe3PTRszuDxYI.png)

你自己也可以画图理解一下

大概的逻辑理解好了之后，我们看看 transfer 全部的代码

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE;
        
        if (nextTab == null) {
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {
                // 可能发生OOM，sizeCtl的值为Integer.MAX_VALUE是OOM发生的标志。
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        
        boolean advance = true;
        boolean finishing = false;
        
        // 循环迁移，迁移一次，advance置为false一次，两者交替进行。
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 确定任务包范围
            while (advance) {
                int nextIndex, nextBound;
                // 仍然存在迁移的元素 || 整个数组的迁移已完成
                if (--i >= bound || finishing)
                    advance = false;
                // 旧数组的迁移已全部完成
                else if ((nextIndex = transferIndex) <= 0) { 
                    i = -1;
                    advance = false;
                }
                // 设置任务包大小，设置 i、bound 的值
                else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                    bound = nextBound;
                    // 现在 i 的值为数组最后一个元素下标
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            // 任务包或者全部迁移是否完成的判断
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 数组迁移后的收尾工作
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 退出迁移任务
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 仍有线程在进行迁移任务
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            // 将空的桶置为ForwardingNode
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // 下面的ln可能表示low n，hn可能表示high n
                        Node<K,V> ln, hn;
                        // 桶f 还没有被迁移
                        if (fh >= 0) {
                            // 区分桶 f 应该被放置在新数组中原来的位置还是多出来位置
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            // 获得链表最后一个元素的引用和它的hash值
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            /**
                             * 不理解下面这里为什么要对runBit进行分情况讨论？
                             * 哈哈，我终于想起来了，有的元素因为扩容了要到新的位置上去。
                             */
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            /**
                             * new出来的元素会一直被覆盖，最后一个元素才会留下来。
                             */
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            // 将老元素的位置设置为forwadNode
                            setTabAt(tab, i, fwd);
                            // 这个位置的元素已迁移
                            advance = true;
                        }
                        // 关于红黑树的迁移
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

### 感谢
* 我的 ConcurrentHashMap 的入门之路 https://www.javadoop.com/post/hashmap
* 我从这篇文章了解了 sizeCtl 和 spread( ) 相关的知识 https://blog.csdn.net/tp7309/article/details/76532366
* 我从这篇文章了解到了 resizeStamp( ) https://juconcurrent.com/2018/12/11/ConcurrentHashMap-source-03-transfer/
* 这篇文章讲到了红黑树和 TreeBin 的 lockState，不过我还没看 https://blog.csdn.net/jy02268879/article/details/88599830
* 讨论了树化阈值 https://blog.csdn.net/wo1901446409/article/details/97388971
* 谈到了ConcurrentHashMap 的死锁 bug，还没看 https://blog.csdn.net/lx1848/article/details/81256443