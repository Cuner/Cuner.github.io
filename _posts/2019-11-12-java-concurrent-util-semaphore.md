---
layout: post
title: "J.U.C之Semaphore"
description: "J.U.C之Semaphore"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2019/11/12/
---
（读此篇前建议完成《J.U.C之AbstractQueuedSynchronizer》的阅读）

# 1 Semaphore介绍

Semaphore（我们也称之为信号量），是用来控制同时访问特定资源的线程数量，他通过协调各个线程，以保证合理的使用公共资源。Semaphore维护了一个许可集，也就是一定数量的“许可证”。线程在访问公共资源前需要先获取到许可证，许可证数量有限，只有获取到许可证的线程才会被允许继续执行，当剩余可用许可证不足后，尝试获取许可证的线程会被阻塞，等待其他线程释放许可证后再尝试获取许可证。

# 2 Semaphore的使用场景

Semaphore可以用来做流量控制，一般适用于公共资源有限的场景（比如数据库连接、单机服务限流）

# 3 Semaphore的使用方式

## 3.1 初始化

```
Semaphore semaphore = new Semaphore(10);

Semaphore semaphore = new Semaphore(10, true);
```

Semaphore提供一个接受一个int类型的参数作为计数器的构造函数，如果想限制可用许可证的上限为N，这里就传入N。

Semaphore还提供另外一个构造函数，Semaphore(int permits, boolean fair)，除了传入一个计数值以外，还接受一个boolean传参fair，这个参数决定线程是公平式还是非公平式地获取许可证，默认是非公平。（这里的公平与非公平的获取资源与ReentrantLock公平与非公平的获取锁类似，不再赘述） 



## 3.2 线程获取许可证

```
semaphore.acquire();

semaphore.acquireUninterruptibly();

semaphore.acquire(2);
```

- 当线程尝试获取许可证时，执行acquire()方法。当线程获取到许可证后，方法立即返回，线程继续执行；而当线程未能获取到许可证，线程会被阻塞，等待许可证被释放后重试获取。
- acquire()方法是对线程中断敏感的，一旦线程在等待获取许可证的过程中被中断，acquire()会抛出中断异常，而如果想要不可中断的获取许可证，即在线程阻塞过程中不会被中断，可使用acquireUninterruptibly()方法来获取许可证
- 如果想要同时获取多个许可证，可在acquire()方法中传入一个int类型的参数，标识你想要获取的许可证数量

## 3.3 实战说明


```
public class Test {
    private static Semaphore semaphore = new Semaphore(10);
    private static Executor executor = Executors.newFixedThreadPool(30);
    public static void main(String[] args) {
        for (int i = 0 ; i < 10 ; i++) {
            executor.execute(() -> {
                try {
                    semaphore.acquire();
                    System.out.println("acquire");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            });
        }
    }
}
```


# 4 实现原理

与寻常并发工具类类似，Semaphore也是通过继承至AbstractQueuedSynchronized的内部类来实现其功能，基本结构几乎和ReentrantLock完全一样，通过内部类分别实现了公平/非公平策略。

```
private final Sync sync;

abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    Sync(int permits) {
        setState(permits);
    }

    final int getPermits() {
        return getState();
    }

    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }

    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}

/**
 * NonFair version
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}

/**
 * Fair version
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}

```

我们可以看到，Semaphore通过两个继承AbstractQueuedSynchronizer的内部类来分别实现公平/非公平地获取同步状态，并且通过实现tryAcquireShared()以及tryReleaseShared()来是实现***共享式***地获取同步状态（采用共享式的原因是因为存在多个线程同时获取到同步状态的情况）。
- 通过构造器传入的许可证最大可用数来初始化同步状态state，这里的state就代表着可用的许可证数量
- tryAcquireShared()：通过将同步状态state线程安全的扣减获取许可证的数量来实现共享式获取同步状态，而当可用的许可证不足的时候，获取同步状态失败，返回小于0的状态值，此时线程会被挂起。公平与非公平获取同步状态的区别在于，公平策略在有前驱节点等待获取同步状态的情况下，会默认获取同步状态失败，构造节点放置于等待队列尾部，这样会保证获取同步状态的线程按照顺利依次获取同步状态；而非公平策略则是当有新的线程获取同步状态时，会直接尝试获取同步状态，一定概率下会成功获取同步状态，非公平策略的性能一般优于公平策略，但是无法保证顺序。
- tryReleaseShared()：通过将同步状态state线程安全的增加释放许可证的数量来实现共享式释放同步状态（因为可能存在多个获取到许可证的线程释放许可证，这里使用compareAndSetState来保障线程安全的数值扣减）




```
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}


public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}


public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}


public void acquireUninterruptibly() {
    sync.acquireShared(1);
}


public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}


public boolean tryAcquire(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}


public void release() {
    sync.releaseShared(1);
}


public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}


public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits);
}


public boolean tryAcquire(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.nonfairTryAcquireShared(permits) >= 0;
}


public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
}


public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}


public int availablePermits() {
    return sync.getPermits();
}


public int drainPermits() {
    return sync.drainPermits();
}


protected void reducePermits(int reduction) {
    if (reduction < 0) throw new IllegalArgumentException();
    sync.reducePermits(reduction);
}


public boolean isFair() {
    return sync instanceof FairSync;
}


public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}


public final int getQueueLength() {
    return sync.getQueueLength();
}

protected Collection<Thread> getQueuedThreads() {
    return sync.getQueuedThreads();
}
```

Semaphore通过委托模式，将内部功能实现委托了内部类Sync，通过构造函数，传入许可证的数量来初始化同步状态state、传入boolean变量来决定使用公平/非公平策略，并主要提供一下几个方法
- acquire()：获取一个许可证，当有剩余许可证时返回，当剩余许可证不足时线程被阻塞，直到许可证被释放或者线程被中断
- acquireUninterruptibly()：获取一个许可证，当有剩余许可证时返回，当剩余许可证不足时线程被阻塞，直到许可证被释放
- tryAcquire()：尝试获取一个许可证，成功获取返回true，否则返回false
- tryAcquire(long timeout, TimeUnit unit)：获取一个许可证，当有剩余许可证时返回，当剩余许可证不足时线程被阻塞，直到许可证被释放或者等待获取超时
- release()：释放一个许可证
- acquire(int permits)：获取指定数量的许可证，当有足够剩余许可证时返回，当剩余许可证不足时线程被阻塞，直到许可证被释放或者线程被中断
- acquireUninterruptibly(int permits)：获取指定数量的许可证，当有足够剩余许可证时返回，当剩余许可证不足时线程被阻塞，直到许可证被释放
- tryAcquire(int permits)：尝试获取指定数量的许可证，成功获取返回true，否则返回false
- tryAcquire(int permits, long timeout, TimeUnit unit)：获取指定数量的许可证，当有足够剩余许可证时返回，当剩余许可证不足时线程被阻塞，直到许可证被释放或者等待获取超时
- release(int permits)：释放指定数量的许可证
- availablePermits()：查询当前可用的许可证数
- drainPermits()：将当前可用许可证数量设置为0
- reducePermits(int reduction)：将当前可用许可证减少指定数量
- hasQueuedThreads()：是否有线程正在等待许可证
- getQueueLength()：返回正在等待获取许可证的线程数