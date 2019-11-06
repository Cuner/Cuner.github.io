---
layout: post
title: "J.U.C之CountDownLatch"
description: "J.U.C之CountDownLatch"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2019/11/6/
---
（读此篇前建议完成《J.U.C之AbstractQueuedSynchronizer》的阅读）

# 1 CountDownLatch介绍

CountDownLatch（我们也称之为闭锁），是一种同步的工具类。类似于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭着的，不允许任何线程通过，当到达结束状态时，这扇门会打开并允许所有的线程通过。且当门打开了，就永远保持打开状态

# 2 CountDownLatch的使用场景

一般在需要等待其他多个指定任务执行完成的场景下使用，这里的多个任务可以分散在不同的线程中执行。最常见的场景：主线程需要等待其他子线程执行完成后才能继续执行，而一般上述的场景我们能在主线程中使用thread.join()来实现，相比来说CountDownLatch也能解决该问题，同时它更具有灵活性，它允许一个或多个线程等待其他线程完成某个特殊的操作

# 3 CountDownLatch的使用方式

## 3.1 初始化

```
CountDownLatch latch = new CountDownLatch(3);
```

CountDownLatch的构造函数接受一个int类型的参数作为计数器，如果想等待N个操作执行完成打开闭锁，这里就传入N（这里的N必须大于等于0，等于0的时候闭锁状态打开）。

## 3.2 标记某个被等待的操作执行完成

```
latch.countDown();
```

在某个需要被等待的操作执行完成之后，执行这行代码，标识操作完成，当我们调用countDown()方法时，构造时候传入的N就会减1，由于countDown()方法可以使用在任何地方，所以这里被等待的N个操作，可以分散在N个不同的线程中，也可以是一个线程中的N个步骤。用在多线程中，只需要把这个CountDownLatch的引用传递到线程中去即可

## 3.3 等待N个操作执行完成

```
latch.await();
```

CountDownLatch的await()方法会阻塞当前线程，直到构造时候传入的N减少至0。另外await()方法支持传入指定时间，这个方法等待指定时间后，就不会阻塞当前线程

## 3.4 注意

CountDownLatch不可能重新初始化或者修改内部计数N的值。

# 4 实现原理

```
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}

private final Sync sync;

public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

```

闭锁CountDownLatch是J.U.C中的同步工具类，采用委托模式委托给了实现AbstractQueuedSynchronizer的同步器子类Sync，且采用了共享式同步状态的获取与释放（因为这里会存在多个线程同时调用await方法，在等待闭锁打开后获取同步状态）。首先使用CountDownLatch初始化传入的N来设置同步器Sync的同步状态state为N，仅当同步状态减少到0的时候才允许共享式获取同步状态，而当释放同步状态时，将state的值线程安全的递减1（因为存在多个等待的操作同时完成，这里用到了compareAndSetState方法保证线程安全）

```

public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public void countDown() {
    sync.releaseShared(1);
}

public long getCount() {
    return sync.getCount();
}
```
- CountDownLatch把countDown()方法委托给了同步器Sync的releaseShared()方法--共享式的释放同步状态方法
- CountDownLatch把await()方法委托给了同步器Sync的acquireSharedInterruptibly()方法--共享式可中断的获取同步状态方法
- CountDownLatch把await(long timeout, TimeUnit unit)方法委托给了同步器Sync的tryAcquireSharedNanos()方法--共享式超时中断的获取同步状态

使用CountDownLatch，在await()的过程中，线程会被挂起，相比线程自主轮训判断更节省CPU资源