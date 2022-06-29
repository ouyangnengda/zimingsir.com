---
layout: default
title: "ReentrantLock源码分析"
data: 2022-06-29 00:44:00 -0000
category: java
---

ReentrantLock 为重入锁，继承自 Lock 接口，内部定义了公平同步器和非公平同步器的实现，两个同步器都继承自同步器抽象类，而同步器抽象类又继承自 AQS 抽象类。

ReentrantLock 内部的方法很少，主要的几个：lock()、 tryLock()、 unlock，这些方法的内部调用的也是同步器的 acquired() 和 release() 方法。

**构造方法**
如果是通过无参构造函数创建的 ReentrantLock 对象，内部的同步器实现为非公平同步器。如果构造对象的时候你指定了是否公平，而你又选择了公平，那么同步器的实现就为公平同步器。

```java
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

```

**实例方法**
类中的实例方法也是相当的简单，实现全丢在 Sync 中了。

注意一下下面这三个方法。lock() 和 unlock() 是跟着 Sync 的实现走的，而 tryLock 调用的这个方法则是定义在 Sync 抽象类中。

```java
    public void lock() {
        sync.lock();
    }

    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    public void unlock() {
        sync.release(1);
    }
```

**Sync**

Sync 类继承自 AQS，它内部实现了获取临界区和释放锁的方法。

其实你可以把这个 nonfairTryAcquire 方法就看做是一个大的 CAS 判断，作用就是讲当前线程设置为头结点。

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {

        abstract void lock();

        final boolean nonfairTryAcquire(int acquires) {
            /* 省略方法实现*/
        }

        protected final boolean tryRelease(int releases) {
            /* 省略方法实现*/
        }
    }
```

> try开头的方法表示尝试获取，它只竞争一次，竞争成功那我就获得锁，返回true；竞争失败，纳表示没有拿到锁，则返回 false。
> 那么没有使用 try 开头的方法他们的含义是怎么样的呢？那 ReentranLock 中的 lock 方法来举例，你仔细看一下，这个方法的返回值为 void，你应该感到好奇，返回值应该为 true 或 false 吗？表示是佛获取到了锁，难道这个方法不关心结果？还是它的结果永远都只有一个？后一种猜测是对的，这个方法的结束只有一种情况，那就是获取到锁，第一次进入的时候它会试图进行一次 CAS判断，如果失败它会进入 AQS 的队列然后被挂起，当前一个等待节点释放了锁，这时候当前线程就会被唤醒，拿到锁进入临界区。

**NonFairSync**

非公平同步器，后进来的线程可能先获得锁。所以这个方法先进行了一次 CAS 判断，这儿体现了它的非公平。

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

**FairSync**

公平的同步器，后进来的线程一定更后地获取到锁。因此这个实现类的 lock 方法没有进行 CAS 的尝试。

看这个类的 tryAcquire() 它用的不是父类的 nonfairTryAcquire() 而是自己定义的。你拿这个方法和父类中定义的方法比对一下，再想一下这个类的 FairSync，看看能想通为什么吗？

hasQueuedPredecessors() ：如果当前线程的前面还有线程在等待获取锁。

这个类实现两个方法全都体现了公平、排队思想。

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

**acquire()**

Sync 抽象类中定义了抽象方法 lock()，而 NonFairSync 和 FairSync 这两个抽象类中又实现了这个抽象方法，ReentranLock 的 lock 方法调用也是 Sync 实现类里实现的方法。
这两个实现类中的 lock 方法又有些许区别，NonFairSync 中首先进行了一次 CAS 的判断，CAS 失败再调用 acquire()，而 FairSync 则是直接调用 acquire() 。

acquire 的实现在 AQS，换言之这是 AQS 层面的获取锁。这个层面的逻辑是，首先调用 Sync 实现类的 tryAcquire 方法去直接获取锁，如果失败的话，那就创建一个 Waiter 对象加入等待队列。

既然 arg 总是 1，为什么还要这样一个参数，写死不好吗？如果说是为了扩展性，那么还有别的什么地方用了？

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

tail 为 null 什么情况下会发生？初始的等待队列是空的，随着元素一个一个进入，队列的变化是怎样的？

既然已经到了 addWaiter 方法，那么这里的乐观尝试措施就不再是将自己 CAS 为头结点，而是将自己 CAS 为等待队列的最后一个节点。如果这个 CAS 失败的话，那就调用 enq，使用自旋的方式入队。这个方法里面也罢前后的这种联系建立起来了。

这个 enq 方法的自旋里面就两个操作，初始化 tail 变量；以 CAS 的方式入队。这个方法 else 区块的代码很有味道，如果你看不懂可以试着画图去理解，挺有意思的。

这个 addWaiter 方法无论如何都会将这个 Node 入队的
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    
```

回看 acquireQueued() ，入参是一个 Node，这时候的 node 已经入队，arg 为 1。下面这段代码的第一个判断仍然是一个乐观的尝试措施，如果尝试失败那就调用接下来的方法，将线程挂起。

shouldParkAfterFailedAcquire() 根据前驱节点的状态来进行对应处理。如果前驱节点为 SINGAL（-1），表示当前节点需要阻塞；如果前驱节点的 waitStatus 大于 0，其实只能为 1 ，这表示前驱节点为取消状态，那么我们往前找一直找到一个小于等于 0 的，并联系起来；最后一种情况 waitStatus 为 0，这时候会将前驱节点的 waitStatus 置位 SINGAL，然后返回 true。下一次的外圈循环会判断，如果前驱节点为 SINGAL，那么返回 true，当前节点上的线程会被挂起。

处于等待链表中 Node 节点的 waitStatus 大多数情况下都是 0，一个 Node 被初始化 waitStatus 也默认为 0。

节点的 waitStatus 什么情况下会被设置为 0 ？

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

**release**

relase() 方法的实现仅仅存在于 AQS 类，不过其中的 tryRelease() 方法的实现却在子类中，例如 Sync。

首先调子类的 tryRelease() 方法递减重入次数，只有当重入次数减为 0 时这个方法才会返回 true，其它情况这个方法只会返回 false。

当 tryRelease() 方法返回 true时，表示 AQS 的 state 方法被减到了 0，这时候就可以去唤醒等待队列中下一个可以被唤醒的线程了。
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    
```

当锁被释放之后会调用 unparkSuccessor() 去唤醒后续的线程。

里面这个循环会从 tail 开始从后往前找，找到一个离 node 最近的 waitStatus 为 SINGAL 或 NORMAL 的 node，如果那个 node 不为 null，唤醒它。这是时候那个线程又会接着 acquireQueued() 去重新获取锁。

```java
    private void unparkSuccessor(Node node) {

        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
