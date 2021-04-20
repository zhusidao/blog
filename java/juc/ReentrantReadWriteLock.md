# ReentrantReadWriteLock源码分析

 

上一篇[ReentrantLock](ReentrantLock.md)中已经介绍过AbstractQueuedSynchronizer相关数据结构非共享锁的实现，本篇中会介绍JUC中常用的ReentrantReadWriteLock，以及在AbstractQueuedSynchronizer共享锁的源码分析。

 **概述**：ReentrantReadWriteLock是Lock的另一种实现方式，我们已经知道了ReentrantLock是一个排他锁，同一时间只允许一个线程访问，而ReentrantReadWriteLock允许多个读线程同时访问，但不允许写线程和读线程、写线程和写线程同时访问。相对于排他锁，提高了并发性。在实际应用中，大部分情况下对共享数据（如缓存）的访问都是读操作远多于写操作，这时ReentrantReadWriteLock能够提供比排他锁更好的并发性和吞吐量。

#### **通常用法**

示例一：利用重入来执行升级缓存后的锁降级

```java
class CachedData {
    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 
    void processCachedData() {
        rwl.readLock().lock();
        if (!cacheValid) {
            //获取写锁前须释放读锁
            rwl.readLock().unlock();
            rwl.writeLock().lock();
            try {
                // Recheck state because another thread might have
                // acquired write lock and changed state before we did.
                if (!cacheValid) {
                    data = ...
                    cacheValid = true;
                }
                               //锁降级，在释放写锁前获取读锁
                rwl.readLock().lock();
            } finally {
                rwl.writeLock().unlock(); // Unlock write, still hold read
            }
        }
 
        try {
            use(data);
        } finally {
            rwl.readLock().unlock();
        }
    }
} 
```

 

示例二：使用 ReentrantReadWriteLock 来提高 Collection 的并发性

　　　　通常在 collection 数据很多，读线程访问多于写线程并且 entail 操作的开销高于同步开销时尝试这么做。

```java
class RWDictionary {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();
    private final Lock w = rwl.writeLock();
 
    public Data get(String key) {
        r.lock();
        try {
            return m.get(key);
        } finally {
            r.unlock();
        }
    }
 
    public String[] allKeys() {
        r.lock();
        try {
            return m.keySet().toArray();
        } finally {
            r.unlock();
        }
    }
 
    public Data put(String key, Data value) {
        w.lock();
        try {
            return m.put(key, value);
        } finally {
            w.unlock();
        }
    }
 
    public void clear() {
        w.lock();
        try {
            m.clear();
        } finally {
            w.unlock();
        }
    }
}
```

####   **源码解析**

读写锁中`Sync`类是继承于`AQS，`首先我们先看ReentrantReadWriteLock 的内部类Sync的一些关键属性和方法，源码如下：

```java
static final int SHARED_SHIFT   = 16;
//共享锁（读锁）状态单位值65536 
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 共享锁线程最大个数65535 
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 排它锁线程最大个数65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
// 实现于ThreadLocal用于统计当前读锁的重入次数
private transient ThreadLocalHoldCounter readHolds;
// 缓存最近使用的HoldCounter类的对象(如在一段时间内只有一个线程请求读锁则可加速对读锁获取的计数)
private transient HoldCounter cachedHoldCounter;
// 第一个获取读锁的线程
private transient Thread firstReader = null;
// 第一个线程获取锁的重入次数
private transient int firstReaderHoldCount; 
```

实现读写锁与实现普通互斥锁的主要区别在于需要分别记录读锁状态及写锁状态，并且等待队列中需要区别处理两种加锁操作。 
 `Sync`使用`state`变量同时记录读锁与写锁状态，将`int`类型的`state`变量分为高16位与第16位，高16位记录读锁状态，低16位记录写锁状态，如下图所示： 

![image-20210414111422592](ReentrantReadWriteLock.assets/image-20210414111422592.png)

`Sync`使用不同的`mode`描述等待队列中的节点以区分读锁等待节点和写锁等待节点。`mode`取值包括`SHARED`及`EXCLUSIVE`两种，分别代表当前等待节点为读锁和写锁。

#### **先来看读锁**

**tryLock**

```java
public void lock() {
    sync.acquireShared(1);
} 
```

**acquireShared**

```java
 public final void acquireShared(int arg) {
        // 尝试获取锁
    if (tryAcquireShared(arg) < 0)
               // 没有获取到锁
        doAcquireShared(arg);
}
```

**tryAcquireShared**

```java
/**
 * 尝试去获取锁
 */
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        // 写锁持有数非0且不是当前线程锁持有
        return -1;
    /*
     * 下面代码执行中写锁持有可能性
     * 1.写锁持有数为0
     * 2.写锁持有数非0且是当前线性持有
     */
  
    // 获取读锁持有数量
    int r = sharedCount(c);
    /*
     * 1.读锁没有被阻塞
     * 2.读锁没有达到最大值65535
     * 3.CAS成功更新读锁的状态(state高位状态+1)
     */
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 读锁为0
        if (r == 0) {
            /*
             * 标记第一个读锁
             * 标记第一个读锁的持有数
             */
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 当前线程就是第一个读锁时，锁持有数相加(重入)
            firstReaderHoldCount++;
        } else {
            // 最后一次操作的线程
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                // 最后一次操作的线程为空 或者 如果当前线程不是最后一次操作的线程
                // 获取当前线程读锁计数器
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                // 最后一个持有读锁计数器不为空 并且 当前线程就是最后一个持有者 并且 当前线程持有读锁数为0
                // 设置当前线程读锁持有
                readHolds.set(rh);
            // 锁的持有数++
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

> 用ThreadLocal来存储线程读锁获取重入次数；引入了第一个线程读锁计数器和最后一个线程读锁计数器，如果是某一个线程在某一个时刻多次进行获取读锁，就能加快范围速度，不需要去ThreadLocal中获取当前线程的重入次数，也使得代码理解难度度变大。

**readerShouldBlock**

readerShouldBlock是一个抽象方法，我们依次来看公平锁(fair)和非公平锁(unfair)中的具体实现

公平锁中的调用

```java
/**
 * 当前线程不是队列中的第二个线程(head.next),则返回true
 * 否则返回false
 */
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

> Head.next是双向队列中即将要获取锁的节点
>

非公平所的调用

```java
/**
 * 队列中的第二个线程(head.next)是写锁，则返回true
 * 否则返回false
 */
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

**fullTryAcquireShared**处理**tryAcquireShared**中未处理的CAS丢失和应该阻塞的情况下重入读锁获取。

```java
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        // 写锁不为0
        if (exclusiveCount(c) != 0) {
            // 不是当前线程持有锁，需要去排队
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
            // 如果是读锁阻塞
        } else if (readerShouldBlock()) {
            // 不存在写锁
            // Make sure we're not acquiring read lock reentrantly
            // 当前线程获取读锁应该被阻塞
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                // 当前线程不是第一个获取到锁的线程
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        // 最后一个线程计算器为空 或者 最后一个线程的计数器不是当前线程
                        
                        rh = readHolds.get();
                        if (rh.count == 0)
                            // 当前线程读锁持有数为0，又因为不能获取到锁，即将进入等待， 
                            // 所以移除该线程的重入统计的ThreadLocal
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    // 当前线程读锁持有数为0
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 能执行到下面，说明有2种可能
        // 1.读锁进行重入
        // 2.不被阻塞（非公平锁:队列中的第二个节点不是写锁，公平锁:队列中的第二个节点正好是当前线程）
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            // 成功获取锁 读锁+1
            // c是之前的状态（没有进行读锁+1的状态）
            if (sharedCount(c) == 0) {
                // 之前不存在读锁
                // 设置第一个读锁
                firstReader = current;
                // 第一个读锁重入次数
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                // 当前线程是第一个读锁
                // 第一个读锁重入数+1
                firstReaderHoldCount++;
            } else {
                // 之前存在其他的读锁，且不是当前持有
                if (rh == null)
                    // 最后一个持有读锁计数器为空
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    // 最后一个持有读锁计数器为空 或者 当前线程不是最后一个持有读锁的线程
                    // 获取当前线程读锁持有数
                    rh = readHolds.get();
                else if (rh.count == 0)
                    // 最后一个持有读锁计数器不为空 并且 当前线程就是最后一个持有者 并且 当前线程持有读锁数为0
        						// 设置当前线程读锁持有
                    readHolds.set(rh);
                // 持有数+1
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

> 从这里我们看出，读锁获取机制：重入获取锁不受限制，否则根据公平性的原则判断是否能进行获取锁。非公平锁:队列中的第二个节点不是写锁，公平锁:队列中的第二个节点正好是当前线程；这两种情况下才能正确获取到读锁。

**doAcquireShared**

```java
/**
 * 首先尝试入队， 判断前一个节点是否是首节点且能获取到锁
 */
private void doAcquireShared(int arg) {
    // 入队
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 前一个节点
            final Node p = node.predecessor();
            // 前一个节点是head
            if (p == head) {
                // 尝试获取锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取锁成功
                    // 重新设置head节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            /*
             * shouldParkAfterFailedAcquire这里做一个说明，代码分析ReentrantLock已经介绍过了
             * 如果node的前一个节点状态是-1,就会调用parkAndCheckInterrupt将当前线程阻塞；
             * 如果node的前一个节点状态>0, node前面状态>0的节点都剔除；
             * 如果node的前一个节点状态为0,就将前一个节点状态变为-1
             */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**addWaiter**

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 尝试快速入队，失败的话调用enq入队
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
```

 **enq**自旋入队

```java
/**
 * 如果首节点是空的构造第一个节点， 否则入队
 */
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

**setHeadAndPropagate**

```java
/**
 * 如果node的下一个节点是共享模式，则调用doReleaseShared去唤醒下一个节点的线程
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    // 设置为头部节点
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            // 下一个节点是共享模式（读锁获取）
             // 将head节点的下一个节点唤醒
            doReleaseShared();
    }
}
```

> 所以当队列中的获取读锁的线程获取到锁后，会不断的继续唤醒紧临的获取读锁的线程

**doReleaseShared**

```java
/**
 * 如果将head节点状态变为0，并将head节点的下一个节点唤醒
 */
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                // head状态值为-1，尝试将head值变为0
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 唤醒head节点的下一个状态<0的节点
                unparkSuccessor(h);
            }
            // 在队列中的节点对应的线程阻塞之前，将前驱节点的waitStatus状态设置为SIGNAL
            // 所以这块ws==0，其实是当前线程通过第一次循环将状态设置为了0，
            // 第二次循环进入的时候头节点还没有被改变
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // 头节点改变就自旋
        if (h == head)                   // loop if head changed
            break;
    }
}
```

**unparkSuccessor** 

```java
/**
 * 将node节点的下一个状态<0的节点唤醒
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
 
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
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



#### **读锁的释放**

**releaseShared**

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // 所有的读锁都已经释放
        // 
        doReleaseShared();
        return true;
    }
    return false;
} 
```

**tryReleaseShared**

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
        // 当前线程是第一个操作的线程
    if (firstReader == current) {
        // 当前线程持有数为1
        if (firstReaderHoldCount == 1)
            // 当前线程锁已经完全释放，firstReader至为null
            firstReader = null;
        else
            // 没有完全释放，重入数-1
            firstReaderHoldCount--;
    } else {
        // 最后一个操作的线程
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            // 最后一个操作的线程为空 或者 当前线程不是最后一个操作的线程
            // 获取当前线程的读锁持有
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            // 重入数量<=1
            // 移除当前线程中的统计重入次数的ThreadLocal
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        // 重入数-1
        --rh.count;
    }
    for (;;) {
        // 获取当前状态
        int c = getState();
        // state高位-1
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // 更新成功
            // 返回读锁是否完全释放
            return nextc == 0;
    }
}
```



相对于**读锁，写锁**的相关实现调用的acquire方法，和ReentrantLock实现差不多，只是获取锁、锁释放的方式有点不同，这里直接列出不同的部分，其他相关方法的逻辑可以看我的上一篇[ReentrantLock](ReentrantLock.md)。

#### **写锁获取**

**lock**

```java
public void lock() {
    sync.acquire(1);
}
```

**acquire**

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        // acquireQueued和selfInterrupt调用的是AQS中的方法这里不展开讲解
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

**tryAcquire**重点是看ReentrantReadWriteLock中的实现的获取锁的方式

```java
/**
 * 返回true表示已经成功获取锁 ，否则获取失败
 */
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 获取写锁
    int w = exclusiveCount(c);
    // 状态值不为0
    if (c != 0) {
        /*
         * 无法获取锁的情况：
         * 1.写锁值为0(因为锁的状态不为0，说明有还有读锁)
         * 2.持有读锁的线程不是当前线程锁持有
         */
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 持有锁的数量已经超过最大值65535
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 写锁重入
        setState(c + acquires);
        return true;
    }
        /*
         * 对于公平锁：
         *     writerShouldBlock调用hasQueuedPredecessors()方法，
         *     执行逻辑当前线程是队列中的第二个线程(head.next)，且状态更新成功，此时才返回true
         * 对于非公平锁：
         *     writerShouldBlock()返回的固定值false
         */ 
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 设置独占线程
    setExclusiveOwnerThread(current);
    return true;
} 
```

#### **写锁释放**

**release**

```java
/**
 * 如果锁释放成功，则取唤醒head的后继节点(head.next)并返回true，
 * 否则返回false
 */
public final boolean release(int arg) {
    // 尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒队列中第一个处于等待状态的节点（从二个元素开始向后遍历）
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

 **tryRelease**

```java
protected final boolean tryRelease(int releases) {
    // 持有锁的线程不是当前线程
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    // 不存在写锁
    boolean free = exclusiveCount(nextc) == 0;
    // state为0，设置锁持有者为空
    if (free)
        // 设置独占线程为null
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

 **unparkSuccessor**

```java
/**
 * 唤醒node的第一个正常(not null && waitStatus>0)后继节点
 */
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    // 若node状态<0，则将node节点的状态变为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
 
    /*
     * 找寻后继状态小于0不且为空的节点
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 唤醒node
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

以上便是读写锁的相关逻辑实现方式。