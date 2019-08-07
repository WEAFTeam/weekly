---
title: Android线程池简介
description: Android线程池简介
tags:
  - ANDROID
author:
  - 夏沫
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/android.jpg'
category: ANDROID
date: '2019-07-22 02:04:44'
---
# 1:线程池的优点
1：相比new thread 来说，线程池复用线程，减少频繁创建的线程的开销，性能更佳；
2：可有效控制最大并发数，避免线程过大抢占资源导致线程阻塞；
3：可以对线程进行有效管理，提供定时，定期执行，并发数控制等。

# 2：ThreadPoolExccutor
ThreadPoolExecutor是jdk5.0后自带的线程池，通过构造方法来配置线程池参数。
首先我们看下具体的都有哪些构造：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019071515352670.png)
从上面的图片我们可以看到，具体有四种方法，我们拿最长的也就是六个参数来说。

```
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
**参数详解**：
**corePoolSize**：线程池的核心线程数。默认情况下，核心线程即使没有任务执行，也会存在，除非我们设置**allowCoreThreadTimeOut**为true，此时闲置的核心线程也会有超时策略，这个超时时间由keepAliveTime参数来决定。
**maximumPoolSize**：最大线程数。等于核心线程+非核心线程。非核心线程当核心线程数达到最大值，且线程池有空余时被创建，当线程池超过最大线程数量时，其余的线程会被阻塞。
**keepAliveTime**：非核心线程的超时时间，当allowCoreThreadTimeOut=true时，也作用于核心线程。
**TimeUnit**：枚举时间单位。
```
 TimeUnit.NANOSECONDS;
 TimeUnit.MICROSECONDS;
 TimeUnit.MILLISECONDS;
 TimeUnit.SECONDS;
 TimeUnit.MINUTES;
 TimeUnit.HOURS;
 TimeUnit.DAYS;
```
**workQueue**：线程池中的任务队列。用于保存等待执行的任务的阻塞队列；具体有以下几种：

**SynchronousQueue**（直接提交或者说同步任务队列）：SynchronousQueue内部仅允许容纳一个元素。当一个线程插入一个元素后会被阻塞，除非这个元素被另一个线程消费。
**ArrayBlockingQueue**（有界队列）：创建队列时指定最大值，如果线程数小于核心线程数，则创建核心线程，如果大于核心线程数，则首先加入等待队列，如果等待队列也满了，则判断是否大于核心线程小于最大线程则创建非核心线程，如果大于核心线程，则执行异常策略。内部是由数组实现，数组的大小在构造函数时指定，以后不再改变。
**LinkedBlockingQueue**（无界任务队列）：构造函数中指定大小，则有界，没有指定大小，则默认为Interger.MAX_VALUE;内部对象按FIFO顺序执行。
**PriorityBlockingQueue**（优先级任务队列）：类似于LinkedBlockingQueue，但是内部执行顺序按优先级（依据对象的自然排序顺序或者是构造函数所带的Comparator决定的顺序）来执行。


**threadFactory**：执行程序在创建新的线程时使用的工厂。
**handler** ：RejectedExecutionHandler ，线程池对拒绝任务的处理策略(默认抛出异常)。
主要分为以下四种：
AbortPolicy：直接抛出异常，默认情况。如果是：corePoolSize < 0，keepAliveTime < 0，maximumPoolSize <= 0，maximumPoolSize < corePoolSize会抛出IllegalArgumentException；如果是workQueue=null ，threadFactory=null，handler=null是会抛出NullPointerException。
CallerRunsPolicy：只用调用者所在线程来运行任务。
DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
DiscardPolicy：不处理，丢弃掉。

# 3：线程池分类
java中使用Excutors来提供四种线程池：

## 3.1：CachedThreadPool
通过Executors的newCachedThreadPool()方法来创建，
```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
上面是newCachedThreadPool的源码，我们可以看到其实还是通过new ThreadPoolExecutor来实现线程池管理，最大线程数是 Integer.MAX_VALUE。
内部队列使用的是SynchronousQueue，核心线程数为0，所以CachedThreadPool所有的线程都是非核心线程，当有新任务时，先判断是否有空闲线程，没有时就创建新的线程。线程池的大小超过了处理任务所需要的线程，则那么就会回收部分空闲（60秒不执行任务）的线程，所以CachedThreadPool适合执行大量且耗时小的任务。





## 3.2：FixedThreadPool
通过Executors的newFixedThreadPool()方法创建，一共有两种传参，如下：

```
 public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

 public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```
由代码可以看出最大线程数和核心线程数都是我们传入的nThreads。
内部队列时LinkedBlockingQueue。所以FixedThreadPool是固定线程数量的线程池，并且**所有的线程都是核心线程**。



## 3.3：ScheduledThreadPool
通过Executors的newScheduledThreadPool()方法创建，

```
  public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
 public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }

```
同样是不限制最大线程数的线程池，该线程池可以执行定时和周期性任务。使用队列为DelayedWorkQueue。DelayedWorkQueue实现了BlockingQueue接口，也就是一个阻塞队列，内部是基于堆的数据结构。


## 3.4：SingleThreadExecutor 
创建一个单线程的线程池。通过Executors的newSingleThreadExecutor()方法实现。
```
 public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```
这个线程池内部只有一个线程，当这个线程因为异常关闭时，会重新创建一个线程替代它，这个线程池保证任务的执行顺序（按照提交顺序执行）。内部队列使用LinkedBlockingQueue。

# 4：线程池状态

```
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```
**RUNNING**    ：线程池初始状态就是RUNNING    ，也就是线程创建后的状态。

**SHUTDOWN**：线程处于SHUTDOWN时，不接受新任务，但可以处理已添加的任务，执行shutdown()后，状态由RUNNING    变成SHUTDOWN;

**STOP**：线程池不接受新任务，也不处理已接收的任务，并且会中断现有任务。执行shutdownNow()方法，状态由RUNNING    或者SHUTDOWN变成STOP。
**TIDYING**    ：所有任务任务已终止，任务数量为0时，进入该状态。
当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。 
当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。
**TERMINATED** ：线程池彻底终止，就变成TERMINATED状态。 
处于TIDYING状态，执行terminated()后就会进入该状态。

# 5：常用方法

## 5.1：execute方法

```
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
       //将corePoolSize设置为线程池的最大线程数，将command传入，
       //并再次获取有效线程数，如果依然小于corePoolSize，则创建新线程，
       //结束execute方法，如果大于等于corePoolSize，继续执行
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //如果是处于running状态，offer，如果阻塞队列未满，将command加入阻塞队列并且返回true。
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //再次判断线程池运行状态，如果处于未运行状态，并且将command移除阻塞队列，
            if (! isRunning(recheck) && remove(command))
                reject(command);
                //再次计算workerCount，如果等于0则将最大线程数设置成maximumPoolSize，并且创建一个线程但是不启动
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //此时，线程池有两种情况
        //1：线程池状态不属于RUNNING
        //2：线程池处于RUNNING状态，但是有效线程数大于核心线程数，并且阻塞队列已满
        //创建新的线程，如果返回true，execute执行成功，否则返回执行拒绝策略，默认抛出异常
        else if (!addWorker(command, false))
            reject(command);
    }
```
根据以上代码：
:首先了解下ctl，根据源码我们可以找到变量。
```
 private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
AtomicInteger 类是继承Number实现Serializable接口。
这个类可以原子更新int的值，不能使用integer替代。
一个ctl变量包含了两部分信息：运行状态（runstate）和线程运行数量（workcount）。int 32位，高三位代表运行状态，低29位代表运行数量。

通过workerCountOf获取当前有效线程数，如果小于核心线程数，则执行
addWorker方法，接着我们简单了解下addWorker方法：

```
/**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
    //retry无限循环，线程状态处于running时，只有当有效线程成功+1才会退出循环，也就是
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //1：当前线程处于stop或者tiding或者terminated时直接返回false
            //2: 当前线程处于SHUTDOWN 状态，firstTask 为null或者阻塞队列为空直接返回false
            //此时，没有创建新线程，提交的任务也未被执行。
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
               
                int wc = workerCountOf(c);
                //如果有效线程数大于或者等于理论最大容量或者实际设置的最大容量，就直接返回false，同样未创建线程也未执行任务。
                //core=true，最大为核心线程数，否则就是maximumPoolSize；
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                 // 有效线程数加一
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
			//使用worker创建线程对象
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
总体总结下execute方法具体的判断逻辑：
1：如果线程池有效线程数小于核心线程数，则创建新的线程。
2：如果线程池有效线程数大于等于核心线程数，但是阻塞队列未满，则将任务添加进入阻塞队列。
3：如果线程池有效线程数大于等于核心线程数，但是小于线程池最大线程数，则创建线程。
4：如果有效线程数大于线程池最大线程数，并且阻塞队列已满，则执行拒绝策略，默认抛出异常。

**线程池关闭**
从上面的线程池状态中，我们知道有两个方法shutdown() 与shutdownNow() ；
那么具体是怎么实现的呢？我们看下源码：
```
 public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
             // 线程池状态设为SHUTDOWN，如果已经是shutdown<stop<tidying<terminated，
             //也就是非RUNING状态则直接返回 
            advanceRunState(SHUTDOWN);
            //中断空闲的没有执行任务的线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```
接着是shutdownNow（）：
```
 public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // STOP状态：不再接受新任务且不再执行队列中的任务。
            advanceRunState(STOP);
            //中断所有任务
            interruptWorkers();
            //清空任务队列
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
这两个方法都可以直接调用，在触发垃圾回收时，也会调用shutdown方法，因为ThreadPoolExecuter重写了finalize方法。
```
 /**
     * Invokes {@code shutdown} when this executor is no longer
     * referenced and it has no threads.
     */
    protected void finalize() {
        shutdown();
    }
```

**线程中断**
线程池中线程的中断方法也有两个：
1：interruptIdleWorkers：中断没有执行任务的线程
2：interruptWorkers：中断所有正在运行的线程。

```
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                //线程未中断，tryLock()内部执行tryAcquire()（尝试加锁）；
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

```
interruptWorkers()是中断所有正在运行的线程。
```
   private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }

 void interruptIfStarted() {
            Thread t;
            //线程状态不为中断
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
```
**扩展：**
如何手写一个线程池？
首先分析下需要的几个核心点：
1：线程池需要设置核心线程数和最大线程数
2：线程池需要设置任务队列。
3：线程并发锁
4：线程工厂以及线程状态等。
根据threadPoolExcutor定义线程池的相关参数：

```
//判断线程池是否正在工作
private volitile boolean RUNNING=true;
//阻塞队列
private static BlockingQueue<Runnable> queue=null;
//实际工作的工作集，使用hashSet表示没有重复的工作任务
private final HashSet<Worker> workers = new HashSet<Worker>();
//线程集
private final ArrayList<Thread> threadList = new ArrayList<Thread>();
//核心线程数
int poolSize = 0;
//空闲线程数
int coreSize = 0;
```
然后开始编写线程池的入口excute。

```
public void excute（Runnable runnable ）{
//判空
  if (runnable == null) throw new NullPointerException();
     //如果核心线程数大于空闲线程数，增加线程
       if(coreSize < poolSize){         
           addThread(runnable);                
       }else{
           //System.out.println("offer" +  runnable.toString() + "   " + queue.size());
           try {
               queue.put(runnable);                //coreSize>poolSzie 加入阻塞队列中去
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
}
```
接着我们编写创建线程的方法：

```
//创建线程，修改工作线程数量，启动线程
 public void addThread(Runnable runnable){
       coreSize ++;                                 //正在工作的线程+1
       Worker worker = new Worker(runnable);        //创建worker
       workers.add(worker);                         
       Thread t = new Thread(worker);              
       threadList.add(t);                          
       try {
           t.start();
       }catch (Exception e){
           e.printStackTrace();
       }
 
   }
```
编写shutdown方法

```
public void shutdown() {
	//更改线程状态
       RUNNING = false;
       if(!workers.isEmpty()){
       		//遍历工作线程，将工作集阻塞
           for (Worker worker : workers){
               worker.interruptIfIdle();
           }
       }
       Thread.currentThread().interrupt();
   }
```



