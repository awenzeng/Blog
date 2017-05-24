---
layout: post
title: "【Java基础】SimpleDateFormat与Calendar使用详解"
date: 5/17/2017 6:20:05 PM 
comments: true
tags: 
	- 技术 
	- Java
	- Java基础
---
---
# 1.SimpleDateFormat类

**Format语法**

| 标识 | 标识代表意 | 
| :-----: | :----: |
|G|年代标志符|
|y|年|
|M|月|
|d|日|
|h|时 在上午或下午 (1~12)|
|H|时 在一天中 (0~23)|
|m|分|
|s|秒|
|S|毫秒|
|E|星期|
|D|一年中的第几天|
|F|一月中第几个星期几|
|w|一年中第几个星期|
|W|一月中第几个星期|
|a|上午 / 下午 标记符| 
|k|时 在一天中 (1~24)|
|K|时 在上午或下午 (0~11)|
|z|时区|
<!-- more -->
**Java代码示例：**
```java
        SimpleDateFormat myFmt  = new SimpleDateFormat("yyyy年MM月dd日 HH时mm分ss秒");
        SimpleDateFormat myFmt1 = new SimpleDateFormat("yy/MM/dd HH:mm");
        SimpleDateFormat myFmt2 = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        SimpleDateFormat myFmt3 = new SimpleDateFormat("yyyy年MM月dd日 HH时mm分ss秒 E ");
        SimpleDateFormat myFmt4 = new SimpleDateFormat("一年中的第 D 天 一年中第w个星期 一月中第W个星期 在一天中k时 z时区");
        Date now=new Date();
        System.out.println(myFmt.format(now));
        System.out.println(myFmt1.format(now));
        System.out.println(myFmt2.format(now));
        System.out.println(myFmt3.format(now));
        System.out.println(myFmt4.format(now));
        System.out.println(now.toGMTString());
        System.out.println(now.toLocaleString());
        System.out.println(now.toString());
````

**输出结果：**
> 2017年05月23日 10时53分28秒
> 17/05/23 10:53
> 2017-05-23 10:53:28
> 2017年05月23日 10时53分28秒 星期二 
> 一年中的第 143 天 一年中第21个星期 一月中第4个星期 在一天中10时 CST时区
> 23 May 2017 02:53:28 GMT
> 2017-5-23 10:53:28
> Tue May 23 10:53:28 CST 2017

# 2.Calendar类

|Letter|	Date or Time Component	|Presentation	|Examples|
| :-----: | :----: | :-----: | :----: |
|G	|Era designator	|Text	|AD|
|y	|Year|	Year|1996; 96|
|Y	|Week year|	Year|	2009; 09|
|M|	Month in year (context sensitive)|	Month|	July; Jul; 07
|L|	Month in year (standalone form)	Month	|July; Jul; 07|
|w|	Week in year|	Number|27|
|W	|Week in month|	Number|	2|
|D	|Day in year	|Number|	189|
|d|	Day in month|	Number|	10|
|F|	Day of week in month|	Number|	2|
|E|	Day name in week|	Text|	Tuesday; Tue|
|u	|Day number of week (1 = Monday, ..., 7 = Sunday)|	Number|	1|
|a|	Am/pm marker|	Text	|PM|
|H|	Hour in day (0-23)|	Number|	0|
|k|	Hour in day (1-24)|	Number|	24|
|K|	Hour in am/pm (0-11)|	Number|	0|
|h|	Hour in am/pm (1-12)|	Number|	12|
|m|	Minute in hour|	Number|	30|
|s	|Second in minute|	Number|	55|
|S|	Millisecond|Number|	978|
|z	|Time zone|	General time zone|	Pacific Standard Time; PST; GMT-08:00|
|Z|	Time zone	|RFC 822 time zone|	-0800|
|X	|Time zone	|ISO 8601 time zone	-08;| -0800; -08:00|

**Java代码示例：**
```java
 // Calendar 取得当前时间的方法  
    // 初始化 (重置) Calendar 对象  
    calendar = Calendar.getInstance();  
    // 或者用 Date 来初始化 Calendar 对象  
    calendar.setTime(new Date());  
  
    // 显示年份  
    int year = calendar.get(Calendar.YEAR);  
    System.out.println("year is = " + String.valueOf(year));  
  
    // 显示月份 (从0开始, 实际显示要加一)  
    int month = calendar.get(Calendar.MONTH);  
    System.out.println("nth is = " + (month + 1));  
  
    // 本周几  
    int week = calendar.get(Calendar.DAY_OF_WEEK);  
    System.out.println("week is = " + week);  
  
    // 今年的第 N 天  
    int DAY_OF_YEAR = calendar.get(Calendar.DAY_OF_YEAR);  
    System.out.println("DAY_OF_YEAR is = " + DAY_OF_YEAR);  
  
    // 本月第 N 天  
    int DAY_OF_MONTH = calendar.get(Calendar.DAY_OF_MONTH);  
    System.out.println("DAY_OF_MONTH = " + String.valueOf(DAY_OF_MONTH));  
  
    // 3小时以后  
    calendar.add(Calendar.HOUR_OF_DAY, 3);  
    int HOUR_OF_DAY = calendar.get(Calendar.HOUR_OF_DAY);  
    System.out.println("HOUR_OF_DAY + 3 = " + HOUR_OF_DAY);  
  
    // 当前分钟数  
    int MINUTE = calendar.get(Calendar.MINUTE);  
    System.out.println("MINUTE = " + MINUTE);  
  
    // 15 分钟以后  
    calendar.add(Calendar.MINUTE, 15);  
    MINUTE = calendar.get(Calendar.MINUTE);  
    System.out.println("MINUTE + 15 = " + MINUTE);  
  
    // 30分钟前  
    calendar.add(Calendar.MINUTE, -30);  
    MINUTE = calendar.get(Calendar.MINUTE);  
    System.out.println("MINUTE - 30 = " + MINUTE);  
  
    str = (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:SS")).format(calendar.getTime());  
    System.out.println(str);  
  
    // 重置 Calendar 显示当前时间  
    calendar.setTime(new Date());  
    str = (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:SS")).format(calendar.getTime());  
    System.out.println(str);  
  
    // 创建一个 Calendar 用于比较时间  
    Calendar calendarNew = Calendar.getInstance();  
  
    // 设定为 5 小时以前，后者大，显示 -1  
    calendarNew.add(Calendar.HOUR, -5);  
    System.out.println("时间比较：" + calendarNew.compareTo(calendar));  
  
    // 设定7小时以后，前者大，显示 1  
    calendarNew.add(Calendar.HOUR, +7);  
    System.out.println("时间比较：" + calendarNew.compareTo(calendar));  
  
    // 退回 2 小时，时间相同，显示 0  
    calendarNew.add(Calendar.HOUR, -2);  
    System.out.println("时间比较：" + calendarNew.compareTo(calendar));  
```

**输出结果：**

>year is = 2017
nth is = 5
week is = 3
DAY_OF_YEAR is = 143
DAY_OF_MONTH = 23
HOUR_OF_DAY + 3 = 14
MINUTE = 7
MINUTE + 15 = 22
MINUTE - 30 = 52
时间比较：-1
时间比较：-1
时间比较：-1

