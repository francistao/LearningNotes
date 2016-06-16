#Zygote和System进程的启动过程
---

##init脚本的启动
---

```
+------------+    +-------+   +-----------+
|Linux Kernel+--> |init.rc+-> |app_process|
+------------+    +-------+   +-----------+
               create and public          
                 server socket
```

linux内核加载完成后，运行init.rc脚本

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server socket zygote stream 666
```

* /system/bin/app_process Zygote服务启动的进程名
* --start-system-server 表明Zygote启动完成之后，要启动System进程。
* socket zygote stream 666 在Zygote启动时，创建一个权限为666的socket。此socket用来请求Zygote创建新进程。socket的fd保存在名称为“ANDROID_SOCKET_zygote”的环境变量中。

##Zygote进程的启动过程
---

```
               create rumtime                            
  +-----------+            +----------+                  
  |app_process+----------> |ZygoteInit|                  
  +-----------+            +-----+----+                  
                                 |                       
                                 |                       
                                 | registerZygoteSocket()
                                 |                       
   +------+  startSystemServer() |                       
   |System| <-------+            |                       
   +------+   fork               | runSelectLoopMode()   
                                 |                       
                                 v         
```

**app_process进程**

---

/system/bin/app_process 启动时创建了一个AppRuntime对象。通过AppRuntime对象的start方法，通过JNI调用创建了一个虚拟机实例，然后运行com.android.internal.os.ZygoteInit类的静态main方法，传递true(boolean startSystemServer)参数。

**ZygoteInit类**

---

ZygoteInit类的main方法运行时，会通过registerZygoteSocket方法创建一个供ActivityManagerService使用的server socket。然后通过调用startSystemServer方法来启动System进程。最后通过runSelectLoopMode来等待AMS的新建进程请求。

1. 在registerZygoteSocket方法中，通过名为ANDROID_SOCKET_zygote的环境获取到zygote启动时创建的socket的fd，然后以此来创建server socket。
2. 在startSystemServer方法中，通过Zygote.forkSystemServer方法创建了一个子进程，并将其用户和用户组的ID设置为1000。
3. 在runSelectLoopMode方法中，会将之前建立的server socket保存起来。然后进入一个无限循环，在其中通过selectReadable方法，监听socket是否有数据可读。有数据则说明接收到了一个请求。
selectReadable方法会返回一个整数值index。如果index为0，则说明这个是AMS发过来的连接请求。这时会与AMS建立一个新的socket连接，并包装成ZygoteConnection对象保存起来。如果index大于0，则说明这是AMS发过来的一个创建新进程的请求。此时会取出之前保存的ZygoteConnection对象，调用其中的runOnce方法创建新进程。调用完成后将connection删除。
这就是Zygote处理一次AMS请求的过程。

##System进程的启动
---

```
 +                                                     
 |                                                     
 |                                                     
 v fork()                                              
 +--------------+                                      
 |System Process|                                      
 +------+-------+                                      
        |                                              
        | RuntimeInit.zygoteInit() commonInit, zygoteInitNative                                             
        | init1()  SurfaceFlinger, SensorServic...     
        |                                              
        |                                              
        | init2() +------------+                       
        +-------> |ServerThread|                       
        |         +----+-------+                       
        |              |                               
        |              | AMS, PMS, WMS...              
        |              |                               
        |              |                               
        |              |                               
        v              v               
```

System进程是在ZygoteInit的handleSystemServerProcess中开始启动的。

1. 首先，因为System进程是直接fork Zygote进程的，所以要先通过closeServerSocket方法关掉server socket。
2. 调用RuntimeInit.zygoteInit方法进一步启动System进程。在zygoteInit中，通过commonInit方法设置时区和键盘布局等通用信息，然后通过zygoteInitNative方法启动了一个Binder线程池。最后通过invokeStaticMain方法调用SystemServer类的静态Main方法。
3. SystemServer类的main通过JNI调用cpp实现的init1方法。在init1方法中，会启动各种以C++开发的系统服务（例如SurfaceFlinger和SensorService）。然后回调ServerServer类的init2方法来启动以Java开发的系统服务。
4. 在init2方法中，首先会新建名为"android.server.ServerThread"的ServerThread线程，并调用其start方法。然后在该线程中启动各种Service（例如AMS，PMS，WMS等）。启动的方式是调用对应Service类的静态main方法。
5. 首先，AMS会被创建，但未注册到ServerManager中。然后PMS被创建，AMS这时候才注册到ServerManager中。然后到ContentService、WMS等。
注册到ServerManager中时会制定Service的名字，其后其他进程可以通过这个名字来获取到Binder Proxy对象，以访问Service提供的服务。
6. 执行到这里，System就将系统的关键服务启动起来了，这时候其他进程便可利用这些Service提供的基础服务了。
7. 最后会调用ActivityManagerService的systemReady方法，在该方法里会启动系统界面以及Home程序。

##Android进程启动

---

```
   +----------------------+       +-------+      +----------+   +----------------+   +-----------+                         
   |ActivityManagerService|       |Process|      |ZygoteInit|   |ZygoteConnection|   |RuntimeInit|                         
   +--------------+-------+       +---+---+      +-----+----+   +-----------+----+   +------+----+                         
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |                              
 startProcessLocked()                 |                |                    |               |                              
+---------------> |                   |                |                    |               |                              
                  |  start()          |                |                    |               |                              
                  |  "android.app.ActivityThread"      |                    |               |                              
                  +-----------------> |                |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |openZygoteSocketIfNeeded()           |               |                              
                  |                   +------+         |                    |               |                              
                  |                   |      |         |                    |               |                              
                  |                   | <----+         |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |sZygoteWriter.write(arg)             |               |                              
                  |                   +------+         |                    |               |                              
                  |                   |      |         |                    |               |                              
                  |                   |      |         |                    |               |                              
                  |                   | <----+         |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   +--------------> |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |                |runSelectLoopMode() |               |                              
                  |                   |                +-----------------+  |               |                              
                  |                   |                |                 |  |               |                              
                  |                   |                | <---------------+  |               |                              
                  |                   |                |   acceptCommandPeer()              |                              
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |                |  runOnce()         |               |                              
                  |                   |                +------------------> |               |                              
                  |                   |                |                    |forkAndSpecialize()                           
                  |                   |                |                    +-------------+ |                              
                  |                   |                |                    |             | |                              
                  |                   |                |                    | <-----------+ |                              
                  |                   |                |                    |  handleChildProc()                           
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |                              
                  |                   |                |                    | zygoteInit()  |                              
                  |                   |                |                    +-------------> |                              
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |in^okeStaticMain()            
                  |                   |                |                    |               +---------------->             
                  |                   |                |                    |               |("android.app.ActivityThread")
                  |                   |                |                    |               |                              
                  |                   |                |                    |               |                              
                  +                   +                +                    +               +                            
```

* AMS向Zygote发起请求（通过之前保存的socket），携带各种参数，包括“android.app.ActivityThread”。
* Zygote进程fork自己，然后在新Zygote进程中调用RuntimeInit.zygoteInit方法进行一系列的初始化（commonInit、Binder线程池初始化等）。
* 新Zygote进程中调用ActivityThread的main函数，并启动消息循环。