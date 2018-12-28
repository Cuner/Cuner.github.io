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

## 2.1 实现重进入
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


## 2.2 公平性与非公平性锁
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

## 3.2 读写锁实现分析

### 3.2.1 读写锁的实现结构
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

### 3.2.2 读写状态的设计

读写锁同样依赖自定义同步器来实现同步功能，而读写状态就是同步器的同步状态。因此同步状态值需要区分读写锁的获取，同时也要考虑都线程重入的情况。   
读写锁将同步状态值分成了两个部分，高16位表示读，低16位表示写。于是写状态等于S & ((1 << 16) - 1)，即将高16为全部抹去，读状态等于S >>> 16（无符号补0右移16位）

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


### 3.2.3 写锁的获取与释放
写锁是一个支持重进入的排他锁。如果当前线程已经获取了死锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（代码中的 c != 0 && w == 0）或者该线程不是已经获取写锁的线程，则获取同步状态失败，当前线程进入到等待状态。

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

写锁的释放与ReenTrantLock释放锁的过程基本一致，每次释放减少同步状态，当写状态为0时则表示写锁被释放。

### 3.2.4 读锁的获取与释放

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



### 3.2.5 公平性与非公平性的选择

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

### 3.2.6 锁降级

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


