#ANR
---
1、ANR排错一般有三种类型

1. KeyDispatchTimeout(5 seconds) --主要是类型按键或触摸事件在特定时间内无响应
2. BroadcastTimeout(10 seconds) --BroadcastReceiver在特定时间内无法处理完成
3. ServiceTimeout(20 secends) --小概率事件 Service在特定的时间内无法处理完成

2、如何避免

1. UI线程尽量只做跟UI相关的工作
2. 耗时的操作(比如数据库操作，I/O，连接网络或者别的有可能阻塞UI线程的操作)把它放在单独的线程处理
3. 尽量用Handler来处理UIThread和别的Thread之间的交互

3、如何排查

1. 首先分析log
2. 从trace.txt文件查看调用stack，adb pull data/anr/traces.txt ./mytraces.txt
3. 看代码
4. 仔细查看ANR的成因(iowait?block?memoryleak?)

4、监测ANR的Watchdog

最近出来一个叫LeakCanary