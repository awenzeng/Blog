---
layout: post
title: "【Android基础】Activity的启动模式"
date: 3/25/2017 12:02:11 PM 
comments: true
tags: 
	- 技术
	- Android 
	- Android基础
---
---
 在Android中，当我们多次启动同一个Activity时，系统会创建多个实例，并把它们按照先进后出的原则一一放入任务栈中，当我们按back键时，就会有一个activity从任务栈顶移除，重复下去，直到任务栈为空，系统就会回收这个任务栈。但是这样以来，系统多次启动同一个Activity时就会重复创建多个实例，这种做法显然不合理，为了能够优化这个问题，Android提供四种启动模式来修改系统这一默认行为。
> 四种启动模式分别为：
> 
- standard
- singleTop
- singleTask
- singleInstance
 
   
>启动模式配置
```xml
<activity android:name=".Activity" android:launchMode="启动模式">
```

# Activity的四种启动模式

----------

**1.Standard-默认模式**

默认模式，可以不用写配置。在这个模式下，都会默认创建一个新的实例。因此，在这种模式下，可以有多个相同的实例，也允许多个相同Activity叠加。
<!-- more -->
**2.SingleTop-栈顶复用模式** 

可以有多个实例，但是不允许多个相同Activity叠加。三种情况：
> 1.如果当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewIntent方法
> 
> 2.当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例
> 
> 3.当前栈中不存在该Activity的实例时，其行为同standard启动模式


>应用场景:
>适合接收通知启动的内容显示页面。例如，某个新闻客户端的新闻内容页面，如果收到10个新闻推送，每次都打开一个新闻内容页面是很烦人的。


**3.SingleTask-栈内复用模式**

只有一个实例。在同一个应用程序中启动他的时候，若Activity不存在，则会在当前task创建一个新的实例，若存在，则会把task中在其之上的其它Activity destory掉并调用它的onNewIntent方法。

如果是在别的应用程序中启动它，则会新建一个task，并在该task中启动这个Activity，singleTask允许别的Activity与其在一个task中共存，也就是说，如果我在这个singleTask的实例中再打开新的Activity，这个新的Activity还是会在singleTask的实例的task中。

>应用场景:
>适合作为程序入口点。例如浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。

**4.SingleInstance-全局唯一模式**

只有一个实例，并且这个实例独立运行在一个task中，这个task只有这个实例，不允许有别的Activity存在。

>应用场景:
>
>适合需要与程序分离开的页面。例如闹铃提醒，将闹铃提醒与闹铃设置分离。


### Note：Activity的标签属性（taskAffinity)
taskAffinity属性
>   每个Activity都有taskAffinity属性，这个属性指出了它希望进入的任务栈。如果一个Activity没有显式的指明该 Activity的taskAffinity，那么它的这个属性就等于Application指明的taskAffinity，如果 Application也没有指明，那么该taskAffinity的值就等于包名。而任务栈也有自己的affinity属性，它的值等于它的根 Activity的taskAffinity的值。taskAffinity代码配置：
```xml
activity android:name=".Activity" android:launchMode="启动模式" android:taskAffinity="任务栈名（如：包名）"/>
```