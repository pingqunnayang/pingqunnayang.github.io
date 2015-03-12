---
layout: post
title:  "threadpool"
date:   2015-03-10 00:18:23 
categories: threadpool
---
之前项目中很多地方用到线程池，线程池的好处：
>- 可以控制系统中线程的数量，根据系统的性能要求调整线程的数目。
>- 减少创建和销毁线程的次数，提高服务器的工作效率。

一个线程池的实现原理可以用如下一张图来描述：

![Alt text](./1426073188527.png)
（图1）
> **流程：** 用户提交线程放到线程池的**Queue**中，线程池启动多个**Worker**线程监听**Queue**，如果**Queue**中有任务则获取并执行，没有则等待或销毁。

从上图我们了解了线程池的一种实现，类似于生产者/消费者模型，那么要设计一套这样的实现我们需要关注`哪些问题`？
>-  **Queue**的大小如何限制，**Queue**满了怎么处理？
>-  **Worker**线程的生命周期怎么控制？**worker**如何创建、销毁。
>-  **线程池**的状态如何控制？如何关闭**线程池**？

带着问题我们来看下java.util.concurrent.**ThreadPoolExecutor**是如何实现线程池的？**ThreadPoolExecutor**是ExecutorService的默认实现，**ThreadPoolExecutor**的构造函数如下：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

>- `corePoolSize:` 是核心线程的数量（图1中**worker**线程的数量）。
>- `maximumPoolSize:` 是最大**worker**线程数。
>- `keepAliveTime:` 当线程数大于核心时，此为终止空闲线程等待新任务的最长时间。
>- `keepAliveTime:` 参数的时间单位。
>- `workQueue:` 保存任务的队列（图1的**Queue**）。
>- `threadFactory:`  创建**worker**线程的工程。
>- `handler:` 超出线程范围和队列容量的处理程序。

**看下ThreadPoolExecutor中一些比较重要的变量：**

```java
    /**
    * ctl是用来统计worker数量和线程池状态的，为了用一个int解决问题
    * workerCount 用低29位，最大worker数量是(2^29)-1
    * runState 使用高3位 状态值如下：
    **/
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int RUNNING    = -1 << COUNT_BITS;  //111[29个0]
    private static final int SHUTDOWN   =  0 << COUNT_BITS;  //000[29个0]
    private static final int STOP       =  1 << COUNT_BITS;  //001[29个0]
    private static final int TIDYING    =  2 << COUNT_BITS;  //010[29个0]
    private static final int TERMINATED =  3 << COUNT_BITS;  //011[29个0]
    // 线程池大小、线程空闲数、实际数、线程池运行状态等等状态改变都不是线程安全的，
    // 需要一个全局锁来协调竞争。
    private final ReentrantLock mainLock = new ReentrantLock();
    // 存放worker，操作需要用获取mainlock。
    private final HashSet<Worker> workers = new HashSet<Worker>();
    
```

**线程池的五种状态**
>- `RUNNING:` 接收新的task。可处理**Queue**里的task。
>- `SHUTDOWN:`不接收新的task，可处理**Queue**里的task。
>- `STOP:` 不接收新的task，不处理queue里的task，同时中断处理中的task。
>- `TIDYING:` 所有task都被终止，workercount=0，并执行terminated()方法。
>- `TERMINATED:` terminated()行完成。

**执行过程：**
> 调用ThreadPoolExecutor.`execute(Runnable)`来提交一个线程到线程池中，线程池会根据当前的**Worker**线程数和**Queue**中的容量来确定操作，有以下三种情况：
>- `第一种情况` 如果运行的**worker**线程少于 corePoolSize，则始终首选添加新的**worker**线程，而不插入到**Queue**中。
>- `第二种情况` 如果运行的**worker**线程等于或多于 corePoolSize，则 始终首选将请求加入**Queue**中，而不添加新的**worker**线程。
>- `第三种情况` 如果无法将请求加入**Queue**，则创建新的**worker**线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。

```java
 public void execute(Runnable command) {
      if (command == null)
           throw new NullPointerException();
       int c = ctl.get();
       /**第一种情况*/
       if (workerCountOf(c) < corePoolSize) {
           if (addWorker(command, true))
               return;
           c = ctl.get();
       }
       /**
        *第二种情况，此处需要做一次double check，因为你插入到queue的时候
        *有可能线程池已经shutdown或者没有worker线程监听了，
        *所以需要check线程池的状态和wokerCount的值。
        */
       if (isRunning(c) && workQueue.offer(command)) {
           int recheck = ctl.get();
           if (! isRunning(recheck) && remove(command))
               reject(command);
           else if (workerCountOf(recheck) == 0)
               addWorker(null, false);
       }
       /**第三种情况*/
       else if (!addWorker(command, false))
           reject(command);
   }
```

>当出现第三种情况的时候，就是**Queue**被占满，这个时候我们会调用`reject(command)` ，这个方法会调用构造函数中传入的RejectedExecutionHandler对task进行处理，默认是抛出RejectedException。_(问题1)_
>可以看到启动一个**worker**线程是通过`addWorker` 方法实现的，下面看下`addWorker` 方法的具体实现，

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }
        /**上面的一段是通过CAS来对workerCount++操作，需要处理线程池状态关闭的情况*/
        
        //开始创建新的worker线程
        Worker w = new Worker(firstTask);
        Thread t = w.thread;
        
        /**mainLock在此处是对workers的操作做同步*/
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            int c = ctl.get();
            int rs = runStateOf(c);
            if (t == null ||
                (rs >= SHUTDOWN &&
                 ! (rs == SHUTDOWN &&
                    firstTask == null))) {
                decrementWorkerCount();
                tryTerminate();
                return false;
            }
            //往workerSet中添加worker
            workers.add(w);
            int s = workers.size();
            if (s > largestPoolSize)
                largestPoolSize = s;
        } finally {
            mainLock.unlock();
        }
        /**worker线程启动*/
        t.start();
        /**处处都需要考虑线程池的状态变化，当线程加入到workers中
         *还没有启动，此时状态变为stop，会导致当前线程未interrupt
         */
        if (runStateOf(ctl.get()) == STOP && ! t.isInterrupted())
            t.interrupt();

        return true;
    }
```

> 通过`addWorker`代码,可以看到worker线程是**Worker**类来实现的，下面我们看下worker类的代码：

```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;
        Runnable firstTask;
        //统计worker执行的task数目
        volatile long completedTasks;
        Worker(Runnable firstTask) {
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        public void run() {
            runWorker(this);
        }
        /**
         * 下面的方法是继承AbstractQueuedSynchronizer，利用单个原子int值来表示状态的大多数同步器
         * 0 表示未锁定状态
         * 1 表示锁定状态
         */
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }
    }
```

> 可以看到**Worker**类是ThreadPoolExceutor的一个内部类，是继承AbstractQueuedSynchronizer实现的一个互斥锁类，AbstractQueuedSynchronizer的详细解说可以参考下面几篇文章：
>- http://ifeve.com/introduce-abstractqueuedsynchronizer/
>- http://ifeve.com/jdk1-8-abstractqueuedsynchronizer/

> 可以看到**Worker**类的`run() `方法的执行体是调用了`runWorker`方法，下面我们看下`runWorker`方法的实现：

```JAVA
final void runWorker(Worker w) {
        /**创建worker时候的firstTask没有插入queue，在这直接运行*/
        Runnable task = w.firstTask;
        w.firstTask = null;
        boolean completedAbruptly = true;
        try {
            /**如果getTask为null则线程停止，否则根据策略操作，具体需要看getTask()的实现*/
            while (task != null || (task = getTask()) != null) {
                w.lock();
                clearInterruptsForTaskRun();
                try {
                    beforeExecute(w.thread, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

> **`runWorker`**方法是轮训获取task，然后执行task的`run`方法，当程序获取task的值为null时，此**worker**线程就表示终止_(问题2)_。
> 所以我们要让**worker**线程的生命周期的结束，只需要将`getTask()`的返回值设置为null就可以了，接下来我们看下`getTask()`的一些实现。

> **钩子 (hook) 方法**
        可重写的 beforeExecute(java.lang.Thread, java.lang.Runnable) 和 afterExecute(java.lang.Runnable, java.lang.Throwable) 方法，这两种方法分别在执行每个任务之前和之后调用。

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            boolean timed;      // Are workers subject to culling?

            for (;;) {
                int wc = workerCountOf(c);
                /**判断线程池是否设置了timeout，如果设置了则会回收线程，
                 * 回收条件是allowCoreThreadTimeOut || workerCount > corePoolSize
                 */
                timed = allowCoreThreadTimeOut || wc > corePoolSize;
                /**出现timeout就会设置timedOut=true，可以判断是否需要回收*/
                if (wc <= maximumPoolSize && ! (timedOut && timed))
                    break;
                if (compareAndDecrementWorkerCount(c))  //返回null，就表示worker回收
                    return null;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
            }

            try {
                //OLL是带超时获取，take是阻塞获取
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

> `getTask`是从**Queue**中获取task，可以看

>```JAVA
Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
```

>这段代码是根据用户是否设置了超时策略用不同的方法来从**Queue**中抓取task，简单说下`pool`与`take`的区别：
 >|           | 抛出异常      |    特殊值 | 阻塞  |超时|
 | :-------- | :-------- | --------:| :--: |
 | 插入   add(e)| offer(e)|put(e) |  offer(e, time, unit )  |
 | 移除   remove()| poll()|   take()|  poll(time, unit)  |
 | 检查   element()|  peek()|   不可用 |不可用 |
>`poll`是在固定时间如果没有则返回null，那么可以通过这个null来确定线程是否有闲置，是否需要回收部分**worker**线程。       

    

