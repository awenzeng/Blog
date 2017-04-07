---
layout: post
title: "Android技术知识要点"
date: 4/5/2017 6:57:14 PM 
comments: true
tags: 
	- 技术 
---
---
# 一、项目中使用库工程问题要点
##### 1.库工程与主工程资源冲突问题

  当运行有引用library工程的android工程时，android工具将会合并library工程与主工程的所有资源。如果一个资源ID将有可能在library工程之间或library工程、主工程之间都有定义，这时候优先级别高的资源ID将覆盖优先级别低的，使用资源时将使用有线级别最高的工程的资源。工程之间优先级别如何判定，请看下一条。

##### 2.库工程之间以及主工程的资源使用上的优先级问题

上图显示一个android工程引用了四个library工程，这四个library工程和主工程之间是有优先级之分的。android主工程的优先级别最高，四个library工程科举上图排序有上到下优先级别依次降低。library工程之间也可以手动排序，选择其中一个，点击up(提高优先级)或者down（降低优先级）。

##### 3.库工程和主工程使用不同的android platform version问题

主工程打包时，android sdk版本使用的是主工程。所以library工程使用的android sdk版本要不高于主工程的sdk版本。如果library工程sdk版本高于主工程，将不能通过编译。
<!-- more -->
# 二、自定义ViewGroup或Canvas获取绘制内容Bitmap
* 可以通过setDrawingCacheEnabled，然后再getDrawingCache(),但这个你得保证onDraw被调用。
* 从Canvas获取Bitmap(自定义View类似),例子：

```java
public Bitmap getBitmap() {  
        Bitmap whiteBgBitmap = Bitmap.createBitmap(mBackgroundBitmap.getWidth(), mBackgroundBitmap.getHeight(),  
                Bitmap.Config.ARGB_8888);  
        Canvas canvas = new Canvas(whiteBgBitmap);  
        canvas.drawColor(Color.WHITE);  
        canvas.drawBitmap(mBackgroundBitmap, 0, 0, null);  
        return whiteBgBitmap;  
    }  
``` 