#Activity启动过程
---

###一些基本的概念

* ActivityManagerServices，简称AMS，服务端对象，负责系统中所有Activity的生命周期
* ActivityThread，App的真正入口。当开启App之后，会调用main()开始运行，开启消息循环队列，这就是传说中的UI线程或者叫主线程。与ActivityManagerServices配合，一起完成Activity的管理工作
* ApplicationThread，用来实现ActivityManagerService与ActivityThread之间的交互。在ActivityManagerService需要管理相关Application中的Activity的生命周期时，通过ApplicationThread的代理对象与ActivityThread通讯。
* ApplicationThreadProxy，是ApplicationThread在服务器端的代理，负责和客户端的ApplicationThread通讯。AMS就是通过该代理与ActivityThread进行通信的。
* Instrumentation，每一个应用程序只有一个Instrumentation对象，每个Activity内都有一个对该对象的引用。Instrumentation可以理解为应用进程的管家，ActivityThread要创建或暂停某个Activity时，都需要通过Instrumentation来进行具体的操作。
* ActivityStack，Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等。通过ActivityStack决定是否需要启动新的进程。
* ActivityRecord，ActivityStack的管理对象，每个Activity在AMS对应一个* ActivityRecord，来记录Activity的状态以及其他的管理信息。其实就是服务器端的Activity对象的映像。
* TaskRecord，AMS抽象出来的一个“任务”的概念，是记录ActivityRecord的栈，一个“Task”包含若干个ActivityRecord。AMS用TaskRecord确保Activity启动和退出的顺序。如果你清楚Activity的4种launchMode，那么对这个概念应该不陌生。

###回答一些问题

**zygote是什么？有什么作用？**

zygote意为“受精卵“。Android是基于Linux系统的，而在Linux中，所有的进程都是由init进程直接或者是间接fork出来的，zygote进程也不例外。

在Android系统里面，zygote是一个进程的名字。Android是基于Linux System的，当你的手机开机的时候，Linux的内核加载完成之后就会启动一个叫“init“的进程。在Linux System里面，所有的进程都是由init进程fork出来的，我们的zygote进程也不例外。

我们都知道，每一个App其实都是

* 一个单独的dalvik虚拟机
* 一个单独的进程

所以当系统里面的第一个zygote进程运行之后，在这之后再开启App，就相当于开启一个新的进程。而为了实现资源共用和更快的启动速度，Android系统开启新进程的方式，是通过fork第一个zygote进程实现的。所以说，除了第一个zygote进程，其他应用所在的进程都是zygote的子进程，这下你明白为什么这个进程叫“受精卵”了吧？因为就像是一个受精卵一样，它能快速的分裂，并且产生遗传物质一样的细胞！

**SystemServer是什么？有什么作用？它与zygote的关系是什么？**

首先我要告诉你的是，SystemServer也是一个进程，而且是由zygote进程fork出来的。

知道了SystemServer的本质，我们对它就不算太陌生了，这个进程是Android Framework里面两大非常重要的进程之一——另外一个进程就是上面的zygote进程。

为什么说SystemServer非常重要呢？因为系统里面重要的服务都是在这个进程里面开启的，比如 
ActivityManagerService、PackageManagerService、WindowManagerService等等，看着是不是都挺眼熟的？

那么这些系统服务是怎么开启起来的呢？

在zygote开启的时候，会调用ZygoteInit.main()进行初始化

```
public static void main(String argv[]) {

     ...ignore some code...

    //在加载首个zygote的时候，会传入初始化参数，使得startSystemServer = true
     boolean startSystemServer = false;
     for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            ...ignore some code...

         //开始fork我们的SystemServer进程
     if (startSystemServer) {
                startSystemServer(abiList, socketName);
         }

     ...ignore some code...

}
```

我们看下startSystemServer()做了些什么

```
    /**留着这个注释，就是为了说明SystemServer确实是被fork出来的
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {

         ...ignore some code...

        //留着这段注释，就是为了说明上面ZygoteInit.main(String argv[])里面的argv就是通过这种方式传递进来的
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };

        int pid;
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        //确实是fuck出来的吧，我没骗你吧~不对，是fork出来的 -_-|||
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

**ActivityManagerService是什么？什么时候初始化的？有什么作用？**

ActivityManagerService，简称AMS，服务端对象，负责系统中所有Activity的生命周期。

ActivityManagerService进行初始化的时机很明确，就是在SystemServer进程开启的时候，就会初始化ActivityManagerService。从下面的代码中可以看到

```
public final class SystemServer {

    //zygote的主入口
    public static void main(String[] args) {
        new SystemServer().run();
    }

    public SystemServer() {
        // Check for factory test mode.
        mFactoryTestMode = FactoryTest.getMode();
    }

    private void run() {

        ...ignore some code...

        //加载本地系统服务库，并进行初始化 
        System.loadLibrary("android_servers");
        nativeInit();

        // 创建系统上下文
        createSystemContext();

        //初始化SystemServiceManager对象，下面的系统服务开启都需要调用SystemServiceManager.startService(Class<T>)，这个方法通过反射来启动对应的服务
        mSystemServiceManager = new SystemServiceManager(mSystemContext);

        //开启服务
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        ...ignore some code...

    }

    //初始化系统上下文对象mSystemContext，并设置默认的主题,mSystemContext实际上是一个ContextImpl对象。调用ActivityThread.systemMain()的时候，会调用ActivityThread.attach(true)，而在attach()里面，则创建了Application对象，并调用了Application.onCreate()。
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }

    //在这里开启了几个核心的服务，因为这些服务之间相互依赖，所以都放在了这个方法里面。
    private void startBootstrapServices() {

        ...ignore some code...

        //初始化ActivityManagerService
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);

        //初始化PowerManagerService，因为其他服务需要依赖这个Service，因此需要尽快的初始化
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // 现在电源管理已经开启，ActivityManagerService负责电源管理功能
        mActivityManagerService.initPowerManagement();

        // 初始化DisplayManagerService
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    //初始化PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, mInstaller,
       mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);

    ...ignore some code...

    }

}
```

经过上面这些步骤，我们的ActivityManagerService对象已经创建好了，并且完成了成员变量初始化。而且在这之前，调用createSystemContext()创建系统上下文的时候，也已经完成了mSystemContext和ActivityThread的创建。注意，这是系统进程开启时的流程，在这之后，会开启系统的Launcher程序，完成系统界面的加载与显示。

你是否会好奇，我为什么说AMS是服务端对象？下面我给你介绍下Android系统里面的服务器和客户端的概念。

其实服务器客户端的概念不仅仅存在于Web开发中，在Android的框架设计中，使用的也是这一种模式。服务器端指的就是所有App共用的系统服务，比如我们这里提到的ActivityManagerService，和前面提到的PackageManagerService、WindowManagerService等等，这些基础的系统服务是被所有的App公用的，当某个App想实现某个操作的时候，要告诉这些系统服务，比如你想打开一个App，那么我们知道了包名和MainActivity类名之后就可以打开

```
Intent intent = new Intent(Intent.ACTION_MAIN);  
intent.addCategory(Intent.CATEGORY_LAUNCHER);              
ComponentName cn = new ComponentName(packageName, className);              
intent.setComponent(cn);  
startActivity(intent); 
```

但是，我们的App通过调用startActivity()并不能直接打开另外一个App，这个方法会通过一系列的调用，最后还是告诉AMS说：“我要打开这个App，我知道他的住址和名字，你帮我打开吧！”所以是AMS来通知zygote进程来fork一个新进程，来开启我们的目标App的。这就像是浏览器想要打开一个超链接一样，浏览器把网页地址发送给服务器，然后还是服务器把需要的资源文件发送给客户端的。

知道了Android Framework的客户端服务器架构之后，我们还需要了解一件事情，那就是我们的App和AMS(SystemServer进程)还有zygote进程分属于三个独立的进程，他们之间如何通信呢？

知道了Android Framework的客户端服务器架构之后，我们还需要了解一件事情，那就是我们的App和AMS(SystemServer进程)还有zygote进程分属于三个独立的进程，他们之间如何通信呢？

App与AMS通过Binder进行IPC通信，AMS(SystemServer进程)与zygote通过Socket进行IPC通信。

那么AMS有什么用呢？在前面我们知道了，如果想打开一个App的话，需要AMS去通知zygote进程，除此之外，其实所有的Activity的开启、暂停、关闭都需要AMS来控制，所以我们说，AMS负责系统中所有Activity的生命周期。

在Android系统中，任何一个Activity的启动都是由AMS和应用程序进程（主要是ActivityThread）相互配合来完成的。AMS服务统一调度系统中所有进程的Activity启动，而每个Activity的启动过程则由其所属的进程具体来完成。

这样说你可能还是觉得比较抽象，没关系，下面有一部分是专门来介绍AMS与ActivityThread如何一起合作控制Activity的生命周期的。

**Launcher是什么？什么时候启动的？**

当我们点击手机桌面上的图标的时候，App就由Launcher开始启动了。但是，你有没有思考过Launcher到底是一个什么东西？

Launcher本质上也是一个应用程序，和我们的App一样，也是继承自Activity

packages/apps/Launcher2/src/com/android/launcher2/Launcher.java

```
public final class Launcher extends Activity
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks,
                   View.OnTouchListener {
                   }
```

Launcher实现了点击、长按等回调接口，来接收用户的输入。既然是普通的App，那么我们的开发经验在这里就仍然适用，比如，我们点击图标的时候，是怎么开启的应用呢？如果让你，你怎么做这个功能呢？捕捉图标点击事件，然后startActivity()发送对应的Intent请求呗！是的，Launcher也是这么做的，就是这么easy！

那么到底是处理的哪个对象的点击事件呢？既然Launcher是App，并且有界面，那么肯定有布局文件呀，是的，我找到了布局文件launcher.xml

```
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res/com.android.launcher"
    android:id="@+id/launcher">

    <com.android.launcher2.DragLayer
        android:id="@+id/drag_layer"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="true">

        <!-- Keep these behind the workspace so that they are not visible when
             we go into AllApps -->
        <include
            android:id="@+id/dock_divider"
            layout="@layout/workspace_divider"
            android:layout_marginBottom="@dimen/button_bar_height"
            android:layout_gravity="bottom" />

        <include
            android:id="@+id/paged_view_indicator"
            layout="@layout/scroll_indicator"
            android:layout_gravity="bottom"
            android:layout_marginBottom="@dimen/button_bar_height" />

        <!-- The workspace contains 5 screens of cells -->
        <com.android.launcher2.Workspace
            android:id="@+id/workspace"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:paddingStart="@dimen/workspace_left_padding"
            android:paddingEnd="@dimen/workspace_right_padding"
            android:paddingTop="@dimen/workspace_top_padding"
            android:paddingBottom="@dimen/workspace_bottom_padding"
            launcher:defaultScreen="2"
            launcher:cellCountX="@integer/cell_count_x"
            launcher:cellCountY="@integer/cell_count_y"
            launcher:pageSpacing="@dimen/workspace_page_spacing"
            launcher:scrollIndicatorPaddingLeft="@dimen/workspace_divider_padding_left"
            launcher:scrollIndicatorPaddingRight="@dimen/workspace_divider_padding_right">

            <include android:id="@+id/cell1" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell2" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell3" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell4" layout="@layout/workspace_screen" />
            <include android:id="@+id/cell5" layout="@layout/workspace_screen" />
        </com.android.launcher2.Workspace>

    ...ignore some code...

    </com.android.launcher2.DragLayer>
</FrameLayout>
```

为了方便查看，我删除了很多代码，从上面这些我们应该可以看出一些东西来：Launcher大量使用标签来实现界面的复用，而且定义了很多的自定义控件实现界面效果，dock_divider从布局的参数声明上可以猜出，是底部操作栏和上面图标布局的分割线，而paged_view_indicator则是页面指示器，和App首次进入的引导页下面的界面引导是一样的道理。当然，我们最关心的是Workspace这个布局，因为注释里面说在这里面包含了5个屏幕的单元格，想必你也猜到了，这个就是在首页存放我们图标的那五个界面(不同的ROM会做不同的DIY，数量不固定)。

接下来，我们应该打开workspace_screen布局，看看里面有什么东东。

workspace_screen.xml

```
<com.android.launcher2.CellLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res/com.android.launcher"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:paddingStart="@dimen/cell_layout_left_padding"
    android:paddingEnd="@dimen/cell_layout_right_padding"
    android:paddingTop="@dimen/cell_layout_top_padding"
    android:paddingBottom="@dimen/cell_layout_bottom_padding"
    android:hapticFeedbackEnabled="false"
    launcher:cellWidth="@dimen/workspace_cell_width"
    launcher:cellHeight="@dimen/workspace_cell_height"
    launcher:widthGap="@dimen/workspace_width_gap"
    launcher:heightGap="@dimen/workspace_height_gap"
    launcher:maxGap="@dimen/workspace_max_gap" />
```

里面就一个CellLayout，也是一个自定义布局，那么我们就可以猜到了，既然可以存放图标，那么这个自定义的布局很有可能是继承自ViewGroup或者是其子类，实际上，CellLayout确实是继承自ViewGroup。在CellLayout里面，只放了一个子View，那就是ShortcutAndWidgetContainer。从名字也可以看出来，ShortcutAndWidgetContainer这个类就是用来存放快捷图标和Widget小部件的，那么里面放的是什么对象呢？

在桌面上的图标，使用的是BubbleTextView对象，这个对象在TextView的基础之上，添加了一些特效，比如你长按移动图标的时候，图标位置会出现一个背景(不同版本的效果不同)，所以我们找到BubbleTextView对象的点击事件，就可以找到Launcher如何开启一个App了。

除了在桌面上有图标之外，在程序列表中点击图标，也可以开启对应的程序。这里的图标使用的不是BubbleTextView对象，而是PagedViewIcon对象，我们如果找到它的点击事件，就也可以找到Launcher如何开启一个App。

其实说这么多，和今天的主题隔着十万八千里，上面这些东西，你有兴趣就看，没兴趣就直接跳过，不知道不影响这篇文章阅读。

BubbleTextView的点击事件在哪里呢？我来告诉你：在Launcher.onClick(View v)里面。

```
/**
     * Launches the intent referred by the clicked shortcut
     */
    public void onClick(View v) {

          ...ignore some code...

         Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            // Open shortcut
            final Intent intent = ((ShortcutInfo) tag).intent;
            int[] pos = new int[2];
            v.getLocationOnScreen(pos);
            intent.setSourceBounds(new Rect(pos[0], pos[1],
                    pos[0] + v.getWidth(), pos[1] + v.getHeight()));
        //开始开启Activity咯~
            boolean success = startActivitySafely(v, intent, tag);

            if (success && v instanceof BubbleTextView) {
                mWaitingForResume = (BubbleTextView) v;
                mWaitingForResume.setStayPressed(true);
            }
        } else if (tag instanceof FolderInfo) {
            //如果点击的是图标文件夹，就打开文件夹
            if (v instanceof FolderIcon) {
                FolderIcon fi = (FolderIcon) v;
                handleFolderClick(fi);
            }
        } else if (v == mAllAppsButton) {
        ...ignore some code...
        }
    }
```

从上面的代码我们可以看到，在桌面上点击快捷图标的时候，会调用

```
startActivitySafely(v, intent, tag);
```

那么从程序列表界面，点击图标的时候会发生什么呢？实际上，程序列表界面使用的是AppsCustomizePagedView对象，所以我在这个类里面找到了onClick(View v)。

com.android.launcher2.AppsCustomizePagedView.java

```
/**
 * The Apps/Customize page that displays all the applications, widgets, and shortcuts.
 */
public class AppsCustomizePagedView extends PagedViewWithDraggableItems implements
        View.OnClickListener, View.OnKeyListener, DragSource,
        PagedViewIcon.PressedCallback, PagedViewWidget.ShortPressListener,
        LauncherTransitionable {

     @Override
    public void onClick(View v) {

         ...ignore some code...

        if (v instanceof PagedViewIcon) {
            mLauncher.updateWallpaperVisibility(true);
            mLauncher.startActivitySafely(v, appInfo.intent, appInfo);
        } else if (v instanceof PagedViewWidget) {
                 ...ignore some code..
         }
     }      
   }
```

可以看到，调用的是

```
mLauncher.startActivitySafely(v, appInfo.intent, appInfo);
```

殊途同归

不管从哪里点击图标，调用的都是Launcher.startActivitySafely()。

下面我们就可以一步步的来看一下Launcher.startActivitySafely()到底做了什么事情。

```
 boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        try {
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        }
        return success;
    }
```

调用了startActivity(v, intent, tag)

```
 boolean startActivity(View v, Intent intent, Object tag) {

        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);

            if (useLaunchAnimation) {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent, opts.toBundle());
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(),
                            opts.toBundle());
                }
            } else {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(), null);
                }
            }
            return true;
        } catch (SecurityException e) {
        ...
        }
        return false;
    }
```

这里会调用Activity.startActivity(intent, opts.toBundle())，这个方法熟悉吗？这就是我们经常用到的Activity.startActivity(Intent)的重载函数。而且由于设置了

```
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
```

所以这个Activity会添加到一个新的Task栈中，而且，startActivity()调用的其实是startActivityForResult()这个方法。

```
@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

所以我们现在明确了，Launcher中开启一个App，其实和我们在Activity中直接startActivity()基本一样，都是调用了Activity.startActivityForResult()。

**Instrumentation是什么？和ActivityThread是什么关系？**

还记得前面说过的Instrumentation对象吗？每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象。当startActivityForResult()调用之后，实际上还是调用了mInstrumentation.execStartActivity()

```
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            ...ignore some code...    
        } else {
            if (options != null) {
                 //当现在的Activity有父Activity的时候会调用，但是在startActivityFromChild()内部实际还是调用的mInstrumentation.execStartActivity()
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
         ...ignore some code...    
    }
```

下面是mInstrumentation.execStartActivity()的实现

```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
            ...ignore some code...
      try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```

所以当我们在程序中调用startActivity()的 时候，实际上调用的是Instrumentation的相关的方法。

Instrumentation意为“仪器”，我们先看一下这个类里面包含哪些方法吧

![](http://i11.tietuku.com/15eb948d37d9d4b3.png)

我们可以看到，这个类里面的方法大多数和Application和Activity有关，是的，这个类就是完成对Application和Activity初始化和生命周期的工具类。比如说，我单独挑一个callActivityOnCreate()让你看看

```
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

对activity.performCreate(icicle);这一行代码熟悉吗？这一行里面就调用了传说中的Activity的入口函数onCreate()，不信？接着往下看

Activity.performCreate()

```
final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
```

没骗你吧，onCreate在这里调用了吧。但是有一件事情必须说清楚，那就是这个Instrumentation类这么重要，为啥我在开发的过程中，没有发现他的踪迹呢？

是的，Instrumentation这个类很重要，对Activity生命周期方法的调用根本就离不开他，他可以说是一个大管家，但是，这个大管家比较害羞，是一个女的，管内不管外，是老板娘~

那么你可能要问了，老板是谁呀？ 
老板当然是大名鼎鼎的ActivityThread了！

ActivityThread你都没听说过？那你肯定听说过传说中的UI线程吧？是的，这就是UI线程。我们前面说过，App和AMS是通过Binder传递信息的，那么ActivityThread就是专门与AMS的外交工作的。

AMS说：“ActivityThread，你给我暂停一个Activity！” 
ActivityThread就说：“没问题！”然后转身和Instrumentation说：“老婆，AMS让暂停一个Activity，我这里忙着呢，你快去帮我把这事办了把~”
于是，Instrumentation就去把事儿搞定了。

所以说，AMS是董事会，负责指挥和调度的，ActivityThread是老板，虽然说家里的事自己说了算，但是需要听从AMS的指挥，而Instrumentation则是老板娘，负责家里的大事小事，但是一般不抛头露面，听一家之主ActivityThread的安排。

**如何理解AMS和ActivityThread之间的Binder通信？**

前面我们说到，在调用startActivity()的时候，实际上调用的是

```
mInstrumentation.execStartActivity()
```

但是到这里还没完呢！里面又调用了下面的方法

```
ActivityManagerNative.getDefault()
                .startActivity
```




