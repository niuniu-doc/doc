---
title: Java线程池初探
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---

#### 线程池类型
```
1. newSingleThreadExecutor 
   -> return new ThreadPoolExecutor(1, 1, 0L, 
                                      TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
   只使用一个线程、使用无界队列 LinkedBlockingQueue, 线程创建后不会超时终止、该线程顺序执行所有任务
   适用于需要确保所有任务呗顺序执行的场合

1. newFixedThreadPool 
   -> ThreadPoolExecutor(nThreads, nThreads, 0L, 
                         TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>())
   保持固定线程数、新的任务进入、会放进任务队列                       
   
2. newCachedThreadPool 
   -> ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, 
                         TimeUnit.SECONDS, new SynchronousQueue<Runnable>())
   任务到达、有空闲线程就复用、无空闲线程就创建
   任务执行完、线程最大空闲时间60s、超过60s会被销毁                     
                         
3. new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors() * 3,
                          200, 1,TimeUnit.MINUTES, new ArrayBlockingQueue<Runnable>(100000), new DefaultThreadFactory())                    
```

#### 核心参数
```
corePoolSize: 线程池中维护的核心线程数、空闲依然存在、除非设置了 allowCoreThreadTimeOut 参数
maximumPoolSize: 最大线程数
keepAliveTime: 线程池中的线程数 > 核心线程数的时候、空闲线程最大idle的时间
workQueue: 任务队列
```

#### Queue类型
```
1. LinkedBlockingQueue
2. SynchronousQueue
3. ArrayBlockingQueue
4. 
```

#### reject
```
1. ThreadPoolExecutor.AbortPolicy：这就是默认的方式，抛出异常
2. ThreadPoolExecutor.DiscardPolicy：静默处理，忽略新任务，不抛异常，也不执行
3. ThreadPoolExecutor.DiscardOldestPolicy：将等待时间最长的任务扔掉，然后自己排队
4. ThreadPoolExecutor.CallerRunsPolicy：在任务提交者线程中执行任务，而不是交给线程池中的线程执行

拒绝策略只有在队列有界，且maximumPoolSize有限的情况下才会触发
如果队列无界，服务不了的任务总是会排队，但这不见得是期望的，因为请求处理队列可能会消耗非常大的内存，甚至引发内存不够的异常
如果队列有界但maximumPoolSize无限，可能会创建过多的线程，占满CPU和内存，使得任何任务都难以完成
在任务量非常大的场景中，让拒绝策略有机会执行是保证系统稳定运行很重要的方面
```

#### 线程池的选择
```
1. 在系统负载很高的情况下、newFixedThreadPool可以通过队列对新任务排队、保证有足够的资源处理实际任务
   而newCachedThreadPool会为每一个任务创建一个线程、导致创建过多的线程竞争CPU和内存资源、
   使得任何任务都难以完成、此时使用newFixedThreadPool更合适
2. 若系统负载不太高、单个任务执行时间也比较短、newCachedThreadPool 因为不用排队、效率可能更高

3. 系统负载可能极高的情况下、两者都不是最好的选择
   newFiexedThreadPool的问题是: 队列过长、占用更多内存
   newCachedThreadPool的问题是: 线程数过多、导致过多线程竞争CPU和内存资源
```

#### 线程池的死锁
```
若任务之间有依赖、可能会出现线程池的死锁
eg. 线程池的大小为5、taskA 提交了5个任务、taskA会提交5个taskB、taskB因为无法创建新的线程只能在等待队列进行等待
    而taskA一直在等待taskB的处理结果、这样就会造成线程池的死锁
    
resolve: 
1. 使用 newCachedThreadPool 替代 fixed: 这样、有新的任务提交就会创建新的线程
2. 使用 SynchronousQueue 这样、入队成功就意味着有线程接受处理、若入队失败、就会触发reject机制、不管怎么样、都不会死锁了
   static ExecutorService executor = new ThreadPoolExecutor(
        THREAD_NUM, THREAD_NUM, 0, TimeUnit.SECONDS, 
        new SynchronousQueue<Runnable>());
```


#### note
```
1. 核心线程不会预先创建，只有当有任务时才会创建
2. 核心线程不会因为空闲而被终止，keepAliveTime参数不适用于它
3. 核心线程干预
   1) public int prestartAllCoreThreads() 预先创建所有的核心线程
   2) public boolean prestartCoreThread() 创建一个核心线程，如果所有核心线程都已创建，返回false
   3) public void allowCoreThreadTimeOut(boolean value) 如果参数为true，则keepAliveTime参数也适用于核心线程
```
