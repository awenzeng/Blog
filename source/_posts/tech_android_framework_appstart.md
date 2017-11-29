---
layout: post
title: "Android应用程序入口源码解析"
date: 11/23/2017 9:43:41 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
	- Android应用程序入口源码解析
---
---
我们在写C、C++或java应用时，都会有一个main函数，但Android的main函数在哪里呢？偶记得从第一个HelloWorld应用开始，就知道，只要在AndroidManifest配置表中对Activity的intent-filter进行配置，action为android.intent.action.MAIN，category为android.intent.category.LAUNCHER，应用程序启动的时候就会首先启动这个Activity，此Activity也就是应用的入口；后来又知道Application这个类，只要有类继承Application,并在AndroidManifest配置表中对application的name进行配置该类，android应用启动后会把该类的attachBaseContext和onCreate方法作为程序开发入口。实际上是不是这样的呢？本篇博文将会从源码角度来分析。

在说Android应用程序开发入口之前，我们有必要了解一下android系统的启动和Android应用程序的启动流程，这样有助于我们对Android系统有一个整体的认识。首先，让我们来简单了解一下Android系统的启动流程。


# 一、Android系统的启动
Android系统的启动流程是怎样的呢？首先先来看一下流程图：

![](/assets/img/tech_android_system_start_chart.png)
<!-- more -->

根据图，我们知Android启动流程的步骤如下：

- **1.启动电源**
当电源按下时引导芯片代码开始从预定义的地方（固化在ROM）开始执行。加载引导程序Bootloader到RAM，然后执行。

- **2.引导程序BootLoader执行**
引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。

- **3.Linux内核启动**
内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置，它首先在系统文件中寻找init.rc文件，并启动init进程。

- **4.init进程启动**
初始化和启动属性服务，并且启动Zygote进程。

- **5.Zygote进程启动**
创建JavaVM并为JavaVM注册JNI，创建服务端Socket，启动SystemServer进程。

- **6.SystemServer进程启动**
启动Binder线程池和SystemServiceManager，并且启动各种系统服务，例如：ActivityManagerService、PowerManagerService、PackageManagerService，BatteryService、UsageStatsService等其他80多个系统服务。

- **7.Launcher启动**
被SystemServer进程启动的ActivityManagerService会启动Launcher，Launcher启动后会将已安装应用的快捷图标显示到界面上。

关于Android系统的启动源码分析，这里推荐大神刘望舒的几篇文章，讲得比较详细：

[Android系统启动流程（一）解析init进程启动过程](http://liuwangshu.cn/framework/booting/1-init.html)

[Android系统启动流程（二）解析Zygote进程启动过程](http://liuwangshu.cn/framework/booting/2-zygote.html)

[Android系统启动流程（三）解析SyetemServer进程启动过程](http://liuwangshu.cn/framework/booting/3-syetemserver.html)

[Android系统启动流程（四）Launcher启动过程与系统启动流程](http://liuwangshu.cn/framework/booting/4-launcher.html)

# 二、Android应用程序启动
上面说到，当Android系统启动完成之后，Lancher也就启动完成了，在我们的桌面上就会看到已安装的应用快捷图标。点击快捷图标，就能启动我们的应用程序。我们知道，android系统中的每一个应用程序，都是独立运行在自己的进程中的，所以在点击应用快捷图标后，如果应用程序还没有进程，首先应该会先建立应用程序的进程。具体流程是怎样的呢？我们先来看流程图：

![](/assets/img/tech_android_app_start_chart.png)

从流程图知，**应用程序在没有创建进程的情况下，会通过ActivitServiceManager去请求服务端Socket，服务端Socket再去请求Zygote进程，让其帮忙建立进程，而Zygote进程会fork自身来创建应用程序进程。应用程序进程创建的同时，应用程序的主线程也会创建，与主线程息息相关的ActivityThread类也会创建，并调用自身的main方法，进行相关的初始化。**

具体进程是怎么创建的，这里也还是推荐大神刘望舒的两篇文章，其中非常详细的分析了应用进程的创建过程，想了解的可以看一下。

[Android应用程序进程启动过程（前篇）](http://liuwangshu.cn/framework/applicationprocess/1.html)

[Android应用程序进程启动过程（后篇）](http://liuwangshu.cn/framework/applicationprocess/2.html)

好了，下面我们来继续说说**ActivityThread的main方法，也即Android应用程序的入口**。

# 三、Android应用程序入口源码分析
通过Android应用程序的启动，我们知道android应用程序的入口，即ActivityThread的main方法。但在我们开发的时候，很少接触ActivityThread类,主要还是Application和Activity，他俩与ActivityThread的关系怎样呢？让我们从源码中来看看，ActivityThread的main方法：
```java
    public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();//1.Looper的创建
        if (sMainThreadHandler == null) {
            sMainThreadHandler = new Handler();
        }

        ActivityThread thread = new ActivityThread();//2.ActivityThread初始化
        thread.attach(false);//3.调用ActivityThread附属方法attach

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();//4.Looper消息开始循环

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
在main方法中，主线程的Looper实现了初始化和消息循环，这与Android的消息机制息息相关。关于消息机制，我已写过一个篇文章为[Android消息机制源码解析(Handler)](http://blog.csdn.net/awenyini/article/details/78593139)，想了解的可以看一下。这里我们主要来看一下ActivityThread的初始化和attach方法，其中ActivityThread初始化构造方法什么也没做，没啥好看的，我们主要来看看attach方法
```java
    private void attach(boolean system) {
        sThreadLocal.set(this);
        mSystemThread = system;
        if (!system) {//false，不是system,普通app
            .......省略
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>");
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            IActivityManager mgr = ActivityManagerNative.getDefault();//1.核心代码
            try {
                mgr.attachApplication(mAppThread);//2.核心代码
            } catch (RemoteException ex) {
                // Ignore
            }
        } else {//true，是system
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            android.ddm.DdmHandleAppName.setAppName("system_process");
            try {
                mInstrumentation = new Instrumentation();
                ContextImpl context = new ContextImpl();
                context.init(getSystemContext().mPackageInfo, null, this);
                Application app = Instrumentation.newApplication(Application.class, context);
                mAllApplications.add(app);
                mInitialApplication = app;
                app.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }
       ......省略
    }
```
ActivityThread调用attach()传入的参数是false，不是system。注释1通过静态方法ActivityManagerNative.getDefault()获取IActivityManager,实际上是获取到ActivityManagerProxy类，让我们来看ActivityManagerNative.getDefault()方法,进入ActivityManagerNative类
```java

    /**
     * Cast a Binder object into an activity manager interface, generating
     * a proxy if needed.
     */
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);//核心方法
    }

    /**
     * Retrieve the system's default/global activity manager.
     */
    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);//核心方法
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
通过查看ActivityManagerProxy类，了解到它实现IActivityManager接口,mgr就是ActivityManagerProxy。知道返回的类后，我们再来看一下注释2，mgr.attachApplication(mAppThread)，其中mAppThread为ApplicationThread，让我们再进入ActivityManagerProxy，看看attachApplication方法
```java
class ActivityManagerProxy implements IActivityManager
{
    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }

    public IBinder asBinder()
    {
        return mRemote;
    }
    .......
    public void attachApplication(IApplicationThread app) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(app.asBinder());
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);//核心代码
        reply.readException();
        data.recycle();
        reply.recycle();
    }
    .......
}
```
从源码中，了解到mRemote是一个IBinder，通过Binder实现进程间通信(Android核心进程通信方法)，从而调用到ActivityServiceManager里面对应的方法
```java
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);//核心方法
            Binder.restoreCallingIdentity(origId);
        }
    }
```
我们来继续看方法attachApplicationLocked()
```java
   private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
            .......
            thread.bindApplication(processName, appInfo, providers,
                    app.instrumentationClass, profileFile, profileFd, profileAutoStop,
                    app.instrumentationArguments, app.instrumentationWatcher, testMode,
                    enableOpenGlTrace, isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, true);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
          .....

        return true;
    }
```
在attachApplicationLocked()方法中,细节比较多，我们省略掉了，主要来看一下核心方法bindApplication(),其中thread就是ActivityThread中ApplicationThread类，让我们再来看ApplicationThread中的bindApplication()方法
```java
        public final void bindApplication(String processName,
                ApplicationInfo appInfo, List<ProviderInfo> providers,
                ComponentName instrumentationName, String profileFile,
                ParcelFileDescriptor profileFd, boolean autoStopProfiler,
                Bundle instrumentationArgs, IInstrumentationWatcher instrumentationWatcher,
                int debugMode, boolean enableOpenGlTrace, boolean isRestrictedBackupMode,
                boolean persistent, Configuration config, CompatibilityInfo compatInfo,
                Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfileFile = profileFile;
            data.initProfileFd = profileFd;
            data.initAutoStopProfiler = false;
            queueOrSendMessage(H.BIND_APPLICATION, data);//核心方法
        }
```
其中queueOrSendMessage方法主要就是向Handler发送了一个Message，让我们来看看具体的方法
```java
 // if the thread hasn't started yet, we don't have the handler, so just
    // save the messages until we're ready.
    private void queueOrSendMessage(int what, Object obj) {
        queueOrSendMessage(what, obj, 0, 0);
    }

    private void queueOrSendMessage(int what, Object obj, int arg1) {
        queueOrSendMessage(what, obj, arg1, 0);
    }

    private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        }
    }
```
容易知道，主要就是向Handler mH中发送一条消息，根据Handler消息循环机制，可以在handMessage查看处理方法，根据H.BIND_APPLICATION
```java
    private class H extends Handler {
           .....
        public static final int BIND_APPLICATION        = 110;
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                .......
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);//核心方法
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                 .......
       }
}
```
继续看handleBindApplication()方法
```java
private void handleBindApplication(AppBindData data) {
         try {

            .........

            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

           ...... 

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
            }
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
 }
```
通过分析知data为AppBindData，data.info是一个LoadeApk对象。data.info.makeApplication()，让我们继续看LoadeApk中的方法
```java
 public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            ContextImpl appContext = new ContextImpl();
            appContext.init(this, null, mActivityThread);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);//1.核心方法
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);//2.核心方法
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }
        
        return app;
    }
```
主要还是调ActivityThread中mInstrumentation.newApplication()方法，并传入了继承Application的类的类名appClass，当Applicaiton建立后，马上就调用了方法2即Application的onCreate(),让我们先来看Instrumentation中的newApplication()方法
```java
  
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        return newApplication(cl.loadClass(className), context);
    }
    
    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);//方法
        return app;
    }
```
到这里，就建立了应用程序的Application，然而在Application中那个方法是最先调用的呢？让我们继续看一下
```java
    /**
     * @hide
     */
    /* package */ final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
```
由此我们知道Application初始化后，首先调用的是attachBaseContext()方法，其次才是Application的onCreate方法，让我们看一下调用onCreate()方法,在前面的makeApplication()方法中，有此代码instrumentation.callApplicationOnCreate(app)，我们继续来看看源码
```java
    public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
```
到这里Android应用程序的入口源码就分析完了。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 四、总结
Android应用程序的入口是ActivityThread的main方法，而Android应用程序的开发入口是Application的attachBaseContext()和onCreate()。如果有类继承Application，并在Androidmanifest中配置了，它就会优先启动。在继承Application类初始化后，首先调用的是attachBaseContext()方法，其次才是onCreate方法。

# 五、相关及参考文档

[Activity启动过程全解析](http://blog.csdn.net/tenggangren/article/details/50925740)

[深入理解ActivityManagerService](http://blog.csdn.net/Innost/article/details/47254381)

[Android系统启动流程（一）解析init进程启动过程](http://liuwangshu.cn/framework/booting/1-init.html)

[Android系统启动流程（二）解析Zygote进程启动过程](http://liuwangshu.cn/framework/booting/2-zygote.html)

[Android系统启动流程（三）解析SyetemServer进程启动过程](http://liuwangshu.cn/framework/booting/3-syetemserver.html)

[Android系统启动流程（四）Launcher启动过程与系统启动流程](http://liuwangshu.cn/framework/booting/4-launcher.html)

[Android应用程序进程启动过程（前篇）](http://liuwangshu.cn/framework/applicationprocess/1.html)

[Android应用程序进程启动过程（后篇）](http://liuwangshu.cn/framework/applicationprocess/2.html)
