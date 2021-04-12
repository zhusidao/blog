# Semaphore源码解析

**信号量（Semaphore）**用来控制同时访问某个特定资源的操作数量，或者同时执行某个制定操作的数量。计数信号量还可以用来实现某种资源池，或者对容器施加边界。

Semaphore中管理者一组虚拟的许可，许可的初始数量可通过构造函数来指定，在执行操作时可以首先获得许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么acquired将阻塞直到有许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么acquire将阻塞直到有许可（或者直到被终端或者操作超时）。release方法将返回一个许可给信号量。计算信号量的一种简化形式是二值信号量，即初始值为1的Semaphore。二值信号量可以用做互斥体（mutex），并具备不可重入的加锁语义：谁拥有了这个唯一的许可，谁就拥有了互斥锁。

**用途**：可以用于实现资源池，例如数据库连接池。我们可以构造一个固定长度的资源池，当池为空时，请求资源将会失败，但你真正希望的行为是阻塞而不是失败，并且当池非空时接触阻塞。

**通常用法：**

```java
class Pool {
    private static final int MAX_AVAILABLE = 3;
    private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);
 
    public Object getItem() throws InterruptedException {
        available.acquire();
        return getNextAvailableItem();
    }
 
    public void putItem(Object x) {
        if (markAsUnused(x)) {
            available.release();
        }
    }
 
    // Not a particularly efficient data structure; just for demo
 
    protected Object[] items = new String[]{"123", "456", "789"};
    protected boolean[] used = new boolean[MAX_AVAILABLE];
 
    protected synchronized Object getNextAvailableItem() {
        for (int i = 0; i < MAX_AVAILABLE; ++i) {
            if (!used[i]) {
                used[i] = true;
                return items[i];
            }
        }
        return null; // not reached
    }
 
    protected synchronized boolean markAsUnused(Object item) {
        for (int i = 0; i < MAX_AVAILABLE; ++i) {
            if (item == items[i]) {
                if (used[i]) {
                    used[i] = false;
                    return true;
                } else {
                    return false;
                }
            }
        }
        return false;
    }
}
```

 

#### 还是直接上源码分析吧

**Semaphore**的构造方法

```java
/**
 * 指定许可证的数量、是否是公平锁。
 */
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
 
 
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

以公平锁为例看其构造方法的调用链

```java
/**
 * 构造方法会执行调用父类的构造方法
 */
NonfairSync(int permits) {
    super(permits);
}
```

**Sync**

```java
/**
 * 最终是初始化了state的值
 */
Sync(int permits) {
    setState(permits);
}
```

 

接下来直接看获取许可的相关方法，因为很多都是AQS（AbstractQueuedSynchronizer）中共享锁的相关调用逻辑，详细可以参考[ReentrantReadWriteLock](http://pms.ipo.com/pages/viewpage.action?pageId=141580618)的读锁的相关实现逻辑，这里也不会赘余给出，看其中的相关不同的实现部分

**acquire**

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

**acquireSharedInterruptibly**

```java
/**
 * 该方法首先判断当前线程是否interrupted状态，如果是true则抛出异常
 * 然后调用tryAcquireShared去获取锁，执行的结果<0则说明获取许可失败，调用doAcquireSharedInterruptibly
 */
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

**tryAcquireShared**有两种实现方式：公平获取、非公平获取

先看**公平获取**的相关实现

```java
/**
 * hasQueuedPredecessors：该方法会判断队列中第二个节点中的线程是否为当前线程，不是当前线程就返回true，否则返回false
 * hasQueuedPredecessors该方法其实就已经实现了公平性，排后面的线程必须继续排队等待，排在第二位(head.next)之的线程才能尝试去获取许可
 */
protected int tryAcquireShared(int acquires) {
  	// 自旋
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        // 获取state
        int available = getState();
        // 所剩余的许可
        int remaining = available - acquires;
        /*
         * 许可小于0
         * 剩余许可>=0，且更新状态成功
         */
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

**非公平获取**的相关实现

```java
/**
 * 调用tryAcquireShared
 */
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
} 
```

**nonfairTryAcquireShared**

```java
/**
 * 与公平锁不同的是，当许可大于0且状态更新成功就有可能获取到锁(不用去关心队列，即发生抢占)
 */
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

**doAcquireSharedInterruptibly**

```java
/**
 * 该方法是核心方法，整个入队出队操作都包含其中
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
        // 入队
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
                       // 前一个节点
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
                       // parkAndCheckInterrupt如果是被打断(interrupt)，则抛出异常
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

该方法的执行流程如图所示：

![img](file:////Users/zhusidao/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image001.png)

以上就是所以的许可获取调用了，相关没有给解释的方法调用，可以去参考[ReentrantReadWriteLock](http://pms.ipo.com/pages/viewpage.action?pageId=141580618)。

 

#### 下面是许可**释放**

```
public void release() {
    sync.releaseShared(1);
} 
```

**releaseShared**

```
/**
 * 释放成功执行doReleaseShared()
 */
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
} 
```

**tryReleaseShared**

```
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
               // 获取当前许可数
        int current = getState();
               // 当前许可和释放许相加
        int next = current + releases;
               // 这里有必要判断？？？
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
               // state更新成功返回true
        if (compareAndSetState(current, next))
            return true;
    }
}
```

 

以上就是所有的相关源码分析，不足之处还望指出，也望多多交流。

 