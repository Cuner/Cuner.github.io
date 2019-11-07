---
layout: post
title: "J.U.C之CyclicBarrier"
description: "J.U.C之CyclicBarrier"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2019/11/08/
---
（读此篇前建议完成《J.U.C之AbstractQueuedSynchronizer》的阅读）

# 1 CyclicBarrier介绍

CyclicBarrier（我们也称之为同步屏障），是一种同步的工具类。让一组线程在达到各个线程中设置的屏障（同步点）的时候阻塞，直到最后一个线程达到屏障时，屏障才会打开，所有被屏障阻塞的线程才会继续运行。

# 2 CyclicBarrier的使用场景

一般在一组线程需要相互等待，等待全部线程达到各自的屏障后，线程组中的线程才能继续运行的场景下使用。CyclicBarrier还提供了额外的功能，能够让一组线程全部达到屏障时，***让最后一个到达同步点的线程执行指定的任务。***

# 3 CyclicBarrier的使用方式

## 3.1 初始化

```
CyclicBarrier barrier = new CyclicBarrier(3);

CyclicBarrier barrier = new CyclicBarrier(3, new Runnable() {
    @Override
    public void run() {
        System.out.println("hello world");
    }
});
```

CyclicBarrier提供一个默认的构造函数，接受一个int类型的参数作为计数器，如果想设置N个屏障点（对应屏障拦截的N个线程），这里就传入N ***（这里的N必须大于等于0，等于0的时候屏障状态打开）***。

CyclicBarrier还提供一个更高级的构造函数，CyclicBarrier(int parties, Runnable barrierAction)，用于在所有线程达到屏障后，由最后一个到达屏障的线程执行barrierAction，在此之后，所有线程才能继续运行



## 3.2 线程标记达到屏障并等待其他线程达到屏障

```
barrier.await();
```

在每个需要被屏障拦截的线程中执行，标记此处为该线程的屏障，且在其他需要被屏障拦截的线程达到各自的屏障之前被阻塞，由于执行await()之后，线程会被阻塞，所以这里的N个屏障只能分散在N个线程（也就是由N个线程分别执行await方法），在多线程中，需要把这个CyclicBarrier的引用传递到线程中去即可


# 4 CyclicBarrier与CountDownLatch的区别

# 5 实现原理
