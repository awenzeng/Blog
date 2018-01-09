---
layout: post
title: "View绘制三大流程源码分析"
date: 1/8/2018 7:39:30 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
在上篇博文[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)中，关于DecorView作为Activity、Window中的顶级View的绘制，我们已经作了一个详细的分析。但在具体说到View的绘制的时候，我们没有详细说明，所以本篇博文将会对View的绘制原理作深度分析。

在开始分析之前，我们需要了解一些概念，如：

- **View：**是所有UI组件的基类,是Android平台中用户界面体现的基础单位。
- **ViewGroup:**是容纳UI组件的容器,它本身也是View的子类。
- **ViewRootImpl:**是View的绘制的辅助类，所有View的绘制都离不开ViewRootImpl。
- **MeasureSpec：** View的内部类，主要就是View测量模式的工具类

# 一、View绘制三大流程分析
在DecorView的具体绘制中，我们涉及了View绘制的三大流程，具体分别为measure(测量)、layout(布局)和draw(绘制)。下面我们就来一一分析：
<!-- more -->

**1.performMeasure(测量)**

我们知道ViewRootImpl是View绘制的辅助类，View的绘制都是在ViewRootImpl的帮助下完成的，所以要了解View的measure(测量)，我们就必须看看ViewRootImpl中的performMeasure()方法
```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
在[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)中，我们知道mView就是DecorView,而**DecorView继承于FrameLayout,FrameLayout又继承于ViewGroup，ViewGroup又继承于View**，根据他们之间的关系，我们知道，mView.measure()是调用了父类View的measure()方法(因为只有View有measure方法)，所以来分析View的measure()方法
```java
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        if ((mPrivateFlags & FORCE_LAYOUT) == FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
            ........

            onMeasure(widthMeasureSpec, heightMeasureSpec);//核心方法

            ........
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
    }
```
此onMeasure()方法，在DecorView，Framelayout和View中都有定义，并且DecorView和FrameLayout重载了此方法，根据调用关系，这里调用了DecorView的onMeasure(),我们来看看此方法
```java
 @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            final DisplayMetrics metrics = getContext().getResources().getDisplayMetrics();
            final boolean isPortrait = metrics.widthPixels < metrics.heightPixels;

            final int widthMode = getMode(widthMeasureSpec);
            final int heightMode = getMode(heightMeasureSpec);

            ......

            super.onMeasure(widthMeasureSpec, heightMeasureSpec);

            ......
        }
```
关于测量模式，这里我们先不说，后面我们会说到，这里调用了父类FrameLayout中的onMeasure()方法，这里我们来看一下源码
```java
  @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        ........//计算top,left,bottom,right的margin值，从而确定FrameLayout的宽高

        //设置宽高
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);//获取子View

                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();//子View配置参数
                int childWidthMeasureSpec;
                int childHeightMeasureSpec;
                
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredWidth() -
                            getPaddingLeftWithForeground() - getPaddingRightWithForeground() -
                            lp.leftMargin - lp.rightMargin,
                            MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }
                
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredHeight() -
                            getPaddingTopWithForeground() - getPaddingBottomWithForeground() -
                            lp.topMargin - lp.bottomMargin,
                            MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }

                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }
```
由于FrameLayout是一个容器，可以装载其他的View，所以这里需要进行遍历其中的子View，并一一进行measure(测量)。

在具体说此方法前，我们先要了解一下View的测量模式及MeasureSpec类，具体我们先来看看MeasureSpec类的源码
```java
    public static class MeasureSpec {

        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         * 父View不对子View有任何限制，子View需要多大就多大
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         * 父View已经测量出子View所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值。对应于match_parent和精确数* 值这两种模式
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;

        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         * 子View的最终大小是父View指定的SpecSize值，并且子View的大小不能大于这个值，即对应wrap_content这种模式。
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;


       /**
         * 用实际值和测量模式组装成measureSpec测量规格
         * 将size和mode打包成一个32位的int型数值
         * 高2位表示SpecMode，测量模式，低30位表示SpecSize，某种测量模式下的规格大小
         */
        public static int makeMeasureSpec(int size, int mode) {
            return size + mode;
        }

        //将32位的MeasureSpec解包，返回SpecMode,测量模式(EXACTLY、UNSPECIFIED或AT_MOST）
        public static int getMode(int measureSpec) {
            return (measureSpec & MODE_MASK);
        }

        
        //将32位的MeasureSpec解包，返回SpecSize
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }
```
这里主要通过位运算，来实现View的三种测量模式UNSPECIFIED、EXACTLY和AT_MOST。相关定义如下：

- **UNSPECIFIED：**父View不对子View有任何限制，子View需要多大就多大
- **EXACTLY：**父View已经测量出子View所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值。对应于match_parent和精确数值这两种模式
- **AT_MOST：** 子View的最终大小是父View指定的SpecSize值，并且子View的大小不能大于这个值，即对应wrap_content这种模式。

我们继续上面FrameLayout的onMeasure()方法继续分析，可以发现，此方法主要就是组装子View宽高的测量规格MeasureSpec，然后作为参数传给子View的measure()方法。这里我们只来看一个组装就好，我们来看宽的组装的测量规格MeasureSpec，我们来看看相关代码
```java
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        ........
               final View child = mMatchParentChildren.get(i);//获取子View

                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();//子View配置参数
                int childWidthMeasureSpec;
                int childHeightMeasureSpec;
                
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredWidth() -
                            getPaddingLeftWithForeground() - getPaddingRightWithForeground() -
                            lp.leftMargin - lp.rightMargin,
                            MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

      ........
}
```
当子View的布局参数lp.width为LayoutParams.MATCH_PARENT时，生成后的测量规格MeasureSpec是以测量模式为MeasureSpec.EXACTLY的值。当lp.width不为LayoutParams.MATCH_PARENT时，这是调用了ViewGroup中的getChildMeasureSpec()方法，来生成相关值，这里我们来看此方法
```java
  public static int getChildMeasureSpec(int spec, int padding, int childDimension) {

        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
这里主要传入了FrameLayout的测量规格MeasureSpec，根据FrameLayout的测量规格和子View具体的padding和childDimension值，从而决定子view宽的测量规格MeasureSpec。具体创建细节，这里就不说。回到FrameLayout的onMeasure()方法，这样当子View的宽高的测量规格都计算出来之后，就会调用子View的measure()方法。如果子View不再是ViewGroup，那样就会调用子View(或自定义View)的onMeasure()方法，从而完成View的测量；如果子View还是ViewGroup，那就会按我们说的逻辑再走一遍measure方法。

**2.performLayout(布局)**
说完View绘制的measure(测量)，我们来看看View绘制的layout(布局)。同样的，我们先来看ViewRootImpl中的performLayout()方法
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
由相关继承类的关系，我们知道，这里调用的是View的layout()方法
```java
 public void layout(int l, int t, int r, int b) {
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        boolean changed = setFrame(l, t, r, b); 
        if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {

            onLayout(changed, l, t, r, b);//核心方法

            mPrivateFlags &= ~LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
        mPrivateFlags &= ~FORCE_LAYOUT;
    }
```
通过源码我们知道，ViewGroup是一个抽象的View的子类，而FrameLayout是ViewGroup的实现类，所以这里onLayout()是FrameLayout中的方法，我们来看一下此方法
```java
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        final int count = getChildCount();

        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        mForegroundBoundsChanged = true;
        
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;

                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }

                final int layoutDirection = getResolvedLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.LEFT:
                        childLeft = parentLeft + lp.leftMargin;
                        break;
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }

                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```
这里主要就是通过padding和margin算出子View的top,left,bottom,right四个顶点的值，从而再调其子View的layout方法。如果子View child不是ViewGroup，最后直接调用子View(或自定义View)的onLayout()方法，如果child是ViewGroup，那就再走一遍流程。

**3.performDraw(绘制)**

从[DecorView绘制流程源码分析](http://blog.csdn.net/awenyini/article/details/78983463)中，我们知道performDraw()绘制有两种方式，分别为Hardware渲染(硬件加速)和Software渲染，因为两种绘制方式最后也都走到调用View的draw()方法，所以这里我们来看看software渲染方式的绘制
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
同上，通过分析知，这里mView.draw(canvas)其实是调用View.draw(canvas)方法，让我们来看看此方法
```java
  public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & DIRTY_MASK) == DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~DIRTY_MASK) | DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // 第一步,如果有背景，绘制背景
        int saveCount;

        if (!dirtyOpaque) {
            final Drawable background = mBackground;
            if (background != null) {
                final int scrollX = mScrollX;
                final int scrollY = mScrollY;

                if (mBackgroundSizeChanged) {
                    background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
                    mBackgroundSizeChanged = false;
                }

                if ((scrollX | scrollY) == 0) {
                    background.draw(canvas);
                } else {
                    canvas.translate(scrollX, scrollY);
                    background.draw(canvas);
                    canvas.translate(-scrollX, -scrollY);
                }
            }
        }
        ........

        //第二步,保存画布的层级
        int paddingLeft = mPaddingLeft;

        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }

        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);

        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }

        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;

        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }

        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }

        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }

        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }

        saveCount = canvas.getSaveCount();

        int solidColor = getSolidColor();
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // 第三步，绘制内容
        if (!dirtyOpaque)
           onDraw(canvas);//核心方法

        //第四步，分发绘制子View
        dispatchDraw(canvas);

        //第五步，绘制fade效果和restore Layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            canvas.drawRect(left, top, right, top + length, p);
        }

        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }

        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            canvas.drawRect(left, top, left + length, bottom, p);
        }

        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            canvas.drawRect(right - length, top, right, bottom, p);
        }

        canvas.restoreToCount(saveCount);

        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
    }
```
从此方法，我们知道View的draw()分五步,分别为：

- 第一步，如果有背景，绘制背景
- 第二步，保存画布的层级
- 第三步，绘制内容
- 第四步，分发绘制子View
- 第五步，绘制fade效果和restore Layers

由于我们的DecorView是FrameLayout,是ViewGroup，所以我们来看一下第四步，分发绘制子View，来看ViewGroup中dispatchDraw()方法(此方法主要是ViewGroup中实现)
```java
  protected void dispatchDraw(Canvas canvas) {
        final int count = mChildrenCount;
        final View[] children = mChildren;
        .......

        if ((flags & FLAG_USE_CHILD_DRAWING_ORDER) == 0) {
            for (int i = 0; i < count; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                    more |= drawChild(canvas, child, drawingTime);
                }
            }
        } else {
            for (int i = 0; i < count; i++) {
                final View child = children[getChildDrawingOrder(count, i)];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                    more |= drawChild(canvas, child, drawingTime);
                }
            }
        }

        // Draw any disappearing views that have animations
        if (mDisappearingChildren != null) {
            final ArrayList<View> disappearingChildren = mDisappearingChildren;
            final int disappearingCount = disappearingChildren.size() - 1;
            // Go backwards -- we may delete as animations finish
            for (int i = disappearingCount; i >= 0; i--) {
                final View child = disappearingChildren.get(i);
                more |= drawChild(canvas, child, drawingTime);
            }
        }
       .......
    }

    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```
通过遍历ViewGroup中的子View，然后在调用子View的draw方法，这里draw()方法和我们前面的view的draw()有点不一样，因为是三个参数的，我们再来看此方法
```java
  /**
     * This method is called by ViewGroup.drawChild() to have each child view draw itself.
     * This draw() method is an implementation detail and is not intended to be overridden or
     * to be called from anywhere else other than ViewGroup.drawChild().
     */
    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
        ......
        if (hasNoCache) {
          .......

            if (!layerRendered) {
                if (!hasDisplayList) {
                    // Fast path for layouts with no backgrounds
                    if ((mPrivateFlags & SKIP_DRAW) == SKIP_DRAW) {
                        mPrivateFlags &= ~DIRTY_MASK;
                        dispatchDraw(canvas);
                    } else {
                        draw(canvas);
                    }
                } else {
                    mPrivateFlags &= ~DIRTY_MASK;
                    ((HardwareCanvas) canvas).drawDisplayList(displayList, null, flags);
                }
            }
        } else if (cache != null) {
           .....
           
        }
         ......

        return more;
    }
```
可以发现，最后还是调用回了View的draw(canvas)方法。所以对于View的draw(绘制)，其实也和measure(测量)和layout(布局)一样，如果View是ViewGroup，就是在draw的时候会进行分发绘制子View，如果view就是View,那就会调用View(或自定义View)的onDraw()方法，绘制内容。

到这里，我们View的三大绘制原理就分析完了。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 三、参考文档
[Android View 测量流程(Measure)完全解析](http://blog.csdn.net/a553181867/article/details/51494058)

[View的绘制原理](http://blog.csdn.net/u014316462/article/details/52054352)










