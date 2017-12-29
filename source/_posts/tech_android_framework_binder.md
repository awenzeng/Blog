---
layout: post
title: "Binder通信机制原理解析"
date: 12/14/2017 7:24:20 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Binder通信机制
	- Android框架源码解析
---
---
Binder是什么？Binder有啥用？作为一个应用开发者，如果我们开发的应用不涉及跨进程通信(IPC)，我想我们也不会去接触Binder。但不知你有没有发现，近来的Andorid面试，都会问及Android跨进程通信方面的知识，这又是为什么呢？如果你喜欢看Android源码，你就会发现Binder无处不在，Android系统中很多服务都是通过Binder来进行跨进程通信，了解Binder,弄懂Binder原理，这就变得非常重要了。

Android的内核是Linux内核,但为什么没有采用Linux常用的跨进程通信方式呢？而是采用Binder呢？这里在知乎有一篇非常详细的帖子:为什么 Android 要采用 Binder 作为 IPC 机制？大家可以看一下，我相信此篇帖子下肚，你就可以了解当年Android之父Andy Rubin为什么会选择Binder作为android系统跨进程通信方式了。

# 一、什么Binder?

Binder是Android系统中非常重要的一种跨进程通信方式(IPC)。

Binder是一种基于C/S的架构，主要包含四个部分：服务端（Server）,客户端（Client）,Binder驱动,ServiceManager。

Android四大组件Activity、Service、BroadcastReceiver和ContentProvider的启动原理也都与Binder IPC机制有关；Android源码中ActivityManagerService、PackageManagerService、WindowManagerService、PowerManagerService等服务的调用也都与Binder IPC机制有关。

# 二、Binder跨进程通信实现原理
**1.应用进程空间分配**
<!-- more -->
![](/assets/img/tech_android_binder_ipc.png)

每个Android应用程序的进程，只能运行在自己进程所拥有的虚拟地址空间上。对应一个4GB的虚拟地址空间，其中3GB是用户空间，1GB是内核空间，当然内核空间的大小是可以通过参数配置调整的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。Client进程向Server进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client端与Server端进程往往采用ioctl等方法跟内核空间的驱动进行交互。

**2.Binder进程通信原理**

![](/assets/img/tech_android_binder_struct.png)

* Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中
* Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server
* Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信
* Client和Server之间的进程间通信通过Binder驱动程序间接实现
* Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力

**3.Binder跨进程通信数据流动图：**

![](/assets/img/tech_android_binder_data.png)

Client端发起通信请求–>服务代理Proxy通过Parcel打包数据–>内核空间Binder驱动实现共享数据–>Server端解包Parcel获取数据。这样就实现了Client端到Server端的跨进程通信。

具体详细实现，可以参考这篇文章[彻底理解Android Binder通信架构](http://www.droidsec.cn/%E5%BD%BB%E5%BA%95%E7%90%86%E8%A7%A3android-binder%E9%80%9A%E4%BF%A1%E6%9E%B6%E6%9E%84/)

# 三、Binder两种跨进程通信方式
**1.注册服务的方式**

Android系统多数服务，都需要先注册，并通过ServiceManager守护进程来管理，需要用时，直接从ServiceManager中获取相关服务。例如ActivityManagerService、PackageManagerService、WindowManagerService、PowerManagerService等系统服务都是通过此种方式。这些服务什么时候注册呢？在Android应用程序入口源码解析这篇文章中有说，当Zygote进程启动后，启动SystemServer进程时，就会注册各种服务。具体系统服务是怎么通信的，下篇博文会介绍。

**2.AIDL方式**

这种方式也是开发中我们经常使用的方式。首先定义一个后缀为aidl的接口，如
```java
interface IMainService {
   void start(String temp);
}
```
项目编译成功后，会自动生成相关的java文件如：
```java
public interface IMainService extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.awen.codebase.IMainService {
        private static final java.lang.String DESCRIPTOR = "com.awen.codebase.IMainService";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.awen.codebase.IMainService interface,
         * generating a proxy if needed.
         */
        public static com.awen.codebase.IMainService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.awen.codebase.IMainService))) {
                return ((com.awen.codebase.IMainService) iin);
            }
            return new com.awen.codebase.IMainService.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_start: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    this.start(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.awen.codebase.IMainService {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void start(java.lang.String temp) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(temp);
                    mRemote.transact(Stub.TRANSACTION_start, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_start = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }
    public void start(java.lang.String temp) throws android.os.RemoteException;
}
```

其中Stub就相当于Server端，代理Proxy就相当于Client端。在AIDL开发中，一般都是结合Service一起使用的，通过bindService把客户端和服务端链接起来，从而就可以进行通信。如果Service在同一个进程，就在进程中通信，如果Service在另外一个进程，就属于跨进程通信。


# 四、总结
Binder跨进程通信，主要就是利用进程间内核空间共享数据来实现的。Binder驱动主要就是在内核空间，通过Binder驱动实现进程间数据共享。

# 五、参考文档
[彻底理解Android Binder通信架构](http://www.droidsec.cn/%E5%BD%BB%E5%BA%95%E7%90%86%E8%A7%A3android-binder%E9%80%9A%E4%BF%A1%E6%9E%B6%E6%9E%84/)

[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)

[简单理解Binder机制的原理](http://www.jianshu.com/p/4920c7781afe?from=jiantop.com)

[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)



