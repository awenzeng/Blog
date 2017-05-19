---
layout: post
title: "【响应式编程】Rxjava学习总结"
date: 3/16/2017 9:06:13 PM 
comments: true
tags: 
	- 技术 
	- Rxjava
	- Rxandroid
	- 响应式编程
---
---
# 1.[RxJava基础详解-扔物线](http://gank.io/post/560e15be2dca930e00da1083) #
# 2.RxJava常用方法总结 #
RxJava 的观察者模式

![](http://ww3.sinaimg.cn/mw1024/52eb2279jw1f2rx4446ldj20ga03p74h.jpg)


![](http://ww3.sinaimg.cn/mw1024/52eb2279jw1f2rx46dspqj20gn04qaad.jpg)


Observable.just(T...)方法
>![](http://ww4.sinaimg.cn/mw1024/52eb2279jw1f2rx489robj20lk0a8my2.jpg)
>Observable.just()动画显示
>![](http://ww3.sinaimg.cn/mw1024/52eb2279jw1f2rx4ay0hrg20ig08wk4q.gif)

Observable.from(T[])分发集合方法(图类似just())

<!-- more -->
Observable.map()直接变换
>![](http://ww1.sinaimg.cn/mw1024/52eb2279jw1f2rx4fitvfj20hw0ea0tg.jpg)

Observable.flatMap()平铺变换
>![](http://ww1.sinaimg.cn/mw1024/52eb2279jw1f2rx4i8da2j20hg0dydgx.jpg)

# 3.RxJava线程调度 #
1.RxJava线程控制
>Observable.subscribeOn(Schedulers.io())指定被观察运行线程(订阅线程)
>
Observable.observeOn(AndroidSchedulers.mainThread)指定观察者运行线程
>
Observable.doOnSubscribe(Schedulers.io())被观察者开始执行前调用

2.Schedulers介绍

 > Schedulers.immediate(): 直接在当前线程运行，相当于不指定线程（这是默认的）。
 > 
 > Schedulers.newThread(): 总是启用新线程，并在新线程执行操作。
 > 
 > Schedulers.io(): I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。
 > 
 >Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
 >
 >AndroidSchedulers.mainThread()，是RxAndroid 中一个对 RxJava 的轻量级扩展为了Android 的主线程提供 Scheduler，它指定的操作将在 Android 主线程运行。   

# 4.结尾 #
以上图片资源皆来至于 [RxJava基础详解-扔物线](http://gank.io/post/560e15be2dca930e00da1083)


