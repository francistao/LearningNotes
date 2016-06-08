##理解Window和WindowManager

Window是一个抽象类，它的具体实现是PhoneWindow。WindowManager是外界访问Window的入口，Window的具体实现位于WindowManagerService中，WindowManager和WindowManagerService的交互是一个IPC过程。Android中所有的视图都是通过Window来呈现的，不管是Activity、Dialog还是Toast，它们的视图实际上都是附加在Window上的，因此Window实际是View的直接管理者。

###8.1 Window和WindowManager

为了分析Window的工作机制，先通过代码了解如何使用WindowManager添加一个Window，下面一段代码将一个Button添加到屏幕坐标为(100, 300)的位置上

```
mFloatingButton = new Button(this);
mFloatingButton.setText("test button");
mLayoutParams = new WindowManager.LayoutParams(
        LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT, 0, 0,
        PixelFormat.TRANSPARENT);//0,0 分别是type和flags参数，在后面分别配置了
mLayoutParams.flags = LayoutParams.FLAG_NOT_TOUCH_MODAL
        | LayoutParams.FLAG_NOT_FOCUSABLE
        | LayoutParams.FLAG_SHOW_WHEN_LOCKED;
mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR;
mLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;
mLayoutParams.x = 100;
mLayoutParams.y = 300;
mFloatingButton.setOnTouchListener(this);
mWindowManager.addView(mFloatingButton, mLayoutParams);
```
Flags参数表示Window的属性，以下列举常用的选项：

* FLAG_NOT_FOCUSABLE：表示Window不需要获取焦点，也不需要接收各种输入事件，此标记会同时启动FLAG_NOT_TOUCH_MODEL，最终事件会传递给下层的具有焦点的Window
* FLAG_NOT_TOUCH_MODAL：在此模式下，系统会将当前Window区域以外的单击事件传递给底层的Window，当前Window区域以内的单击事件则自己处理。这个标记很重要，一般来说都需要开启此标记，否则其他Window将无法收到单击事件。
* FLAG_SHOW_WHEN_LOCKED：开启此模式可以让显示在锁屏的界面




Type参数表示Window的类型，Window有三种类型，分别是应用Window、子Window和系统Window。应用类Window对应着一个Activity。子Window不能单独存在，它需要附属在特定的父Window之中，比如常见的一些Dialog就是一个子Window。系统Window是需要声明权限才能创建的Window，比如Toast和系统状态栏这些都是系统Window。

Window是分层的，每个Window都有对应的z-ordered，层级最大的会覆盖在层级小的Window上面，这和HTML中的z-index的概念是完全一致的。在三类Window中，应用Window的层级范围是1~99，子Window的层级范围是1000~1999，系统Window的层级范围是2000~2999，这些层级属性范围对应着WindowManager.LayoutParams的type参数。

如果采用TYPE_SYSTEM_ERROR，只需要为type参数指定这个层级即可：

mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR

同时声明权限：

<uses-permissionandroid:name="android.permission.SYSTEM_ALERT_WINDOW" />

WindowManager所提供的功能很简单，常用的只有三个方法，即添加View、更新View和删除View，这三个方法定义在ViewManager中，而WindowManager继承了ViewManager。

###8.2 Window的内部机制

Window是一个抽象的概念，并不是实际存在的，它是以View的形式存在，每一个Window都对应着一个View和一个ViewRootImpl，Window和View通过ViewRootImpl来建立联系。在实际使用中无法直接访问Window，对Window的访问必须通过WindowManager。

####8.2.1 Window的添加过程

Window的添加过程需要通过WindowManager的addView来实现，WindowManager是一个接口，它的真正实现是WindowManagerImpl类。WindowManager的实现类对于addView、updateView和removeView方法都是委托给WindowManagerGlobal类。

WindowManagerGlobal的addView方法分为如下几步：

1. 检查参数是否合法，如果是子Window那么还需要调整一些布局参数
2. 创建ViewRootImpl并将View添加到列表中
3. 通过ViewRootImpl来更新界面并完成Window的添加过程

####8.2.2 Window的删除过程

和添加过程一样，都是先通过WindowManagerImpl后，再进一步通过WindowManagerGlobal来实现的。
真正删除View的逻辑在dispatchDetachedFromWindow方法的内部实现。dispatchDetachedFromWindow方法主要做四件事：

1. 垃圾回收的工作，比如清除数据和消息，移除回调。
2. 通过Session的remove方法删除Window，mWindowSession.remove(mWindow)，这同样是一个IP C过程，最终会调用WindowManagerService的removeWindow方法
3. 调用View的dispatchDetachedFromWindow方法，在内部调用View的onDetachedFromWindow()以及onDetachedFromWindowInternal()。
4. 调用WindowManagerGlobal的doRemoveView方法刷新数据，包括mRoots、mParams以及mDyingViews，需要将当前Window所关联的这三类对象从列表中删除。

####8.2.3 Window的更新过程

首先需要更新View的LayoutParams并替换掉老的LayoutParams，接着再更新ViewRootImpl中的LayoutParams，这一步是通过ViewRootImpl的setLayoutParams方法来实现的。在ViewRootImpl中会通过scheduleTrversals方法来对View重新布局，包括测量、布局、重绘三个过程。除了View本身的重绘以外，ViewRootImpl还会通过WindowSession来更新Window的视图，这个过程最终是由WindowManagerService的relayoutWindow()来具体实现的，同样是一个IPC过程。

##Window的创建过程

###8.3.1 Activity的Window创建过程

1、Activity的启动过程很复杂，最终会由ActivityThread中的performLaunchActivity()来完成整个启动过程，在这个方法内部会通过类加载器创建Activity的实例对象，并调用其attach方法为其关联运行过程中所依赖的一系列上下文环境变量。

2、Activity实现了Window的Callback接口，当Window接收到外界的状态变化时就会调用Activity的方法，例如onAttachedToWindow、onDetachedFromWindow、dispatchTouchEvent等。

3、Activity的Window是由PolicyManager来创建的，它的真正实现是Policy类，它会新建一个PhoneWindow对象，Activity的setContentView的实现是由PhoneWindow来实现的。
PhoneWindow方法大致遵循如下几个步骤：

1. 如果没有DecorView，那么就创建它
2. 将View添加到DecorView的mContentParent中
3. 回调Activity的onCreateChanged方法通知Activity视图已经发生改变

###8.3.2 Dialog的Window创建过程

Dialog的Window的创建过程和Activity类似，有如下步骤：

1. 创建Window:Diolog中Window的创建同样是通过PolicyManager的makeNewWindow方法来完成的，创建后的对象实际上就是PhoneWindow。
2. 初始化DecorView并将Dialog的视图添加到DecorView中
3. 将DecorView添加到Window中并显示：普通的Dialog有一个特殊之处，就是必须采用Activity的Context，如果采用Application的Context，那么就会报错。应用token只有Activity拥有，所以这里只需要Activity作为Context来显示对话框即可。

###8.3.3 Toast的Window创建过程

在Toast的内部有两类IPC过程，第一类是Toast访问NotificationManagerService，第二类是NotificationManagerService回调Toast里的TN接口。

Toast属于系统Window，它内部的视图由两种方式指定：一种是系统默认的演示，另一种是通过setView方法来指定一个自定义的View

Toast具有定时取消功能，所以系统采用了Handler。Toast的显示和隐藏是IPC过程，都需要NotificationManagerService（NMS）来实现，在Toast和NMS进行IPC过程时，NMS会跨进程回调Toast中的TN类中的方法，TN类是一个Binder类，运行在Binder线程池中，所以需要通过Handler将其切换到当前发送Toast请求所在的线程，所以Toast无法在没有Looper的线程中弹出。

对于非系统应用来说，mToastQueue最多能同时存在50个ToastRecord，这样做是为了防止DOS(Denial of Service，拒绝服务)。因为如果某个应用弹出太多的Toast会导致其他应用没有机会弹出Toast。









