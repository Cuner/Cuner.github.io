---
layout: post
title: "J.U.C之锁"
description: "J.U.C之锁"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2018/12/27/
---

# 1 Lock接口
锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源（但有些锁支持多个线程同时访问共享资源，如读写锁）。在Lock接口出现之前，java程序是靠synchronized关键字实现锁功能的。在java SE5之后，并发包中新增了Lock接口（以及相关实现类）用来实现锁功能，它提供了与synchronized关键字类似的同步功能，只是在使用的时候需要显式的获取跟释放锁。虽然它缺少了隐式获取和释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、非阻塞获取锁、可中断获取锁以及超时获取锁等多种特性。

## 1.1 Lock接口的API
- **void lock()**  
获取锁，调用该方法当前线程将会获取锁，当获取锁后，从该方法返回
- **void lockInterruptibly() throws InterruptedException**  
可中断的获取锁，和lock()方法不同在于该方法可以响应中断，即在锁获取中可中断当前线程
- **boolean tryLock()**  
尝试非阻塞获取锁，调用该方法后立刻返回，如果能获取到锁则返回true，否则返回false
- **boolean tryLock(long time, TimeUnit unit) throws InterruptedException**  
超时获取锁，当前线程在以下三种情况下返回
  - 当前线程获取到了锁，返回true
  - 当前线程在超时时间内被中断，抛出InterruptedException异常
  - 获取超时时间结束，返回false
- **void unlock()**  
释放锁
- **Condition newCondition()**  
获取等待通知组件，该组件和当前的锁绑定，当前线程只有获取了锁，才能调用该组件的await方法，而调用后，当前线程将释放锁

# 2 重入锁ReentrantLock
顾名思义，就是支持重进入的锁，它表示该锁支持一个线程对资源重复加锁。除此之外，该锁还支持获取锁时的公平与非公平性选择。

## 2.1 使用示例

```
Lock lock = new ReentrantLock();
lock.lock();
try {
    //相关操作
} finally {
    lock.unlock();
}
```

## 2.2 实现重进入
重进入是指任意线程在获取锁之后能够再次获取该锁而不被阻塞，该特性的实现需要解决以下两个问题
- 同一线程再次获取锁：锁需要识别获取锁的线程是够为当前占据锁的线程，如果是，则再次获取成功。
- 锁的最终释放：线程重复n次获取了锁，随后在第n次释放锁之后，其他线程才能获取到锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数为0时表示锁已经释放

ReentrantLock是通过委托给继承队列同步器的静态内部类，来实现（非）公平锁的获取与释放，以非公平性（默认）的实现为例，获取以及释放同步状态的代码如下：

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
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

nonfairTryAcquire()方法：增加了线程再次获取同步状态的处理逻辑：通过判断当前线程是否是获取锁的线程来决定获取同步状态是否成功，如果是获取锁的线程再次请求获取锁，则将同步状态值增加并返回true，表示获取同步状态成功。其中同步状态值即为线程重复获取锁的次数。（这里自增的操作因为是单线程，所以不需要考虑同步的问题）。    
tryRelease()方法：每次释放锁则将同步状态值自减，只有当同步状态完全释放了，才能返回true。可以看到该方法将同步状态值是否为0作为最终释放条件，当同步状态值为0，将占有线程置为null，并返回true，表示释放成功。（这里释放过程同样也不会产生线程竞争）


## 2.3 公平性与非公平性锁
如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的。公平的获取锁，也就是等待时间最长的线程最先获取锁。事实上公平性锁机制往往没有非公平的效率高，但是公平性锁的优势在于能够减少“饥饿”发生的概率。

在ReentrantLock中，根据公平锁以及非公平锁获取获取锁顺序的区别，分别实现了两个静态内部类（继承AbstractQueuedSynchronized），并把锁的获取委托给它们。


```
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
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

/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
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
}
```

对于非公平锁，线程请求锁的时候直接尝试获取同步状态，只要CAS设置同步状态成功，那么当前线程则获取到了锁；而公平锁不同，与nonfairTryAcquire()方法相比较，唯一不同的地方是判断条件多了hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更先请求锁，因此不能直接尝试获取锁，需要构造同步节点加入到同步队列中，等待前驱节点获取并释放锁后才能去尝试获取同步状态。

# 3 读写锁

读写锁维护了一对锁，一个读锁和一个写锁，分别对应共享锁和排它锁。简单来说，当某个线程获取到了写锁，其他线程无法再去获取写锁或者读锁；当某个线程获取到了读锁，其他线程无法再去获取写锁，允许再去获取读锁。读写锁通过读写分离，除了保证写操作对读操作的可见性之外，在读多于写的情况下，还能提供更好的并发性和吞吐量。

## 3.1 读写锁的特性

- 公平性选择
- 重进入
- **锁降级**

前两点特性与ReentrantLock一致，而锁降级特性是遵循获取写锁、获取读锁在释放写锁的次序，写锁能够降级成为读锁。

## 3.2 读写锁的示例

```
public class Cache {
    private static Map<String, Object> map = new HashMap<>();
    private static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private static Lock readLock = readWriteLock.readLock();
    private static Lock writeLock = readWriteLock.writeLock();

    //获取map中某个key对应的value
    public static Object get(String key) {
        readLock.lock();
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }

    //设置某个key对应的value，并返回旧的value
    public static void set(String key, Object value) {
        writeLock.lock();
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

上述示例中，Cache组合了一个非线程安全的HashMap作为缓存的实现，同时使用读写锁来保证Cache是线程安全的。在读操作中，需要获取读锁，这使得并发读取时不会相互阻塞；写操作中需要获取写锁，获取到写锁后，其他线程对于读锁或者写锁的获取均被阻塞。Cache使用读写锁保证了读操作的并发性，也保证了每次写操作对所有的读写操作的可见性。


## 3.3 读写锁实现分析

### 3.3.1 读写锁的实现结构
- ReentrantReadWriteLock对外封装了获取读锁与写锁的方法，这里的读锁与写锁都实现了锁接口Lock

```
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
    
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

- ReentrantReadWrite通过委托给继承队列同步器AQS的静态内部类（FairSync/NonfairSync），来实现（非）公平锁的获取与释放

```
abstract static class Sync extends AbstractQueuedSynchronizer {...}
static final class NonfairSync extends Sync {...}
static final class FairSync extends Sync {...}
```

- 写锁的获取与释放，委托给Sync类独占式的获取与释放同步状态
- 读锁的获取与释放，委托给Sync类共享式的获取与释放同步状态

### 3.3.2 读写状态的设计

读写锁同样依赖自定义同步器来实现同步功能，而读写状态就是同步器的同步状态。因此同步状态值需要区分读写锁的获取，同时也要考虑都线程重入的情况。   
读写锁将同步状态值分成了两个部分，高16位表示读，低16位表示写。于是写状态等于S & ((1 &lt;&lt;&lt; 16) - 1)，即将高16为全部抹去，读状态等于S &gt;&gt;&gt; 16（无符号补0右移16位）

```
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

### 3.3.3 写锁的获取与释放
写锁是一个支持重进入的排他锁。如果当前线程已经获取了死锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（代码中的 c != 0 &amp;&amp; w == 0）或者该线程不是已经获取写锁的线程，则获取同步状态失败，当前线程进入到等待状态。

```
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

写锁的释放与ReentrantLock释放锁的过程基本一致，每次释放减少同步状态，当写状态为0时则表示写锁被释放。

### 3.3.4 读锁的获取与释放

读锁是一个支持重进入的共享搜，它能够被多个线程同时获取，在没有其他写线程访问时，读锁总能被成功获取，此时需要线程安全的增加读状态。如果当前线程在获取读锁时，写锁已经被其他线程获取，则该线程进入等待状态。这里读锁的获取释放代码做了部分删减（删减掉了每个线程维护各自获取读锁次数的相关代码）。

在tryReleaseShared()方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS）增加读状态，成功获取锁。

可以看到这里有个fullTryAcquireShared()方法，其中的for循环主要应对多个线程同时获取共享式同步状态导致CAS失败的重试。

```
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        return 1;
    }
    return fullTryAcquireShared(current);
}

final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
        } else if (readerShouldBlock()) {
            return -1;
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            return 1;
        }
    }
}

protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```



### 3.3.5 公平性与非公平性的选择

```
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();//队列中第一个等待的节点是否为独占式
    }
}

static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();//是否有前驱节点
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}

```

### 3.3.6 锁降级

锁降级是指写锁降级成读锁。如果当前线程拥有写锁，然后将其释放，最后在获取读锁，这种分段过程不能称之为锁降级。锁降级是指把持住当前拥有的写锁，再获取到读锁，随后释放先前拥有的写锁的过程。

```
private void process() {
    readLock.lock();
    if (update) {
        //需要写入，此时释放掉读锁
        readLock.unlock();
        //锁降级过程从写锁获取开始
        writeLock.lock();
        try {
            if (update) {
                //准备数据写入
                update = false;
            }
            readLock.lock();
        } finally {
            writeLock.unlock();
        }
        //锁降级完成，写锁降级为读锁
    }
    try {
        //读取数据
    } finally {
        readLock.unlock();
    }
}
```

当数据发生变更后，update（布尔类型且被volatile修饰）变量被置为true，此时所有访问process()方法的线程都将感知到变更，但只有一个线程能够获取到写锁，其他线程会被阻塞在读锁或者写锁的lock()方法上。当前线程获取写锁完成数据准备之后，在获取读锁，随后释放写锁，完成锁降级。

锁降级过程中再次获取读锁是为了保证数据的可见性，避免在次过程中，其他线程的写入操作使得当前线程无法感知

在前面获取锁的代码中可以看到，在获取到读锁的时候，如果当前线程已经获取到写锁，这个时候读锁是会被成功获取的；而获取写锁的时候，当前线程获取到了读锁，这个时候线程将被阻塞，处于等待状态。可以得知ReenTrantReadWriteLock支持锁降级，而不支持锁升级（把持读锁，获取写锁，释放读锁的过程）。目的也是为了保障可见性，如果读锁已经被多个线程获取，其中任意线程获取了写锁并更新了数据，则器更新的数据对其他获取读锁的线程是不可见的。

# 4 Condition接口

我我们知道，任何一个Java都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()、notifyAll()方法，这些方法与synchronized同步关键字配合使用，可以实现等待通知模式。Condition接口也提供了类型Object监视器方法，与Lock配合可以实现等待通知模式。但是两者在使用方式以及功能特性上还是有一些区别的。

| 对比项  | Object Monitor Methods | Condition |
| --- | --- | --- |
| 前置条件 | 获取对象锁 | 调用Lock.lock()获取锁，调用lock.newCondition()获取Condition对象 |
| 等待队列个数 | 一个 | 多个 |
| 当前线程释放锁进入等待状态 | 支持 | 支持 |
| 当前线程释放锁进入等待状态，在等待状态中不响应中断 | 不支持 | 支持 |
| 当前线程释放锁进入超时等待状态 | 支持 | 支持 |
| 当前线程释放锁进入等待状态到将来的某个时间点 | 不支持 | 支持 |
| 唤醒等待队列中的一个线程 | 支持 | 支持 |
| 唤醒等待队列中的全部线程 | 支持 | 支持 |

实现Condition接口的ConditionObject类是在AbstractQueuedSynchronized中定义的，但是ConditionObject的使用都是与Lock绑定使用的，因此放到这里来讲。

## 4.1 Condition接口

- **void await() throws InterruptedException**  
当前线程进入等待状态，直到被通知或者中断。如果当前线程从await()方法返回，那么表明该线程已经获取了Condition对象所对应的锁。
- **void awaitUninterruptibly()**  
当前线程进入等待状态直到被通知，该方法对中断不敏感
- **long awaitNanos(long nanosTimeout) throws InterruptedException**     
当前线程进入等待状态，直到被通知、中断或者超时。返回值表示剩余的时间，如果返回值小于等于0，表示等待超时
- **boolean awaitUntil(Date deadline) throws InterruptedException**   
当前线程进入等待状态，直到被通知、中断或者到某个时间点。如果没有到指定时间就被通知，方法返回true，否则表示已经到了指定时间还未被通知，此时返回false。
- **void signal()**    
唤醒一个等待在Condition上的线程，该线程从等待时间返回前必须获得与Condition相关联的锁
- **void signalAll()**   
唤醒所有等待在Condition上的线程，能够从等待方法返回的线程必须获取与Condition相关联的锁

## 4.2 Condition实例

Condition定义了等待通知两种类型的方法，当线程调用终这些方法时，需要提前获取到Condition对象关联的锁。

```
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void conditionWait() throws InterruptedException {
    lock.lock();
    try {
        condition.await();
    } finally {
        lock.unlock();
    }
}

public void conditionSignl() {
    lock.lock();
    try {
        condition.signal();
    } finally {
        lock.unlock();
    }
}
```

当调用await()方法后，当前线程会释放锁并在此等待，而其他线程调用Condition对象的signal()方法后，通知当前线程后，当前线程从await()方法返回，并且在返回前已经获取到了锁。

## 4.3 Condition实现分析

### 4.3.1 等待队列
ConditionObject是同步器AbstractQueuedSynchronizer的内部类，每个Condition对象都包含一个等待队列，该队列是Conditon实现等待通知功能的关键。

等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造等待节点加入等待队列并进入等待状态。这里的节点复用了同步器中的节点，也就是说，同步队列以及等待队列中的节点类型都是同步器的静态内部类AbstractQueuedSynchronizer.Node。

一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点（lastWaiter）。在Object监视器上，一个对象拥有一个同步队列和等待队列，而并发包中的Lock拥有一个同步队列和多个等待队列。

<center><img src="/assets/images/post/2018-12-27-java-concurrent-util-lock/waitQueue.jpg" width="500px"/></center>


## 4.3.2 等待

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 当前线程加入等待队列
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 阻塞线程
    while (!isOnSyncQueue(node)) {
        // 解除阻塞
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 自旋尝试获取同步状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
该方法的线程成功获取到了锁，也就是同步队列中的首节点。该方法将会以当前线程构造新的等待节点（addConditionWaiter()方法），并将节点从尾部加入等待队列，同时释放锁，唤醒同步队列中的后继节点，最后当前线程会进入等待状态。Condition拥有首尾节点的引用，而新增节点只需要将原有的尾节点lastWaiter指向它，并且更新尾节点即可。上述节点引用更新的过程并没有使用CAS保证，原因在于调用await()方法的线程必定是获取了锁的线程，也就是说该过程是线程安全的。

当等待队列中的节点被唤醒，则唤醒的线程开始尝试获取同步状态，如果等待线程被中断会抛出InterruptedException。

<center><img src="/assets/images/post/2018-12-27-java-concurrent-util-lock/addToWaitQueue.jpg" width="500px"/></center>

## 4.3.3 通知

调用Condition中的signal()方法，将会唤醒等待队列中的等待时间最长的节点（也就是首节点），在唤醒节点之前，会将节点移动到同步队列中。

```
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
            (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
    // 修改节点状态，恢复到默认状态
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 加入到同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        // 唤醒等待状态
        LockSupport.unpark(node.thread);
    return true;
}
```

调用该方法的前置条件是当前线程必须获取到了锁，可以看到signal()方法进行了isHeldExclusively()检查，也就是当前线程必须是获取了锁的线程。接着获取等待队列中的首节点，将其移动到同步队列并用LockSupport唤醒节点中的线程。

通过调用同步器的enq()方法，等待队列中的首节点线程安全的加入到同步队列。当节点移动到同步队列之后，当前线程再唤醒等待线程。

被唤醒后的线程，将从await()方法中的while循环中退出（节点已经在同步队列中，isOnSyncQueue()方法返回true），进而调用同步器的acquireQueued()方法加入到获取同步状态的竞争中。

成功获取同步状态之后，被唤醒的线程将先从先前调用的await()方法中返回，此时该线程已经成功地获取了锁。

Condition的signalAll()方法，相当于对等待队列中的每一个节点均进行一次signal()方法，效果就是将等待队列中的所有节点移动到同步队列中，并唤醒每个节点的线程

<center><img src="/assets/images/post/2018-12-27-java-concurrent-util-lock/backToSynQueue.jpg" width="500px"/></center>

