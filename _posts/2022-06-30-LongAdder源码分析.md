---
layout: default
title: "LongAdder源码分析"
data: 2022-06-30 00:45:00 -0000
category: java
---


```java
/**
 * 这个add方法返回值是void，所以我们不能通过这个方法得知操作是成功还是失败。
 * 这个方法有点异步的意思：这个操作我做出去了，但是我不关心它的结果
 */
public void add(long x) {
    Cell[] as;
    Cell a;
    long b;
    long v;
    int m;

    /**
     * 把cell数组复制给as，如果数组不为null则直接进入下面的代码块
     * 如果数组为null则还要进行后面的判断:base为原值，b+x为结果，如果对他们cas失败了则也会进进入下面的代码。
     * 进入if代码块的条件:cell数组存在 或者 cas失败
     */
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        /**
         *             ||       as中有元素          ||   本线程在cell数组的对应位置为null   ||    将cas结果赋值给uncontended
         */
        if (as == null || (m = as.length - 1) < 0 || (a = as[getProbe() & m]) == null || !(uncontended = a.cas(v = a.value, v + x)))
            //          要增加的值        false
            longAccumulate(x, null, uncontended);
    }
}
```

从上面最后这个长判断，我们知道longAccumulate会做三件事：初始化Cell数组，初始化Cell元素，对Cell元素进行循环的CAS

不断尝试

```java
/**
 * @param x              要增加的值
 * @param fn             fn包括了一个Long的左操作数和右操作数，当update的时候fn会传一个对象进来，如果是add操作则是null。
 * @param wasUncontended 传入的时候为false
 *                       <p>
 *                       试图获取对象的锁，1表示上锁，0表示没有上锁
 *                       final boolean casCellsBusy() {
 *                       return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
 *                       }
 */
final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {
    int h; // 随机数
    if ((h = getProbe()) == 0) {
        
        ThreadLocalRandom.current(); // force initialization 初始化随机数生成器
        h = getProbe(); // 获取随机数
        wasUncontended = true;
    }
    // collide:碰撞
    boolean collide = false;                // True if last slot nonempty
    // 明明是个死循环里面的两个符号竟然还保持着代码规范
    for (; ; ) {
        Cell[] as;
        Cell a;
        int n; // 表示数组的length
        long v; //cell的值
        //     cell数组存在        且    有元素
        if ((as = cells) != null && (n = as.length) > 0) {
            // 线程对应槽为null
            if ((a = as[(n - 1) & h]) == null) {
                // 试图去获得一个新槽
                if (cellsBusy == 0) {       // Try to attach new Cell
                    // 将自己要增加的值放入槽中
                    Cell r = new Cell(x);   // Optimistically create
                    /**
                     * casCellsBusy()这个方法应该是用来测试忙不忙的，不忙则返回true
                     */
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs;
                            int m, j;
                            //  cell数组存在                    数组中有元素             线程对应槽为null
                            if ((rs = cells) != null && (m = rs.length) > 0 && rs[j = (m - 1) & h] == null) {
                                // 把上面new的cell放到对应槽中
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        // 上面的操作都成功了那就直接退出这个死循环
                        if (created)
                            break;
                        // 创建失败
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
                // 把wasUncontended置为true
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
                // cas成功就直接退出死循环
            else if (a.cas(v = a.value, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
                break;
            else if (n >= NCPU || cells != as)
                collide = false;            // At max size or stale
                // 将false置为true
            else if (!collide)
                collide = true;
            // 扩容
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    // 如果还是原来那个数组，那么对数组进行扩容
                    if (cells == as) {      // Expand table unless stale
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            // 扰乱h
            h = advanceProbe(h);
            /**
             * 初始化数组
             * 初始化cell数组并且把x赋进去
             * 条件是cell数组为null 或 数组长度为0
             */
        }
        // 初始化cells数组
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            // 在cells == as 和 casCellsBusy()操作获得锁之间的时间，cells被替换了，初始化就会失败。
            if (init)
                break;

        }
        // 数组中没有元素 且 初始化不是由自己完成的，则执行下面的cas
        else if (casBase(v = base, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
```


LongAdder 的自我提问？

LongAdder 一开始针对一个long类型 cas， cas 失败之后会进化为对一个数组进行cas，分散竞争的压力。

LongAdder 进化后会退化吗？

LongAdder 进化为数组之后，发现对应下标没有元素，为什么不做一次优先的尝试？

LongAdder 方法会做三件事：cell 数组的初始化，cell 下标元素的初始化，最普通的 cas 赋值

LongAdder 初始化的时候会怎么做：拿到busy锁，然后创建数组，再将当前long放到里面来。 

LongAdder 初始化数组元素只有两个，有点想不通？