---
layout: post
title: "Activity布局加载流程源码分析"
date: 12/29/2017 7:20:32 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
最近阅读Android源码，似乎有点发现新大陆的感觉。以前经常接触Android知识，在阅读源码中，开始变得豁然开朗。前两天才写完两篇博文[Activity启动流程源码分析(应用中)](http://blog.csdn.net/awenyini/article/details/78906030)和[Activity启动流程源码分析(Launcher中)](http://blog.csdn.net/awenyini/article/details/78915225)，今天，就急不可耐的想写写Activity布局加载流程，其实，也就是想趁热打铁，好好梳理梳理这部分知识。

在开始梳理之前，我们需要了解一些概念，如：

- **Window：** 是一个抽象类，表示是一个窗口。Android系统中的界面，也都是以窗口的形式存在的。
- **PhoneWindow：** 是Window类具体实现类，Activity中布局加载逻辑主要就是在此类中完成的。
- **WindowManager：** 是Window的管理类，管理着Window的添加、更新和删除。
- **WindowManagerService(AMS)：**是系统窗口管理服务类，具体管理着系统各种各样的Window.
- **DecorView：**是Window的顶级View，主要负责装载各种View。

# 一、Activity布局加载分析
我们知道，设置Activity布局内容，主要是在Activity的onCreate()中调用setContentView()方法，下面让我们来看看此方法
```java
   
    public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);//核心代码
        initActionBar();
    }
```
<!-- more -->

这里主要调用了getWindow().setContentView()方法，我们来看看Activity中getWindow()
```java
    public Window getWindow() {
        return mWindow;
    }
```
由此知mWindow是Activity一个属性变量，在前面Activity启动流程介绍中，我们知道在Activity启动前都会先调用attach()，而这mWindow就是在attach初始化的时候赋值的，我们来看看Activity的attach源码
```java
 final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config) {
        attachBaseContext(context);
        mFragments.attachActivity(this);
        
        mWindow = PolicyManager.makeNewWindow(this);//核心代码

        ......

        mUiThread = Thread.currentThread();
        
        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;

        mWindow.setWindowManager(null, mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```
这里我们来关注一下PolicyManager.makeNewWindow(this)方法，创建Window，我们来看看PolicyManager类
```java
public final class PolicyManager {  
     
   private static final String POLICY_IMPL_CLASS_NAME =  
        "com.android.internal.policy.impl.Policy";  
  
    private static final IPolicy sPolicy;  
  
    static {  
        try {  
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);  
            sPolicy = (IPolicy)policyClass.newInstance();//反射初始化Policy

        } catch (ClassNotFoundException ex) {  
            throw new RuntimeException(  
                    POLICY_IMPL_CLASS_NAME + " could not be loaded", ex);  
        } catch (InstantiationException ex) {  
            throw new RuntimeException(  
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);  
        } catch (IllegalAccessException ex) {  
            throw new RuntimeException(  
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);  
        }  
    }  
   
    public static Window makeNewWindow(Context context) {  
        return sPolicy.makeNewWindow(context); //核心方法
    }  
    .......
}
```
由上易知，这里主要是通过反射初始化Policy，然后利用设计模式[里氏替换原则](http://blog.csdn.net/awenyini/article/details/78793233)调用Policy的makeNewWindow()方法，我们继续来看Policy中的方法
```java
public class Policy implements IPolicy {  
   
    ........

    public PhoneWindow makeNewWindow(Context context) {  
        return new PhoneWindow(context);//核心代码
    }  
    ......
}  
```
我们可以发现mWindow其实就是PhoneWindow,在Activity中getWindow().setContentView()方法，就是调用PhoneWindow中的setContentView方法，所以我们这里来看看PhoneWindow中的setContentView()方法
```java
    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null){
            installDecor();//1.安装装饰器
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);//2.填充我们的布局文件
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
从注释2,我们知道布局填充器mLayoutInflater向mContentParent填充我们的布局内容，而mContentParent是一个ViewGroup,它是怎么赋值的呢？这里我们要来看注释1，当mContentParent为空时，会安装装饰器，我们继续来看phoneWindow中installDecor()方法
```java
  private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();//1.生成装饰器
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);//2.对填充我们布局的ViewGroup赋值

            mDecor.makeOptionalFitsSystemWindows();

            mTitleView = (TextView)findViewById(com.android.internal.R.id.title);
            if (mTitleView != null) {
                if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                  .......
                } else {
                    mTitleView.setText(mTitle);//设置Activity的title
                }
            } else {
                mActionBar = (ActionBarView) findViewById(com.android.internal.R.id.action_bar);
                if (mActionBar != null) {
                   .......mActionBar的处理
                }
            }
        }
    }
```
首先，我们先来看看注释1装饰器的生成方法generateDecor()
```java
    protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
```
这里主要就是装饰器DecorView的初始化，我们再来看一下DecorView的源码
```java
   private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
        ......

        public DecorView(Context context, int featureId) {
            super(context);
            mFeatureId = featureId;
        }
       .......
```
DecorView类包括内容还有许多，这里就不介绍了，我们只需知道**DecorView是PhoneWindow的内部类，DecorView继承于FrameLayout，实现RootViewSurfaceTaker接口**。下面我们来看一下mContentParent的生成，即generateLayout(mDecor)方法
```java
 protected ViewGroup generateLayout(DecorView decor) {
        
        .......//初始化一些window属性

        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();

        //通过判断Activity的不同feature加载不同的系统默认布局
        if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = com.android.internal.R.layout.screen_title_icons;
            }
            
            removeFeature(FEATURE_ACTION_BAR);
           
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            //带进度系统布局
            layoutResource = com.android.internal.R.layout.screen_progress;
            
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {      
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = com.android.internal.R.layout.screen_custom_title;
            }
           
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            //无Titile系统布局
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                if ((features & (1 << FEATURE_ACTION_BAR_OVERLAY)) != 0) {
                    layoutResource = com.android.internal.R.layout.screen_action_bar_overlay;
                } else {
                    layoutResource = com.android.internal.R.layout.screen_action_bar;
                }
            } else {
                layoutResource = com.android.internal.R.layout.screen_title;
            }
            
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = com.android.internal.R.layout.screen_simple_overlay_action_mode;
        } else {
           
            layoutResource = com.android.internal.R.layout.screen_simple;//1.一般系统布局
           
        }

        mDecor.startChanging();

        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));//2.向装饰View加入系统布局View

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);//3.获取我们Activity能填充的内容ViewGroup
        ......
        }

        mDecor.finishChanging();

        return contentParent;
    }
```
这里我们来看一下，系统默认的几种Activity的头部布局xml文件，布局文件的源码位置为：android4.1.1_r1\frameworks\base\core\res\res\layout,我们挑两个文件来看一下：

第一个screen_title.xml
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:fitsSystemWindows="true">
    <!-- Popout bar for action modes -->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content" />
    <FrameLayout
        android:layout_width="match_parent" 
        android:layout_height="?android:attr/windowTitleSize"
        style="?android:attr/windowTitleBackgroundStyle">
        <TextView android:id="@android:id/title" 
            style="?android:attr/windowTitleStyle"
            android:background="@null"
            android:fadingEdge="horizontal"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </FrameLayout>
    <FrameLayout android:id="@android:id/content"
        android:layout_width="match_parent" 
        android:layout_height="0dip"
        android:layout_weight="1"
        android:foregroundGravity="fill_horizontal|top"
        android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
第二个，screen_simple.xml
```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
通过对比，我们发现这两个布局文件都有一个共同id为@android:id/content的FrameLayout,其实这也就是我们Activity布局填充容器。我们还发现，这两个布局父布局都是一个线性布局LinearLayout，并且方向都是垂直的，这也验证了我们Activity内容布局一般都是状态栏的下边的模式。我们再来看后面的代码
```java
  View in = mLayoutInflater.inflate(layoutResource, null);
  decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));//2.向装饰View加入系统布局View

  ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);//3.获取我们Activity能填充的内容ViewGroup
```
这里向装饰器添加了系统布局View，并从系统布局View中获取了Activity填充内容的容器ViewGroup。其中ID_ANDROID_CONTENT就是com.android.internal.R.id.content，通过(ViewGroup)findViewById(ID_ANDROID_CONTENT)就获取了布局文件中的FrameLayout，即Activity内容填充布局的ViewGroup。这样我们再回到PhoneWindow的setContentView方法
```java
    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);//核心代码
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
现在mContentParent已经赋完值了，再通过布局填充器mLayoutInflater的inflate()方法，这样我们就把Activity的布局文件添加到装饰器上了。然而，现在虽然装饰器DecorView上已经有了Activity布局内容，但是是什么时候添加到Window上的呢？这里就需要了解[Activity的启动流程](http://blog.csdn.net/awenyini/article/details/78906030)，在Activity的启动流程最后几步会执行ActivityThread中handleLaunchActivity()方法，我们接着此方法继续分析
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
      
        .......

        Activity a = performLaunchActivity(r, customIntent);//1.创建Activity实例

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;

            handleResumeActivity(r.token, false, r.isForward);//2.调用Activity onResume方法

            .......
        } else {
            .......
        }
    }
```
在注释1处，已经建立Activity的实例，并且执行Activity生命周期的attach()和onCreate()方法。我们知道setContentView()也就在onCreate()方法中调用的,所以这个时候，我们Activity布局文件内容已经装入了装饰器DecorView中，接下来就是把DecorView和Window关联起来，所以下面我们继续来看handleResumeActivity()方法
```java
 final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        ActivityClientRecord r = performResumeActivity(token, clearHide);//1.执行Activity的onResume方法

        if (r != null) {
            final Activity a = r.activity;

            ........
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(a.getActivityToken());//2.Activity显示可见
                } catch (RemoteException e) {
                }
            }
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
                    wm.addView(decor, l);//3.通过WindowManager将DecorView加入Window,从而显示Window，Activity变为可见。
                }
            } else if (!willBeVisible) {
                r.hideForNow = true;
            }

            cleanUpPendingRemoveWindows(r);

            if (!r.activity.mFinished && willBeVisible&& r.activity.mDecor != null && !r.hideForNow) {
             
                WindowManager.LayoutParams l = r.window.getAttributes();
                if ((l.softInputMode
                        & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                        != forwardBit) {
                    l.softInputMode = (l.softInputMode
                            & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                            | forwardBit;
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);//4.更新DecorView,更新Activity界面
                    }
                }
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
            }

          .......

        } else {
          .......
        }
    }
```
在注释1处，调用performResumeActivity(token, clearHide)方法，实际上就是调用activity生命周期的onResume()方法。注释2处，通过Binder跨进程通信，调用ActivityManagerService中willActivityBeVisible()获取显示Activity的控制开关，从而在注释3处，通过WindowManager添加装饰器DecorView到Window,然后，再调用相关View的绘制流程，这样一个有布局的Activity就被加载出来了。

到这里，Activity布局加载流程就是梳理完了。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 二、参考文档
[ Binder通信机制原理解析](http://blog.csdn.net/awenyini/article/details/78806893)

[Activity启动流程源码分析(应用中)](http://blog.csdn.net/awenyini/article/details/78906030)

[Activity启动流程源码分析(Launcher中)](http://blog.csdn.net/awenyini/article/details/78915225)
