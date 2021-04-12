# ReentrantLock源码分析

可重入锁，继承于[Lock](http://pms.ipo.com/display/JGY/lock)，对于ReentrantLock需要了解到：

1.通常这样使用：

```java
private final ReentrantLock lock = new ReentrantLock();
public void m() {
    lock.lock();  // block until condition holds
    try {
      // ... method body
    } finally {
      lock.unlock()
    }
}
```

2.可以实现公平锁、非公平锁（默认是非公平锁）

```java
public ReentrantLock() {
    this.sync = new ReentrantLock.NonfairSync();
}
 
// true则为公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### 结构分析

```
ReentrantLock是通过AbstractQueuedSynchronizer实现，AbstractQueuedSynchronizer核心的数据结构是一个双向队列，
具体结构示意图如图：
```

![img](file:////Users/zhusidao/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image001.png)

```java
AbstractQueuedSynchronizer中定义双向队列中Node节点中的几个重要状态(这里我们只列举出ReentrantLock中所使用到的状态)，
状态说明见代码中的注释
// 共享模式
static final Node SHARED = new Node();
// 独占锁
static final Node EXCLUSIVE = null;
 
// 线程已经取消
static final int CANCELLED =  1;
// 后续的线程等待被唤醒
static final int SIGNAL    = -1;
 
/**
 * 等待状态：CANCELLED、SIGNAL、0(初始化状态)
 */
volatile int waitStatus;
 
// 上一个节点
volatile Node prev;
 
// 下一个节点
volatile Node next;
 
// 队列中的线程
volatile Thread thread;
```

### 这里开始上源码分析吧

#### **lock**

```java
public void lock() {
    sync.lock();
}
```

sync.lock()有两种实现：FairSync公平锁和NonFairSync非公平锁

 

这里我们先看**公平锁**实现

```java
// 直接调用acquire(1)
final void lock() {
    acquire(1);
}
```

**acquired**方法：获取锁的核心方法

```java
/**
 * 整体分为四大步骤
 * 1、tryAcquire：获取锁，其实就是设置state，
 * 成功了，if语句就终止，失败了，if语句就继续下面的方法（短路），
 * 在AQS只是一个空的方法，需要其他的实现者去实现该方法
 * 2、addWaiter：使用独占模式，
 * 将一个node节点放到AQS维护的队列的队尾
 * 3、acquireQueued：主要的阻塞方法，会进行状态的翻转和再次确认等操作，
 * 如果没有中断返回false，如果中断了，返回true，下面详细分析
 * 4、selfInterrupt：设置当前线程的中断状态
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

ReentrantLock的对**tryAcquired**方法的实现如下：

```java
/**
 * 如果状态为0(说明是此时锁是空闲状态)：
 *     当前队列是空，或者当前线程是head节点的后一个节点的线程
 *     状态更改为1
 *     设置当前拥有独占访问权的线程
 * 否则判断当前线是否是之前设置的独占线程(锁重入)
 *     设置state值更新
 * 这里可以说明：若所获取成功则返回为true，否则返回未false
 */
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
```

**hasQueuedPredecessors**

```java
/*
 * 当前线程不是队列中的第二个节点的线程(head的后一个节点)返回true
 */
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

**addWaiter**入队操作

```java
/**
 * 入队操作
 */
private Node addWaiter(Node mode) {
    // 创建一个Node节点
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试快速入队
    Node pred = tail;
    // 队列不为空
    if (pred != null) {
        node.prev = pred;
        // CAS算法保证tail不被修改
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 正常入队
    enq(node);
    return node;
}
```

**enq**

```java
/**
 * 当为空队列的时候会首先创建一个空的节点，然后再将node添加到队尾
 */
private Node enq(final Node node) {
    // 自旋到成功添加node节点
    for (;;) {
        Node t = tail;
        // 空队列
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 添加节点到尾部
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
 
```

**acquireQueued**该方法主要是用来获取锁，或者线程进入阻塞

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果前一个节点是队列的head，尝试去获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            /*
             * shouldParkAfterFailedAcquire进行状态标记
             * parkAndCheckInterrupt会让线程进入block状态
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

 **shouldParkAfterFailedAcquire**方法需要说明的地方：

1.当前添加的node节点的前一个节点如果状态为SIGNAL(-1)，返回true，执行parkAndCheckInterrupt使先当前线程进入block状态

2.如果前一个节点的状态大于0（说明为1，已取消状态），就将队列前面的状态大于1的节点都剔除掉

3.否则就将前一个节点的状态变为SIGNAL(-1)

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * 前一个的node已经设置过了SIGAL状态
         * 当前入队的node能进入block状态
         */
        return true;
    if (ws > 0) {
        /*
         * 前一个node的状态已经被取消，剔除前面已经被取消的node
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus一定是0或者是PROPAGATE，将前一个node的状态设置成SIGNAL，可能会设置失败，
         * 那就回进入到外层的死循环进行重试
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

**parkAndCheckInterrupt**

```java
/**
 * 这个方法是在shouldParkAfterFailedAcquire方法返回true之后才能执行
 * 其实就是可以阻塞方法，阻塞的条件是：前置节点是SIGNAL状态
 */
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 线程进入block状态
    return Thread.interrupted(); //如果被唤醒，查看自己是不是被中断的
}
```

 

发现如果非正常结束的情况下会走finally里面的if分支里面的方法：

```java
private void cancelAcquire(Node node) {
        // 校验当前node是否存在
        if (node == null)
            return;
 
        // 将节点中持有的线程对象扔掉
        node.thread = null;
 
        // 如果当前节点的前置节点都是取消状态的，那么就一直找到非取消状态的前置节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
 
        // 拿到前置节点的后置节点，其实就是当前节点，不过为了后面的设置有用
        Node predNext = pred.next;
 
 
        // waitStatus是int的，原子赋值为取消的状态
   			//这样一赋值，后面的再次添加到队列中的结点就能忽略当前这个取消的结点了
        node.waitStatus = Node.CANCELLED;
 
        // 如果当前取消的结点是尾节点的话，就直接将尾节点设置成前置节点，这样当前节点就解链了
        if (node == tail && compareAndSetTail(node, pred)) {
            // 将原先前置节点的后置节点置为空，这样当前节点就是一个可gc的对象了
            compareAndSetNext(pred, predNext, null);
        } else {
            /**
             * 这个分支稍有复杂：
             * 走到这个分支，首先取消的结点并不是尾节点；
             * 另外如果此前置节点也不是头结点，并且是后置节点等待资源的状态（SIGNAL），
             * 那么将当前取消节点的后置节点，提前；
             * 如果前置节点是头节点或者前置节点已经是取消的状态、或者前置节点设置状态位失败、或者
             * 前置节点的内部线程是一个空，那么，
             * 将当前取消节点的后置节点唤醒去争取资源（unparkSuccessor）
             */
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node);
            }
 
            node.next = node; // help GC
        }
    }
```

 

以上便是所有的获取锁定操作了，做一个小quire的总结，相关代码执行流程如图所示：

![img](file:////Users/zhusidao/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image002.png)

上面整个获取锁的过程已经结束，接下来看**锁的释放**

```java
public void unlock() {
    sync.release(1);
} 
```

**release**

```java
/**
 * 如果锁释放成功，则取唤醒head的后继节点并返回true，
 * 否则返回false
 */
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

 **tryRelease**

```java
/**
 * 若成功释放锁返回true， 否则返回false
 */
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // state为0，说明当前锁处于空闲状态
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

 **unparkSuccessor**

```java
/**
 * 唤醒node的第一个正常(not null && state>0)后继节点
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

以上就是所有ReentrantLock公平锁的实现，当线程成功持有锁的时候，state会从0变为1。

当该线程发生重入时，state会加1，而锁的释放就是依次减1，当state为0的时候，说明锁是空闲的状态。

方法之中用了许多CAS保证操作的原子性。

 

**非公平锁**和公平锁其实差不多，还是直接看其中lock方法实现， 和tryAcquire方法重写。

**lock**

```java
/**
 * 首先直接尝试进行锁的抢占.
 * 是否还有必要先去尝试进行锁抢占呢
 */
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

**tryAcquire**

```java
// 获取锁
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
} 
```

**nonfairTryAcquire**

```java
/*
 * 与公平锁的实现来看，少了一个hasQueuedPredecessors的判断，
 * 这里是只要锁是空闲的状态，可以直接获取到锁，不管线程是刚创建还是之前入队的线程，
 * 存在抢占的可能性
 * 公平锁加了hasQueuedPredecessors判断之后，刚创建的线程是无法满足条件判断的，则无法获取锁
 * compareAndSetState方法，可以结合两者做比对
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 进行锁重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

 

说完了获取锁公平和非公平的方式，接下来看下**tryLock**

```java
/**
 * 以非公平的方式去获取锁（无论创建的ReentrantLock是公平还是非公平的方式），获取成功则立刻返回true， 否则返回false
 */
public boolean tryLock() {
    // 调用的是非公平的方式去获取锁
    return sync.nonfairTryAcquire(1);
}
```

以上便是所有的源码分析， 不足之处，还望大佬指出和交流