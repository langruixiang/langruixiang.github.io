---
layout: post
title: wait、notify与Condition
categories: [Java]
description: some word here
keywords: Java, Concurrency
---


# 具体场景
假设我们需要实现一个固定大小的支持多线程的环形缓冲区（ring buffer），该环形缓冲区支持FIFO顺序的get，put，以及大小相关的size，isFull，isEmpty操作，interface定义如下：

    public interface Ringbuffer {
    	public Integer get();
    	public void add(Integer num);
    	public boolean isFull();
    	public boolean isEmpty();
    	public int size();
    }
# 分析
很多场景下，线程只需要拿到锁就满足了可执行条件直接执行就可以。固定大小的环形缓冲区则不太一样，考虑以下场景:

* 线程A要执行put操作，并且已经拿到了锁，然而缓冲区当前是满的 /(ㄒoㄒ)/~~
* 线程B要执行get操作，并且已经拿到了锁，然而缓冲区当前是空的 /(ㄒoㄒ)/~~

在线程中使用while操作去轮询是一种方案，这种情况下，线程会占用cpu，一直检查条件是够满足，怎样实现A线程在不满足条件时休眠，条件满足后程序自动唤醒A线程？

# wait、notify
Java中任何一个对象都带有一个monitor锁，同时，任何一个对象都带有一个等待队列，通过wait把线程加入该对象的等待队列，通过notify去唤醒该对象等待队列的线程。需要注意的是wait和notify操作都需要在线程已获取monitor锁的情况下进行。因此，使用wait和notify的框架如下所示[1]：
    
    synchronized (obj) {
         while (<condition does not hold>)
             obj.wait();
         ... // Perform action appropriate to condition
    }
* obj.wait()在synchorized块中，保证wait调用时，当前线程已获得monitor锁。wait调用后，当前线程释放monitor锁，并加入等待队列，被唤醒时，线程需要重新获取monitor锁，并继续执行wait函数后的代码
* while(<condition does not hold>)很重要，我们常常使用notifyAll去唤醒等待队列中的线程，这些被唤醒的线程中的一个执行后可能导致被唤醒的其他线程不再满足congdition **（假唤醒）**，所以使用while循环，重新判断等待条件是否满足，不满足则继续加入等待队列等待下次唤醒。一个用if替换while导致**错误的示例**可参考Appendix，

用wait和notify实现的唤醒缓冲区代码如下，

    
    public class RingbufferWithMonitor implements Ringbuffer{
    	private int capacity;
    	private int[] buffer;
    	
    	private int head = 0;
    	private int tail = 0;
    	private int size = 0;
    	
    	public RingbufferWithMonitor(int capacity) {
    		this.capacity = capacity;
    		this.buffer = new int[capacity];
    	}
    
    
    	public void add(Integer num) {
    		synchronized (buffer) {
    			while (size == capacity) {
    				try {
    					buffer.wait();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			
    			buffer[tail] = num;
    			size++;
    			tail = (tail + 1) % capacity;
    			
    			buffer.notifyAll();
    		}
    
    	}
    
    	public Integer get() {
    		int ret;
    		synchronized (buffer) {
    			while(size == 0) {
    				try {
    					buffer.wait();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			
    			ret = buffer[head];
    			size--;
    			head = (head + 1) % capacity;
    			
    			buffer.notifyAll();
    		}
    		
    		return ret;
    	}
    
    
    	public boolean isFull() {
    		return size == capacity;
    	}
    
    	public boolean isEmpty() {
    		return size == 0;
    	}
    
    	public int size() {
    		return size;
    	}
    }
    
# Condition
上例中，有一个缺陷是get与add操作是互斥的，当唤醒缓冲区有大量元素时，get操作与add操作在访问ringbuffer的两个不同元素，可以不互斥。同时，由于get和add都要更改size的值，size需要变成互斥的。也就是说，我们可以加两个锁使得互斥的范围由add、get函数缩小到size值得更新。通过synchorized两个对象是可以实现的，本部分介绍如何通过ReentrantLock的Condition实现。

Condition是专为线程获取锁后需要等待其他条件而设置的一个类。ReentrantLock充当了Condition的工厂，即通过如下当时可以获取ReentrantLock对应的Condition。一个ReentrantLock可对应多个Condition

    ReentrantLock getLock = new ReentrantLock();
    Condition notEmpty = getLock.newCondition();
    Condition otherCondition = getLock.newCondition();
调用condition.await可将线程加入condition的等待队列，通过condition.signal可将condition等待队列的线程唤醒，与wait、notify类似，condition有如下约束
* await与signal函数必须在获取对应的ReentrantLock后调用。调用notEmpty的await和signal必选在getLock之后调用。


    ReentrantLock getLock = new ReentrantLock();
    Condition notEmpty = getLock.newCondition();
    try{
        getLock.lock();
        while(<condition does not hold>){
            notEmpty.await();   //该行需要捕获异常
        }
    }finally{
        getLock.unlock();
    }
* 为了避免假唤醒，写在while循环内

一个用condition实现的唤醒缓冲区代码如下
    public class RingbufferWithCondition implements Ringbuffer{
    	
    	private Integer[] buffer;
    	private int capacaity;
    	private int head = 0;
    	private int tail = 0;
    	private AtomicInteger size = new AtomicInteger(0);
    	
    	private ReentrantLock getLock = new ReentrantLock();
    	private Condition notEmpty = getLock.newCondition();
    	
    	private ReentrantLock addLock = new ReentrantLock();
    	private Condition notFull = addLock.newCondition();
    	
    	public RingbufferWithCondition(int capacity) {
    		this.capacaity = capacity;
    		buffer = new Integer[capacity];
    	}
    
    	public Integer get() {
    		int ret;
    		try {
    			getLock.lock();
    			
    			while(size.intValue() == 0) {
    				try {
    					notEmpty.await();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			
    			ret = buffer[head];
    			head = (head + 1) % capacaity;
    			size.decrementAndGet();
    			if(size.intValue() > 0) {
    				notEmpty.signal();
    			}
    		} finally {
    			getLock.unlock();
    		}
    		
    		signalNotFull();
    		return ret;
    	}
    
    	public void add(Integer num) {
    		try {
    			addLock.lock();	
    			
    			while(size.intValue() == capacaity) {
    				try {
    					notFull.await();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			
    			buffer[tail] = num;
    			tail = (tail + 1) % capacaity;
    			size.incrementAndGet();
    			
    			if(size.intValue() < capacaity) {
    				notFull.signal();
    			}
    		} finally {
    			addLock.unlock();
    		}
    		
    		signalNotEmpty();		
    	}
    
    	public boolean isFull() {
    		return size.intValue() == capacaity;
    	}
    
    	public boolean isEmpty() {
    		return size.intValue() == 0;
    	}
    
    	public int size() {
    		return size.intValue();
    	}
    	
    	private void signalNotEmpty() {
    		try {
    			getLock.lock();
    			notEmpty.signal();
    		}finally{
    			getLock.unlock();
    		}
    	}
    	
    	private void signalNotFull() {
    		try {
    			addLock.lock();
    			notFull.signal();
    		} finally {
    			addLock.unlock();
    		}
    	}
    
    }
* signalNotEmpty函数用于在add调用后唤醒get的等待线程，signalNotFull类似
* signalNotEmpty调用一定不要位于addLock.lock()和addLock.unlock之间，signalNotEmpty要调用getLock.lock,容易死锁。我第一次写在这踩了坑，对比[2]才找出来的 /(ㄒoㄒ)/~~


# Reference
[1] 摘自JDK1.8 wait函数的javadoc文档

[2] JDK1.8 LinkedBlockingQueue的实现及Javadoc文档
# Appendix
if替换RingbufferWithmonitor中的while

    public class WrongRingbufferWithmonitor implements Ringbuffer{
    
    	private int capacity;
    	private int[] buffer;
    	
    	private int head = 0;
    	private int tail = 0;
    	private int size = 0;
    	
    	public WrongRingbufferWithmonitor(int capacity) {
    		this.capacity = capacity;
    		this.buffer = new int[capacity];
    	}
    
    
    	public void add(Integer num) {
    		synchronized (buffer) {
    			if (size == capacity) {
    				try {
    					buffer.wait();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			
    			buffer[tail] = num;
    			size++;
    			tail = (tail + 1) % capacity;
    			
    			buffer.notifyAll();
    		}
    
    	}
    
    	public Integer get() {
    		int ret;
    		synchronized (buffer) {
    			if(size == 0) {
    				try {
    					buffer.wait();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			
    			ret = buffer[head];
    			size--;
    			head = (head + 1) % capacity;
    			
    			buffer.notifyAll();
    		}
    		
    		return ret;
    	}
    
    
    	public boolean isFull() {
    		return size == capacity;
    	}
    
    	public boolean isEmpty() {
    		return size == 0;
    	}
    
    	public int size() {
    		return size;
    	}
    }
测试代码：

    final WrongRingbufferWithmonitor ringbufferWithmonitor = new WrongRingbufferWithmonitor(10);
    ExecutorService executorService = Executors.newFixedThreadPool(50);
	
	for(int i = 0; i < 10; i++) {
		executorService.execute(new Runnable() {
			
			public void run() {
				System.out.println(ringbufferWithmonitor.get() + " size: " + ringbufferWithmonitor.size());;
				
			}
		});
	}
	
	try {
		Thread.sleep(1000);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
	
	executorService.execute(new Runnable() {
		
		public void run() {
			ringbufferWithmonitor.add(0);
			
		}
	});

    
