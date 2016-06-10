#ANR
---

1、ANR排错一般有三种类型

1. KeyDispatchTimeout(5 seconds) --主要是类型按键或触摸事件在特定时间内无响应
2. BroadcastTimeout(10 seconds) --BroadcastReceiver在特定时间内无法处理完成
3. ServiceTimeout(20 secends) --小概率事件 Service在特定的时间内无法处理完成

2、哪些操作会导致ANR
在主线程执行以下操作：
1. 高耗时的操作，如图像变换
2. 磁盘读写，数据库读写操作
3. 大量的创建新对象


3、如何避免

1. UI线程尽量只做跟UI相关的工作
2. 耗时的操作(比如数据库操作，I/O，连接网络或者别的有可能阻塞UI线程的操作)把它放在单独的线程处理
3. 尽量用Handler来处理UIThread和别的Thread之间的交互

4、解决的逻辑
1. 使用AsyncTask
	1. 在doInBackground()方法中执行耗时操作
	2. 在onPostExecuted()更新UI
2. 使用Handler实现异步任务
	1. 在子线程中处理耗时操作
	2. 处理完成之后，通过handler.sendMessage()传递处理结果
	3. 在handler的handleMessage()方法中更新UI
	4. 或者使用handler.post()方法将消息放到Looper中
	

5、如何排查

1. 首先分析log
2. 从trace.txt文件查看调用stack，adb pull data/anr/traces.txt ./mytraces.txt
3. 看代码
4. 仔细查看ANR的成因(iowait?block?memoryleak?)

6、监测ANR的Watchdog

最近出来一个叫LeakCanary

#FC(Force Close)
##什么时候会出现
1. Error
2. OOM，内存溢出
3. StackOverFlowError
4. Runtime,比如说空指针异常

##解决的办法
1. 注意内存的使用和管理
2. 使用Thread.UncaughtExceptionHandler接口
