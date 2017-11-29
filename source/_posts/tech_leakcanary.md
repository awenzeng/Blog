---
layout: post
title: "LeakCanary框架源码解析"
date: 11/16/2017 5:43:41 PM 
comments: true
tags: 
	- 技术 
	- Android
	- 开源框架源码解析
	- LeakCanary框架源码分析
---
---
如果我们开发的程序，出现内存泄漏，导致程序奔溃，造成用户卸载APP。这样的结果,是我们不想见到的。作为一名向上的程序员，如何避免内存泄漏，这就成为必须要解决的问题。良心企业Square,开源了LeakCanary框架，可以轻松集成，让检测内存泄漏变得十分容易。

**什么是内存泄漏？**
内存泄漏是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

# 一、什么是LeakCanary?
LeakCanary 是一个检测内存泄露的开源类库。你可以在 debug 包种轻松检测内存泄露。

LeakCanary源码地址：[https://github.com/square/leakcanary](https://github.com/square/leakcanary)

# 二、LeakCanary源码解析

**1.LeakCanary入口**

```java
public class ExampleApplication extends Application {
  @Override public void onCreate() {
    super.onCreate();

    LeakCanary.install(this);//1.核心方法

  }
}

```
<!-- more -->
进入LeakCanary类
```java
  /**
   * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
   * references (on ICS+).
   */
  public static RefWatcher install(Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }

  /** Builder to create a customized {@link RefWatcher} with appropriate Android defaults. */
  public static AndroidRefWatcherBuilder refWatcher(Context context) {
    return new AndroidRefWatcherBuilder(context);
  }

```
refWatcher()返回AndroidRefWatcherBuilder对象，listenerServiceClass、excludedRefs和buildAndInstall皆为
AndroidRefWatcherBuilder的方法。这里我们先看AndroidRefWatcherBuilder中的buildAndInstall的方法

```java
  /**
   * Creates a {@link RefWatcher} instance and starts watching activity references (on ICS+).
   */
  public RefWatcher buildAndInstall() {
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      LeakCanary.enableDisplayLeakActivity(context);
      ActivityRefWatcher.install((Application) context, refWatcher);//2.核心方法
    }
    return refWatcher;
  }
```
这里创建了RefWatcher，并把其传给了ActivityRefWatcher。进入ActivityRefWatcher类
```java
  public static void install(Application application, RefWatcher refWatcher) {
    new ActivityRefWatcher(application, refWatcher).watchActivities();
  }

  /**
   * Constructs an {@link ActivityRefWatcher} that will make sure the activities are not leaking
   * after they have been destroyed.
   */
  public ActivityRefWatcher(Application application, RefWatcher refWatcher) {
    this.application = checkNotNull(application, "application");
    this.refWatcher = checkNotNull(refWatcher, "refWatcher");
  }
```
创建ActivityRefWatcher类的对象，并且调用了watchActivities()方法，我们继续看
```java
 private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new Application.ActivityLifecycleCallbacks() {
        @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }

        @Override public void onActivityStarted(Activity activity) {
        }

        @Override public void onActivityResumed(Activity activity) {
        }

        @Override public void onActivityPaused(Activity activity) {
        }

        @Override public void onActivityStopped(Activity activity) {
        }

        @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }

        @Override public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);//3.核心方法
        }
      };


  public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);//4.核心方法，设置监听所有Activity的生命周期
  }
```
**application.registerActivityLifecycleCallbacks(lifecycleCallbacks)，设置监听了应用Activity的生命周期，可以监听所有Activity**。当Activity调onDestory方法时，都会调 ActivityRefWatcher.this.onActivityDestroyed(activity)，我们继续看
```java
  void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);//5.核心方法
  }
```
**2.核心类RefWatcher**
```java
  /**
   * Identical to {@link #watch(Object, String)} with an empty string reference name.
   *
   * @see #watch(Object, String)
   */
  public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }

  /**
   * Watches the provided references and checks if it can be GCed. This method is non blocking,
   * the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
   * with.
   *
   * @param referenceName An logical identifier for the watched object.
   */
  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);//6.核心方法

    ensureGoneAsync(watchStartNanoTime, reference);//7.核心方法
  }
```
这里对watchedReference即Activity对象建立了一个弱应用KeyedWeakReference，并且对KeyedWeakReference加了一个引用队列queue（ReferenceQueue）。当KeyedWeakReference对象可以回收时，会添加到ReferenceQueue中，我们继续
```java
  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);//8.核心方法
      }
    });
  }
```
通过watchExecutor开启了一个线程，执行ensureGone。**ensureGone方法可以说是LeakCanary框架最最核心的方法，核心原理都在这里。**
```java
@SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();//i.回收弱引用和删除key

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {//ii.判断弱引用是否回收，若回收true,不存在泄漏
      return DONE;
    }
    gcTrigger.runGc();//iii.手动GC
    removeWeaklyReachableReferences();//iv.再次回收弱引用和删除key
    if (!gone(reference)) {//v.再判断弱引用是否回收，若回收true,不存在泄漏，false则存在泄漏
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(//vi.分析Hprof文件,找出泄漏的最短
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }

  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

  private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```
其中，方法removeWeaklyReachableReferences()是回收弱引用及删除key，方法gone(reference)判断弱引用对象是否被回收。


如代码中的注释，通过多次判断Activity的弱应用是否被回收，判断Activity是否内存泄漏。如泄漏，生成Hprof文件，通过Square的haha开源库分析泄漏引用链，从而将其应用链传给界面展示出来，让开发者知道。

到此，LeakCanary原理分析就告一段落了。如果你还想知道LeakCanary是怎么找到泄漏引用链，并传给界面展示的，你还可以继续分析 heapdumpListener.analyze();入口地方AndroidRefWatcherBuilder中的listenerServiceClass的方法与DisplayLeakService类。

# 三、总结
通过阅读LeakCanary框架的源码，知LeakCanary框架原理还是比较简单的，主要就是通过Activity弱引用(KeyedWeakReference)是否被回收，来判断是否内存是否泄漏。


# 四、相关及参考文档
[LeakCanary 中文使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)

[LeakCanary核心原理源码浅析](http://blog.csdn.net/cloud_huan/article/details/53081120)

[LeakCanary 原理浅析](http://www.jianshu.com/p/3f1a1cc1e964)

[leakcanary原理分析与AppsFly内存泄漏](http://blog.csdn.net/ahong222/article/details/52295844)

[十分钟理解Java中的弱引用](http://www.importnew.com/21206.html)




