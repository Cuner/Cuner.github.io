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

ReentrantLock是通过委托实现队列同步器的静态内部类，来实现（非）公平锁的获取与释放，以非公平性（默认）的实现为例，获取以及释放同步状态的代码如下：
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


# 3 读写锁


