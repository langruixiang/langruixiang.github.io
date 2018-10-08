---
layout: post
title: ReentrantLock
categories: [Java]
description: 
keywords: JDK, AQS
---

##引言
***ReentrantLock***是JDK提供的一个可重入互斥锁，所谓可重入就是同一个锁允许被已经获得该锁的线程重新获得。可重入锁的好处可以在递归算法中使用锁，不可重入锁则导致无法在递归算法中使用锁。因为第二次递归时由于第一次递归已经占有锁，而导致死锁。本文我们将探讨JDK中ReentrantLock的实现。</br>
***Semaphore***是JDK提供的一个可共享的同步组建，有n个许可，多个线程可以共同去获得许可，当线程申请的许可小于n时即可成功申请，否则申请失败。</br>
***AQS（AbstractQueuedSynchronizer）***是Java实现同步组建的基础框架，一般以静态内部类的形式实现在某个同步组件类中，通过代理的方式向外提供同步服务，ReentrantLock和Semaphore都是基于AQS实现的同步组件，前者是独占式同步组建，即一个线程获得后，其他线程无法获得。后者是共享式同步组件，一个线程获得后，在满足的条件下，其他线程也可以获得。</br>
##AQS工作原理
AQS是Java实现同步组建的基础框架，其基本思想是用一个volatile int state变量来表示当前同步组件的状态，用getState()获取同步组件的状态，用compareAndSet(int expect, int update)来对state状态进行操作，compareAndSet可以保证对state变量更新值的原子性。AQS中很多方法是final的，即不允许用户覆盖，用户自定义的方法一般有：</br>

* tryAcquire: 独占式获取同步状态，该函数一般首先查询state的值，如果state不允许继续被获取，直接返回false。如果state允许继续被获取，CAS尝试更新state的值，成功返回true，失败返回false
* tryAcquireShared：共享式的获取同步状态，该一般是在CAS死循环获取state的值，计算state被获取后的值，如果该值为负数，直接返回负数表示失败，如果该值为正值，则用CAS更新该值，当CAS更新失败时，重复上述步骤，直至返回负数或CAS更新成功返回正值。
* tryRelease：独占式的释放同步状态
* tryReleaseShared：共享式的释放同步状态，一般在CAS死循环中反复尝试，直至释放成功
* isHeldExclusively：判断当前同步器是否被当前线程占有</br>
 
AQS提供的模板方法有：</br>

* acquire:独占式的获取同步状态，获取成功则返回，获取失败则会进入等待队列，该方法会调用用户自定义的tryAcquire函数
* acquireInterruptibly：与acquire类似，不同在于当进入等待队列时，遇到中断会抛出InterruptedException异常，用户可以处理该中断异常
* tryAcquireNanos：在acquireInterruptibly的基础上增加了时间限制，一定时间内没有成功获取则返回false
* acquireShared：共享式的获取同步状态，成功则返回，失败则进入等待队列，该方法会调用用户自定义的tryAcquired函数
* acquireSharedInterruptibly：在等待队列可以相应中断，与上类似
* tryAcquireShared：在acquireSharedInterruptibly增加了超时限制
* release：独占式的释放同步状态，会调用用户自定义tryRelease函数
* releaseShared：共享式的释放同步状态，会调用用户自定义tryReleaseShared函数
* getQueuedThreads：获取等待队列线程集合

##ReentrantLock源码分析
ReentrantLock的默认构造函数是

	public ReentrantLock() {
	        sync = new NonfairSync();
	}
NonfairSync继承了Sync，Sync是一个抽象类，并继承了抽象类AbstractQueuedSynchronizer。
ReentrantLock是一个独占式的锁，所以它需要实现tryAcquire函数和tryRelease函数</br>
**tryAcquire函数源码如下**

	protected final boolean tryAcquire(int acquires) {
	        return nonfairTryAcquire(acquires);
	}
nonfairTryAcquire(acquires)源码如下

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
* 先得到当前线程
* 查询当前state值，如果为0则说明当前锁还未被其他线程获取，则尝试CAS获得锁，成功则把占有锁的线程设置为当前线程，返回true。失败返回false。
* 如果state不为0则说明该锁已经被其他线程获取，则检查获得锁的线程是否是当前线程以实现可重入特性，如果是，则更新state的值，并返回true。此处更新不需要CAS，因为只有当前线程可以操作state。
* 其他情况返回false

**tryRelease函数源码如下**

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

* 首先得到释放之后的状态值c
* 检查当前释放锁的线程，如果不是已占有锁的线程则抛出异常，因为ReentrantLock是独占式锁，释放锁的线程一定是占有锁的线程
* 如果c是等于0的，说明获取锁的所有函数都已经返回，则锁释放成功
* 如果c不等于0，说明只是部分递归的函数返回，部分递归函数还未返回，则释放失败，锁依然被占有

**Lock**函数源码

	public void lock() {
        sync.lock();
    }
sunc的lock函数

	final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
     }
该函数首先直接尝试CAS操作，成功则设置当前函数为占有锁的函数，返回，失败则调用acquire函数。acquire函数为AQS实现的模板方法，它尝试获得锁，成功则返回，不成功则进入等待队列直至获取成功。

**unLock**函数源码
	
	public void unlock() {
        sync.release(1);
    }
调用tryRelease函数释放锁。

##Semaphore的源码
Semaphore构造函数如下：

	public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
与ReentrantLock代码结构非常相似。Semaphore是一个共享式的同步组建，它应该实现tryAcquireShared和tryReleaseShared

**tryAcquireShared**函数源码：
	
	protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
    }
nonfairTryAcquireShared源码：

	final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
    }
* 函数首先取得当前的可用许可数，并计算被获取acquires个许可后剩余的许可数。
* 如果剩余的许可数小于0直接返回剩余的许可数，即负值
* 如果大于0则尝试使用CAS循环更新state的值，更新失败则重试上述步骤，直至返回负值更新失败，或者返回非负值更新成功。</br>
***tips：***与独占式的tryAcquire逻辑不太一样，独占式的tryAcquire在CAS操作失败后，直接返回失败。本人觉得共享式的tryAcquiredShared在CAS操作失败后，因为组件是共享的，所以再次尝试获取同步组件成功的可能性较大，所以在CAS失败后，尝试再次更新。而独占式的CAS更新失败后，组件已经被其他线程获取，再次尝试成功的可能性较小，所以没有重新尝试。纯属个人观点。

**tryReleasedShared**源码

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

* 函数计算释放后的state值并验证是否溢出
* CAS更新state的值直至成功

**acquire**函数源码
	
	public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
acquireSharedInterruptibly为AQS提供的模板方法，调用了tryAcquireShared，成功直接返回，不成功加入等待队列，并添加了处理中断的机制

**release**函数源码

	public void release() {
        sync.releaseShared(1);
    }
releaseShared为AQS提供的模板方法，调用了tryReleaseShared

##总结
完整的ReentrantLock和Semaphore实现非常复杂，本文旨在介绍AQS框架，并通过ReentrantLock和Semaphore一个独占式的同步组件和一个非独占式的同步组件来学习怎么使用AQS实现通组件，具体来说分为以下步骤：

* 待实现的同步组件是独占式的还是共享式的
* 独占式的同步组件实现tryAcquire和tryRelease，非独占式的实现tryAcquireShared和tryReleaseShared
* 将我们实现的同步组建相应的方法如Lock和unLock代理到AQS对应的函数包括用户自定义的函数和AQS提供的模板函数

AQS的方便之处在于我们只需要实现tryAcquire和tryRelease或tryAcquireShared和tryReleaseShared就可以使用AQS帮我们实现好的阻塞的acquire函数，可中断的acquire函数，带超时的acquire函数等模板函数，大大简化了用户的开发量和难度。
