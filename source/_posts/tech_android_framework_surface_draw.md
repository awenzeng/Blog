---
layout: post
title: "Android显示原理源码分析"
date: 3/5/2018 8:42:59 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
在博文[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)中，我们对Android的显示原理简单的说了一下，但没有深入。在博文中我们只知道Choreographer(舞蹈指挥者)只是post了一个操作，但后面到底怎么执行的？按啥逻辑执行的？我们都不清楚，作为一个喜欢刨根问底的程序员，是必须要分析分析的。

在开始分析之前，我们需要了解一些概念，如：

- **ViewRootImpl:**是View的绘制的辅助类，所有View的绘制都离不开ViewRootImpl。
- **Choreographer：**是"舞蹈指挥"者，控制同步处理输入(Input)、动画(Animation)、绘制(Draw)三个UI操作。
- **DisplayEventReceiver：**是一个抽象类，主要是接收显示绘制帧的垂直脉冲vsync,从而开始绘制帧。
- **FrameDisplayEventReceiver：** Choreographer的内部类，也是DisplayEventReceiver具体实现类。

# 一、Android的显示原理
**Android的显示过程：**
- i.应用层通过执行View三大绘制流程，把数据缓存在Surface上；
- ii.应用层通过跨进程通信机制，将数据传给系统层的SurfaceFlinger服务，SurfaceFlinger服务再通过硬件渲染到屏幕上；
- iii.通过Android刷新机制(每隔16ms会发出VSYNC信号),刷新界面。

<!-- more -->

** 1.应用层（Android应用程序）**
我们都知道一个Android的UI界面layout是整体一棵由很多不同层次的View组成的树形结构，它们存在着父子关系，子View在父View中，这些View都经过一个相同的流程最终显示到屏幕上。

关于View的绘制流程，在前面两篇博文[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)和[View绘制三大流程源码分析](http://blog.csdn.net/awenyini/article/details/79006432)中已经说过，这里就不再细说了，想了解的同学，可以回看一下前面的文章。

通过绘制流程，最后绘制数据都缓存到Surface上。

**2.系统层（SurfaceFlinger服务）**
Android是通过系统级进程中的SurfaceFlinger服务来把真正需要显示的数据渲染到屏幕上。SurfaceFlinger的主要工作是：

- 响应客户端事件，创建Layer与客户端的Surface建立连接。
- 接收客户端数据及属性，修改Layer属性，如尺寸、颜色、透明度等。
- 将创建的Layer内容刷新到屏幕上。
- 维持Layer的序列，并对Layer最终输出做出裁剪计算。

因应用层和系统层分别是两个不同进程，需要一个跨进程的通信机制来实现数据传输，在Android的显示系统中，使用了Android的匿名共享内存：SharedClient。每一个应用和SurfaceFlinger之间都会创建一个SharedClient，每个SharedClient中，最多可以创建31个SharedBufferStack，每个Surface都对应一个SharedBufferStack，也就是一个window。这意味着一个Android应用程序最多可以包含31个窗口，同时每个SharedBufferStack中又包含两个(<4.1)或三个(>=4.1)缓冲区。

![](http://img.blog.csdn.net/20170612225948542?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlwZW5nc2hpd28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

总结：应用层绘制到缓冲区，SurfaceFlinger把缓存区数据渲染到屏幕，两个进程之间使用Android的匿名共享内存SharedClient缓存需要显示的数据。

** 3.Android显示刷新机制**
Android系统一直在不断的优化、更新，但直到4.0版本发布，有关UI显示不流畅的问题仍未得到根本解决。

从Android4.1版本开始，Android对显示系统进行了重构，引入了三个核心元素：VSYNC, Tripple Buffer和Choreographer。VSYNC是Vertical Synchronized的缩写，是一种定时中断；Tripple Buffer是显示数据的缓冲区；Choreographer起调度作用，将绘制工作统一到VSYNC的某个时间点上，使应用的绘制工作有序进行。

Android在绘制UI时，会采用一种称为“双缓冲”的技术，双缓冲即使用两个缓冲区（在SharedBufferStack中），其中一个称为Front Buffer，另外一个称为Back Buffer。UI总是先在Back Buffer中绘制，然后再和Front Buffer交换，渲染到显示设备中。理想情况下，一个刷新会在16ms内完成（60FPS），下图就是描述的这样一个刷新过程（Display处理前Front Buffer，CPU、GPU处理Back Buffer。

i.没有VSYNC信号同步时

![](http://img.blog.csdn.net/20170612230450013?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlwZW5nc2hpd28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

但实际运行时情况并不一定如此

- 第一个16ms开始：Display显示第0帧，CPU处理完第一帧后，GPU紧接其后处理继续第一帧。三者都在正常工作。
- 进入第二个16ms：因为早在上一个16ms时间内，第1帧已经由CPU，GPU处理完毕。故Display可以直接显示第1帧。显示没有问题。但在本16ms期间，CPU和GPU却并未及时去绘制第2帧数据（前面的空白区表示CPU和GPU忙其它的事），直到在本周期快结束时，CPU/GPU才去处理第2帧数据。
- 进入第三个16ms，此时Display应该显示第2帧数据，但由于CPU和GPU还没有处理完第2帧数据，故Display只能继续显示第一帧的数据，结果使得第1帧多画了一次（对应时间段上标注了一个Jank），导致错过了显示第二帧。


通过上述分析可知，此处发生Jank的关键问题在于，为何第1个16ms段内，CPU/GPU没有及时处理第2帧数据？原因很简单，CPU可能是在忙别的事情，不知道该到处理UI绘制的时间了。可CPU一旦想起来要去处理第2帧数据，时间又错过了。 为解决这个问题，Android 4.1中引入了VSYNC，核心目的是解决刷新不同步的问题。

ii.引入VSYNC信号同步后

![](http://img.blog.csdn.net/20170612230222465?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlwZW5nc2hpd28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

在加入VSYNC信号同步后，每收到VSYNC中断，CPU就开始处理各帧数据。已经解决了刷新不同步的问题。
但是上图中仍然存在一个问题：CPU和GPU处理数据的速度似乎都能在16ms内完成，而且还有时间空余，也就是说，CPU/GPU的FPS（帧率）要高于Display的FPS。由于CPU/GPU只在收到VSYNC时才开始数据处理，故它们的FPS被拉低到与Display的FPS相同。但这种处理并没有什么问题，因为Android设备的Display FPS一般是60，其对应的显示效果非常平滑。

但如果CPU/GPU的FPS小于Display的FPS，情况又不同了，将会发生如下图的情况：

![](http://img.blog.csdn.net/20170612230210136?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlwZW5nc2hpd28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 在第二个16ms时间段，Display本应显示B帧，但却因为GPU还在处理B帧，导致A帧被重复显示。
- 同理，在第二个16ms时间段内，CPU无所事事，因为A Buffer被Display在使用。B Buffer被GPU在使用。注意，一旦过了VSYNC时间点，CPU就不能被触发以处理绘制工作了。

为什么CPU不能在第二个16ms处开始绘制工作呢？原因就是只有两个Buffer（Android 4.1之前）。如果有第三个Buffer的存在，CPU就能直接使用它，而不至于空闲。于是在Android4.1以后，引出了第三个缓冲区：Tripple Buffer。Tripple Buffer利用CPU/GPU的空闲等待时间提前准备好数据，并不一定会使用。

iii.引入Tripple Buffer后

引入Tripple Buffer后的刷新时序如下图：

![](http://img.blog.csdn.net/20170612230156826?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlwZW5nc2hpd28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

上图中，第二个16ms时间段，CPU使用C Buffer绘图。虽然还是会多显示A帧一次，但后续显示就比较顺畅了。

是不是Buffer越多越好呢？回答是否定的。由上图可知，在第二个时间段内，CPU绘制的第C帧数据要到第四个16ms才能显示，这比双Buffer情况多了16ms延迟。所以缓冲区并不是越多越好。

**注：2和3来源于[Android绘制优化----系统显示原理](http://blog.csdn.net/lipengshiwo/article/details/73143222)**

# 三、Android显示原理源码分析
我们接着[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)中Choreographer(舞蹈指挥者)post一个操作继续分析，我们来看相关源码，ViewRootImp中的scheduleTraversals()方法
```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            ......
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);//核心代码
            ......
        }
    }
```
这里主要mChoreographer(舞蹈指挥者)作了一个postCallback操作，主要Action为mTraversalRunnable,我们再来看此变量：
```java
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```
从DecorView绘制流程知，doTraversal()方法主要功能就是执行绘制流程，也就是我们上面说应用层，主要就是把绘制数据缓存到surface上。前面两篇博文已经介绍过了，这里就不介绍了。

我们具体来看看 mChoreographer.postCallback()方法，mChoreographer为ViewRootImp的属性变量，其初始化主要也就是在ViewRootImp的构造方法中，具体我们来看一下：
```java
   public ViewRootImpl(Context context) {
        super();
        .......
        mChoreographer = Choreographer.getInstance();
        .......
    }
```
这里主要用了单例模式来初始化mChoreographer，我们继续来看看Choreographer中的getInstance()方法
```java
    public final class Choreographer {
    .......
    // Thread local storage for the choreographer.
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();//1.
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            return new Choreographer(looper);//2.
        }
    };

    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
   .......
}
```
注释1处，通过Looper.myLooper()获取Looper，我们知道Choreographer主要是在ViewRootImpl的构造函数中初始化的，并且ViewRootImpl是运行在主线程中的，所以此处的Looper也即主线程的Looper。下面让我们继续来看看Choreographer中的postCallback()方法：
```java
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }
```
我们继续看看postCallbackDelayedInternal()方法
```java
 private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;

            //1.将一个遍历操作Action加入数组队列,callbackType = Choreographer.CALLBACK_TRAVERSAL
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            //2.根据延迟时间来执行不同的操作
            if (dueTime <= now) {
                scheduleFrameLocked(now);//3
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
```
此方法主要就是将操作Action加入数组队列mCallbackQueues中，然后通过判断延迟时间执行操作，下面我们继续来看一下注释3，方法scheduleFrameLocked()
```java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {//是否用VSYNC脉冲
                if (DEBUG) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }
                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```
由英文注释知，当用VSYNC脉冲时，看是否在Looper线程也即主线程，如果在直接执行，如果不在就利用Handler消息机制，发送消息，然后执行；如果不用VSYNC脉冲，也是利用handler消息机制发送MSG_DO_FRAME消息执行，我们先来看立即执行方法scheduleVsyncLocked(),然后再看handler消息处理方法
```java
    private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
```
这里的mDisplayEventReceiver为FrameDisplayEventReceiver，也即调用FrameDisplayEventReceiver的scheduleVsync()方法，这里我们知道：**FrameDisplayEventReceiver是绘制帧显示的接收器，专门接收系统层发送来的绘制消息**。我们来看一下FrameDisplayEventReceiver
```java
  private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper) {
            super(looper);
        }

        @Override
        public void onVsync(long timestampNanos, int frame) {
        .....
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```
由上知道FrameDisplayEventReceiver继承于抽象类DisplayEventReceiver，我们再来看抽象类的源码
```java
public abstract class DisplayEventReceiver {
    private static final String TAG = "DisplayEventReceiver";

    private final CloseGuard mCloseGuard = CloseGuard.get();

    private int mReceiverPtr;

    // We keep a reference message queue object here so that it is not
    // GC'd while the native peer of the receiver is using them.
    private MessageQueue mMessageQueue;

    //原生方法
    private static native int nativeInit(DisplayEventReceiver receiver,
            MessageQueue messageQueue);

    private static native void nativeDispose(int receiverPtr);
    private static native void nativeScheduleVsync(int receiverPtr);

    /**
     * Creates a display event receiver.
     *
     * @param looper The looper to use when invoking callbacks.
     */
    public DisplayEventReceiver(Looper looper) {
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }

        mMessageQueue = looper.getQueue();
        mReceiverPtr = nativeInit(this, mMessageQueue);
        mCloseGuard.open("dispose");
    }
    .......

    /**
     * Called when a vertical sync pulse is received.
     * The recipient should render a frame and then call {@link #scheduleVsync}
     * to schedule the next vertical sync pulse.
     * 当收到一个VSYNC脉冲时就会执行此方法
     *
     * @param timestampNanos The timestamp of the pulse, in the {@link System#nanoTime()}
     * timebase.
     * @param frame The frame number.  Increases by one for each vertical sync interval.
     */
    public void onVsync(long timestampNanos, int frame) {
    }

    /**
     * Schedules a single vertical sync pulse to be delivered when the next
     * display frame begins.
     */
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }

    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, int frame) {
        onVsync(timestampNanos, frame);
    }
}
```
从此类知，最后调用了原生方法nativeScheduleVsync()，系统发出一个VSYNC脉冲。

我们再来看非Looper线程的情况，Handler消息是怎么处理操作的，我们继续来看handler的处理方法：
```java
  private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME://直接执行绘制帧
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();//通过上面，我们知道这方法最后会发出一个VSYNC信号，从而触发绘制操作
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```
我们来看看doScheduleCallback(msg.arg1)方法
```java
    void doScheduleCallback(int callbackType) {
        synchronized (mLock) {
            if (!mFrameScheduled) {
                final long now = SystemClock.uptimeMillis();
                if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                    scheduleFrameLocked(now);//核心方法
                }
            }
        }
    }
```
此方法最后又回到我们前面的方法，又会按相关逻辑执行一遍。所以这里我们主要来看执行绘制帧的方法doFrame()
```java
 void doFrame(long frameTimeNanos, int frame) {

        ......
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        ......
    }
```
在doFrame()中，我们了解到在Choreographer中，主要就是同步处理输入(CALLBACK_INPUT)、动画(CALLBACK_ANIMATION)、绘制(CALLBACK_TRAVERSAL)三个UI操作，也即应用层绘制操作，doFrame()方法主要就是绘制帧。我们具体来看看doCallbacks()方法
```java
  void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            // We use "now" to determine when callbacks become due because it's possible
            // for earlier processing phases in a frame to post callbacks that should run
            // in a following phase, such as an input event that causes an animation to start.
            final long now = SystemClock.uptimeMillis();
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(now);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;
        }
        try {
            for (CallbackRecord c = callbacks; c != null; c = c.next) {//循环操作数组队列
                if (DEBUG) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
        }
    }
```
通过此方法知，主要就是执行数组队列中的run方法，从而实现View的绘制。

主动执行绘制操作的流程我们清楚了，但当系统层发出VSYNC信号，Android系统又是怎么接收的呢？我们在阅读FrameDisplayEventReceiver和DisplayEventReceiver源码时，通过注释发现，当FrameDisplayEventReceiver收到VSYNC信号时，就会调用onVsync()方法，我们来看看此方法：
```java
       @Override
        public void onVsync(long timestampNanos, int frame) {
            // Post the vsync event to the Handler.
            // The idea is to prevent incoming vsync events from completely starving
            // the message queue.  If there are no messages in the queue with timestamps
            // earlier than the frame time, then the vsync event will be processed immediately.
            // Otherwise, messages that predate the vsync event will be handled first.
            .......
            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / NANOS_PER_MS);
        }
```
这里msg.what没有赋值，我们知道默认值为0，而MSG_DO_FRAME的值也为0，也就说最后调用了doFrame()方法，所以当收到系统层发出VSYNC信号时，就会执行绘制帧的方法。所以当系统间隔16ms发出VSYNC信号脉冲，就会执行绘制方法doFrame()。

到此，Android显示原理的源码就分析完了。


**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 四、参考文档
[Android绘制优化----系统显示原理](http://blog.csdn.net/lipengshiwo/article/details/73143222)

[Android UI绘制原理(二)](http://blog.csdn.net/itluochen/article/details/52506173)

[Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d)

[Android SurfaceFlinger服务启动过程源码分析](http://blog.csdn.net/yangwen123/article/details/11890941)










