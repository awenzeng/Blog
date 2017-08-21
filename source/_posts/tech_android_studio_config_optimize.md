---
layout: post
title: "【转载】Android Studio Gradle编译优化方法"
date: 8/8/2017 2:30:24 PM 
comments: true
tags: 
	- 技术 
	- AS Gradle优化
---
---
前言：最近发现Gradle项目编译越来越慢，有时甚至需要好几十分钟，实在是让人难以忍受。于是，便搜寻于网络，经过验证，发现此方案尤其有效，故留此博客，以备以后之需。

# 第1步：
打开AS安装所在的位置，用记事本打开studio64.exe.vmoptions文件。具体如图：
![](/assets/img/as_gradle_first.jpg)

# 第2步：
打开“studio64.exe.vmoptions”文件后修改里面的值，修改后如下：

```java
#
# *DO NOT* modify this file directly. If there is a value that you would like to override,
# please add it to your user specific configuration file.
#
# See http://tools.android.com/tech-docs/configuration
#
-Xms2048m
-Xmx2048m
-XX:MaxPermSize=2048m
-XX:ReservedCodeCacheSize=1024m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-da
-Djna.nosys=true
-Djna.boot.library.path=
 
-Djna.debug_load=true
-Djna.debug_load.jna=true
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError
-Didea.paths.selector=AndroidStudio2.0
-Didea.platform.prefix=AndroidStudio
```
<!-- more -->
# 第3步：
打开你的项目中的“gradle.properties”文件复制如下内容：

```java
# Project-wide Gradle settings.
# IDE (e.g. Android Studio) users:
# Settings specified in this file will override any Gradle settings
# configured through the IDE.
# For more details on how to configure your build environment visit
# http://www.gradle.org/docs/current/userguide/build_environment.html
# The Gradle daemon aims to improve the startup and execution time of Gradle.
# When set to true the Gradle daemon is to run the build.
# TODO: disable daemon on CI, since builds should be clean and reliable on servers
org.gradle.daemon=true
# Specifies the JVM arguments used for the daemon process.
# The setting is particularly useful for tweaking memory settings.
# Default value: -Xmx10248m -XX:MaxPermSize=256m
org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
# When configured, Gradle will run in incubating parallel mode.
# This option should only be used with decoupled projects. More details, visit
# http://www.gradle.org/docs/current/userguide/multi_project_builds.html#sec:decoupled_projects
org.gradle.parallel=true
# Enables new incubating mode that makes Gradle selective when configuring projects.
# Only relevant projects are configured which results in faster builds for large multi-projects.
# http://www.gradle.org/docs/current/userguide/multi_project_builds.html#sec:configuration_on_demand
org.gradle.configureondemand=true
```
# 第4步：
修改gradle-wrapper.properties文件,如图：
![](/assets/img/as_gradle_fourth.jpg)

# 第5步：
Gradle官网下载地址：http://services.gradle.org/distributions
如图：
![](/assets/img/as_gradle_fifth.jpg)

# 第6步：
具体如图：
![](/assets/img/as_gradle_sixth.jpg)

# 第7步：
重新编译一下项目，结果如图：
![](/assets/img/as_gradle_finish.jpg)


参考资料：
[Android Studio Gradle优化方法(一般人我不告诉他)](http://www.cnblogs.com/leichentao0905/p/5464844.html)