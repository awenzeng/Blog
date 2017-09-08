---
layout: post
title: "HTTP文件断点续传原理解析"
date: 9/8/2017 7:59:12 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Http文件断点续传
---
---
生活中，有许多事物，在没有被揭开面纱之前，我们往往会觉得很神秘很高深，认为它一定很难，进而望而却步，失去了解它的机会。然而，很多事，只要我们自己能沉下心来，细细研究，那些神秘高深的，也会变得简单明了。"HTTP文件断点续传"就是这样一个好例子，深入了解背后之理，“HTTP文件断点续传原理”其实很简单。

# 一、什么是断点续传

**1.定义：**

可以从下载或上传断开点继续开始传输，就叫断点续传。

**2.核心实现原理：**

**i.RandomAccessFile(文件任意位置保存)**
方法seek():可以移动到保存文件任意位置，在该位置发生下一个读取或写入操作

**ii.HttpURLConnection.setRequestProperty()(任意位置请求返回剩余文件)**
HttpURLConnection.setRequestProperty("Range", "bytes=" + start + "-" + end)


# 二、实例分析
<!-- more -->
流程图

![](/assets/img/tech_breakpoint_flowchart.png)

实现步骤

- 1.建立数据库：保存文件下载信息
- 2.下载服务类(DownloadService)
- 3.两个线程：文件信息线程(FileInfoThread)和文件下载线程(DownloadThread)
- 4.广播(BroadcastReceiver)：UI进度更新

------------
**1.建立数据库**
按常规数据库建立方法，具体(略)。数据保存信息为：
```java
/**
 * 下载信息类
 */
public class DownloadInfo {
    private int id;
    private String url;//下载链接
    private long start;//开始大小
    private long end;//最终大小
    private long progress;//下载进度
}
```
**2.下载服务类**
利用service多次启动只调用onStartCommand()方法，处理开始或暂停下载逻辑。

```java
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent.getAction().equals(ACTION_START)) {
            FileInfo fileInfo = (FileInfo) intent.getSerializableExtra(TAG_FILEINFO);
            mFileInfoThread = new FileInfoThread(fileInfo,mHandler);
            mFileInfoThread.start();

        } else if (intent.getAction().equals(ACTION_PAUSE)) {
            if (mDownloadThread != null) {
                mDownloadThread.setPause(true);
            }
        }
        return super.onStartCommand(intent, flags, startId);
    }
```
**3.两个线程**
**i.文件信息线程(FileInfoThread)**

通过网络获取下载文件大小，并建立对应大小的保存文件路径。
```java
 HttpURLConnection conn = null;
        RandomAccessFile raf = null;
        try {
            URL url = new URL(mFileInfo.getUrl());
            conn = (HttpURLConnection) url.openConnection();//连接网络文件
            conn.setConnectTimeout(3000);
            conn.setRequestMethod("GET");
            int length = -1;
            Log.e(TAG,"HttpResponseCode=="+ conn.getResponseCode() + "");
            if (conn.getResponseCode() == HttpURLConnection.HTTP_OK) {
                //获取文件长度
                length = conn.getContentLength();
            }
            if (length < 0) {
                return;
            }
            File dir = new File(DownloadService.DOWNLOAD_PATH);
            if (!dir.exists()) {
                dir.mkdir();
            }
            //在本地创建文件
            File file = new File(dir, mFileInfo.getFileName());
            raf = new RandomAccessFile(file, "rwd");
            //设置本地文件长度
            raf.setLength(length);
            mFileInfo.setLength(length);
            Log.e(TAG,"下载文件大小(size)"+ mFileInfo.getLength() + "");
            mHandler.obtainMessage(DownloadService.MSG_FILEINFO, mFileInfo).sendToTarget();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (conn != null && raf != null) {
                    raf.close();
                    conn.disconnect();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
```

**ii.文件下载线程(DownloadThread)**
断点续传原理核心类。
**1.判断下载进度是否有保存，若无，数据插入一条数据。**
```java
        if (!mDatabaseOperation.isExists(downloadInfo.getUrl(), downloadInfo.getId())) {
            mDatabaseOperation.insert(downloadInfo);
        }
```
**2.设置网络请求Range参数，从请求位置返回数据**
```java
   //设置下载位置
   long start = downloadInfo.getStart() + downloadInfo.getProgress();
   connection.setRequestProperty("Range", "bytes=" + start + "-" + downloadInfo.getEnd());
```
**3.通过RandomAccessFile从进度保存位置保存文件**
```java
  RandomAccessFile raf;
  File file = new File(DownloadService.DOWNLOAD_PATH, mFileInfo.getFileName());
  raf = new RandomAccessFile(file, "rwd");
  raf.seek(start);
  ...
  //写入文件
  raf.write(buffer, 0, len);
```
**4.用户暂停时，保存下载进度**
```java
 //下载暂停时，保存进度
  if (isPause) {
     Log.e(TAG,"保存进度文件(size):"+progress + "");
     mDatabaseOperation.update(mFileInfo.getUrl(), mFileInfo.getId(), progress);
     return;
  }
```
**4.广播(BroadcastReceiver)**：
每秒广播一次，刷新UI
```java
 long time = System.currentTimeMillis();
 ......
 if (System.currentTimeMillis() - time > 1000) {//超过一秒，就刷新UI
     time = System.currentTimeMillis();
     sendBroadcast(intent,(int)(progress * 100 / mFileInfo.getLength()));
     Log.e(TAG,"进度：" + progress * 100 / mFileInfo.getLength() + "%");
 }
```

**DownloadThread类源码：**
```java
/**
 * 文件下载线程
 * Created by AwenZeng on 2017/9/6.
 */

public class DownloadThread extends Thread {
    private DownloadInfo downloadInfo;
    private FileInfo mFileInfo;
    private long progress = 0;
    private boolean isPause;
    private DatabaseOperation mDatabaseOperation;
    private Context mContext;
    private static final String TAG = "DownloadThread";

    public DownloadThread(Context context, DatabaseOperation databaseOperation, DownloadInfo threadInfo, FileInfo fileInfo) {
        this.downloadInfo = threadInfo;
        mContext = context;
        mDatabaseOperation = databaseOperation;
        mFileInfo = fileInfo;
    }


    public void setPause(boolean pause) {
        isPause = pause;
    }

    @Override
    public void run() {
        if (!mDatabaseOperation.isExists(downloadInfo.getUrl(), downloadInfo.getId())) {
            mDatabaseOperation.insert(downloadInfo);
        }
        HttpURLConnection connection;
        RandomAccessFile raf;
        InputStream is;
        try {
            URL url = new URL(downloadInfo.getUrl());
            connection = (HttpURLConnection) url.openConnection();
            connection.setConnectTimeout(3000);
            connection.setRequestMethod("GET");
            //设置下载位置
            long start = downloadInfo.getStart() + downloadInfo.getProgress();
            connection.setRequestProperty("Range", "bytes=" + start + "-" + downloadInfo.getEnd());

            //设置文件写入位置
            File file = new File(DownloadService.DOWNLOAD_PATH, mFileInfo.getFileName());
            raf = new RandomAccessFile(file, "rwd");
            raf.seek(start);

            progress += downloadInfo.getProgress();
            Log.e(TAG,"下载文件进度(size)："+ downloadInfo.getProgress() + "");
            Log.e(TAG,"HttpResponseCode ==="+connection.getResponseCode() + "");
            //开始下载
            if (connection.getResponseCode() == HttpURLConnection.HTTP_PARTIAL) {
                Log.e(TAG,"剩余文件(size):"+connection.getContentLength() + "");
                Intent intent = new Intent(DownloadService.ACTION_UPDATE);//广播intent
                is = connection.getInputStream();
                byte[] buffer = new byte[1024 * 4];
                int len = -1;
                long time = System.currentTimeMillis();
                while ((len = is.read(buffer)) != -1) {
                    //下载暂停时，保存进度
                    if (isPause) {
                        Log.e(TAG,"保存进度文件(size):"+progress + "");
                        mDatabaseOperation.update(mFileInfo.getUrl(), mFileInfo.getId(), progress);
                        return;
                    }
                    //写入文件
                    raf.write(buffer, 0, len);
                    //把下载进度发送广播给Activity
                    progress += len;
                    if (System.currentTimeMillis() - time > 1000) {//超过一秒，就刷新UI
                        time = System.currentTimeMillis();
                        sendBroadcast(intent,(int)(progress * 100 / mFileInfo.getLength()));
                        Log.e(TAG,"进度：" + progress * 100 / mFileInfo.getLength() + "%");
                    }
                }
                sendBroadcast(intent,100);
                /**
                 *  删除下载信息（重新下载）
                 */
                mDatabaseOperation.delete(mFileInfo.getUrl(), mFileInfo.getId());
                is.close();
            }
            raf.close();
            connection.disconnect();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void sendBroadcast(Intent intent,int progress){
        intent.putExtra("progress",progress);
        mContext.sendBroadcast(intent);
    }
}
```

# 三、源码地址
如果你觉得还不错，欢迎star或fork。
[https://github.com/awenzeng/BreakPointDemo](https://github.com/awenzeng/BreakPointDemo) 

# 四、参考文献
[RandomAccessFiley详解](http://www.cnblogs.com/tp123/p/6430395.html)

[http断点续传原理：http头 Range、Content-Range](http://blog.csdn.net/lv18092081172/article/details/51457525)

[InputStream中read()与read(byte[] b)](http://blog.csdn.net/jdsjlzx/article/details/8875758)









