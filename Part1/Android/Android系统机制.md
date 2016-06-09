#Android系统机制
---
###APP启动过程

1. Launcher线程捕获onclick的点击事件，调用Launcher.startActivitySafely，进一步调用Launcher.startActivity，最后调用父类Activity的startActivity。
2. Activity和ActivityManagerService交互，引入Instrumentation，将启动请求交给Instrumentation，调用Instrumentation.execStartActivity
3. 

###Android内核解读-应用的安装过程

[http://blog.csdn.net/singwhatiwanna/article/details/19578947](http://blog.csdn.net/singwhatiwanna/article/details/19578947)
apk的安装过程分为两步：

1. 将apk文件复制到程序目录下(/data/app/)
2. 为应用创建数据目录(/data/data/package name/)、提取dex文件到指定目录(/data/delvik-cache/)、修改系统包管理信息。


###View的事件体系



###Handler消息机制


###AsyncTask
AyncTask是一个抽象类。

需要重写的方法有四个：
1. onPreExecute()
	这个方法会在后台任务开始之前调用，用于进行一些界面上的初始化操作，比如显示一个进度条对话框等
2. doInBackGround(Params...)
	在子线程中运行，不可更新UI，返回的是执行结果，第三个参数为Void不返回
3. onProgressUpdate(Progress...)
	利用参数可以进行UI操作。
4. onPostExecute(Result)
	返回的数据会作为参数传递到此方法中，可以利用返回的一些数据来进行一些UI操作。

```
class DownloadTask extends AsyncTask<Void, Integer, Boolean> {  
  
    @Override  
    protected void onPreExecute() {  
        progressDialog.show();  
    }  
  
    @Override  
    protected Boolean doInBackground(Void... params) {  
        try {  
            while (true) {  
                int downloadPercent = doDownload();  
                publishProgress(downloadPercent);  
                if (downloadPercent >= 100) {  
                    break;  
                }  
            }  
        } catch (Exception e) {  
            return false;  
        }  
        return true;  
    }  
  
    @Override  
    protected void onProgressUpdate(Integer... values) {  
        progressDialog.setMessage("当前下载进度：" + values[0] + "%");  
    }  
  
    @Override  
    protected void onPostExecute(Boolean result) {  
        progressDialog.dismiss();  
        if (result) {  
            Toast.makeText(context, "下载成功", Toast.LENGTH_SHORT).show();  
        } else {  
            Toast.makeText(context, "下载失败", Toast.LENGTH_SHORT).show();  
        }  
    }  
}  
```

启动这个任务：

```
new DownloadTask().execute();
```







###图片缓存机制




