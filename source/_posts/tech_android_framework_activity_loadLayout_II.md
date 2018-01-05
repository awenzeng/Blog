---
layout: post
title: "Activity布局加载流程源码分析(II)"
date: 1/3/2018 6:55:03 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
在[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)文章中，已经详细分析了setContentView()加载流程，但对于装饰器DecorView怎么被加载到Window上的，上篇博文没有说到，所以本篇博文将会接着上篇博文，继续分析Activity布局的加载流程。

在开始分析之前，我们需要了解一些概念，如：

- **Window：** 是一个抽象类，表示是一个窗口。Android系统中的界面，也都是以窗口的形式存在的。
- **PhoneWindow：** 是Window类具体实现类，Activity中布局加载逻辑主要就是在此类中完成的。
- **DecorView：**是PhoneWindow中的一个内部类，也是Window的顶级View，主要负责装载各种View。
- **WindowManager：** 是Window的管理类，管理着Window的添加、更新和删除。
- **WindowManagerService(AMS)：**是系统窗口管理服务类，具体管理着系统各种各样的Window。
- **ViewRootImpl:**是View的绘制的辅助类，所有View的绘制都离不开ViewRootImpl。

# 一、Activity布局及DecorView加载分析
这里，我们接着Activity布局加载流程继续分析。在布局加载流程最后，主要是通过WindowManager添加装饰器DecorView到Window中，从而实现Activity布局的加载，这里继续来看那部分代码
<!-- more -->

```java
   final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward) {

        ActivityClientRecord r = performResumeActivity(token, clearHide);
        .......
        if (r != null) {
            final Activity a = r.activity;
            ......
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
            ......
        }
    }
```
这里，简单解释一下参数。其中r.window为PhoneWindow,decor就是装饰器DecorView,wm就是WindowManager。最后通过wm.addView(decor, l)方法，实现Activity布局的加载。这里还注意到WindowManager.LayoutParams的type参数为WindowManager.LayoutParams.TYPE_BASE_APPLICATION，也即是应用窗口类型(所有程序窗口的base窗口，其他应用程序窗口都显示在它上面),具体有关Window的窗口属性，可以参考博文[Android悬浮窗原理解析(Window)](http://blog.csdn.net/awenyini/article/details/78265284),想了解的同学，可以点击看看，这里有比较全的Window属性解释。

我们再来看一下wm,这里定义的类型是接口ViewManager，其实它就是WindowManager，这里主要是使用设计模式的里氏替换原则(源码中很多地方都用这原则)。wm主要是通过a.getWindowManager()赋值的，所以我们主要来看看Activity中的getWindowManager()方法
```java
    public WindowManager getWindowManager() {
        return mWindowManager;
    }
```
mWindowMananger是Activity的一个属性变量，通过[Activity的启动加载流程](http://blog.csdn.net/awenyini/article/details/78906030),Activity初始化过程中就会对mWindowManager进行赋值，而Activity初始化主要通过attach方法完成，所以我们继续来看Activity的attach方法
```java
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config) {
        .......
        
        mWindow = PolicyManager.makeNewWindow(this);
        mWindow.setCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);

        ......
        mWindow.setWindowManager(null, mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```
这里是通过mWindow.getWindowManager()来赋值mWindowManager，而mWindow即PhoneWindow，这点在[Activity布局加载流程](http://blog.csdn.net/awenyini/article/details/78934390)中，已分析过。由于PhoneWindow是继承至Window，通过阅读源码分析知道，getWindowManager()方法，主要是window中完成实现的，所以我们具体来看看Window中的getWindowManager()方法
```java

    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        if (wm == null) {
            wm = WindowManagerImpl.getDefault();//1.核心代码
        }
        mWindowManager = new LocalWindowManager(wm, hardwareAccelerated);//2.核心代码
    }

    public WindowManager getWindowManager() {
        return mWindowManager;
    }
```
这里，我们先来看一下注释1，WindowManagerImpl.getDefault()，这很明显是单例模式。我们继续来看看源码
```java

    private final static WindowManagerImpl sWindowManager = new WindowManagerImpl();

    public static WindowManagerImpl getDefault() {
        return sWindowManager;
    }
```
这里wm就是WindowManagerImpl。我们来看看WindowManagerImpl类
```java
public class WindowManagerImpl implements WindowManager {
 ......省略细节
}
```
我们再来看看注释2,也即LocalWindowManager类
```java
private class LocalWindowManager extends WindowManagerImpl.CompatModeWrapper {

        private static final String PROPERTY_HARDWARE_UI = "persist.sys.ui.hw";

        private final boolean mHardwareAccelerated;

        LocalWindowManager(WindowManager wm, boolean hardwareAccelerated) {
            super(wm, getCompatInfo(mContext));
            mHardwareAccelerated = hardwareAccelerated ||
                    SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        }

        public boolean isHardwareAccelerated() {
            return mHardwareAccelerated;
        }
        
        public final void addView(View view, ViewGroup.LayoutParams params) {
            ........
            super.addView(view, params);
        }
    }
```
我们继续来看看LocalWindowManager类继承的WindowManagerImpl的内部类CompatModeWrapper
```java
 static class CompatModeWrapper implements WindowManager {
        private final WindowManagerImpl mWindowManager;
        private final Display mDefaultDisplay;
        private final CompatibilityInfoHolder mCompatibilityInfo;

        CompatModeWrapper(WindowManager wm, CompatibilityInfoHolder ci) {
            mWindowManager = wm instanceof CompatModeWrapper
                    ? ((CompatModeWrapper)wm).mWindowManager : (WindowManagerImpl)wm;
            if (ci == null) {
                mDefaultDisplay = mWindowManager.getDefaultDisplay();
            } else {
                mDefaultDisplay = Display.createCompatibleDisplay(
                        mWindowManager.getDefaultDisplay().getDisplayId(), ci);
            }

            mCompatibilityInfo = ci;
        }

        @Override
        public void addView(View view, android.view.ViewGroup.LayoutParams params) {
            mWindowManager.addView(view, params, mCompatibilityInfo);
        }

        @Override
        public void updateViewLayout(View view, android.view.ViewGroup.LayoutParams params) {
            mWindowManager.updateViewLayout(view, params);

        }

        @Override
        public void removeView(View view) {
            mWindowManager.removeView(view);
        }

        @Override
        public Display getDefaultDisplay() {
            return mDefaultDisplay;
        }

        @Override
        public void removeViewImmediate(View view) {
            mWindowManager.removeViewImmediate(view);
        }

        @Override
        public boolean isHardwareAccelerated() {
            return mWindowManager.isHardwareAccelerated();
        }
    }
```
WindowManagerImpl的内部类CompatModeWrapper实现了WindowManager接口，而WindowManager又实现了ViewManager接口
```java
public interface WindowManager extends ViewManager {
  ......
}
```
我们来看看ViewManager接口
```java
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```
这里实现ViewManager接口的有WindowManager,而WindowManagerImpl和CompatModeWrapper也都实现WindowManager接口，从而间接实现了ViewManager接口，也都实现的添加，更新和删除View的方法。

所以，在最开始处，以ViewManager定义的wm其实也就是LocalWindowManager,通过相互继承调用，其实最后调用的是WindowManagerImpl中的addView()方法，我们继续来看看此方法
```java
 public void addView(View view) {
        addView(view, new WindowManager.LayoutParams(
            WindowManager.LayoutParams.TYPE_APPLICATION, 0, PixelFormat.OPAQUE));
    }

    public void addView(View view, ViewGroup.LayoutParams params) {
        addView(view, params, null, false);
    }
    
    public void addView(View view, ViewGroup.LayoutParams params, CompatibilityInfoHolder cih) {
        addView(view, params, cih, false);
    }
    
    private void addView(View view, ViewGroup.LayoutParams params,
            CompatibilityInfoHolder cih, boolean nest) {
        if (false) Log.v("WindowManager", "addView view=" + view);

        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException(
                    "Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams
                = (WindowManager.LayoutParams)params;
        
        ViewRootImpl root;
        View panelParentView = null;
        .......
            root = new ViewRootImpl(view.getContext());
            root.mAddNesting = 1;
            if (cih == null) {
                root.mCompatibilityInfo = new CompatibilityInfoHolder();
            } else {
                root.mCompatibilityInfo = cih;
            }

            view.setLayoutParams(wparams);
            
            if (mViews == null) {
                index = 1;
                mViews = new View[1];
                mRoots = new ViewRootImpl[1];
                mParams = new WindowManager.LayoutParams[1];
            } else {
                index = mViews.length + 1;
                Object[] old = mViews;
                mViews = new View[index]; 
                System.arraycopy(old, 0, mViews, 0, index-1);
                old = mRoots;
                mRoots = new ViewRootImpl[index];
                System.arraycopy(old, 0, mRoots, 0, index-1);
                old = mParams;
                mParams = new WindowManager.LayoutParams[index];
                System.arraycopy(old, 0, mParams, 0, index-1);
            }
            index--;

            mViews[index] = view;
            mRoots[index] = root;
            mParams[index] = wparams;
        }
        // do this last because it fires off messages to start doing things
        root.setView(view, wparams, panelParentView);
    }
```
这里有三个数组mViews，mRoots，mParams变量，mViews主要保存向Window中添加的View,mRoots主要保存实现绘制View的ViewRootImpl,mParams主要保存添加的Window参数，最后每一个添加的View都会调用root.setView(view, wparams, panelParentView)方法，实现View的绘制。关于这三个参数的作用，是因为WindowManager中可能添加多个Window，多个View，所以需要保存起来，方便删除和更新。

关于WindowManagerImpl的源码，这里需要注意一下，由于我用的版本是Andorid4.1.1_r1，源码如上，而在大于Andorid4.1.1_r11的版本，如Android5.1.1中的WindowManagerImpl源码的addView方法，主要是通过WindowManagerGlobal来实现的,逻辑也都和Andorid4.1.1_r1一样，只是把部分逻辑封装成WindowManagerGlobal，这里我就不多说，想了解的同学，可以自行查看。

到这里，DecorView添加入Window的流程就分析完了。接下来主要就是DecorView的绘制流程，也即View的绘制流程。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 二、参考文档

[Activity布局加载流程源码分析](http://blog.csdn.net/awenyini/article/details/78934390)

[Activity启动流程源码分析(应用中)](http://blog.csdn.net/awenyini/article/details/78906030)

[Android悬浮窗原理解析(Window)](http://blog.csdn.net/awenyini/article/details/78265284)
