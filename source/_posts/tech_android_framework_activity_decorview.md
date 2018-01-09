---
layout: post
title: "DecorView绘制流程源码分析"
date: 1/5/2018 5:24:04 PM 
comments: true
tags: 
	- 技术 
	- Android
	- DecorView绘制流程
	- Android框架源码解析
---
---
通过[Activiyt布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)和[Activiyt布局加载流程源码分析(II)](http://blog.csdn.net/awenyini/article/details/78964353)两篇博文，我们知道，首先，Activity的布局内容被加载进入装饰器DecorView中，然后WindowManager将DecorView添加到PhoneWindow中，也即Window中，最后ViewRootImpl对DecorView进行绘制操作，将其内容显示到手机上。但前两篇博文中，对于DecorView的绘制原理，没有作详细说明，所以本篇博文重在梳理这部分逻辑。

在开始分析之前，我们需要了解一些概念，如：

- **DecorView：**是PhoneWindow中的一个内部类，也是Window的顶级View，主要负责装载各种View和Activity布局。
- **ViewRootImpl:**是View的绘制的辅助类，所有View的绘制都离不开ViewRootImpl。
- **Choreographer：**是"舞蹈指挥"者，控制同步处理输入(Input)、动画(Animation)、绘制(Draw)三个UI操作。
- **DisplayEventReceiver：**是一个抽象类，主要是接收显示绘制帧的垂直脉冲vsync,从而开始绘制帧。
- **FrameDisplayEventReceiver：** Choreographer的内部类，也是DisplayEventReceiver具体实现类。

在说DecorView的绘制之前，我们先来说说Android的绘制原理，这样方便我们理解后面内容。

# 一、Android的绘制原理简介
Android系统每隔16ms会发出VSYNC信号重绘我们的界面(Activity)。为什么是16ms, 因为Android设定的刷新率是60FPS(Frame Per Second), 也就是每秒60帧的刷新率, 约合16ms刷新一次。如下图所示：
<!-- more -->
![](/assets/img/tech_android_draw_flow.png)



# 二、DecorView绘制原理分析
在Activity布局加载流程分析中，我们知道DecorView被添加进入了WindowManager,并且最后ViewRootImpl通过setView()方法开始绘制DecorView，所以下面我们就来看看ViewRootImpl的setView()方法
```java
 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;//1.DecorView赋值为mView
                mFallbackEventHandler.setView(view);
                ......
                
                requestLayout();//2.DecorView的绘制
                
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();

                    //3.Window的权限判断，主要是Window添加的控制，由于本篇博文重在DecorView绘制，所以这里将不会分析
                    res = sWindowSession.add(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mAttachInfo.mContentInsets,
                            mInputChannel);

                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
                
               ........
            }
        }
    }
```
首先，我们来关注一下注释1，这里主要是对mView进行赋值DecorView，mView是ViewRootImpl的属性变量，这里需要注意一下，因为后面绘制需要用到。我们再来看注释2，ViewRootImpl的requestLayout()方法，我们具体来看看其方法逻辑
```java
    public void requestLayout() {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();//核心方法
    }
```
这里我们直接来看核心方法scheduleTraversals()
```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            scheduleConsumeBatchedInput();
        }
    }
```
这里我们需要特别关注mChoreographer，即Choreographer类，从字面意思来说是“舞蹈指挥”者，是Android绘制原理的核心类，控制Android显示帧的绘制。关于Choreographer类，这里不做过多的分析，想了解其原理的同学，可以看看博文[Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d)。

由Android绘制原理，我们知道每隔16ms,Android系统就会发出垂直信号VSYNC脉冲重绘我们的界面，而Choreographer中postCallback()方法主要功能就是向系统添加回调并加入绘制帧，从而实现View的绘制。这里我们来看看添加的回调mTraversalRunnable
```java
   final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();//核心方法
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```
我们继续来看doTraversal()方法
```java
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "performTraversals");
            try {
                performTraversals();//核心方法
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```
我们继续分析方法performTraversals()
```java
   private void performTraversals() {
        
            .......//1.代码省略。省略主要内容，Surface和SurfaceHolder初始化及条件判断

            if (!mStopped) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerImpl.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged) {
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    
                 
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);//2.执行View的宽高测量
    
                    ........
    
                    layoutRequested = true;
                }
            }
        }

        final boolean didLayout = layoutRequested && !mStopped;
        boolean triggerGlobalLayoutListener = didLayout
                || attachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {

            performLayout();//3.执行View的布局

            .....
        }

        ......

        boolean cancelDraw = attachInfo.mTreeObserver.dispatchOnPreDraw() ||
                viewVisibility != View.VISIBLE;

        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                ........
                performDraw();//执行View绘制
            }
        } else {
           ........
        }
    }
```
此方法，可以说是Android系统绘制的核心方法。**View绘制原理的三大流程:View的测量onMeasure -> View的布局onLayout -> View的绘制onDraw，**都在此方法中提现出来了。下面我们一一来看一下相关方法，首先我们来看一下performMeasure()方法
```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);//核心方法
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
根据上面的分析，我们知道mView就是DecorView,所以这里就是调用DecorView的measure()方法。由[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)博文，我们知道DecorView是继承至FrameLayout,而FrameLayout又继承至ViewGroup，ViewGroup又继承至View,所以这里的measure()方法就是调用View中的measure()方法，具体怎么调用，这里不细说了，想了解的同学可以看看这篇博文[View的绘制原理](http://blog.csdn.net/u014316462/article/details/52054352)。下面让我们来看看performLayout()方法
```java
 private void performLayout() {
        mLayoutRequested = false;
        mScrollMayChange = true;

        final View host = mView;
        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
            Log.v(TAG, "Laying out " + host + " to (" +
                    host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
        }

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
这里逻辑与测量measure类似，也就是调用DecorView的layout方法，具体View的布局控制细节略。我们再来看看performDraw()方法
```java
 private void performDraw() {
        if (!mAttachInfo.mScreenOn && !mReportNextDraw) {
            return;
        }

        final boolean fullRedrawNeeded = mFullRedrawNeeded;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            draw(fullRedrawNeeded);//核心方法
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        ......
    }

```
我们继续来看看draw()方法
```java
 private void draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        .......

        if (!dirty.isEmpty() || mIsAnimating) {
            if (attachInfo.mHardwareRenderer != null && attachInfo.mHardwareRenderer.isEnabled()) {
                // Draw with hardware renderer.
                mIsAnimating = false;
                mHardwareYOffset = yoff;
                mResizeAlpha = resizeAlpha;

                mCurrentDirty.set(dirty);
                mCurrentDirty.union(mPreviousDirty);
                mPreviousDirty.set(dirty);
                dirty.setEmpty();

                //1.Hardware渲染(Hardware加速)
                if (attachInfo.mHardwareRenderer.draw(mView, attachInfo, this,animating ? null :mCurrentDirty)) {
                    mPreviousDirty.set(0, 0, mWidth, mHeight);
                }

            } else if (!drawSoftware(surface, attachInfo, yoff, scalingRequired, dirty)) {//2.Software渲染
                return;
            }
        }
        if (animating) {
            mFullRedrawNeeded = true;
            scheduleTraversals();
        }
    }
```
这里的绘制方法涉及到两种绘制方式，分别为Hardware渲染(硬件加速)和Software渲染，关于选择那种绘制方式，这里还需要回溯到ViewRootImpl的setView()方法,我们再来看看此方法
```java
 /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                ......

                if (view instanceof RootViewSurfaceTaker) {//1.mSurfaceHolder赋值
                    mSurfaceHolderCallback =
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    if (mSurfaceHolderCallback != null) {
                        mSurfaceHolder = new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                    }
                }

                ........

                if (mSurfaceHolder == null) {//2.是否需要硬件加速
                    enableHardwareAcceleration(mView.getContext(), attrs);
                }
            .......
       }
}
```
我们知道DecorView是实现了RootViewSurfaceTaker接口的，所以当View为DecorView时，就不会开启硬件加速，不会走Hardware渲染，而其他的View会选择Hardware渲染。因为WindowManager添加的View可能使DecorView，也可能不是DecorView，也可能是一般的View。我们来看看enableHardwareAcceleration()方法
```java
  private void enableHardwareAcceleration(Context context, WindowManager.LayoutParams attrs) {
        mAttachInfo.mHardwareAccelerated = false;
        mAttachInfo.mHardwareAccelerationRequested = false;

        if (mTranslator != null) return;

        final boolean hardwareAccelerated = 
                (attrs.flags & WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED) != 0;

        if (hardwareAccelerated) {
            if (!HardwareRenderer.isAvailable()) {
                return;
            }

            final boolean fakeHwAccelerated = (attrs.privateFlags &
                    WindowManager.LayoutParams.PRIVATE_FLAG_FAKE_HARDWARE_ACCELERATED) != 0;
            final boolean forceHwAccelerated = (attrs.privateFlags &
                    WindowManager.LayoutParams.PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATED) != 0;

            if (!HardwareRenderer.sRendererDisabled || (HardwareRenderer.sSystemRendererDisabled
                    && forceHwAccelerated)) {
                ........
                final boolean translucent = attrs.format != PixelFormat.OPAQUE;
                mAttachInfo.mHardwareRenderer = HardwareRenderer.createGlRenderer(2, translucent);//核心方法
                mAttachInfo.mHardwareAccelerated = mAttachInfo.mHardwareAccelerationRequested
                        = mAttachInfo.mHardwareRenderer != null;

            } else if (fakeHwAccelerated) {

                mAttachInfo.mHardwareAccelerationRequested = true;
            }
        }
    }
```
这方法主要是对mAttachInfo.mHardwareRenderer进行赋值，从而在performDraw()方法中可以执行绘制。下面我们来看看上面的绘制方式1，Hardware渲染(硬件加速)，由上知主要是通过attachInfo.mHardwareRenderer.draw()绘制，所以我们来看看HardwareRenderer中的draw()方法
```java
  @Override
        boolean draw(View view, View.AttachInfo attachInfo, HardwareDrawCallbacks callbacks,
                Rect dirty) {
            if (canDraw()) {
               .......

                    try {
                        .......

                        DisplayList displayList;//渲染列表

                        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "getDisplayList");
                        try {
                            displayList = view.getDisplayList();
                        } finally {
                            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                        }

                       .......

                        if (displayList != null) {
                            .....
                            try {
                                status |= canvas.drawDisplayList(displayList, mRedrawClip,
                                        DisplayList.FLAG_CLIP_CHILDREN);
                            } finally {
                                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                            }

                            .....

                            handleFunctorStatus(attachInfo, status);
                        } else {
                            view.draw(canvas);//核心方法
                        }
                    } finally {
                       ....
                    }

                    ......

                    return dirty == null;
                }
            }

            return false;
        }
```
当displayList为空的时候，也就会调用 view.draw(canvas)方法，即DeocorView的draw()方法。关于DisplayList这里也不细说，它主要是View中的显示列表记录，具体作用这里不作详述了。我们再来看看第二种绘制方式SoftWare渲染,drawSoftware()方法
```java
 private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int yoff,
            boolean scalingRequired, Rect dirty) {

        // Draw with software renderer.
        Canvas canvas;
        try {
            .......

            canvas = mSurface.lockCanvas(dirty);

           ......
          
        } catch (Surface.OutOfResourcesException e) {
           .....
        } catch (IllegalArgumentException e) {
          .....
        }

        try {
            .......
            try {

               ......

                mView.draw(canvas);//核心方法

                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            } finally {
                if (!attachInfo.mSetIgnoreDirtyState) {
                    // Only clear the flag if it was not set during the mView.draw() call
                    attachInfo.mIgnoreDirtyState = false;
                }
            }
        } finally {
          .....
        }
        return true;
    }
```
这里发现，最后也还是调用DecorView的draw方法，具体流程也与measure和layout类似。可以说两种绘制方式最后也还是调用了View的draw方法，可以说是殊途同归。

到这里我们就把添加的回调绘制帧mTraversalRunnable这个说完了。上面说到，Choreographer通过postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null)方法向系统添加回调并加入绘制帧，然后Android系统通过16ms间隔脉冲实现帧的绘制，从而才将布局内容显示到手机上。

说到这里，DecorView的绘制流程我们就说完了。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 三、参考文档
[Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d)

[Android App卡顿分析，以及使用Choreographer进行帧率统计监测](https://www.jianshu.com/p/6a680186b95f)

[View的绘制原理](http://blog.csdn.net/u014316462/article/details/52054352)





