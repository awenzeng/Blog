---
layout: post
title: "Android跨进程通信方式(IPC)解析"
date: 12/15/2017 4:05:27 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
在Android的圈子里，一直有一个声音，那就是：要学会看Android源码。在前期，android开发者比较缺乏阶段，似乎大家也没那么重视，但随着时间的发展，Android开发者早已供大于求，需要具备的技能也不在仅限于应用开发，还需要对Android运行机制原理有一个深度的了解，从而更好的为项目服务。所以，阅读Android源码，也就成为Android开发人员必须要做的事。

在阅读Android源码时，我们发现，Android系统中跨进程通信主要就是Binder。关于Binder跨进程通信原理，我已写过一篇文章[Binder通信机制原理解析](http://blog.csdn.net/awenyini/article/details/78806893),想了解的同学可以看一下。其中也有说到为什么Andorid系统跨进程通信要使用Binder。在Android系统中多数服务如ActivitManagerService,WindowManagerService,PackageManagerService等都是通过Binder进行通信的，在阅读源码时，我们会时时与其打交道，所以本篇博文主要是想梳理一下Andorid源码中常见的Binder跨进程通信的方式，以便自己在读源码时，可以更好的理解。

在[Binder通信机制原理解析](http://blog.csdn.net/awenyini/article/details/78806893)博文中，我们说到Binder跨进程通信方式有两种，分别为AIDL方式、注册服务方式。AIDL方式在开发中是我们经常使用的方式，这里将会采用对比的方式来解析系统服务的Binder跨进程通信。

# 一、常用AIDL方式
**1.aidl接口创建**

以aidl为后缀创建一个接口类。如
```java

interface IMainService {
   void start(String temp);
}

```
<!-- more -->
项目编译时，系统会自动生成相对应的java文件，如
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
抽象类Stub相当于服务端，代理类Proxy相当于客户端。抽象类Stub继承于Binder，代理类Proxy依赖于IBinder接口。

**2.AIDL的使用**

AIDL的使用是以Service为载体，进而实现跨进程通信。我们知道Android的四大组件，在Androidmanifest中注册的时候可以通过android：process来指定组件所在的进程，当组件间不在同进程时，就需要跨进程通信了。AIDLService代码如下：

```java
public class AIDLService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        LogUtil.androidLog("Received start command.");
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        LogUtil.androidLog("Received binding.");
        return mBinder;
    }

    private final IMainService.Stub mBinder = new IMainService.Stub() {
        @Override
        public void start(String temp) throws RemoteException {
            LogUtil.androidLog("AIDLService服务端打印日志："+temp);
        }
    };
}
```
其中mBinder通过匿名内部类的形式初始化了Stub抽象类，进而AIDLService也就变成了Server端。当AIDLService与项目不在同一进程时，其他组件想与其通信，就必须要跨进程通信了。我们来看Activity与AIDLService通信，如
```java
public class AIDLServiceConnection implements ServiceConnection {

    private IMainService mService;

    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
             mService = IMainService.Stub.asInterface(service);//核心代码
             try{
                 mService.start("Android IPC机制，Bindler跨进程通信~~~~~~~");
             }catch (RemoteException e){
                 e.printStackTrace();
             }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
        LogUtil.androidLog("AIDL服务断开连接");
    }
}

public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //AIDL跨进程通信
        Intent intent = new Intent(this, AIDLService.class);
        AIDLServiceConnection connection = new AIDLServiceConnection();
        bindService(intent,connection,BIND_AUTO_CREATE);
    }
```
Activity通过bindService的方式，建立与AIDLService服务的联系。这中间主要是通过ServiceConnection这个接口，我们来看一下注释中的核心代码，IMainService.Stub.asInterface(service)，这里我们再来看一下，aidl接口生成的java类的asInterface方法。

```java
    public static com.awen.codebase.IMainService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);//查询是否本地进程
            if (((iin != null) && (iin instanceof com.awen.codebase.IMainService))) {
                return ((com.awen.codebase.IMainService) iin);
            }
            return new com.awen.codebase.IMainService.Stub.Proxy(obj);
        }
```

这里主要判断IBinder是否是跨进程，如果不是就返回本身，如果是则返回代理类Proxy，从而进行跨进程通信。具体Binder是怎么跨进程通信的，可以参考我的博文[Binder通信机制原理解析](http://blog.csdn.net/awenyini/article/details/78806893)。最后运行程序，结果如下

![](/assets/img/tech_android_ipc_aidl.png)

这里跨进程通信AIDL方式就讲解完了，AIDL方式源码地址：[https://github.com/awenzeng/AndroidCodeLibrary](https://github.com/awenzeng/AndroidCodeLibrary),欢迎star，fork。

# 二、注册服务方式
Android的各种系统服务在Android系统启动的时候就也会启动和注册，具体启动流程，可以参考[Android应用程序入口源码解析](http://blog.csdn.net/awenyini/article/details/78619361)，这篇博文中有介绍，想了解的同学可以看一下。系统服务启动和注册流程具体如下：

![](/assets/img/tech_android_ipc_regist.png)


通过此图，我想大家对系统服务的启动流程已有一个大概认识。各种系统服务启动后，都会在ServiceManager进行注册备注，以方便应用进程调用，这ServiceManager相当于各种系统服务的大管家。另外，Andorid的各种系统服务都运行在system_server进程中，应用进程想要获取系统服务，就需要与system_server进程通信，Binder在其中就起着桥梁的作用。

Android系统中服务大约有八十多个，我们也没必要一一分析，遇到相关服务时，再进一步分析就好。本篇博文主要是针对跨进程通信(IPC)，所以也主要分析Andorid源码中常见的通过Binder通信的C/S端，来加深对Android源码的理解。常见的Android源码Binder通信C/S端有：

* ActivityManagerService(AMS),ActivityManagerNative(AMN),ActivityManagerProxy(AMP)
* ApplicationThread(AT),ApplicationThreadNative(ATN)，ApplicationThreadProxy(ATP)

**1.AMS跨进程通信**
首先我们来看一下ActivityManagerNative源码，如下
```java
/** {@hide} */
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
    ....
    public ActivityManagerNative() {
        attachInterface(this, descriptor);
    }

    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            String profileFile = data.readString();
            ParcelFileDescriptor profileFd = data.readInt() != 0
                    ? data.readFileDescriptor() : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            int result = startActivity(app, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags,
                    profileFile, profileFd, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }
        }
        }
        return super.onTransact(code, data, reply, flags);
    }

    public IBinder asBinder() {
        return this;
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");//1.核心代码
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);//2.核心代码
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
}

class ActivityManagerProxy implements IActivityManager
{
    private IBinder mRemote;

    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }

    public IBinder asBinder()
    {
        return mRemote;
    }

    public int startActivity(IApplicationThread caller, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        ......
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
   ........
 }
}
```
从此源码来看，这与我们AIDL方式的接口生成的java代码很像，抽象类ActivityManagerNative就相当于抽象类Stub,代理类ActivityManagerProxy就相当于代理类Proxy，所以抽象类AMN就相当于Server端，代理类ActivityManagerProxy就相当于Client端。我们再来看一下ActivityManagerService类
```java

public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
......省略代码
}
```
可以看出ActivityManagerService继承至ActivityManagerNative，所以ActivityManagerService也是Server端，类似AIDL方式的AIDLService。我们再来看看ActivityManagerService的获取，在ActivityManagerNative源码中

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
  .......
 static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);//核心代码
    }

 static public IActivityManager getDefault() {
        return gDefault.get();
    }

 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");//1.核心代码
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);//2.核心代码
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
.........
}
```
主要通过一个静态方法ActivityManagerNative.getDefault()获取，而gDefault就是一个单例，从注释1：ServiceManager.getService("activity")，我们知主要就是从大管家ServiceManager中获取ActivityManagerService服务，但由于AMS是在不同的进程，所以通过asInterface()获取代理类ActivityManagerProxy来进行Binder跨进程通信。通过调用代理类AMP中的方法，然后跨进程通信，从而调用AMS中的相关方法。

到这里ActivitManagerService的Binder跨进程通信方式就简单介绍完了。

对于AMS，我不得不提一下，因为Android中四大组件Activity、Service、BroadcastReceiver和ContentProvider启动和使用都与其有关，可以说Andorid系统中比较重要的一个类。

**2.ApplicationThread跨进程通信**

同样的，首先我们先来看一下ApplicationThreadNative此类，源码如下
```java
public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
    /**
     * Cast a Binder object into an application thread interface, generating
     * a proxy if needed.
     */
    static public IApplicationThread asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IApplicationThread in =
            (IApplicationThread)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ApplicationThreadProxy(obj);
    }
    
    public ApplicationThreadNative() {
        attachInterface(this, descriptor);
    }
    
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean finished = data.readInt() != 0;
            boolean userLeaving = data.readInt() != 0;
            int configChanges = data.readInt();
            schedulePauseActivity(b, finished, userLeaving, configChanges);
            return true;
        }
        .........
        }

        return super.onTransact(code, data, reply, flags);
    }

    public IBinder asBinder()
    {
        return this;
    }
}

class ApplicationThreadProxy implements IApplicationThread {

    private final IBinder mRemote;
    
    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }
    
    public final IBinder asBinder() {
        return mRemote;
    }
    
    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(finished ? 1 : 0);
        data.writeInt(userLeaving ? 1 :0);
        data.writeInt(configChanges);
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
    ......
}
```
Binder跨进程通信定义方式，差不多都一样，ApplicationThread跨进程通信也和AIDL方式类似。其中抽象类ApplicationThreadNative为Server端，代理类ApplicationThreadProxy为Client端。我们再来看ApplicationThread类，通过阅读源码，我们知ApplicationThread是ActivityThread中内部类，这里我们也来看看ApplicationThread的源码
```java
public final class ActivityThread {

final ApplicationThread mAppThread = new ApplicationThread();
......
private class ApplicationThread extends ApplicationThreadNative {
......
}
}
```
可以发现，ApplicationThread继承至ApplicationThreadNative，所以ApplicationThread也是AT跨进程通信的Server端，这里与AIDL的调用方式类似，通过一定的转换获取到代理类ApplicationThreadProxy，从而进行跨进程通信。

ApplicationThread跨进程通信方式，在Android系统中还是比较重要的，它主要与AMS跨进程通信一起使用，当ActivityManagerService执行完响应的操作后，再通过跨进程通信方式与应用进程通信(ApplicationThread是在应用进程中)，从而对Andorid的四大组件进行调度，Activity，Service等的启动及生命周期，也就是通过AMS跨进程通信和AT跨进程通信实现的。这点在阅读Activity及Service启动源码的时候，会接触的比较频繁。

到这里，我想说的就说完了。

**注：源码采用android-4.1.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 三、相关文档
[Binder通信机制原理解析](http://blog.csdn.net/awenyini/article/details/78806893)

[Android应用程序入口源码解析](http://blog.csdn.net/awenyini/article/details/78619361)


