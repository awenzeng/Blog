---
layout: post
title: "Android事件分发机制源码分析"
date: 1/9/2018 7:17:17 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
昨天我们对[View绘制三大流程源码](http://blog.csdn.net/awenyini/article/details/79006432)已做了深入分析，所以关于View的绘制流程，我相信大家也有了一个大致的了解(如果不了解，请回看博文)。然而对于View，还有一个知识点，也是极其重要的，那就是View的事件分发机制(也即Android事件分发机制)。所以，今天我们就来谈谈View的事件分发机制，从源码的角度，跟随Touch事件流，走一遍流程。

在开始分析之前，我们需要了解一些概念，如一次Touch事件，可能包括下面三个事件：

- **MotionEvent.ACTION_DOWN：** 表示手指按下事件，一个事件的开始。
- **MotionEvent.ACTION_MOVE：** 表示手指移动事件，事件的持续移动。
- **MotionEvent.ACTION_UP：** 表示手指抬起事件，一个事件的结束。

# 一、View事件分发流程图
在具体分析之前，我们先来看一下事件分发流程图，以便我们更好的理解内容。图如下

<!-- more -->

![](/assets/img/tech_android_dispatch_event_flow.png)

# 二、View事件分发机制分析
由Android系统的启动，Lancher系统的启动相关知识，我们知道，当我们点击手机屏幕，主要是通过硬件传感器传输事件，传感器会将其Touch事件传给我们界面使者Activity。当事件传给Activity后，Activity会进行事件分发，会调用Activity的dispatchTouchEvent()方法进行事件分发，我们就从此方法开始来分析。我们来看具体源码
```java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {//1.
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {//2.事件继续分发
            return true;
        }
        return onTouchEvent(ev);//3.Activity自身onTouchEvent()方法
    }
```
在注释1处，主要对事件进行判断，当Touch事件为MotionEvent.ACTION_DOWN事件时，会执行onUserInteraction()方法，查看源码发现，此方法是一个空方法。主要作用就是当各种事件key，touch或trackball分发到Activity时，都会执行此方法。

我们先来看注释3，当事件都没有被消费，及getWindow().superDispatchTouchEvent(ev)返回falses时，就会调用Activity自己的onTouchEvent()方法，自己对事件进行消费。

我们再来看注释2，getWindow().superDispatchTouchEvent(ev)，从[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)中，我们知道getWindow()返回的是PhoneWindow,所以我们来看看PhoneWindow中的superDispatchTouchEvent()方法
```java
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```
由[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)，我们也知，mDecor就是DecorView，所以我们继续来看DecorView中的superDispatchTouchEvent()方法
```java
        public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
        }
```
由[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)博文，我们知道，DecorView继承于FrameLayout,FrameLayout由继承于ViewGroup,ViewGroup又继承于View。通过分析相互关系，知最后调用的是ViewGroup的dispatchTouchEvent()方法(FrameLayout没有实现此方法)，所以我们继续来看看ViewGroup中的方法
```java
   @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {//1.过滤Touch事件
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            //初始化down事件
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                
                cancelAndClearTouchTargets(ev);//清除前一个事件的目标及事件状态
                resetTouchState();//重置事件状态
            }

            // 检测是否拦截事件
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);//2.拦截事件调用方法
                    ev.setAction(action); //重置Action事件，以防被修改
                } else {
                    intercepted = false;
                }
            } else {
                //没有touch目标直接拦截事件
                intercepted = true;
            }

            // 检测事件是否取消
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;


            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {//当事件没有取消并没有被拦截时，执行事件分发
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                    .......

                    final int childrenCount = mChildrenCount;
                    if (childrenCount != 0) {
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final View[] children = mChildren;
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);

                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final View child = children[i];
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);

                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {//3.子View事件分发
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                mLastTouchDownIndex = i;
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                        }
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

        ........//省略部分，如果事件被取消，那就分发取消事件

        return handled;
    }
```
我们来看注释1处，事件过滤onFilterTouchEventForSecurity(),此方法为View中的方法，我们来看看此方法
```java
    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // 当Window被遮盖，就丢弃此事件
            return false;
        }
        return true;
    }
```
这里主要就是对Window是否被遮盖进行判断，从而决定事件是否进行传递。事件要进行传递，首先就是Window没有被遮盖。我们再来看看注释2，拦截方法onInterceptTouchEvent(),此方法是ViewGroup特有的，也只有ViewGroup可以进行事件拦截。我们来看看ViewGroup的此方法
```java
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }
```
这里是直接返回了false，也就是不拦截事件。如果返回true,也就会拦截事件。**在我们开发的过程中，经常会出现一些事件冲突，而往往解决这些事件冲突的途径，也都是在我们自定义的ViewGroup中复写拦截方法onInterceptTouchEvent()，重写返回值，从而解决事件冲突问题。**我们继续往下走，当不拦截事件后，我们就对子View进行事件分发，这里我们继续来看注释3，方法dispatchTransformedTouchEvent()
```java
  private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        //当为取消事件时，就分发取消事件ACTION_CANCEL
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        ........
        
        //ViewGroup是否有子View
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```
这里主要对ViewGroup是否有子View做了一个判断，如果ViewGroup无子View，那直接调用ViewGroup父类View的dispatchTouchEvent()方法;如果有子View，那就调用子View的dispatchTouchEvent()方法；其实也都是View类的dispatchTouchEvent()方法，但这里需要注意一下，如果子View又是ViewGroup，那样当调用dispatchTouchEvent()方法时，那就调用ViewGroup的dispatchTouchEvent()事件分发方法，需要重走一遍分发流程。我们这里把子View就看成View了，所以我们就来看看此方法
```java
 public boolean dispatchTouchEvent(MotionEvent event) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {//1.实现OnTouchListener接口
                return true;
            }

            if (onTouchEvent(event)) {//2.View自身的OnTouchEvent事件
                return true;
            }
        }

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }
        return false;
    }
```
从注释1处知，当事件分发到View后，首先调用的接口OnTouchListener的实现方法(在我们开发的时候，经常会对View或ViewGroup设置一些Touch的监听事件)，然后才调用注释2的OnTouchEvent()方法，我们也来看看View的OnTouchEvent()方法
```java
    /**
     * Touch事件的具体实现方法
     *
     * @param event The motion event.
     * @return 返回true，此事件被消费，返回false,则没被消费
     */
    public boolean onTouchEvent(MotionEvent event) {
        final int viewFlags = mViewFlags;
       
        ........

        //对View是否可以消费点击事件做判断，是否设置点击事件，是否可点击
        if (((viewFlags & CLICKABLE) == CLICKABLE ||(viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP://手指抬起up事件
                    boolean prepressed = (mPrivateFlags & PREPRESSED) != 0;
                    if ((mPrivateFlags & PRESSED) != 0 || prepressed) {
                        
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            setPressed(true);//Button按压状态变化通知
                       }

                        if (!mHasPerformedLongPress) {
                            
                            removeLongPressCallback();//去除长按状态

                            //执行点击事件
                            if (!focusTaken) {
                               
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();//消费点击事件
                                }
                            }
                        }

                        .......

                        removeTapCallback();
                    }
                    break;

                case MotionEvent.ACTION_DOWN://手指按下down事件
                     ......
                    break;

                case MotionEvent.ACTION_CANCEL://事件取消
                    setPressed(false);
                    removeTapCallback();
                    break;

                case MotionEvent.ACTION_MOVE://手指移动move事件
                    final int x = (int) event.getX();
                    final int y = (int) event.getY();
                    .......
                    break;
            }
            return true;
        }

        return false;
    }
```
此方法主要是消费事件的方法，当View设置了点击事件或长按事件，那就会对事件进行消费。当手指抬起，也就是ACTION_UP事件时，就执行点击事件方法performClick(),我们也再来看看此方法
```java
    public boolean performClick() {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);//实现OnClickListener接口
            return true;
        }

        return false;
    }
```
到这里，主要就是实现了View的onClickListener接口的方法onClick(),也就消费了点击事件。但如果View没有设置点击事件，那就不会消费此方法，而在ViewGroup分发事件的时候就已判断过是否有子View，此时当子View和ViewGroup都没有设置点击事件时，就会直接返回false给上一级，上一级如果也是ViewGroup，那也是类似，如果都返回false，那样事件就将会被Activity的OnTouchEvent()消费掉。

到这里，Android的事件传递我们就分析完了。


**注意：**

1. **只要有一个View消费了ACTION_DOWN事件，剩余的所有事件(ACTION_MOVE、ACTION_UP等)都将由此View消费。**
2. **源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 三、参考文档

[Android View 事件分发机制 源码解析（ViewGroup篇）](http://blog.csdn.net/dfskhgalshgkajghljgh/article/details/53492488)

[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)

[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)

