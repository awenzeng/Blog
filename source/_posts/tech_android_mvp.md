---
layout: post
title: "MVP模式学习及使用"
date: 4/11/2017 6:41:32 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android基础
	- MVP
---
---
Google官方推出MVP模式有一段时间，MVP模式在android项目中使用也越来越广泛。作为一个Android开发人员，学会使用MVP模式，那也相当的重要。
# 一、什么是MVP
MVP从MVC架构模式演化而来， MVC分别代表模型、视图和控制器，在Android中，定义Class类作为模型，Layout XML表示视图，而Activity用作控制器，这样一来，在Activity中充斥了大量代码，无论是从扩展性和重用性都无法达到理想的效果。所以个人认为，MVC分层在Android App开发中没有解决问题。而MVP解决了这个问题。

 MVP即为Model、View和Presenter。Model表示模型，实现数据存储与业务逻辑；View表示视图，提供用户交互的接口；Presenter表示主导器，相当于MVC中的Controller但比Controller更灵活。MVP的关系如图所示。

![](http://img.blog.csdn.net/20160605143622019?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### **从上图可以看出：**

A） View将功能委托给Presenter完成，Presenter调用Model完成业务功能与数据存储，并再次通过View更新UI；

B） View和Model没有直接关联，无法相互调用；

C） Presenter和View可以相互调用；

D） Presenter调用Model完成业务功能。

<!-- more -->
### **MVP的优点：**

A） 各个层次之间的职责更加单一清晰；

B） 很大程度上降低了代码的耦合度；

C） 复用性大大提高；

D） 面向接口编程，定义与实现分离，方便测试与维护；

E） 代码更简洁。
 
 

### **MVP的缺点：**

A） 类变得更多了；

B） 组件与组件之间的关系很复杂。

# 二、MVP的使用
Google官方在推出MVP模式时，也给出了一个使用MVP模式的DEMO([TODO-MVP](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)),通过此Demo，细细阅读，细细评味。相信，很快你就会掌握MVP模式的使用方法。

DEMO项目截图:

![](http://7xohx8.com1.z0.glb.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-04-10%20%E4%B8%8B%E5%8D%883.53.20.png)

使用注意：虽然MVP模式非常的给力，但我们也不能乱用。需要结合我们项目实际情况来使用，对于功能比较简单的界面，其实MVC模式相对来说还是比较适合，毕竟功能简单，代码量也不会很多。对于功能比较复杂的界面，建议使用MVP模式来优化流程。

# 三、相关链接

[Android Architecture Blueprints 学习之 TODO-MVP（一）](http://www.tuicool.com/articles/zqiiu2y);

[Android Architecture Blueprints 学习之 TODO-MVP（二）](http://www.tuicool.com/articles/qyIVV3q);

[Android Architecture Blueprints 学习之 TODO-MVP（三）](http://www.tuicool.com/articles/UrmMfyB);