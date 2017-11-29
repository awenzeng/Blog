---
layout: post
title: "Android消息机制源码解析(Handler)"
date: 11/21/2017 3:44:59 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
	- Android消息机制源码解析
---
---
Android消息机制，其实也就是Handler机制，主要用于UI线程和子线程之间交互。众所周知，一般情况下，出于安全的考虑，所有与UI控件的操作都要放在主线程即UI线程中，而一些耗时操作应当放在子线程中。当在子线程中完成耗时操作并要对UI控件进行操作时，就要用Handler来控制。另外，Android系统框架内，Activity生命周期的通知等功能也是通过消息机制来实现的。本篇博文主要是想通过Handler源码解析，来加深我自己对Android消息机制的理解。

# 一、Handler使用

使用例子：
```java
private Handler handler = new Handler(){//1.Handler初始化,一个匿名内部类
    @Override
    public void handleMessage(Message msg) {
         super.handleMessage(msg);
         textView.setText("对UI进行操作");
    }
};
@Override
protected void onCreate(Bundle savedInstanceState){
       super.onCreate(savedInstanceState);
       setContentView(R.layout.activity_main);
       textView = (TextView) findViewById(R.id.mytv);
       new Thread(new Runnable() {
           @Override
           public void run() {
               //模拟耗时操作
               SystemClock.sleep(3000);
               handler.sendMessage(new Message());//2.在子线程中sendMessage();
           }
       }).start();

   }
```
**1.我们先来看看，Handler初始化。**
Handler初始化的同时，实现了消息处理方法handleMessage()。查看Handler源码
<!-- more -->
```java
    final MessageQueue mQueue;
    final Looper mLooper;
    final Callback mCallback;
    /**
     * Default constructor associates this handler with the queue for the
     * current thread.
     *
     * If there isn't one, this handler won't be able to receive messages.
     */
    public Handler() {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();//3.核心代码。获取一个Looper
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;//4.核心代码。从Looper获取一个消息队列
        mCallback = null;
    }
```
在源码中，Handler定义了一个MessageQueue消息队列mQueue和一个Looper对象mLooper，并都进行了初始化，分别对mQueue和mLooper进行了赋值，其中mLooper是通过Looper.myLooper()赋值，mQueues是Looper中的mQueue。通过了解，知Looper.myLooper()是一个静态方法。让我们进入Looper类看看
```java
/**
  * Class used to run a message loop for a thread.  Threads by default do
  * not have a message loop associated with them; to create one, call
  * {@link #prepare} in the thread that is to run the loop, and then
  * {@link #loop} to have it process messages until the loop is stopped.
  * 
  * <p>Most interaction with a message loop is through the
  * {@link Handler} class.
  * 
  * <p>This is a typical example of the implementation of a Looper thread,
  * using the separation of {@link #prepare} and {@link #loop} to create an
  * initial Handler to communicate with the Looper.
  *
  * <pre>
  *  class LooperThread extends Thread {
  *      public Handler mHandler;
  *
  *      public void run() {
  *          Looper.prepare();
  *
  *          mHandler = new Handler() {
  *              public void handleMessage(Message msg) {
  *                  // process incoming messages here
  *              }
  *          };
  *
  *          Looper.loop();
  *      }
  *  }</pre>
  */
public class Looper {
    private static final String TAG = "Looper";

    // sThreadLocal.get() will return null unless you've called prepare().

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;  // guarded by Looper.class

    final MessageQueue mQueue;
    final Thread mThread;
    volatile boolean mRun;

    private Printer mLogging;

     /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
       ......
    }

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
   ......
}
```
从Looper源码的注释中，我们知道**Looper是一个专门为线程提供消息循环的类，通过调用prepare()和loop()就可以为线程提供一个消息循环机制。**线程本来是没有消息循环机制的，想要消息循环机制就必须自己建立。如：
```java
   class LooperThread extends Thread {
        public Handler mHandler;
  
        public void run() {
            Looper.prepare();
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };
            Looper.loop();
        }
   }
```
在Looper源码中，有两个方法prepare()和prepareMainLooper()对Looper进行了初始化,Looper.myLooper()核心代码为sThreadLocal.get()，主要也是从sThreadLocal中取值。两个初始化方法的源码为
```java
     /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
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
从源码中知道prepare()创建的Looper为允许退出循环的，而prepareMainLooper()方法创建的是不应许退出循环的，通过分析，**很明显知道prepare()方法创建的是一般线程的Looper,而通过而prepareMainLooper()创建的，就是主线程消息循环的Looper。**

现在，虽然我们知道了Handler中对MessageQueue队列和Looper进行了赋值，但是Looper啥时候通过prepareMainLooper()初始化的呢？什么是开始调loop()开始循环的呢？这里我们先停一下，后面我们会说道。

**2.我们再看例子中的注释方法，在子线程中handler.sendMessage(message)**

我们继续看Handler源码
```java
    ......
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    public boolean sendMessageAtTime(Message msg, long uptimeMillis)
    {
        boolean sent = false;
        MessageQueue queue = mQueue;
        if (queue != null) {
            msg.target = this;//1.对Message中的target赋值Handler
            sent = queue.enqueueMessage(msg, uptimeMillis);//2.向循环队列中，加入消息
        }
        else {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
        }
        return sent;
    }
....
```
阅读Handler源码知，发送消息的方法还有许多种，sendMessage()是其中一种，如果还想具体了解还有哪些，可以下载Handler源码看一下，这里就不一一介绍了。**从上面三个方法中我们了解到方法sendMessageAtTime()是最后调用的，这个方法主要是，对Message的target赋值为发送主体Handler，并把Message加入消息队列MessageQueue中，等待消息队列循环处理。**

Handler发送主体为Message，Message是啥呢？Message主要就是对一些数据做封装处理，其中有int变量what,arg1,arg2,Object变量obj等，具体可以查看Message源码，这里就不详细说了。

# 二、Looper的创建及循环机制
上面说到，Looper的建立有两种方式prepare()和prepareMainLooper()，其中prepare建立的为一般子线程Looper，可以取消循环；而prepareMainLooper()建立的为主线程的Looper，不可以取消循环。到底而prepareMainLooper建立的是不是主线程循环呢？让我们继续分析

**1.主线程Looper建立**

主线程即UI线程，说到UI线程，我们知道应用程序一启动，主(UI)线程就开始启动，而线程的建立必须要在进程的基础上。**通过对Android应用程序启动的分析，我们知道，应用程序启动，首先会通过Zygote复制自身fork出一个进程，然后再由进程创建一个主线程，主线程的建立和ActivityThread息息相关，通过分析，知ActivityThread的main方法就是应用程序启动的入口。**具体可以参考：[Android应用程序进程启动过程（前篇）](http://liuwangshu.cn/framework/applicationprocess/1.html)

让我们来看一下ActivityThread类的main方法：
```java
 public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();//1.主线程Looper创建
        if (sMainThreadHandler == null) {
            sMainThreadHandler = new Handler();
        }

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();//2.主线程Looper循环

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
从源码知道，正如我们想的那样prepareMainLooper()建立的Looper就是主线程的Looper。

**2.Looper的消息循环**

从上面ActivityThread的main方法中，我们发现Looper.loop()消息循环方法。Looper是怎么循环的，这里让我们来看一下Looper.loop()
```java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
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

        for (;;) {//for循环
            Message msg = queue.next(); //从消息队列中取值
            if (msg == null) {//消息为空就返回
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);//分发消息

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

            msg.recycle();
        }
    }
```
从loop()源码中我们知道，建立了一个for循环从消息队列中取数据，然后通过msg.target.dispatchMessage(msg)分发消息，从前面我们知道target就是handler,这里我们再看一下Handler的消息分发方法dispatchMessage()
```java
    /**
     * Handle system messages here.
     */
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
在这里我们就看到，Handler最后会调用handleMessage()方法，只要message中callback为空，就是调用handleMessage(),从而实现消息的处理。

到这里，我们Android Handler消息分发机制解析就分解完了。但这里需要注意一下的是，在loop循环中，如果消息为空就会跳出循环，而我们的主线程Looper循环应该是死循环才对。针对这个问题，我们继续深入源码看一下，前面说prepare()和prepareMainLooper()是两种建立Looper的方式，两者的区别是一个是可取消循环的，一个是不可以取消循环的，这里让我们再来看看一下Looper的源码
```java

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mRun = true;
        mThread = Thread.currentThread();
    }

```
通过查看源码发现，是否可以取消消息循环，主要控制是MessageQueue里面，这里我们可以知道，主线程的消息循环控制应该就在 queue.next()方法里,好了，让我们来看MessageQueue的next方法
```java
  final Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;

        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(mPtr, nextPollTimeoutMillis);//1.核心代码

            synchronized (this) {
                if (mQuiting) {
                    return null;
                }
                
            .......省略代码，获取消息队列中的Message      

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
在next()方法中，有一个原生方法nativePollOnce(),它的作用是干啥的呢？是不是就是控制主线程循环的呢？**通过进一步阅读C++源码，我们知道这里是利用Linux系统中epoll_wait方法来进行阻塞，形成一个等待状态，也就是说，当消息队列中消息为空时，nativePollOnce()方法不会返回，会进行阻塞，形成一个等待状态，等有新消息进入消息队列，才会返回，从而获取消息。**这里我们也来看一下消息队列的插入方法
```java
  final boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        boolean needWake;
        synchronized (this) {
            if (mQuiting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }
            msg.when = when;
            Message p = mMessages;
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
        }
        if (needWake) {
            nativeWake(mPtr);//核心代码
        }
        return true;
    }
```
在消息队列中加入消息之后，会调用一个原生方法 nativeWake()，这个原生的C++的方法，也就是通知nativePollOnce()返回的方法，通过方法nativeWake和nativePollOnce的一唱一和，从而实现主线程的消息队列的无限循环。

具体C++代码是怎么实现的，这里推荐一篇博文[从源码角度分析native层消息机制与java层消息机制的关联](http://blog.csdn.net/andywuchuanlong/article/details/48179165)。

好了，分析就到这里了。

# 三、总结
Android消息分发机制，也就是Handler处理消息机制。流程如下：

- 1.应用程序在启动的时候，通过Zygote复制自身fork出应用程序的进程，然后该进程又以ActivityThread创建主线程。
- 2.主线程启动时，在ActivityThread的main方法中初始化了Looper和执行消息队列的循环。
- 3.使用过程中，Handler初始化，获取了主线程的Looper和消息队列MessageQueue，并实现消息处理方法handlerMessage
- 4.Handler通过sendMessage方法将消息插入消息队列
- 5.通过Looper消息队列的循环，从而执行处理方法，实现了UI线程和子线程之间的交互。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 四、相关及参考文档
[Android应用程序进程启动过程（前篇）](http://liuwangshu.cn/framework/applicationprocess/1.html)

[Android消息机制学习笔记](https://zhuanlan.zhihu.com/p/25222485?refer=levent-j)

[从源码角度分析native层消息机制与java层消息机制的关联](http://blog.csdn.net/andywuchuanlong/article/details/48179165)。

[ActivityThread](http://blog.csdn.net/zhangfei2018/article/details/46518615)

[Java单链表、双端链表、有序链表实现](http://blog.csdn.net/a19881029/article/details/22695289)

