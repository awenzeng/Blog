---
layout: post
title: "Activity启动流程源码分析(应用中)"
date: 12/26/2017 8:25:27 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Activity启动流程
	- Android框架源码解析
---
---
在移动应用开发中，Android四大组件之一Activity是最常用的。很多界面，如：闪屏、主界面、次功能界面等都需要Activity来作为主要的载体；界面与界面之间，即不同的Activity之间也都存在跳转切换，弄懂这其中跳转切换原理，将有助于我们更好的理解Android中Activity之间的交互逻辑，从而更好的开发Android应用。本篇博文将会重点介绍Android应用中的Activity的启动流程。

在开始介绍之前，我们需要了解一些概念，如：

- **ActivityThread：** 应用的启动入口类，当应用启动，会首先执行其main方法，开启主线程消息循环机制。
- **ApplicationThread：** ActivityThread的内部类，主要与系统进程AMS通信，从而对应用进程的具体Activity操作进行管理。
- **Instrumentation：** ActivityThread的属性变量，主要辅助ActivityThread类调用Activity的生命周期相关方法。
- **ActivityManagerService(AMS)：** Activity管理系统服务类，主要是对所有的Activity进行管理。
- **ActivityStack：** Activity任务栈，AMS的属性变量，AMS中Activtiy的实际管理者。


# 一、Activity启动流程
Activity启动流程图：
<!-- more -->

![](/assets/img/tech_activity_start_flow.png)

此流程图，主要是根据Android源码中代码执行顺序来梳理的。浅绿色部分为应用进程，浅蓝色部分为系统服务进程，两个进程间通过Binder驱动来进行通信，第一次Binder通信主要的类有：ActivityManagerService(AMS),ActivityManagerNative(AMN),ActivityManagerProxy(AMP)；第二次Binder通信主要的类有:ApplicationThread(AT),ApplicationThreadNative(ATN)，ApplicationThreadProxy(ATP)。

# 二、Activity启动流程源码分析
根据上面流程图，下面让我们一起来看看源码，首先从Activity的startActivity开始：
```java

    @Override
    public void startActivity(Intent intent) {
        startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

    public void startActivityForResult(Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }

    public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
        if (mParent == null) {//1.核心代码
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }
        } else {//2.核心代码
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);

            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }

```
在Activity源码中，startActivity之后都会调用startActivityForResult；在注释1处，当mParent为空时，会直接调用Instrumentation中的execStartActivity方法，当mParent不为空时，调用mParent.startActivityFromChild方法。通过跟踪查找发现，mParent也是Activity，在Activity attach的时候会初始化，从ActivityRecord中获得值。我们继续来看看startActivityFromChild方法
```java
    public void startActivityFromChild(Activity child, Intent intent,
            int requestCode) {
        startActivityFromChild(child, intent, requestCode, null);
    }

    public void startActivityFromChild(Activity child, Intent intent, 
            int requestCode, Bundle options) {
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, child,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, child.mEmbeddedID, requestCode,
                ar.getResultCode(), ar.getResultData());
        }
    }
```
由此发现，startActivityForResult之后都调用了Instrumentation中的execStartActivity方法。我们继续来看看execStartActivity方法：
```java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        ......
        try {
            intent.setAllowFds(false);
            intent.migrateExtraStreamToClipData();
            //核心代码
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```
这里主要是调用了ActivityManagerNative.getDefault()中的startActivity(...)方法，这里就涉及到Binder的一次跨进程通信，通过跨进程通信调用了ActivityManagerService中的startActivity方法。具体Binder怎么跨进程通信的，我已写过文章[ Android跨进程通信方式(IPC)解析](http://blog.csdn.net/awenyini/article/details/78815629)，想了解的同学，可以点击看看。下面我们继续来看看AMS中的startActivity方法：
```java
    public final int startActivity(IApplicationThread caller,
            Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags,
            String profileFile, ParcelFileDescriptor profileFd, Bundle options) {
        enforceNotIsolatedCaller("startActivity");
        ......
        return mMainStack.startActivityMayWait(caller, -1, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profileFile, profileFd,
                null, null, options, userId);
    }
```
在AMS的startActivity方法中，又调用ActivityStack中的startActivityMayWait()方法,我们再来看看ActivityStack的源码：
```java
 final int startActivityMayWait(IApplicationThread caller, int callingUid,
            Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, String profileFile,
            ParcelFileDescriptor profileFd, WaitResult outResult, Configuration config,
            Bundle options, int userId) {
       
            ......
            
            //核心代码
            int res = startActivityLocked(caller, intent, resolvedType,
                    aInfo, resultTo, resultWho, requestCode, callingPid, callingUid,
                    startFlags, options, componentSpecified, null);
            
           ......
            
           return res;
        }
    }
```
我们这里主要分析启动流程，所以省略掉部分细节。让我们继续看ActivityStack中的startActivityLocked()方法
```java
  final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo, IBinder resultTo,
            String resultWho, int requestCode,
            int callingPid, int callingUid, int startFlags, Bundle options,
            boolean componentSpecified, ActivityRecord[] outActivity) {

        ......
        
        //创建一个新的ActivityRecord
        ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
                intent, resolvedType, aInfo, mService.mConfiguration,
                resultRecord, resultWho, requestCode, componentSpecified);
        ......

        err = startActivityUncheckedLocked(r, sourceRecord,
                startFlags, true, options);
         ......
        return err;
    }
```
同上，也省略的部分细节。我们继续
```java
    final int startActivityUncheckedLocked(ActivityRecord r,
            ActivityRecord sourceRecord, int startFlags, boolean doResume,
            Bundle options) {
       ......

        if (sourceRecord == null) {
            if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            }
        } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {        
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {        
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        }
        
        ......//省略代码：Activity四种启动模式standard,singleTop,singleTask,singleInstance的判断

        if (sourceRecord != null) {
           .......
           if (!addingToTask &&(launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
                // In this case, we are launching an activity in our own task
                // that may already be running somewhere in the history, and
                // we want to shuffle it to the front of the stack if so.
               
                int where = findActivityInHistoryLocked(r, sourceRecord.task.taskId);
                if (where >= 0) {
                    ActivityRecord top = moveActivityToFrontLocked(where);
                    logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                    top.updateOptionsLocked(options);
                    top.deliverNewIntentLocked(callingUid, r.intent);
                    if (doResume) {
                        resumeTopActivityLocked(null);//核心代码
                    }
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            }
            // An existing activity is starting this new activity, so we want
            // to keep the new one in the same task as the one that is starting
            // it.
            r.setTask(sourceRecord.task, sourceRecord.thumbHolder, false);
            if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                    + " in existing task " + r.task);

        } else {
          ......
        }
        ......
        return ActivityManager.START_SUCCESS;
    }

```
在startActivityUncheckedLocked()方法中，主要针对Activity的启动模式进行了检测判断，从而启动Activity。我们知道，Activity有四种启动模式，分别为standard,singleTop,singleTask和singleInstance,但这里我们主要是分析Activity的启动流程，所以具体启动模式的判断逻辑细节，这里就不展开分析了。我们主要来看一下，把Activity启动放到栈顶的方法resumeTopActivityLocked()
```java
  final boolean resumeTopActivityLocked(ActivityRecord prev) {
        return resumeTopActivityLocked(prev, null);
    }

    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
       
        //找到一个栈顶的未finish的Activity的ActivityRecord
        ActivityRecord next = topRunningActivityLocked(null);

        ......

        if (next == null) {//栈顶无Activity，直接启动Launcher        
            if (mMainStack) {
                ActivityOptions.abort(options);
                return mService.startHomeActivityLocked(0);
            }
        }

        ......

        //如果Activity所在的进程已经存在
        if (next.app != null && next.app.thread != null) {
           ......
            
            try {
                .......
                //重新显示Activity
                if (next.newIntents != null) {
                    next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
                }

                EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY,
                        System.identityHashCode(next),
                        next.task.taskId, next.shortComponentName);
                
                next.sleeping = false;
                showAskCompatModeDialogLocked(next);
                next.app.pendingUiClean = true;
                //执行Activity onResume方法
                next.app.thread.scheduleResumeActivity(next.appToken,
                        mService.isNextTransitionForward());
                
                checkReadyForSleepLocked();

            } catch (Exception e) {

                //如果启动异常，就重启Activity
                if (DEBUG_STATES) Slog.v(TAG, "Resume failed; resetting state to "
                        + lastState + ": " + next);
                next.state = lastState;
                mResumedActivity = lastResumedActivity;
                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else {
                    if (SHOW_APP_STARTING_PREVIEW && mMainStack) {
                        mService.mWindowManager.setAppStartingWindow(
                                next.appToken, next.packageName, next.theme,
                                mService.compatibilityInfoForPackageLocked(
                                        next.info.applicationInfo),
                                next.nonLocalizedLabel,
                                next.labelRes, next.icon, next.windowFlags,
                                null, true);
                    }
                }
                startSpecificActivityLocked(next, true, false);//核心代码，重启Activity
                return true;
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.visible = true;
                completeResumeLocked(next);
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception");
                return true;
            }

            // Didn't need to use the icicle, and it is now out of date.
            if (DEBUG_SAVED_STATE) Slog.i(TAG, "Resumed activity; didn't need icicle of: " + next);
            next.icicle = null;
            next.haveState = false;
            next.stopped = false;

        } else {
            //Activity所在的进程不存在，启动Activity
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mService.mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
            }
            startSpecificActivityLocked(next, true, true);//启动Activity
        }

        return true;
    }
```
通过上面注释中的分析，在判断Activity进程之后，就会通过startSpecificActivityLocked()方法来启动Activity,我们继续看
```java
 private final void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid);
        
        if (r.launchTime == 0) {
            r.launchTime = SystemClock.uptimeMillis();
            if (mInitialStartTime == 0) {
                mInitialStartTime = r.launchTime;
            }
        } else if (mInitialStartTime == 0) {
            mInitialStartTime = SystemClock.uptimeMillis();
        }
        
        if (app != null && app.thread != null) {//Activity所在进程判断，进程存在时，直接启动Activity
            try {
                app.addPackage(r.info.packageName);

                realStartActivityLocked(r, app, andResume, checkConfig);//核心代码

                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false);
    }
```
在startSpecificActivityLocked()方法中也对Activity的进程是否存在做了判断，当进程存在时直接调用realStartActivityLocked()方法启动Activity；当Activity的进程不存在时，就会调用AMS的startProcessLocked()方法创建进程，这里其实是Activity的另一种启动流程，从Laucher启动，只有从Launcher启动才会没有进程，这里先不做深度分析，后续我们针对Activity的Launcher启动再写一篇博文。已补博文[Activity启动流程源码分析(Launcher中)](http://blog.csdn.net/awenyini/article/details/78915225)。下面让我们继续看realStartActivityLocked()方法：
```java
  final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
             
            .......

            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    new Configuration(mService.mConfiguration),
                    r.compat, r.icicle, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profileFile, profileFd,
                    profileAutoStop);
            
           ......
        
        return true;
    }
```
这里主要通过调用app.thread.scheduleLaunchActivity(...)方法实现了跨进程通信，这里主要实现了流程图中的第二次Binder跨进程通信。通过Binder跨进程通信调用了ApplicationThread中的scheduleLaunchActivity(...)方法，具体Binder怎么跨进程通信的，我已写过文章[ Android跨进程通信方式(IPC)解析](http://blog.csdn.net/awenyini/article/details/78815629)，想了解的同学，可以点击看看。下面我们继续来看看ApplicationThread中的scheduleLaunchActivity方法：
```java
  public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
                Bundle state, List<ResultInfo> pendingResults,
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
                String profileName, ParcelFileDescriptor profileFd, boolean autoStopProfiler) {
            ActivityClientRecord r = new ActivityClientRecord();
            ......

            queueOrSendMessage(H.LAUNCH_ACTIVITY, r);
   }

  public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);//核心代码
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
             .......
           }
}
```
由上易知，通过Handler消息循环机制，从而执行handleLaunchActivity()方法，我们继续来看此方法
```java
  private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
      
        .......

        Activity a = performLaunchActivity(r, customIntent);

        ......
    }

   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        Activity activity = null;
        try {
            //1.核心代码
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader(); 
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                ContextImpl appContext = new ContextImpl();
                appContext.init(r.packageInfo, r.token, this);
                appContext.setOuterContext(activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                        
                //2.核心代码
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //3.核心代码
                mInstrumentation.callActivityOnCreate(activity, r.state);

                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }

                if (!r.activity.mFinished) {
                    if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }

                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```
注释1处，通过mInstrumentation.newActivity()方法对Activity进行初始化
```java
    public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```
由上我们知道，主要通过反射机制实现Activity的初始化。再来看注释2，调用了Activity.attach(...)方法
```java
 final void attach(Context context, ActivityThread aThread, Instrumentation instr, IBinder token,
            Application application, Intent intent, ActivityInfo info, CharSequence title, 
            Activity parent, String id, NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config) {
        attach(context, aThread, instr, token, 0, application, intent, info, title, parent, id,
            lastNonConfigurationInstances, config);
    }
    
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config) {

        attachBaseContext(context);

        mFragments.attachActivity(this);
        
        mWindow = PolicyManager.makeNewWindow(this);
        mWindow.setCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
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
此方法主要就是对Activity进行了初始化，初始化了许多的属性，具体如上。我们再看注释3，方法mInstrumentation.callActivityOnCreate(activity, r.state)，我们也来看看源码
```java
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        if (mWaitingActivities != null) {
            synchronized (mSync) {
                final int N = mWaitingActivities.size();
                for (int i=0; i<N; i++) {
                    final ActivityWaiter aw = mWaitingActivities.get(i);
                    final Intent intent = aw.intent;
                    if (intent.filterEquals(activity.getIntent())) {
                        aw.activity = activity;
                        mMessageQueue.addIdleHandler(new ActivityGoing(aw));
                    }
                }
            }
        }
        
        activity.performCreate(icicle);//核心代码
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
```
其实，主要也就是调用了Activity的OnCreate()方法，我们继续来看看
```java
    final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
    }
```
的确如此，最后调用了Activity的OnCreate方法，从而就启动了Activity。好了，到这里，Activity的启动流程就说完了。


**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 三、参考文档

[Activity启动流程源码分析(Launcher中)](http://blog.csdn.net/awenyini/article/details/78915225)

[Android Activity启动流程源码全解析（1）](https://www.jianshu.com/p/8a8ec5c17495)

[Android Activity启动流程源码全解析（2）](https://www.jianshu.com/p/067acea47ba6)
