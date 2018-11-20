---
layout: post
title: "J.U.C之AQS队列同步器"
description: "J.U.C之AQS队列同步器"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2018/11/09/
---

# 1 简介
队列同步器AbstractQueuedSynchronizer（简称同步器），是用来构建同步组件（包括同步容器以及同步工具）的基本框架，也是java.concurrent.util包的实现核心。它使用了一个int类型的成员变量来表示同步状态（J.U.C同样提供了AbstractQueuedLongSynchronizer，它使用long类型的成员变量来表示同步状态），通过内置的的FIFO队列来完成资源的获取线程的排队工作。
同步器的主要使用方式是继承，子类通过继承同步器并实现抽象方法来管理同步状态（基于模本方法模式）。子类被推荐定义为自定义同步组件的静态内部类（作为代理配合实现同步组件），同步器自身没有实现任何同步接口，它仅仅定义了同步状态的获取与释放来供自定义同步组件使用，同步器既可以独占式的获取同步状态也可以共享式的获取同步状态。
        

# 2 队列同步器的接口
同步器的设计是基于模板方法模式的，子类需要继承同步器并重写指定的方法，而后将同步器组合在自定义同步组件的实现中（委托模式），并调用同步器的模板方法，这些模板方法将会调用同步器子类中重写的方法
        
## 2.1 访问、修改同步状态
- int getState()：获取当前同步状态
- void setState(int newState)：设置当前同步状态
- boolean compareAndSetState(int expect, int update)：使用CAS设置当前同步状态

## 2.2 同步器可重写方法

**独占式**
- boolean tryAcquire(int arg)：独占式获取同步状态，实现该方法需要查询当前同步状态并判断是否符合预期，然后同步CAS设置同步状态。
- boolean tryRelease(int arg)：独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态。

**共享式**
- int tryAcquireShared(int arg)：共享式获取同步状态，返回大于等于0表示获取成功，反之获取失败。
- boolean tryReleaseShared(int arg)：共享式释放同步状态。
- boolean isHeldExclusively()：当前同步器是否被当前线程独占

## 2.3 同步器提供的模板方法

**独占式**
- void acquire(int arg)：独占式获取同步状态，如果当前线程获取同步状态成功，则返回，否则将会进入同步队列等待，该方法将会调用重写的tryAcquire(int arg)方法
- void acquireInterruptibly(int arg)：与acquire(int arg)方法相同，但是该方法响应中断，当前线程未获取到同步状态而进入到同步队列中，如果当前线程被中断，则抛出InterruptedException并返回。
- boolean tryAcquireNanos(int arg, long nanosTimeout)：在acquireInterruptibly(int arg)基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么将返回false，如果获取到了则返回true。
- boolean release(int arg)：独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒。

**共享式**
- void acquireShared(int arg)：共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的获取主要区别在于同一时刻能够有多个线程获取到同步状态。
- void acquireSharedInterruptibly(int arg)：与acquireShared(int arg)方法相同，该方法响应中断。
- boolean tryAcquireSharedNanos(int arg, long nanosTimeout)：在acquireSharedInterruptibly(int arg)方法的基础上增加了超时限制。
- boolean releaseShared(int arg)：共享式的释放同步状态。

**查询同步队列中等待线程情况**
- Thread getFirstQueuedThread()：获取在同步队列中等待的第一个线程。
- boolean isQueued(Thread thread)：判断该线程是否处于同步队列中。
- boolean apparentlyFirstQueuedIsExclusive()：判断同步队列中的第一个线程是否在等待独占式获取同步状态
- final boolean hasQueuedPredecessors()：同步队列中，当前线程对应的节点是否有前驱节点。
- int getQueueLength()：获取同步队列长度。
- Collection<Thread> getQueuedThreads()：获取等待在同步队列上的线程集合。
- Collection<Thread> getExclusiveQueuedThreads()：获取等待在同步队列上等待独占式获取同步状态的线程集合。
- Collection<Thread> getSharedQueuedThreads()：获取等待在同步队列上等待共享式获取同步状态的线程集合。

# 3 队列同步器的实现分析

## 3.1 同步队列
同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成一个节点（Node），并将其加入到同步队列中，同时会阻塞当前线程，当同步状态释放是，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

### 3.1.1 节点
同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点。

```
static final class Node {
    static final Node SHARED = new Node();//标识共享式节点，对应的线程等待共享式获取同步状态
    static final Node EXCLUSIVE = null;//标识独占式节点，对应的线程等待独占式获取同步状态

    /** waitStatus value to indicate thread has cancelled */
    /** 由于在同步队列中等待的线程等待超时或被中断，需要从队列中取消等待，节点进入该状态后将不会变化 */
    static final int CANCELLED =  1;
    
    /** waitStatus value to indicate successor's thread needs unparking */
    /** 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行 */
    static final int SIGNAL    = -1;
    
    /** waitStatus value to indicate thread is waiting on condition */
    /** 节点在等待队列中，节点的线程在等待condition上，当其他线程对Condition调用了signal()方法后，该节点将会从等待队列中转移到同步队列中，加入到对同步状态的获取中 */
    static final int CONDITION = -2;
    
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
     /** 表示下一次共享式同步状态获取将会无条件地被传播下去 */
    static final int PROPAGATE = -3;
    
    volatile int waitStatus;//对应上述多种状态值，其中0表示初始值
    
    volatile Node prev;//前驱节点

    volatile Node next;//后继节点

    volatile Thread thread;//获取同步状态的线程

    Node nextWaiter;//等待队列中的后继节点。在同步队列中，该字段用来标识当前节点的类型（共享or独占）

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

关于节点的状态state变化：当节点被创建，状态值默认为0；当有后继节点在等待，节点状态会被置为-1（SINGAL）；当节点共享式获取到同步状态，且无后继节点等待，节点状态会被设置为-3（PROPAGATE）；当节点获取同步状态超时或者被中断，节点状态将被置为1（CANCELLED）。

主意：Node节点不仅用在同步队列中，AbstractQueuedSynchronizer中的Condition中的等待队列也是使用Node节点构建，所以其中有些字段在两种队列的共用，需要注意区分，Condition将在后文阐述。

### 3.1.2 队列
节点是构成同步队列的基础，同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的尾部。

- 同步队列的基本结构
同步器包含了两个节点类型的引用，一个指向首节点，一个指向尾节点。

<center><img src="/assets/images/post/2018-11-09-java-concurrent-util-AQS/queue.jpg" width="500px"/></center>

- 节点加入到同步队列中
当一个线程成功的获取了同步状态，其他线程无法获取到同步状态，需要被构造成节点并加入到同步队列中，而加入这个队列的过程必须保证是线程安全的，因此同步器提供了一个基于CAS设置尾节点的方法：compareAndSetTail(Node expect, Node update)。

<center><img src="/assets/images/post/2018-11-09-java-concurrent-util-AQS/setHead.jpg" width="500px"/></center>

- 首节点的设置
同步队列遵循FIFO，首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，会唤醒后继节点，而后继节点将会在获取同步状态成功的同时将自己设置为首节点，由于只有一个线程能获取到同步状态，因此头结点的设置不需要CAS来保证。

<center><img src="/assets/images/post/2018-11-09-java-concurrent-util-AQS/setTail.jpg" width="500px"/></center>


## 3.2 独占式同步状态的获取与释放

### 3.2.1 acquire方法

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
上述代码主要完成了同步状态获取、节点构造、加入同步队列以及再同步队列中自旋等待等相关工作：首先调用自定义同步器实现的tryAcquire(int arg)方法，该方法保证线程安全地获取同步状态，如果同步状态获取失败，则构造独占式同步节点，并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node, int arg)方法，使得该节点以“死循环”的方式获取同步状态（这里我们也叫做自旋）。如果获取不到则阻塞当前线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。
        
```
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

节点进入同步队列之后，就进入了一个自旋的过程，每个节点都在观察自己，当条件满足且获取到了同步状态之后，就从这个自旋状态退出。
        
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();//获取前驱节点
            if (p == head && tryAcquire(arg)) {//前驱节点是首节点时 尝试获取同步状态
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

/** 当获取同步状态失败后 是否需要阻塞当前线程：
当前驱节点等待状态是SIGNAL时，返回true */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {//处理等待状态为删除的节点
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

/** 阻塞当前线程并检查是否中断 */
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//阻塞当前线程
    return Thread.interrupted();
}

/** 取消当前线程获取同步状态的操作：
1.如果需要取消的节点是尾节点，将尾节点指向前驱节点
2.如果需要取消的节点属于中间节点（前驱节点不是首节点），将前驱节点的next指向后继节点（越过当前节点）
3.上述两种case是为了订正next指针，通过后续唤醒后继节点，在shouldParkAfterFailedAcquire方法中订正pre指针
*/
private void cancelAcquire(Node node) {
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                        (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);//唤醒后继节点
        }

        node.next = node; // help GC
    }
}

/** 唤醒后继节点 */
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

头节点是成功获取同步状态的节点，而头节点释放同步状态后，将会唤醒后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否为首节点，这样也保证了队列的“先进先出”原则

### 3.2.2 acquireInterruptibly方法
```
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
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
acquireInterruptibly方法与acquire方法基本一致，只是在检查到中断后抛出InterruptedException。

### 3.2.3 tryAcquireNanos方法
```
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
}

private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
tryAcquireNanos方法方法与acquireInterruptibly方法基本一致，只是增加了超时失败的机制。

### 3.2.4 release方法
```
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
主要是调用unparkSuccessor方法唤醒阻塞中的后继节点，唤醒后的后继节点会开始尝试获取同步状态。

## 3.3 共享式同步状态的获取与释放

### 3.3.1 acquireShared方法
```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);//1.新增SHARED类型的节点
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);//2.设置头节点并唤醒后继节点(因为共享式获取同步状态的特点，如果后继节点也是共享式节点，允许后继节点直接获取同步状态)
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
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

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

    //试图通知首节点的后继节点
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);//唤醒后继节点
            }
            else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
acquireShared方法与acquire方法的区别主要在于当获取到同步状态之后，除了将自身设置为首节点之外，还需要试图告知后继节点，允许共享式获取同步状态。（这里的doReleaseShared试图唤醒共享式节点）

### 3.3.2 acquireSharedInterruptibly方法
```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
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
acquireSharedInterruptibly方法与acquireShared方法基本一致，只是在检查到中断后抛出InterruptedException。

### 3.3.3 tryAcquireSharedNanos方法
```
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
}

private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

tryAcquireSharedNanos方法与acquireSharedInterruptibly方法基本一致，只是增加了超时失败的机制。

### 3.3.4 releaseShared方法
```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

同样releaseShared方法唤醒后继节点（这里doReleaseShared试图唤醒共享节点以及独占节点）

## 4 Condition


