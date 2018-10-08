---
layout: post
title: Executors
categories: [Java]
description: 
keywords: JDK, Executors
---

##1. 引言
Executors类是一个工厂设计模式的类，从返回值来看，分为返回Callable、ThreadFactory、ExecutorsService、ScheduleExecutorsService的，现分别阐述。

##2. 返回值为Callable的函数
返回值为Callable的函数主要起适配器的作用，例如将Runnable对象封装为Callable对象，将PrivilegedAction封装为Callable对象等。

##3. 返回值为ThreadFactory的函数
ThreadFactory是用于生产某种特定属性线程的工厂方法比如特定优先级的线程，属性是守护进程的线程等。

##4. 返回值为ExecutorsService或ScheduledExecutorService的函数

###4.1 
* ***newCachedThreadPool***</br>
* ***newFixedThreadPool***</br>
* ***newSingleThreadPool***</br>
* ***newScheduledThreadPool***</br>

这四个函数归根结底都调用的ThreadPoolExecutor函数。只是参数不一样：


	public static ExecutorService newCachedThreadPool() {
		return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
		  						     60L, TimeUnit.SECONDS,
		  							 new SynchronousQueue<Runnable>());
	}

	public static ExecutorService newFixedThreadPool(int nThreads) {
    	return new ThreadPoolExecutor(nThreads, nThreads,
                                 	 0L, TimeUnit.MILLISECONDS,
                                 	 new LinkedBlockingQueue<Runnable>());
	}

	public static ExecutorService newSingleThreadExecutor() {
	    return new FinalizableDelegatedExecutorService
	            (new ThreadPoolExecutor(1, 1,
	                                    0L, TimeUnit.MILLISECONDS,
	                                    new LinkedBlockingQueue<Runnable>()));
	}

	public static ScheduledExecutorService newScheduledThreadPool(
		            int corePoolSize, ThreadFactory threadFactory) {
		        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
	}
	public ScheduledThreadPoolExecutor(int corePoolSize,
	                                       ThreadFactory threadFactory) {
	        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
	              new DelayedWorkQueue(), threadFactory);
	}
	public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
ThreadPoolExecutor函数的与原型如下：</br>

	public ThreadPoolExecutor(int corePoolSize,
							  int maximumPoolSize,
							  long keepAliveTime,
							  TimeUnit unit,
							  BlockingQueue<Runnable> workQueue,
							  ThreadFactory threadFactory) {
		this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
		threadFactory, defaultHandler);
	}

各参数含义如下：</br>

* corePoolSize：核心线程池大小，具体解释请往下看</br>
* maximumPoolSize：最大线程池大小，具体解释请往下看</br>
* keepAliveTime：空闲线程存活时间</br>
* unit：keepAliveTime的单位</br>
* wordQueue：存放待执行任务的阻塞队列</br>
* threadFactory：线程池线程的默认属性工厂</br>
* defaultHandler：拒绝策略

ThreadPoolExecutor工作原理如下：</br>
(1) 新来一个任务，首先查看线程池当前存活线程的数量alive，如果alive < corePoolSize, 则使用threadFactory创建新的线程，执行该任务。(可视为线程池的预热)</br>
(2) 如果alive >= corePoolSize, 则查看当前阻塞队列是否已满，如果没满，则加入阻塞队列</br>
(3) 如果阻塞队列已满，则创建新的线程，直至alive = maximumPoolSize</br>
(4) 当alive == maximumPoolSize，则使用defaultHandler执行拒绝策略，即线程池已达到最大负载，拒绝提供服务的策略</br>
基于以上四点，我们来看一下这三个线程池，</br>

* newCachedThreadPool的corePoolSize为0，maximumuPoolSize为Integer.MAX\_VALUE，阻塞队列为SynchronousQueue。corePoolSize为0，意味着新来一个任务，则加入阻塞队列，SynchronousQueue是一个没有容量的阻塞队列，所以这时，必将创建一个新的线程，直至线程个数达到Integer.MAX_VALUE。也就是说，每来一个任务创建一个线程。
* newFixedThreadPool的corePoolSize为nThread，maximumPoolSize为nThread，阻塞队列为LinkedBlockingQueue。当alive小于nThread时，来一个任务创建一个线程，直至alive == nThread。此时，新来任务加入阻塞队列，LinkedBlockingQueue是一个无界队列(其容量是Integer.MAX\_VALUE，往往达不到Integer.MAX\_VALUE时，已经OOM了，暂称其为无界队列)，所以永远不可能创建大于nThread个数的线程，即线程池最大也是nThread。
* newSingleThreadExecutor是nThread = 1的newFixedThreadPool
* newScheduledThreadPool的corePoolSize为corePoolSize，maximumuPoolSize为Integer.MAX\_VALUE，阻塞队列为DelayedWorkQueue，DelayedWorkQueue是一个无界的优先队列，按照任务休眠时间排序，休眠时间少的在队列前端，因此，可以实现任务按照时间调度的目的。

###4.2
* ***newWorkStealingPool***</br>

newWorkStealingPool代码如下：

	public static ExecutorService newWorkStealingPool() {
	        return new ForkJoinPool
	            (Runtime.getRuntime().availableProcessors(),
	             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
	             null, true);
	}
其调用的是ForkJoinPool，为了说明ForkJoinPool，首先了解Java的ForkJoin框架。

***ForkJoin框架***</br>
ForkJoin框架是一种特别适合执行分治任务的线程池框架，一般执行集成了RecursiveTask和RecursiveAction类的任务，在RecursiveTask/RecursiveAction有一个compute函数，其一般逻辑是这样的：</br>
	
	Function compute
		if 当前任务规模 < 阈值Threshold
			执行当前任务；
			return；
	 		
		将当前任务分为两个子任务，task1，task2
	    invoke(task1, task2)   //invoke函数会执行task1和task2的compute函数
因此ForkJoin框架会产生大量的小任务在线程池中，ForkJoinPool采用了“工作窃取”的机制，一方面避免多个处理器提取任务是发生冲突，另一方面，减少处理器空闲的时间。工作窃取机制如下，假设当前ForkJoinPool的并行度为4，则将ForkJoinPool的所有任务分为4个队列，每个处理其只执行自己队列的任务，因此与4个处理器去一个队列提取任务相比，大大减少了冲突的可能。另一方面，当某个处理器处理完自己队列的任务时，他回去其他处理器队列尾部“窃取”任务执行，因此，避免了处理器出现空闲的状况。</br>

回到newWorkStealingPool，它使用的“工作窃取”机制的线程池，只不过其执行的不是RecursiveTask/RecursiveAction，而是普通的Runnable/Callable，因此，适合大量小规模的Runnable/Callable的执行。

##5 总结
总而言之，Executors框架的线程池分为两类，一类是对ThreadPoolExecutor的封装，一类是对ForkJionPool的封装。我们也可以直接调用ThreadPoolExecutor，ForkJionPool类定义自己的线程池。

	
	







