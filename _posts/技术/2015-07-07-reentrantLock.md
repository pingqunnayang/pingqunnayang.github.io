---
layout: post
title:  ReentrantLock的原理和锁机制
titlename:   ReentrantLock 的原理和锁机制
date:  	2015-07-06 18:48:00  
category: 技术
tags: Java Lock 
description:
---
###  **1. 锁的种类**
<p style="text-indent: 2em">java.util.concurrent.lock 中的 Lock 允许把锁的实现作为java类，而不是作为语言的特性来实现。ReentrantLock 实现了Lock，它拥有与synchronized 相同的并发性，它可以完全替代synchronized 关键字，还能够提供超过关键字本身的多种功能，在看ReentrantLock 之前我们先了解下几种锁。</p>

#### **1.1 自旋锁**

<p style="text-indent: 2em">自旋锁就是当前线程不断地在循环体中执行，当循环体的条件被其他线程改变时，才能进入临界区。
	
```java
	private AtomicReference<Thread> owner = new AtomicReference<>();
	/**
	 * 使用了CAS原子操作，lock函数将owner设置为当前线程，并且预测原来的值为空.
	 */
	public void lock(){
		Thread current = Thread.currentThread();
		while(!owner.compareAndSet(null, current)){
			
		}
	}
	public void unlock(){
		Thread current = Thread.currentThread();
		owner.compareAndSet(current, null);
	}

```

<p style="text-indent: 2em">由于自旋锁只是将当前线程不停地执行循环体，不进行线程状态的改变， 所以响应速度更快。但当线程数不停增加时，性能下降明显， 因为每个线程都需要执行，占用CPU时间。如果线程竞争不激烈， 并且保持锁的时间段。适合使用自旋锁。

####  **1.2 自旋公平锁**

<p style="text-indent: 2em">公平锁主要的功能是解决顺序访问，ReentrantLock的构造函数中可以设置是否实现公平锁，公平锁会影响性能，公平锁的实现有很多种，我们用链表的形式来实现，这也是ReentrantLock的实现方式，其他可以参考
http://ifeve.com/java_lock_see2/。

```java
	public static class CLHNode{
		private volatile boolean isLocked = true;
	}
	private volatile CLHNode tail;
	private static final ThreadLocal<CLHNode> local = new ThreadLocal<CLHNode>();
	private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode> update = 
			AtomicReferenceFieldUpdater.newUpdater(CLHLock.class, CLHNode.class, "tail");
	public void lock(){
		CLHNode node = new CLHNode();
		local.set(node); 
		CLHNode preNode = update.getAndSet(this, node);
		if(preNode!=null){
			while(preNode.isLocked){ //判断前一个节点是否被lock
			}
			preNode=null;
			local.set(node);
		}
	}
	public void unlock(){
		CLHNode node = local.get();
		if(!update.compareAndSet(this, node, null)){
			node.isLocked=false;
		}		
		node = null;
	}
```

#### **1.3 阻塞锁**

<p style="text-indent: 2em">阻塞锁与自旋锁不同，阻塞锁改变了线程的状态，将线程状态由运行变为阻塞，当获取相应的信号后才可以进入线程的就绪状态，就绪的线程会通过竞争获取锁。java能够进入/退出的方法有：synochronized，ReentrantLock，object.wait()\notify(),LockSupport.park/unpark()。

```java
	private static class CLHNode{
		private volatile Thread isLock; 
	}
	private volatile CLHNode tail;
	private static final ThreadLocal<CLHNode> local = new ThreadLocal<CLHNode>();
	private static final AtomicReferenceFieldUpdater<CLHBlockLock, CLHNode> update = AtomicReferenceFieldUpdater.newUpdater(CLHBlockLock.class, CLHNode.class, "tail");

	public void lock(){
		CLHNode node = new CLHNode();
		local.set(node);
		System.out.println(Thread.currentThread().getName());
		CLHNode preNode = update.getAndSet(this, node);
		if(preNode!=null){
			preNode.isLock = Thread.currentThread();
			LockSupport.park(this);  //阻塞当前线程
			preNode = null;
			local.set(node);
		}
		
	}
	
	public void unlock(){
		CLHNode node = local.get();
		if(!update.compareAndSet(this,node, null)){
			System.out.println("unpack=== " +Thread.currentThread().getName());
			LockSupport.unpark(node.isLock); //退出阻塞状态
		}
		node = null;
	}
```

<p style="text-indent: 2em">阻塞锁的优势在于，阻塞的线程不会占用cpu时间， 不会导致 CPu占用率过高，但进入时间以及恢复时间都要比自旋锁略慢,在竞争激烈的情况下 阻塞锁的性能要明显高于 自旋锁。理想的情况则是; 在线程竞争不激烈的情况下使用自旋锁，竞争激烈的情况下使用阻塞锁。

#### **1.4 重入锁**

<p style="text-indent: 2em">如果自旋锁和自旋公平锁调用了下面这段代码会出现什么问题？

```java
	SpinLock lock = new SpinLock（）；
	lock.lcok();
	lock.lock(); //此处会出现死循环，无法往下运行。
	******
	lock.unlock();
	lock.unlock();
```

我们可以发现自旋锁一个线程调用2次以上lock，会出现死循环，这说明锁是不可重入的，我们需要做些修改让自旋锁变为可重入锁，ReentrantLock 是可重入锁。

```java
AtomicReference<Thread> lock = new AtomicReference<Thread>();
	private int count;
	
	public void lock(){
		Thread currThread = Thread.currentThread();
		if(currThread == lock.get()){
			count++;
			return; 
		}
		while(!lock.compareAndSet(null, currThread)){};
	}
	
	public void unlock(){
		Thread currThread = Thread.currentThread();
		if(count!=0){
			count--;
			return;
		}
		lock.compareAndSet(currThread, null);
	}
```

我们用一个count来记录当前线程的锁数量就可以实现可重入锁，在了解以上几种锁后，我们可以来查看ReentrantLock 的具体实现，ReentrantLock 是一种可重入的自旋锁，同时也实现了设置公平、非公平、阻塞等机制，主要的实现机制是通过AQS来做的。

###  **2. ReentrantLock 原理**

ReentrantLock 的常用方法：

```java
ReentrantLock unfairLock = new ReentrantLock(); //非公平锁
ReentrantLock fairLock = new ReentrantLock(true); //公平锁
fairLock.lock()
//do something
fairLock.unlock();
```
#### **2.1 lock函数分析**

ReentrantLock会保证 do something在同一时间只有一个线程在执行这段代码，或者说，同一时刻只有一个线程的lock方法会返回。其余线程会被挂起，直到获取锁。ReentrantLock使用了AQS的独占API来实现。AQS的全称为（AbstractQueuedSynchronizer），这个类是在java.util.concurrent.locks下面，看下ReentrantLock 的两个构造函数和lock函数：

```java
 public ReentrantLock() {
     sync = new NonfairSync();
 }
 public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
 }
 public void lock() {
        sync.lock();
 }
```

FairSync和NonfairSync两个类都继承了AbstractQueuedSynchronizer类，我们可以看到公平锁与非公平锁的实现分别由FairSync和NonfairSync来实现，我们可以看到`1.2自旋公平锁`中的公平锁实现，每个线程都有个标志位来确定是否获取了当前锁，我们可以继续查看FairSync的lock实现代码：

```java
 final void lock() {
      acquire(1);
 }
```

调用了AQS的acquire的方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

先尝试获取锁，如果没有获取成功，就尝试添加到Queued中，从这点可以看出公平锁是用Queue来实现的，AQS没有实现tryAcquire方法，这个方法需要子类覆盖，我们查看FairSync类的tryAcquire方法如下：

```java
 protected final boolean tryAcquire(int acquires) {
   final Thread current = Thread.currentThread(); //获取当前线程
   int c = getState(); //获取标志位，类似1.2节的isLocked标识
   if (c == 0) {
       if (!hasQueuedPredecessors() && // 如果队列没有其他线程，说明没有线程占用锁
           compareAndSetState(0, acquires)) { //尝试修改状态来获取锁
           setExclusiveOwnerThread(current); //标识当前线程获取了锁
           return true;
       }
   }
   else if (current == getExclusiveOwnerThread()) { //重入锁的实现1.4中说过
       int nextc = c + acquires;
       if (nextc < 0)
           throw new Error("Maximum lock count exceeded");
       setState(nextc);
       return true;
   }
   return false;
}
```

如果如果获取锁，tryAcquire返回true，反之，返回false。回到AQS的acquire方法如果没有获取锁会调用acquireQueued(addWaiter(Node.EXCLUSIVE), arg)方法，我们可以先查看addWaiter方法：

```java
  private Node addWaiter(Node mode) {
     Node node = new Node(Thread.currentThread(), mode);  //构建一个Node对象，Queue中的一个节点
      // Try the fast path of enq; backup to full enq on failure
      Node pred = tail;  //是否存在队列尾
      if (pred != null) {
          node.prev = pred;
          if (compareAndSetTail(pred, node)) {  //将当前节点设置为尾节点
              pred.next = node;
              return node;
          }
      }
      enq(node);   // CAS方式循环获取锁
      return node;
  }
```

这段代码和`1.3`中的lock方法差不多，构建一个Node节点，尝试加入到queue中，不成功则调用enq (node)自旋方式修改。

```java
  private Node enq(final Node node) {
       for (;;) { // 自旋
           Node t = tail;
           if (t == null) { //第一个节点是一个空节点，代表有一个线程已经获取锁
               if (compareAndSetHead(new Node()))  
                   tail = head;
           } else {
               node.prev = t;
               if (compareAndSetTail(t, node)) {
                   t.next = node;
                   return t;
               }
           }
       }
   }
```

最后看下`acquireQueued`方法的实现：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) { // 自旋获取锁
            final Node p = node.predecessor(); //获取当前节点的前一个节点
            if (p == head && tryAcquire(arg)) { //如果是head节点就尝试获取锁
                setHead(node);  //当前节点设置为head。移除head
                p.next = null; // help GC
                failed = false;
                return interrupted; //返回false表示获取到了锁@see acquire方法
	         }
            if (shouldParkAfterFailedAcquire(p, node) && //检查前一个节点的状态位，看当前获取锁失败的线程是否需要挂起。
                parkAndCheckInterrupt()) //如果需要，用pard挂起当前线程
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

我们可以看到这块也是用自旋的方式来确定自己是不是Head的next节点，然后尝试获取锁，如果获取没有成功或者不是head节点，那么需要检查前一个节点waitStatus的状态，waitStatus是Node的一个状态位，在刚进入队列时都是0，如果等待被取消状态被设置Node.CANCELLED，若线程释放锁时候需要唤醒等待队列里的其他线程则被置为Node.SIGNAL。

```java
    /** waitStatus value to indicate thread has cancelled */
      static final int CANCELLED =  1;  // 节点取消
      /** waitStatus value to indicate successor's thread needs unparking */
      static final int SIGNAL    = -1;  // 节点等待触发
      /** waitStatus value to indicate thread is waiting on condition */
      static final int CONDITION = -2;  // 节点等待条件
      /**
       * waitStatus value to indicate the next acquireShared should
       * unconditionally propagate
       */
      static final int PROPAGATE = -3; //节点状态需要向后传播
```

从这里可以看到Reentrantlock也是一种阻塞线程。一个线程在获取锁的过程中，如果成功获取锁就不会进入队列，如果失败会被挂起，等待下次唤醒继续获取锁，那么Reentrantlock是如何实现公平与非公平竞争的排列策略呢?我们看下非公平锁的lock函数：

```java
 final void lock() {
            if (compareAndSetState(0, 1)) //不排队直接抢占
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
  }
```

非公平锁的lock方法的处理方式是: 在lock的时候先直接cas修改一次state变量，如果锁没有被任何线程锁定且加锁成功则设定当前线程为锁的拥有者。

#### **2.2 unlock函数分析**

<p style="text-indent: 2em">unlock要实现的就是修改当前线程的标志位state，然后unpark阻塞的线程，如果有重入，则需要对count--等操作。
 
```java
public void unlock() {
      sync.release(1);
}

public final boolean release(int arg) {
   if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0) //需要unpark操作
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

实现在tryRelease和unparkSuccessor两个方法中，tryRelease需要子类实现，AQS没有实现。我们往下看：

```java
protected final boolean tryRelease(int releases) {
	int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // 可重入的判断
        free = true;
        setExclusiveOwnerThread(null); 
    }
    setState(c); //设置标识位
    return free;
}
```

释放锁成功后，如果waitStatus 状态不等于0，需要做唤醒操作：

```java
 private void unparkSuccessor(Node node) {
     /*
       * If status is negative (i.e., possibly needing signal) try
       * to clear in anticipation of signalling.  It is OK if this
       * fails or if status is changed by waiting thread.
       */
      int ws = node.waitStatus;
      if (ws < 0)
          compareAndSetWaitStatus(node, ws, 0);

      /*
       * Thread to unpark is held in successor, which is normally
       * just the next node.  But if cancelled or apparently null,
       * traverse backwards from tail to find the actual
       * non-cancelled successor.
       */
      Node s = node.next;
      if (s == null || s.waitStatus > 0) {
          s = null;
          //队列尾部开始往前去找的最前面的一个waitStatus小于0的节点。
          for (Node t = tail; t != null && t != node; t = t.prev) 
              if (t.waitStatus <= 0)
                  s = t;
      }
      if (s != null)
          LockSupport.unpark(s.thread); //唤醒操作
  }
```

从分析可以看到，ReentrantLock是一个可设置公平、非公平的锁，同时也是可重入的，并没有采用自旋锁而是阻塞锁的原理来实现，这样可以防止CPU消耗过大，但在低竞争中速度没有自旋锁的速度快。



> - 参考连接
>-  AbstractQueuedSynchronizer的实现分析（上）http://ifeve.com/jdk1-8-abstractqueuedsynchronizer/
>-  java锁的种类以及辨析  http://ifeve.com/java_lock_see1/