---
title: 学习总结 -- Android 中的线程池基本介绍
date: 2017-05-09 20:15:43
category: 学习笔记
tags: 线程池

---

首先概括一下线程池的好处：

> - 重用线程池中的线程，避免因为线程的创建和销毁带来的性能开销；
> - 能有效控制线程池的最大并发数，避免大量的线程之间因互相抢占系统资源而导致的阻塞现象；
> - 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能。

Android 中的线程池来源于 Java 的 Executor，Executor 是个接口，真正的实现是 ThreadPoolExecutor。ThreadPoolExecutor 提供了一系列的参数来配置线程池，从而创建不同的线程池。

<!-- more -->

## ThreadPoolExecutor

ThreadPoolExecutor 是线程池的真正实现，也是线程池中最核心的一个类。它有四个构造方法，按照惯例直接看参数最多的那个构造方法：

```
public ThreadPoolExecutor (int corePoolSize,
                           int maximumPoolSize,
                           long keepAliveTime,
                           TimeUnit unit,
                           BlockingQueue<Runnable> workQueue,
                           ThreadFactory threadFactory,
                           RejectedExecutionHandler handler)
```

接下来解释一下各个参数的含义：

### corePoolSize

线程池中的核心线程数，默认情况下，核心线程会在线程池中一直存活，即使它们处于空闲状态；但是如果调用 ThreadPoolExecutor 的 allowCoreThreadTimeOut(boolean value) 方法并设置参数为 true，那么此时空闲的核心线程就有可能被终止。

创建线程池后，默认情况下，线程池中并没有任何线程，当有任务到来时才会创建线程去执行任务，除非调用了 prestartCoreThread() 或 prestartAllCoreThreads() 方法来预先创建线程。

### maximumPoolSize

线程池所能容纳的最大线程数，当活动线程数达到这个数值后，后续的新任务将会被阻塞。

### keepAliveTime

表示空闲的线程最多能存活的时间。默认情况下，当线程池中的线程数大于 corePoolSize 时，除核心线程外如果某个线程空闲的时间达到 keepAliveTime，该线程就会终止；但是如果调用了 allowCoreThreadTimeOut(boolean value) 方法，即使线程池中的线程数不大于 corePoolSize，keepAliveTime 参数也会起作用，此时任何空闲线程的空闲时间达到 keepAliveTime 都会被终止。

### unit

用于指定 keepAliveTime 的时间单位，是个枚举。有以下几种：

```
TimeUnit.NANOSECONDS;    // 纳秒
TimeUnit.MICROSECONDS;   // 微妙
TimeUnit.MILLISECONDS;   // 毫秒
TimeUnit.SECONDS;        // 秒
TimeUnit.MINUTES;        // 分钟
TimeUnit.HOURS;          // 小时
TimeUnit.DAYS;           // 天
```

### workQueue

线程池中的任务队列，只有通过线程池的 execute 方法提交的 Runnable 对象才会存储在这个参数中。常用的有以下几种：

- ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；
- LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；
- synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是直接新建一个线程来执行新来的任务。

### threadFactory

线程工厂，为线程池提供创建线程的功能。

### handler

当线程池无法执行新任务（由于超出线程范围或队列容量）时而执行的处理程序。有以下几种：

- ThreadPoolExecutor.AbortPolicy：丢弃任务并抛出RejectedExecutionException异常。
- ThreadPoolExecutor.DiscardPolicy：丢弃任务，但是不抛出异常。
- ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）。
- ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务。

ThreadPoolExecutor 的参数解释就到这了，ThreadPoolExecutor 执行任务时大致遵循如下规则：

[![img](http://oh0xbzubu.bkt.clouddn.com/image/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png)](http://oh0xbzubu.bkt.clouddn.com/image/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png)

在 Android 6.0 版本的 ThreadPoolExecutor 文档中有这么一段话：

> However, programmers are urged to use the more convenient `Executors` factory methods `newCachedThreadPool()` (unbounded thread pool, with automatic thread reclamation), `newFixedThreadPool(int)` (fixed size thread pool) and `newSingleThreadExecutor()` (single background thread), that preconfigure settings for the most common usage scenarios.

意思就是建议大家通过 Executors 的三个工厂方法来创建线程池，接下来分别看看这三个方法。

## newCachedThreadPool()

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

该方法会创建一个线程数量为 `Integer.MAX_VALUE` 的线程池，线程池中的每个线程空闲超过 60 秒都会被回收，并且当有新任务到来时立刻创建一个新线程来执行新任务。

> 对于执行很多短期异步任务的程序而言，该线程池通常可提高程序性能。调用 `execute()` 方法将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线程并添加到池中。因此，长时间保持空闲的线程池不会使用任何资源。

## newFixedThreadPool(int)

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

该方法会创建一个可重用固定线程数的线程池，当线程处于空闲状态时并不会被回收，除非线程池被关闭。当所有线程都处于活动状态时，新任务会存储在任务队列中等待，直到有线程空闲出来。

## newSingleThreadExecutor()

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

该方法会创建一个单线程的线程池。这个线程池只有一个线程在工作，也就是相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

实际上以上的三个创建线程池的工厂方法都是通过配置 ThreadPoolExecutor 的参数来实现不同功能的，这三种线程池可以满足大多数场合。

关于线程池基本介绍就到这，如果想了解线程池的原理，可以看看这篇文章：[深入理解Java之线程池](http://www.cnblogs.com/exe19/p/5359885.html)