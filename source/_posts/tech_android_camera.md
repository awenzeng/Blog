---
layout: post
title: "Android Camera相机开发详解"
date: 8/21/2017 7:34:21 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Android相机
	- Camera
---
---
在应用软件开发中，图片数据，对于一个公司来说是十分重要的，例如：上传图片资料，修改用户头像等，而这其中就离不开相机和相册的使用。对于ios平台来说，直接调用系统相机或相册，就可搞定一切。然而对于Android平台来说，直接调用系统相机或相册，在适配和体验上问题比较多，具体原因，相比大家也知道，安卓品牌太多太杂，性能不一。鉴于此，在开发的过程中，遇到类似问题，建议自己实现相机或相册功能，以保证体验完整。本篇博文将会重点介绍Camera相机的实现。

----

首先，推荐两个github项目，可以直接使用的相机和相册；另外，也推荐一个联系人选择器：

相机：[CameraDemo(自定义相机)](https://github.com/awenzeng/CameraDemo)

相册：[ImageSelector(仿微信图片选择相册)](https://github.com/ioneday/ImageSelector)

联系人：[ContactSelector(联系人选择器)](https://github.com/awenzeng/ContactSelector)


# 一、打开Camer
```java
        try {
                mCamera = Camera.open();//开启相机
            } catch (RuntimeException e) {
                e.printStackTrace();
                LogUtil.d(TAG, "摄像头异常，请检查摄像头权限是否应许");
                ToastUtil.getInstance().toast("摄像头异常，请检查摄像头权限是否应许");
                return;
            }
```

# 二、设置Camera参数

默认尺寸可以自由设置，这里取手机的分辨率为默认尺寸。

**1.根据指定分辨率查找相机最佳适配分辨率并设置**

```java
    private void setCameraParams(int width, int height) {
        LogUtil.i(TAG, "setCameraParams  width=" + width + "  height=" + height);

        Camera.Parameters parameters = mCamera.getParameters();

        List<Camera.Size> pictureSizeList = parameters.getSupportedPictureSizes();
        sort(pictureSizeList);//排序
        for (Camera.Size size : pictureSizeList) {
            LogUtil.i(TAG, "摄像头支持的分辨率：" + " size.width=" + size.width + "  size.height=" + size.height);
        }
        Camera.Size picSize = getBestSupportedSize(pictureSizeList, ((float) height / width));//从列表中选取合适的分辨率
        if (null == picSize) {
            picSize = parameters.getPictureSize();
        }

        LogUtil.e(TAG, "我们选择的摄像头分辨率：" + "picSize.width=" + picSize.width + "  picSize.height=" + picSize.height);
        // 根据选出的PictureSize重新设置SurfaceView大小
        parameters.setPictureSize(picSize.width, picSize.height);
        ....
```
<!-- more -->
**2.根据指定分辨率查找相机最佳预览分辨率并设置**

```java
    private void setCameraParams(int width, int height) {
         LogUtil.i(TAG, "setCameraParams  width=" + width + "  height=" + height);

        Camera.Parameters parameters = mCamera.getParameters();

        /*************************** 获取摄像头支持的PreviewSize列表********************/
        List<Camera.Size> previewSizeList = parameters.getSupportedPreviewSizes();
        sort(previewSizeList);
        for (Camera.Size size : previewSizeList) {
            LogUtil.i(TAG, "摄像支持可预览的分辨率：" + " size.width=" + size.width + "  size.height=" + size.height);
        }
        Camera.Size preSize = getBestSupportedSize(previewSizeList, ((float) height) / width);
        if (null != preSize) {
            LogUtil.e(TAG, "我们选择的预览分辨率：" + "preSize.width=" + preSize.width + "  preSize.height=" + preSize.height);
            parameters.setPreviewSize(preSize.width, preSize.height);
        }
       ......
```

**3.最佳分辨率适配算法(先排序)**

```java
    /**
     * 如包含默认尺寸，则选默认尺寸，如没有，则选最大的尺寸
     * 规则：在相同比例下，1.优先寻找长宽分辨率相同的->2.找长宽有一个相同的分辨率->3.找最大的分辨率
     *
     * @param sizes 尺寸集合
     * @return 返回合适的尺寸
     */
    private Camera.Size getBestSupportedSize(List<Camera.Size> sizes, float screenRatio) {
        Camera.Size largestSize = null;
        int largestArea = 0;
        for (Camera.Size size : sizes) {
            if ((float) size.height / (float) size.width == screenRatio) {
                if (size.width == DEFAULT_PHOTO_WIDTH && size.height == DEFAULT_PHOTO_HEIGHT) {
                    // 包含特定的尺寸，直接取该尺寸
                    largestSize = size;
                    break;
                } else if (size.height == DEFAULT_PHOTO_HEIGHT || size.width == DEFAULT_PHOTO_WIDTH) {
                    largestSize = size;
                    break;
                }
                int area = size.height + size.width;
                if (area > largestArea) {//找出最大的合适尺寸
                    largestArea = area;
                    largestSize = size;
                }
            } else if (size.height == DEFAULT_PHOTO_HEIGHT || size.width == DEFAULT_PHOTO_WIDTH) {
                largestSize = size;
                break;
            }
        }
        if (largestSize == null) {
            largestSize = sizes.get(sizes.size() - 1);
        }
        return largestSize;
    }
```

**4.对焦模式选择**
PixelFormat中有多种模式，源码有解。

```java
   //对焦模式的选择 
        if(cameraId == Camera.CameraInfo.CAMERA_FACING_BACK){//前置摄像头
            parameters.setFocusMode(Camera.Parameters.FOCUS_MODE_AUTO);//手动区域自动对焦
        }
```

**5.图片质量**
PixelFormat中有多种模式，源码有解。

```java
        //图片质量
        parameters.setJpegQuality(100); // 设置照片质量
        parameters.setPreviewFormat(PixelFormat.YCbCr_420_SP); // 预览格式
        parameters.setPictureFormat(PixelFormat.JPEG); // 相片格式为JPEG，默认为NV21
```

**6.闪关灯及横竖屏镜头调整**

```java
        // 关闪光灯
        parameters.setFlashMode(Camera.Parameters.FLASH_MODE_OFF);

        // 横竖屏镜头自动调整
        if (mContext.getResources().getConfiguration().orientation != Configuration.ORIENTATION_LANDSCAPE) {
            mCamera.setDisplayOrientation(90);
        } else {
            mCamera.setDisplayOrientation(0);
        }
```

**7.相机异常监听**

```java
        //相机异常监听
        mCamera.setErrorCallback(new Camera.ErrorCallback() {

            @Override
            public void onError(int error, Camera camera) {
                String error_str;
                switch (error) {
                    case Camera.CAMERA_ERROR_SERVER_DIED: // 摄像头已损坏
                        error_str = "摄像头已损坏";
                        break;

                    case Camera.CAMERA_ERROR_UNKNOWN:
                        error_str = "摄像头异常，请检查摄像头权限是否应许";
                        break;

                    default:
                        error_str = "摄像头异常，请检查摄像头权限是否应许";
                        break;
                }
                ToastUtil.getInstance().toast(error_str);
                Log.i(TAG, error_str);
            }
        });
```

完整参数设置代码：

```java
  /**
     * 设置分辨率等参数
     *
     * @param width  宽
     * @param height 高
     */
    private void setCameraParams(int width, int height) {
        LogUtil.i(TAG, "setCameraParams  width=" + width + "  height=" + height);

        Camera.Parameters parameters = mCamera.getParameters();


        /*************************** 获取摄像头支持的PictureSize列表********************/
        List<Camera.Size> pictureSizeList = parameters.getSupportedPictureSizes();
        sort(pictureSizeList);//排序
        for (Camera.Size size : pictureSizeList) {
            LogUtil.i(TAG, "摄像头支持的分辨率：" + " size.width=" + size.width + "  size.height=" + size.height);
        }
        Camera.Size picSize = getBestSupportedSize(pictureSizeList, ((float) height / width));//从列表中选取合适的分辨率
        if (null == picSize) {
            picSize = parameters.getPictureSize();
        }

        LogUtil.e(TAG, "我们选择的摄像头分辨率：" + "picSize.width=" + picSize.width + "  picSize.height=" + picSize.height);
        // 根据选出的PictureSize重新设置SurfaceView大小
        parameters.setPictureSize(picSize.width, picSize.height);


        /*************************** 获取摄像头支持的PreviewSize列表********************/
        List<Camera.Size> previewSizeList = parameters.getSupportedPreviewSizes();
        sort(previewSizeList);
        for (Camera.Size size : previewSizeList) {
            LogUtil.i(TAG, "摄像支持可预览的分辨率：" + " size.width=" + size.width + "  size.height=" + size.height);
        }
        Camera.Size preSize = getBestSupportedSize(previewSizeList, ((float) height) / width);
        if (null != preSize) {
            LogUtil.e(TAG, "我们选择的预览分辨率：" + "preSize.width=" + preSize.width + "  preSize.height=" + preSize.height);
            parameters.setPreviewSize(preSize.width, preSize.height);
        }

        /*************************** 对焦模式的选择 ********************/
        if(cameraId == Camera.CameraInfo.CAMERA_FACING_BACK){
            parameters.setFocusMode(Camera.Parameters.FOCUS_MODE_AUTO);//手动区域自动对焦
        }
        //图片质量
        parameters.setJpegQuality(100); // 设置照片质量
        parameters.setPreviewFormat(PixelFormat.YCbCr_420_SP); // 预览格式
        parameters.setPictureFormat(PixelFormat.JPEG); // 相片格式为JPEG，默认为NV21

        // 关闪光灯
        parameters.setFlashMode(Camera.Parameters.FLASH_MODE_OFF);

        // 横竖屏镜头自动调整
        if (mContext.getResources().getConfiguration().orientation != Configuration.ORIENTATION_LANDSCAPE) {
            mCamera.setDisplayOrientation(90);
        } else {
            mCamera.setDisplayOrientation(0);
        }

        //相机异常监听
        mCamera.setErrorCallback(new Camera.ErrorCallback() {

            @Override
            public void onError(int error, Camera camera) {
                String error_str;
                switch (error) {
                    case Camera.CAMERA_ERROR_SERVER_DIED: // 摄像头已损坏
                        error_str = "摄像头已损坏";
                        break;

                    case Camera.CAMERA_ERROR_UNKNOWN:
                        error_str = "摄像头异常，请检查摄像头权限是否应许";
                        break;

                    default:
                        error_str = "摄像头异常，请检查摄像头权限是否应许";
                        break;
                }
                ToastUtil.getInstance().toast(error_str);
                Log.i(TAG, error_str);
            }
        });
        mCamera.cancelAutoFocus();
        mCamera.setParameters(parameters);
    }
```

# 三、对焦

要实现点击对焦，并有对焦环，需要自定义实现对焦环View.

**1.自定义对焦环View-CameraFocusView**

核心功能，就是对焦环缩小，并变绿。利用动画改变对焦环半径即可。
```java
     @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if(isShow){
            if(radius == GREEN_RADIUS){
                mPaint.setColor(Color.GREEN);
            }
            if(centerPoint!=null){
                canvas.drawCircle(centerPoint.x, centerPoint.y, radius, mPaint);
            }
        }
    }

    private void showAnimView() {
        isShow = true;
        if (lineAnimator == null) {
            lineAnimator = ValueAnimator.ofInt(0, 20);
            lineAnimator.setDuration(DURATION_TIME);
            lineAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    int animationValue = (Integer) animation
                            .getAnimatedValue();
                    if(lastValue!=animationValue&&radius>=(int) ((mScreenWidth * 0.1)-20)){
                        radius = radius - animationValue;
                        lastValue = animationValue;
                    }
                    isShow = true;
                    invalidate();
                }
            });
            lineAnimator.addListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    super.onAnimationEnd(animation);
                    isShow = false;
                    lastValue = 0;
                    mPaint.setColor(Color.WHITE);
                    radius = (int) (mScreenWidth * 0.1);
                    invalidate();
                }
            });
        }else{
            lineAnimator.end();
            lineAnimator.cancel();
            lineAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
            lineAnimator.start();
        }
    }
```

**2.布局界面**

让对焦环自定义View获取整个界面的触摸事件

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.awen.camera.widget.CameraSurfaceView
        android:id="@+id/cameraSurfaceView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <com.awen.camera.widget.CameraFocusView
        android:id="@+id/cameraFocusView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
    ......
```

**3.定义对焦接口**

i.定义接口

```java
    /**
     * 聚焦的回调接口
     */
    public interface IAutoFocus {
        void autoFocus(float x,float y);
    }
```

ii.对焦环View触摸事件中触发接口：

```java

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                int x = (int) event.getX();
                int y = (int) event.getY();
                lastValue = 0;
                mPaint.setColor(Color.WHITE);
                radius = (int) (mScreenWidth * 0.1);
                centerPoint = null;
                if(y>TOP_CONTROL_HEIGHT&&y<ScreenSizeUtil.getScreenHeight()-BETTOM_CONTROL_HEIGHT){//状态栏和底部禁止点击获取焦点（显示体验不好）
                    centerPoint = new Point(x, y);
                    showAnimView();
                    //开始对焦
                    if (mIAutoFocus != null) {
                        mIAutoFocus.autoFocus(event.getX(),event.getY());
                    }
                }
                break;
        }
        return true;
    }
```

**4.在CameraSurfaceView实现对焦**

i.计算对焦区域

```java
   private Rect caculateFocusPoint(int x, int y) {
        Rect rect = new Rect(x - 100, y - 100, x + 100, y + 100);
        int left = rect.left * 2000 / getWidth() - 1000;
        int top = rect.top * 2000 / getHeight() - 1000;
        int right = rect.right * 2000 / getWidth() - 1000;
        int bottom = rect.bottom * 2000 / getHeight() - 1000;
        // 如果超出了(-1000,1000)到(1000, 1000)的范围，则会导致相机崩溃
        left = left < -1000 ? -1000 : left;
        top = top < -1000 ? -1000 : top;
        right = right > 1000 ? 1000 : right;
        bottom = bottom > 1000 ? 1000 : bottom;
        return new Rect(left, top, right, bottom);
    }
```

ii.设置参数进行对焦

```java
      private void camerFocus(Rect rect) {
        if (mCamera != null) {
            Camera.Parameters parameters = mCamera.getParameters();
            if(cameraId == Camera.CameraInfo.CAMERA_FACING_BACK){
                parameters.setFocusMode(Camera.Parameters.FOCUS_MODE_AUTO);//手动区域自动对焦
            }
            if (parameters.getMaxNumFocusAreas() > 0) {
                List<Camera.Area> focusAreas = new ArrayList<Camera.Area>();
                focusAreas.add(new Camera.Area(rect, 1000));
                parameters.setFocusAreas(focusAreas);
            }
            mCamera.cancelAutoFocus(); // 先要取消掉进程中所有的聚焦功能
            mCamera.setParameters(parameters);
            mCamera.autoFocus(this);
        }
```

# 四、拍照

为了图片方便预览，需要对图片进行处理，所以需要知道相机的拍照时的方向，故在拍照应先设置照片的方向参数

**1.CameraOrientationDetector(Camera方向监听器)**

```java
/**
 * 方向变化监听器，监听传感器方向的改变
 * Created by AwenZeng on 2017/2/21.
 */

public class CameraOrientationDetector extends OrientationEventListener {
    int mOrientation;

    public CameraOrientationDetector(Context context, int rate) {
        super(context, rate);
    }

    @Override
    public void onOrientationChanged(int orientation) {
        this.mOrientation = orientation;
        if (orientation == OrientationEventListener.ORIENTATION_UNKNOWN) {
            return;
        }
        //保证只返回四个方向,分别为0°、90°、180°和270°中的一个
        int newOrientation = ((orientation + 45) / 90 * 90) % 360;
        if (newOrientation != mOrientation) {
            mOrientation = newOrientation;
        }
    }

    public int getOrientation() {
        return mOrientation;
    }
}
```

**2.设置照片方向参数**

```java
    /**
     * 拍照
     *
     * @param callback
     */
    public void takePicture(Camera.PictureCallback callback) {
        if (mCamera != null) {
            int orientation = mCameraOrientation.getOrientation();
            Camera.Parameters cameraParameter = mCamera.getParameters();
            if (orientation == 90) {
                cameraParameter.setRotation(90);
                cameraParameter.set("rotation", 90);
            } else if (orientation == 180) {
                cameraParameter.setRotation(180);
                cameraParameter.set("rotation", 180);
            } else if (orientation == 270) {
                cameraParameter.setRotation(270);
                cameraParameter.set("rotation", 270);
            } else {
                cameraParameter.setRotation(0);
                cameraParameter.set("rotation", 0);
            }
            mCamera.setParameters(cameraParameter);
        }
        mCamera.takePicture(null, null, callback);
    }
```

**3.保存图片** 

为了方便预览，对不同方向的图片，需要做正向处理。

```java
  public String handlePhoto(byte[] data, int cameraId) {
        String filePath = FileUtil.saveFile(data, "/DCIM");
        if (!TextUtils.isEmpty(filePath)) {
            int degree = BitmapUtil.getPhotoDegree(filePath);
            Log.i(TAG, degree + "");
            Bitmap bitmap = BitmapFactory.decodeFile(filePath);
            Bitmap tBitmap = null;
            try {
                Log.i(TAG, "保存图片大小："+"width = " + bitmap.getWidth() + "   ------ height = " + bitmap.getHeight());
                if (cameraId == Camera.CameraInfo.CAMERA_FACING_BACK) {
                    switch (degree) {
                        case 0:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 90);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                        case 90:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 180);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                        case 180:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 270);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                        case 270:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 360);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                    }
                } else if (cameraId == Camera.CameraInfo.CAMERA_FACING_FRONT) {
                    switch (degree) {
                        case 0:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 270);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                        case 90:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 180);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                        case 180:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 90);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                        case 270:
                            tBitmap = BitmapUtil.rotateBitmap(bitmap, 360);
                            filePath = BitmapUtil.saveBitmap(tBitmap == null ? bitmap : tBitmap, filePath);
                            break;
                    }
                }

            } catch (Exception e) {
                e.printStackTrace();
                // 重新拍照
                return "";
            } finally {
                if (bitmap != null) {
                    bitmap.recycle();
                    bitmap = null;
                }
                if (tBitmap != null) {
                    tBitmap.recycle();
                    tBitmap = null;
                }
                ScannerByReceiver(mContext, filePath);//图库扫描
            }
            return filePath;
        }
        return null;
    }
```

# 五、切换摄像头

```java
   /**
     * 切换摄像头
     */
    public void changeCamera(int camera_id) {
        mCamera.stopPreview();
        mCamera.release();
        try {
            openCamera(camera_id);
            mCamera.setPreviewDisplay(holder);
            setCameraParams(DEFAULT_PHOTO_WIDTH, DEFAULT_PHOTO_HEIGHT);
            mCamera.startPreview();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public boolean openCamera(int camera_id) {
        LogUtil.i(TAG, "openCamera id = " + camera_id);
        try {
            mCamera = Camera.open(camera_id); // 打开摄像头
            cameraId = camera_id;

        } catch (Exception e) {
            e.printStackTrace();
            ToastUtil.getInstance().toast("请先开启摄像头权限");
            LogUtil.i(TAG, "请先开启摄像头权限");
            return false;
        }

        return true;
    }
```

# 六、打开或关闭闪光灯

```java
    /**
     * 设置闪光灯
     *
     * @param isOpen
     */
    public void changeFlashMode(boolean isOpen, Camera mCamera, int cameraId) {
        if (cameraId == Camera.CameraInfo.CAMERA_FACING_BACK) { // 后摄像头才有闪光灯
            Camera.Parameters parameters = mCamera.getParameters();
            PackageManager pm = mContext.getPackageManager();
            FeatureInfo[] features = pm.getSystemAvailableFeatures();
            for (FeatureInfo f : features) {
                if (PackageManager.FEATURE_CAMERA_FLASH.equals(f.name)) { // 判断设备是否支持闪光灯
                    if (isOpen) {
                        parameters.setFlashMode(Camera.Parameters.FLASH_MODE_TORCH); // 开闪光灯

                    } else {
                        parameters
                                .setFlashMode(Camera.Parameters.FLASH_MODE_OFF); // 关闪光灯

                    }
                }
            }
            mCamera.setParameters(parameters);
        }
    }
```

# 注意事项
- Android6.0以上权限收紧，所以在使用相机前，请用PermissionsModel做好权限判断。[具体Android6.0权限](http://www.cnblogs.com/cr330326/p/5181283.html)
- 部分智能手机，前置摄像头无对焦模式，对焦参数设置应区分前置摄像头
- Android5.0以后，官方推荐使用Camera2,本例子未使用新版本。


