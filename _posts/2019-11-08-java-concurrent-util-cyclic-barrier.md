---
layout: post
title: "J.U.C之并发工具：CyclicBarrier"
description: "J.U.C之并发工具：CyclicBarrier"
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

## 3.3 实战说明

CyclicBarrier可以用于多线程计算数据，最后合并数据的场景，例如四个线程分别通过计算得到四个数值，最后求和得到最终结果

```
public class CountNumService implements Runnable {

    private CyclicBarrier barrier = new CyclicBarrier(4, this);

    private Executor executor = Executors.newFixedThreadPool(4);

    private ConcurrentHashMap<String, Integer> countMap = new ConcurrentHashMap<>();

    public void count() {
        for (int i = 0; i < 4; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    //执行计算过程
                    //存储计算结果
                    countMap.put(Thread.currentThread().getName(), 1);
                    try {
                        barrier.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }
    
    @Override
    public void run() {
        int result = 0;
        for (Integer num : countMap.values()) {
            result += num;
        }
        System.out.println("result:" + result);
    }

    public static void main(String[] args) {
        CountNumService countNumService = new CountNumService();
        countNumService.count();
    }
}
```

# 4 CyclicBarrier与CountDownLatch的区别

我们发现使用CountDownLatch也适用上述的案例：在每个子线程中完成计算任务后执行counDown()，在主线程中执行await()方法，并在此之后执行求和操作

```
public class CountNumService {

    private CountDownLatch latch = new CountDownLatch(4);

    private Executor executor = Executors.newFixedThreadPool(4);

    private ConcurrentHashMap<String, Integer> countMap = new ConcurrentHashMap<>();

    public void count() {
        for (int i = 0; i < 4; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    //执行计算过程
                    //存储计算结果
                    countMap.put(Thread.currentThread().getName(), 1);
                    latch.countDown();
                }
            });
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        int result = 0;
        for (Integer num : countMap.values()) {
            result += num;
        }
        System.out.println("result:" + result);
    }

    public static void main(String[] args) {
        CountNumService countNumService = new CountNumService();
        countNumService.count();
    }
}
```

的确如此，在一些等待其他线程执行任务的场景下，使用CountDownLatch与CyclicBarrier都能解决问题，但是两者还是有一定的区别：

- 使用CountDownLatch的时候，每个子线程标识完成任务是执行countDown()方法，在此之后线程并不会被阻塞，可以执行下去；而在使用CyclicBarrier的时候，每个子线程标识任务完成（达到屏障）是通过执行await()方法，在此之后线程被阻塞
- CountDownLatch不能重新初始化，所以只能被使用一次，而CyclicBarrier的计数器可以通过reset()方法重新初始化被循环使用。所以CyclicBarrier能够支持更加复杂的场景，例如发生计算错误的情况，可以重置计数器并让线程重新执行一次。
- CyclicBarrier还提供其他有用的方法，比如getNumberWaiting()可以获取CyclicBarrier阻塞的线程数量，isBroken()用来获取阻塞的线程是否被中断

# 5 实现原理

```
public class CyclicBarrier {

    //每个barrier实例都会对应一个Genetation实例，而当barrier迭代被循环使用或者重置的时候，Genetation实例会被重新初始化（主要标识是否被中断）
    private static class Generation {
        boolean broken = false;
    }

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    
    //循环使用时计数器默认大小
    private final int parties;
    
    //所有线程达到屏障后执行的任务
    private final Runnable barrierCommand;
    
    /** The current generation */
    private Generation generation = new Generation();

    //实时计数器，每当线程达到屏障时减一（parties-->0）
    private int count;
    
    
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }
    
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    //使用ReentrantLock的Condition的await()方法使得到达屏障的线程阻塞
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();//当最后一个线程到达屏障时，执行构造器传入的Runnable任务后打开屏障
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
    
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
    
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }

    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }

    public int getParties() {
        return parties;
    }
    
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }


    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}
```

- 与CountDownLatch不同，CyclicBarrier直接将内部实现委托给了ReentrantLock，在到达屏障点的时候执行Condition.await()方法阻塞当前线程，当最后一个线程达到屏障的时候执行Condition.notifyAll()方法唤醒所有线程。
- 当某个阻塞的线程中断或者超时后，唤醒所有线程，且所有线程都抛出相对应的异常
- 当最后一个线程到达屏障之后，会去执行构造是传入的可执行任务，然后计数器复位开始下一次循环迭代。
- reset()方法执行后，唤醒所有等待的子线程，然后计数器复位开始下一次循环迭代。