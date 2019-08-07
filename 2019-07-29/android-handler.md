---
title: Android Handler相关
description: Android Handler相关
tags:
  - ANDROID
author:
  - 夏沫
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/android.jpg'
category: ANDROID
date: '2019-07-25 14:15:39'
---
# 1：handler使用
handler有两个用途：
1：安排消息或者runnable在某个时间执行
2：将不同线程的消息排入消息队列

## 1.1：sendMessage
```
 //创建handler
    Handler handler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (msg.what == 1) {
                Toast.makeText(MainActivity.this, "刷新UI", Toast.LENGTH_SHORT).show();
            }
        }
    };


 class  MyThread extends Thread{

        @Override
        public void run() {
            super.run();
            //发送消息
            handler.sendEmptyMessage(1);
        }
    }
//启动线程
new MyThread.start();
```
上面代码就是sendMessage方法，我们在子线程中发送消息，在主线程接收消息，更新ui。
这里产生了一个问题：
1：子线程的消息如何变成了主线程？
这里先不做回答，后面统一解决这些问题。

## 1.2：post
```
  Handler mHandler = new Handler();
  mHandler.post(new Runnable() {
            @Override
            public void run() {
                Toast.makeText(MainActivity.this,"ui",Toast.LENGTH_SHORT).show();
            }
        });
```
当然除了post，我们还可以使用postAtTime(Runnable r, long uptimeMillis)定时发送等。

# 2：handler源码分析

## 2.1 handler构造

首先我们从handler的构造方法开始：

```
 public Handler() {
        this(null, false);
    }

    public Handler(Callback callback) {
        this(callback, false);
    }

    public Handler(Looper looper) {
        this(looper, null, false);
    }
    
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

 
```
可以看到当我们new handler(),也就是调用空构造时，执行的handler两个参数的构造方法，这个方法中主要初始化了looper，messageQueue，Callback。
接着我们看下是如何初始化的，首先Looper.myLooper()：
```
  public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
这里我们看到looper是从sThreadLocal中获取的，而sThreadLocal是ThreadLocal。
```
 static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
那这个ThreadLocal是什么东西呢。
简单翻译下源码中对这个类的描述：
```
该类提供局部线程变量，每个线程通过get或者set方法获取自己的变量副本。
```
接着我们看下get 和set方法

```
   public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
首先获取当前线程，通过getMap返回ThreadLocalMap ，
```
 ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
可以看到ThreadLocalMap 是从线程的threadLocals变量中获取。
而ThreadLocalMap的key传入的是this，也就是变量threadLocals。

```
public  class Thread implements Runnable {
	  ThreadLocal.ThreadLocalMap threadLocals = null;
}
```
set很简单，就是传入value，根据map是否为空来做new map还是直接set的处理：
```
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
 void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
 }
```
总结下：ThreadLocal的值是放入了当前线程的一个ThreadLocalMap实例中，所以只能在本线程中访问，其他线程无法访问。也就是说，在多线程处理同一个
ThreadLocal进行操作时，不影响其他线程，最终获取的还是当前线程的数据。
我们验证下这个结论：

```
public class MainActivity extends AppCompatActivity {

    ThreadLocal<Integer> threadLocal=new ThreadLocal<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        threadLocal.set(100);

        for (int i=0;i<10;i++){
            LocalThread localThread1=new LocalThread (i);
            localThread1.start();
        }
        try {
            Thread.sleep(3000);
            Log.e("xx","threadLocal-->"+threadLocal.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
	
    class LocalThread extends Thread{
        int pos;
        public LocalThread1(int size){
            pos=size;
        }
        @Override
        public void run() {
            super.run();
                threadLocal.set(pos);
                Log.e("xx","threadLocal-->"+threadLocal.get()+"--x="+(++s));
        }
    }
}

```
看下打印结果：
```
07-25 16:50:36.287 3575-3598/? E/xx: threadLocal-->1--x=1
07-25 16:50:36.287 3575-3597/? E/xx: threadLocal-->0--x=2
07-25 16:50:36.287 3575-3599/? E/xx: threadLocal-->2--x=3
07-25 16:50:36.287 3575-3600/? E/xx: threadLocal-->3--x=4
07-25 16:50:36.287 3575-3601/? E/xx: threadLocal-->4--x=5
07-25 16:50:36.287 3575-3602/? E/xx: threadLocal-->5--x=6
07-25 16:50:36.287 3575-3603/? E/xx: threadLocal-->6--x=7
07-25 16:50:36.287 3575-3604/? E/xx: threadLocal-->7--x=8
07-25 16:50:36.287 3575-3605/? E/xx: threadLocal-->8--x=9
07-25 16:50:36.297 3575-3606/? E/xx: threadLocal-->9--x=10
07-25 16:50:39.297 3575-3575/com.zh.myview E/xx: threadLocal-->100
```
可以看到虽然我们在不同线程操作threadLocal，但是通过get获取到的值是不会受到影响的。而且在经过set方法后，我们传入的value，也是和当前线程进行了绑定。
这里threadLocal就差不多分析完了，我们回到正题handler的初始化，从当前线程获取到looper后，我们看到源码中looper进行了判空操作：
```
 if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
```
意思大概是在调用Looper.myLooper方法之前必须调用Looper.prepare方法，否则就会抛出异常。
我们看下prepare做了什么操作：

```
 public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
好吧，两步操作：
1：判断不为空就抛出异常，所以，这里我们可以得到一个结论，一个线程只有一个looper。
2：new looper。

这里会有人问了，上面handler的空构造中调用myLooper时，没有使用prepare方法啊，为什么没有报错呢。
好吧，这里activity已经帮我们做了这个操作了，准备来说是activityThread中调用了，我们看下activityThread的main方法。
```
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
        // It will be in the format "seq=114"
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
瞅了半天发现没有prepare方法，但是有一个Looper.prepareMainLooper()：
```
  public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
这里looper.myLooper()结束。继续分析looper.mQueue;

```
final MessageQueue mQueue;
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
         }
```
我们知道在prepare调用了set，从而初始化了Looper，所以，mQueue这里就直接获取了。

## 2.2 sendMessage()相关
```
 public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
 public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
 }
  public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
可以看到不管是sendMessage还是sendEmptyMessage最后调用的都是sendMessageDelayed，而sendMessageDelayed内部调用的是sendMessageAtTime，所以我们来看看sendMessageAtTime内部做了哪些操作：

```
  public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
两步操作：获取当前的消息队列，将消息当做参数，传入enqueueMessage：
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
这里将this，也就是当前handler传给了message的target。
然后调用messagequeue的enqueueMessage方法：
```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

```
enqueueMessage方法主要进行message插入messagequeue的操作。
那怎么从messagequeue取出message呢？
答案是Looper.loop:

```
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }

```
大致先获取当前线程的looper，从looper中获取messagequeue。
然后通过for循环（死循环）取出queue中的message，取出的方法是queue.next().
其中处理的方法为:
```
 msg.target.dispatchMessage(msg);
```
我们知道在sendmessage时，通过handler的enqueueMessage方法将handler赋值给了msg.target。所以dispatchMessage还是handler的方法:
```
  public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
这里分为两种情况：
1：msg.callback ！=null，
```
 private static void handleCallback(Message message) {
        message.callback.run();
    }
```
callback是message的变量Runnable ，
也就是说最后调用的是runnable的run方法。
2：当msg.callback==null时，判断mCallback 是不是为null；mCallback是在handler的构造方法中传入，如果不为null，就判断传入callback的handleMessage。如果是true，则直接返回；
也就是说，如果我们在handler的创建时传入callback，并且重写handleMessage，将返回值改成true，handler就会被拦截。
流程最后会调用   handleMessage(msg);
```
 public void handleMessage(Message msg) {
    }
```
handleMessage是空方法，需要我们自己处理，也就是在这里处理ui更新等操作。

# 3：handler相关问题

## 3.1：子线程的消息如何变成了主线程？
这个问题在开头时，我们提出来，这里分析完源码我们应该可以总结出来了吧？

在handler发送message时，调用enquemessage与message绑定，并插入messagequeue，当执行looper.loop方法循环取出message时，通过msg.targer也就是handler的handlemessage处理，而我们无参创建的handler是主线程，

## 3.2：怎样实现一个带有消息循环(Looper)的线程？
在handler的构造中可以传入looper参数，所以我们先创建一个handler：

```
private Handler mHandler;
Looper myLooper;
public void createHandler(){
 	handler=new Handler(myLooper){
  		@Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
           //ui操作
        };
	}
}

```
接着开启子线程，

```
new Thread(new Runnable() {
        @Override
        public void run() {
           //创建Looper和MessageQueue对象
            Looper.prepare(); 
            // 获取当前线程下的Looper对象
            mLooper = Looper.myLooper(); 
            
            createHandler();
            // 开启Looper循环 
            Looper.loop(); 
        }
    }).start();
```

