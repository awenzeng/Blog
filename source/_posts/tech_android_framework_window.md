---
layout: post
title: "Android悬浮窗原理解析"
date: 10/17/2017 11:35:14 AM 
comments: true
tags: 
	- 技术 
	- Android
	- 悬浮窗
	- WindowManager
---
---
悬浮窗，在大多数应用中还是很少见的，目前我们接触到的悬浮窗，差不多都是一些系统级的应用软件，例如：360安全卫士，腾讯手机管家等；在某些服务行业如金融，餐饮等，也会在应用中添加悬浮窗，例如：美团的偷红包，博闻金融快捷联系等。但两种悬浮窗还是有区别的：

- **系统悬浮窗：所有界面都会展示，包括主屏、锁屏**
- **应用悬浮窗：只在应用Activity中展示。**
 
# 一、窗口Window
在了解悬浮窗之前，首先我们需要认识一下Android窗口Window。Android Framework将窗口分为三个类型：

1. 应用窗口：所谓应用窗口指的就是该窗口对应一个Activity，因此，要创建应用窗口就必须在Activity中完成了。
2. 子窗口：所谓子窗口指的是必须依附在某个父窗口之上，比如PopWindow，Dialog。
3. 系统窗口：所谓系统窗口指的是由系统进程创建，不依赖于任何应用或者不依附在任何父窗口之上，如：Toast,来电窗口等。

Framework定义了三种窗口类型，这三种类型定义在WindowManager的内部类LayoutParams中，WindowManager将这三种类型 进行了细化，把每一种类型都用一个int常量来表示，这些常量代表窗口所在的层，WindowManagerService在进行窗口叠加的时候，会按照常量的大小分配不同的层，常量值越大，代表位置越靠上面，所以我们可以猜想一下，应用程序Window的层值常量要小于子Window的层值常量，子Window的层值常量要小于系统Window的层值常量。Window的层级关系如下所示。

![](/assets/img/tech_android_window.png)
<!-- more -->

- 应用窗口：层级范围是1~99
- 子窗口：层级范围是1000~1999
- 系统窗口：层级范围是2000~2999

**1.各级别type值在WindowManager中的定义分别为：**

i.应用窗口（1~99）

```java
   //第一个应用窗口
   public static final int FIRST_APPLICATION_WINDOW = 1;
   //所有程序窗口的base窗口，其他应用程序窗口都显示在它上面
   public static final int TYPE_BASE_APPLICATION   = 1;
   //所有Activity的窗口，只能配合Activity在当前APP使用
   public static final int TYPE_APPLICATION        = 2;
   //目标应用窗口未启动之前的那个窗口
   public static final int TYPE_APPLICATION_STARTING = 3;
   //最后一个应用窗口
   public static final int LAST_APPLICATION_WINDOW = 99;
```
ii.子窗口（1000~1999）

```java
  //第一个子窗口
  public static final int FIRST_SUB_WINDOW        = 1000;
  // 面板窗口，显示于宿主窗口的上层,只能配合Activity在当前APP使用
  public static final int TYPE_APPLICATION_PANEL  = FIRST_SUB_WINDOW;
  // 媒体窗口（例如视频），显示于宿主窗口下层
  public static final int TYPE_APPLICATION_MEDIA  = FIRST_SUB_WINDOW+1;
  // 应用程序窗口的子面板，只能配合Activity在当前APP使用(PopupWindow默认就是这个Type)
  public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW+2;
  //对话框窗口,只能配合Activity在当前APP使用
  public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW+3;
  //
  public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW+4;
  //最后一个子窗口
 public static final int LAST_SUB_WINDOW         = 1999;
```

iii.系统窗口（2000~2999）

```java
        //系统窗口，非应用程序创建
        public static final int FIRST_SYSTEM_WINDOW     = 2000;
        //状态栏，只能有一个状态栏，位于屏幕顶端，其他窗口都位于它下方
        public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;
        //搜索栏，只能有一个搜索栏，位于屏幕上方
        public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1; 
        //
        //电话窗口，它用于电话交互（特别是呼入），置于所有应用程序之上，状态栏之下,属于悬浮窗(并且给一个Activity的话按下HOME键会出现看不到桌面上的图标异常情况)
        public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;
        //
        //系统警告提示窗口，出现在应用程序窗口之上,属于悬浮窗, 但是会被禁止
        public static final int TYPE_SYSTEM_ALERT       = FIRST_SYSTEM_WINDOW+3;
        //
        //信息窗口，用于显示Toast, 不属于悬浮窗, 但有悬浮窗的功能, 缺点是在Android2.3上无法接收点击事件
        public static final int TYPE_TOAST              = FIRST_SYSTEM_WINDOW+5;
        //
        public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;
        //锁屏窗口
        public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;
        //系统顶层窗口，显示在其他一切内容之上，此窗口不能获得输入焦点，否则影响锁屏
        public static final int TYPE_SYSTEM_OVERLAY     = FIRST_SYSTEM_WINDOW+6;
        //电话优先，当锁屏时显示，此窗口不能获得输入焦点，否则影响锁屏
        public static final int TYPE_PRIORITY_PHONE     = FIRST_SYSTEM_WINDOW+7;
        //系统对话框窗口
        public static final int TYPE_SYSTEM_DIALOG      = FIRST_SYSTEM_WINDOW+8;
        //锁屏时显示的对话框
        public static final int TYPE_KEYGUARD_DIALOG    = FIRST_SYSTEM_WINDOW+9;
        //系统内部错误提示，显示在任何窗口之上
        public static final int TYPE_SYSTEM_ERROR       = FIRST_SYSTEM_WINDOW+10;
        //内部输入法窗口，显示于普通UI之上，应用程序可重新布局以免被此窗口覆盖
        public static final int TYPE_INPUT_METHOD       = FIRST_SYSTEM_WINDOW+11;
        //内部输入法对话框，显示于当前输入法窗口之上
        public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;
        //墙纸窗口
        public static final int TYPE_WALLPAPER          = FIRST_SYSTEM_WINDOW+13;
        //状态栏的滑动面板
        public static final int TYPE_STATUS_BAR_PANEL   = FIRST_SYSTEM_WINDOW+14;
        //安全系统覆盖窗口，这些窗户必须不带输入焦点，否则会干扰键盘
        public static final int TYPE_SECURE_SYSTEM_OVERLAY = FIRST_SYSTEM_WINDOW+15;
        //最后一个系统窗口
        public static final int LAST_SYSTEM_WINDOW      = 2999;
```

**2.窗口flags显示属性在WindowManager中也有定义：**

```java
        //窗口特征标记
        public int flags;
        //当该window对用户可见的时候，允许锁屏
        public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON     = 0x00000001;
        //窗口后面的所有内容都变暗
        public static final int FLAG_DIM_BEHIND        = 0x00000002;
        //Flag：窗口后面的所有内容都变模糊
        public static final int FLAG_BLUR_BEHIND        = 0x00000004;
        //窗口不能获得焦点
        public static final int FLAG_NOT_FOCUSABLE      = 0x00000008;
        //窗口不接受触摸屏事件
        public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;
        //即使在该window在可获得焦点情况下，允许该窗口之外的点击事件传递到当前窗口后面的的窗口去
        public static final int FLAG_NOT_TOUCH_MODAL    = 0x00000020;
        //当手机处于睡眠状态时，如果屏幕被按下，那么该window将第一个收到触摸事件
        public static final int FLAG_TOUCHABLE_WHEN_WAKING = 0x00000040;
        //当该window对用户可见时，屏幕出于常亮状态
        public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;
        //：让window占满整个手机屏幕，不留任何边界
        public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;
        //允许窗口超出整个手机屏幕
        public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;
        //window全屏显示
        public static final int FLAG_FULLSCREEN      = 0x00000400;
        //恢复window非全屏显示
        public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;
        //开启窗口抖动
        public static final int FLAG_DITHER             = 0x00001000;
        //安全内容窗口，该窗口显示时不允许截屏
        public static final int FLAG_SECURE             = 0x00002000;
        //锁屏时显示该窗口
        public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;
        //系统的墙纸显示在该窗口之后
        public static final int FLAG_SHOW_WALLPAPER = 0x00100000;
        //当window被显示的时候，系统将把它当做一个用户活动事件，以点亮手机屏幕
        public static final int FLAG_TURN_SCREEN_ON = 0x00200000;
        //该窗口显示，消失键盘
        public static final int FLAG_DISMISS_KEYGUARD = 0x00400000;
        //当该window在可以接受触摸屏情况下，让因在该window之外，而发送到后面的window的触摸屏可以支持split touch
        public static final int FLAG_SPLIT_TOUCH = 0x00800000;
        //对该window进行硬件加速，该flag必须在Activity或Dialog的Content View之前进行设置
        public static final int FLAG_HARDWARE_ACCELERATED = 0x01000000;
        //让window占满整个手机屏幕，不留任何边界
        public static final int FLAG_LAYOUT_IN_OVERSCAN = 0x02000000;
        //透明状态栏
        public static final int FLAG_TRANSLUCENT_STATUS = 0x04000000;
        //透明导航栏
        public static final int FLAG_TRANSLUCENT_NAVIGATION = 0x08000000;
```

**3.添加View到Window的流程代码：**

```java

      contactView = new ContactView(mContext);
        if (layoutParams == null) {
            layoutParams = new WindowManager.LayoutParams();
            layoutParams.width = contactView.width;
            layoutParams.height = contactView.height;
            layoutParams.x += ScreenSizeUtil.getScreenWidth();
            layoutParams.y += ScreenSizeUtil.getScreenHeight() - ScreenSizeUtil.dp2px(150);
            layoutParams.gravity = Gravity.TOP | Gravity.LEFT;
            if (Build.VERSION.SDK_INT > 18 && Build.VERSION.SDK_INT < 23) {
                layoutParams.type = WindowManager.LayoutParams.TYPE_TOAST;//Type设置
            } else {
                layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION;
            }
            layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;//Flags设置,窗口属性
            layoutParams.format = PixelFormat.RGBA_8888;
        }
        windowManager.addView(contactView, layoutParams);//向窗口中添加View
```

悬浮窗添加原理：[View添加到Window源码解析](http://www.jianshu.com/p/634cd056b90c)

Window在Android系统中十分重要，其Activity，Dialog的创建都离不开Window.具体Activity，Dialog是怎么添加到Window上的，详情如：
[Activity的加载流程](http://www.jianshu.com/p/c5c3ef2b1b03);     
[Dialog加载绘制流程](http://www.jianshu.com/p/f9303d30eb2b)

# 二、两种悬浮窗
在前言中已说到，悬浮窗在app中，分为两种：系统级别和应用级别。其中系统级别可以在任何界面展示，包括主屏、锁屏（看需要），应用级别只在应用中展示。通过上面对Window的认识，我们可以知道实现方式：

- **系统级别可以通过type设置为：TYPE_TOAST、TYPE_PHONE、TYPE_SYSTEM_ALERT；**
- **应用级别可以通过type设置为：TYPE_APPLICATION、TYPE_APPLICATION_ATTACHED_DIALOG；**

**悬浮窗添加流程：**

WindowManager.addView -> ViewRootImpl.setView -> WindowSession.addToDisplay(AIDL进行IPC) -> WindowManagerService.addWindow() -> ViewRootImpl.setView

**1.系统悬浮窗**

对于系统级别的悬浮窗来说，不同的设置，不同的Android版本，需要权限和交互都不同。通过阅读源码android4.4知：

android 7.1.1的addWindow方法为：

```java

   public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, InputChannel outInputChannel) {
        int[] appOp = new int[1];
        int res = mPolicy.checkAddPermission(attrs, appOp);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res;
        }
        ......
        final int type = attrs.type;
        ......
        } else if (type == TYPE_TOAST) {//android 7.1.1 添加代码（其他版本无）
                // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                addToastWindowRequiresToken = doesAddToastWindowRequireToken(attrs.packageName,
                        callingUid, attachedWindow);
                if (addToastWindowRequiresToken && token.windowType != TYPE_TOAST) {
                    Slog.w(TAG_WM, "Attempted to add a toast window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
        }....
        synchronized(mWindowMap) {
            ......

            mPolicy.adjustWindowParamsLw(win.mAttrs);
            ......
        }
        ......

        return res;
    }
```

权限检查checkAddPermission()方法：

```java

    //权限检查
    @Override
    public int checkAddPermission(WindowManager.LayoutParams attrs, int[] outAppOp) {
        int type = attrs.type;

        outAppOp[0] = AppOpsManager.OP_NONE;

        if (type < WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW
                || type > WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
            return WindowManagerGlobal.ADD_OKAY;
        }
        String permission = null;
        switch (type) {
            case TYPE_TOAST:
                // XXX right now the app process has complete control over
                // this...  should introduce a token to let the system
                // monitor/control what they are doing.
                break;
            case TYPE_DREAM:
            case TYPE_INPUT_METHOD:
            case TYPE_WALLPAPER:
            case TYPE_PRIVATE_PRESENTATION:
                // The window manager will check these.
                break;
            case TYPE_PHONE:
            case TYPE_PRIORITY_PHONE:
            case TYPE_SYSTEM_ALERT:
            case TYPE_SYSTEM_ERROR:
            case TYPE_SYSTEM_OVERLAY:
                permission = android.Manifest.permission.SYSTEM_ALERT_WINDOW;
                outAppOp[0] = AppOpsManager.OP_SYSTEM_ALERT_WINDOW;
                break;
            default:
                permission = android.Manifest.permission.INTERNAL_SYSTEM_WINDOW;
        }
        if (permission != null) {
            if (mContext.checkCallingOrSelfPermission(permission)
                    != PackageManager.PERMISSION_GRANTED) {
                return WindowManagerGlobal.ADD_PERMISSION_DENIED;
            }
        }
        return WindowManagerGlobal.ADD_OKAY;
    }
```
window的flags属性添加adjustWindowParamsLw()方法

```java
//window的flags属性添加
//Android 2.0 - 2.3.7 PhoneWindowManager
    public void adjustWindowParamsLw(WindowManager.LayoutParams attrs) {
        switch (attrs.type) {
            case TYPE_SYSTEM_OVERLAY:
            case TYPE_SECURE_SYSTEM_OVERLAY:
            case TYPE_TOAST:
                // These types of windows can't receive input events.
                attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
                break;
        }
    }

    //Android 4.0.1 - 4.3.1 PhoneWindowManager
    public void adjustWindowParamsLw(WindowManager.LayoutParams attrs) {
        switch (attrs.type) {
            case TYPE_SYSTEM_OVERLAY:
            case TYPE_SECURE_SYSTEM_OVERLAY:
            case TYPE_TOAST:
                // These types of windows can't receive input events.
                attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
                attrs.flags &= ~WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
                break;
        }
    }


   //Android 4.4 PhoneWindowManager
    @Override
    public void adjustWindowParamsLw(WindowManager.LayoutParams attrs) {
        switch (attrs.type) {
            case TYPE_SYSTEM_OVERLAY:
            case TYPE_SECURE_SYSTEM_OVERLAY:
                // These types of windows can't receive input events.
                attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
                attrs.flags &= ~WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
                break;
        }
    }
```

由上可知：

- **Type为TYPE_TOAST：版本低于android4.4的，不能接受触摸事件，无法操作；版本高于Android 7.1.1的无法添加悬浮窗**
- **Type为TYPE_PHONE：所有android版本都需要权限，版本低于android6.0的manifest中添加权限android.Manifest.permission.SYSTEM_ALERT_WINDOW即可，高于andorid6.0的需要判断权限且手动添加。**
- **Type为TYPE_SYSTEM_ALERT：同TYPE_PHONE**

所以系统级别的悬浮窗，android不同版本需要特别处理。

**2.应用悬浮窗**

对于应用悬浮窗来说，android版本对其影响不大。

- **Type为TYPE_APPLICATION：只要Activity建立了，就可以添加。**
- **Type为TYPE_APPLICATION_ATTACHED_DIALOG：需要在Activity获取焦点，并且用户可操作时才可添加。**



# 三、悬浮窗的实现
  
悬浮窗添加比较简单,主要是由WindowManager接口的实现类WindowManagerImpl进行操作，WindowManager接口又继承至ViewManager,其中主要方法为：

```java
    public void addView(View view, ViewGroup.LayoutParams params);//添加View到Window
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);//更新View在Window中的位置
    public void removeView(View view);//删除View
```
主要实现代码：

```java

public class ContactWindowUtil {
    private ContactView contactView;
    private View dialogView;
    private WindowManager windowManager;
    private WindowManager.LayoutParams layoutParams;
    private WindowManager.LayoutParams dialogParams;
    private Context mContext;
    private ContactWindowListener mListener;
    private ValueAnimator valueAnimator;
    private int direction;
    private final int LEFT = 0;
    private final int RIGHT = 1;

    public interface ContactWindowListener {
        void onDataCallBack(String str);
    }

    public void setDialogListener(ContactWindowListener listener) {
        mListener = listener;
    }

    //私有化构造函数
    public ContactWindowUtil(Context context) {
        mContext = context;
        windowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
//      windowManager = (WindowManager) FloatWindowApp.getAppContext().getSystemService(Context.WINDOW_SERVICE);
    }

    public void showContactView() {
        hideContactView();
        contactView = new ContactView(mContext);
        if (layoutParams == null) {
            layoutParams = new WindowManager.LayoutParams();
            layoutParams.width = contactView.width;
            layoutParams.height = contactView.height;
            layoutParams.x += ScreenSizeUtil.getScreenWidth();
            layoutParams.y += ScreenSizeUtil.getScreenHeight() - ScreenSizeUtil.dp2px(150);
            layoutParams.gravity = Gravity.TOP | Gravity.LEFT;
            if (Build.VERSION.SDK_INT > 18 && Build.VERSION.SDK_INT < 23) {
                layoutParams.type = WindowManager.LayoutParams.TYPE_TOAST;
            } else {
                layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION;
            }
            layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;
            layoutParams.format = PixelFormat.RGBA_8888;
        }

        contactView.setOnTouchListener(touchListener);
        contactView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                showContactDialog();
            }
        });
        windowManager.addView(contactView, layoutParams);
    }

    /**
     * 显示联系弹框
     */
    private void showContactDialog() {
        hideDialogView();
        if (dialogParams == null) {
            dialogParams = new WindowManager.LayoutParams();
            dialogParams.width = WindowManager.LayoutParams.MATCH_PARENT;
            dialogParams.height = WindowManager.LayoutParams.MATCH_PARENT;
            dialogParams.gravity = Gravity.CENTER;
            if (Build.VERSION.SDK_INT > 18 && Build.VERSION.SDK_INT < 25){
                dialogParams.type = WindowManager.LayoutParams.TYPE_TOAST;
            } else {
                dialogParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG;
            }
            dialogParams.flags = WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                    | WindowManager.LayoutParams.FLAG_FULLSCREEN;
            dialogParams.format = PixelFormat.RGBA_8888;
        }
        LayoutInflater layoutInflater = (LayoutInflater) mContext.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        dialogView = layoutInflater.inflate(R.layout.window_contact, null);
        TextView okTv = dialogView.findViewById(R.id.mOkTv);
        TextView cancleTv = dialogView.findViewById(R.id.mCancleTv);
        final TextView contentTv = dialogView.findViewById(R.id.mContentTv);
        contentTv.setText(String.format("您确认拨打%s客服电话吗", "4008-111-222"));
        cancleTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                hideDialogView();
            }
        });
        okTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                hideDialogView();
                mListener.onDataCallBack("");
            }
        });
        dialogView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                hideDialogView();
            }
        });
        windowManager.addView(dialogView, dialogParams);
    }


    public void hideAllView() {
        hideContactView();
        hideDialogView();
    }

    public void hideContactView() {
        if (contactView != null) {
            windowManager.removeView(contactView);
            contactView = null;
            stopAnim();
        }
    }

    public void hideDialogView() {
        if (dialogView != null) {
            windowManager.removeView(dialogView);
            dialogView = null;
        }
    }

    View.OnTouchListener touchListener = new View.OnTouchListener() {
        float startX;
        float startY;
        float moveX;
        float moveY;
        @Override
        public boolean onTouch(View v, MotionEvent event) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    startX = event.getRawX();
                    startY = event.getRawY();

                    moveX = event.getRawX();
                    moveY = event.getRawY();
                    break;
                case MotionEvent.ACTION_MOVE:
                    float x = event.getRawX() - moveX;
                    float y = event.getRawY() - moveY;
                    //计算偏移量，刷新视图
                    layoutParams.x += x;
                    layoutParams.y += y;
                    windowManager.updateViewLayout(contactView, layoutParams);
                    moveX = event.getRawX();
                    moveY = event.getRawY();
                    break;
                case MotionEvent.ACTION_UP:
                    //判断松手时View的横坐标是靠近屏幕哪一侧，将View移动到依靠屏幕
                    float endX = event.getRawX();
                    float endY = event.getRawY();
                    if (endX < ScreenSizeUtil.getScreenWidth() / 2) {
                        direction = LEFT;
                        endX = 0;
                    } else {
                        direction = RIGHT;
                        endX = ScreenSizeUtil.getScreenWidth() - contactView.width;
                    }
                    if(moveX != startX){
                        starAnim((int) moveX, (int) endX,direction);
                    }
                    //如果初始落点与松手落点的坐标差值超过5个像素，则拦截该点击事件
                    //否则继续传递，将事件交给OnClickListener函数处理
                    if (Math.abs(startX - moveX) > 5) {
                        return true;
                    }
                    break;
            }
            return false;
        }
    };


    private void starAnim(int startX, int endX,final int direction) {
        if (valueAnimator != null) {
            valueAnimator.cancel();
            valueAnimator = null;
        }
        valueAnimator = ValueAnimator.ofInt(startX, endX);
        valueAnimator.setDuration(500);
        valueAnimator.setRepeatCount(0);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                if(direction == LEFT){
                    layoutParams.x = (int) animation.getAnimatedValue()-contactView.width/2;
                }else{
                    layoutParams.x = (int) animation.getAnimatedValue();
                }
                if (contactView != null) {
                    windowManager.updateViewLayout(contactView, layoutParams);
                }
            }
        });
        valueAnimator.setInterpolator(new LinearInterpolator());
        valueAnimator.start();
    }

    private void stopAnim() {
        if (valueAnimator != null) {
            valueAnimator.cancel();
            valueAnimator = null;
        }
    }
}
```

**悬浮窗源码:**[https://github.com/awenzeng/FloatWindowDemo](https://github.com/awenzeng/FloatWindowDemo)

# 四、注意

1.应用悬浮窗WindowManager的获取环境必须是Activity环境，系统悬浮窗可以Activity环境，也可以是全局的环境。如：

```java
        windowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
//      windowManager = (WindowManager) FloatWindowApp.getAppContext().getSystemService(Context.WINDOW_SERVICE);
```

2.应用级别悬浮窗也可以通过系统级别的方式实现，主要控制一下显示与隐藏就好。**但如果只在应用中展示悬浮窗，建议使用应用级别，那样会省去许多不必要的麻烦。**

3.系统级别Type为TYPE_PHONE、TYPE_SYSTEM_ALERT是权限判断及设置代码：

```java
 /** 
     * 请求用户给予悬浮窗的权限 
     */  
    public void askForPermission() {  
        if (!Settings.canDrawOverlays(this)) {  
            Toast.makeText(TestFloatWinActivity.this, "当前无权限，请授权！", Toast.LENGTH_SHORT).show();  
            Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,  
                    Uri.parse("package:" + getPackageName()));  
            startActivityForResult(intent, OVERLAY_PERMISSION_REQ_CODE);  
        } else {  
            startService(floatWinIntent);  
        }  
    }  
```

# 五、参考文献

[Android源码解析Window系列第（一）篇---Window的基本认识和Activity的加载流程](http://www.jianshu.com/p/c5c3ef2b1b03)

[Android 悬浮窗的小结](https://www.liaohuqiu.net/cn/posts/android-windows-manager/)

[Android悬浮窗TYPE_TOAST小结: 源码分析](http://www.jianshu.com/p/634cd056b90c)

[Android无需权限显示悬浮窗, 兼谈逆向分析app](http://www.jianshu.com/p/167fd5f47d5c)

[Android 悬浮窗权限各机型各系统适配大全](http://blog.csdn.net/self_study/article/details/52859790)