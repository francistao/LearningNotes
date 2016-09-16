#EventBus
---

###概述

EventBus是一款针对Android优化的发布/订阅（publish/subscribe）事件总线。主要功能是替代Intent,Handler,BroadCast在Fragment，Activity，Service，线程之间传递消息。简化了应用程序内各组件间、组件与后台线程间的通信。优点是开销小，代码更优雅。以及将发送者和接收者解耦。比如请求网络，等网络返回时通过Handler或Broadcast通知UI，两个Fragment之间需要通过Listener通信，这些需求都可以通过EventBus实现。

###EventBus作为一个消息总线，有三个主要的元素：

* Event：事件。可以是任意类型的对象
* Subscriber：事件订阅者，接收特定的事件。在EventBus中，使用约定来指定事件订阅者以简化使用。即所有事件订阅都都是以onEvent开头的函数，具体来说，函数的名字是onEvent,onEventMainThread，onEventBackgroundThread，onEventAsync这四个，这个和
ThreadMode（下面讲）有关。
* Publisher：事件发布者，用于通知 Subscriber 有事件发生。可以在任意线程任意位置发送事件，直接调用eventBus.post(Object) 方法，可以自己实例化 EventBus 
对象，但一般使用默认的单例就好了：EventBus.getDefault()， 根据post函数参数的类型，会自动调用订阅相应类型事件的函数。

###关于ThreadMode

前面说了，Subscriber的函数只能是那4个，因为每个事件订阅函数都是和一个ThreadMode相关联的，ThreadMode指定了会调用的函数。有以下四个ThreadMode：

* PostThread：事件的处理在和事件的发送在相同的进程，所以事件处理时间不应太长，不然影响事件的发送线程，而这个线程可能是UI线程。对应的函数名是onEvent。
* MainThread: 事件的处理会在UI线程中执行。事件处理时间不能太长，这个不用说的，长了会ANR的，对应的函数名是onEventMainThread。
* BackgroundThread：事件的处理会在一个后台线程中执行，对应的函数名是onEventBackgroundThread，虽然名字是BackgroundThread，事件处理是在后台线程，但事件处理时间还是不应该太长，因为如果发送事件的线程是后台线程，会直接执行事件，如果当前线程是UI线程，事件会被加到一个队列中，由一个线程依次处理这些事件，如果某个事件处理时间太长，会阻塞后面的事件的派发或处理。
* Async：事件处理会在单独的线程中执行，主要用于在后台线程中执行耗时操作，每个事件会开启一个线程（有线程池），但最好限制线程的数目。

根据事件订阅都函数名称的不同，会使用不同的ThreadMode，比如果在后台线程加载了数据想在UI线程显示，订阅者只需把函数命名onEventMainThread。

对相应的函数名，进一步解释一下：

**onEvent**:如果使用onEvent作为订阅函数，那么该事件在哪个线程发布出来的，onEvent就会在这个线程中运行，也就是说发布事件和接收事件线程在同一个线程。使用这个方法时，在onEvent方法中不能执行耗时操作，如果执行耗时操作容易导致事件分发延迟。

**onEventMainThread**:如果使用onEventMainThread作为订阅函数，那么不论事件是在哪个线程中发布出来的，onEventMainThread都会在UI线程中执行，接收事件就会在UI线程中运行，这个在Android中是非常有用的，因为在Android中只能在UI线程中跟新UI，所以在onEvnetMainThread方法中是不能执行耗时操作的。

**onEventBackground**:如果使用onEventBackgrond作为订阅函数，那么如果事件是在UI线程中发布出来的，那么onEventBackground就会在子线程中运行，如果事件本来就是子线程中发布出来的，那么onEventBackground函数直接在该子线程中执行。

**onEventAsync**：使用这个函数作为订阅函数，那么无论事件在哪个线程发布，都会创建新的子线程在执行onEventAsync。

##基本用法

###引入EventBus:

```
compile 'org.greenrobot:eventbus:3.0.0'
```

定义事件:

```
public class MessageEvent { /* Additional fields if needed */ }
```

注册事件接收者：

```
eventBus.register(this);
```

发送事件:

```
eventBus.post(event)
```

接收消息并处理:

```
// 3.0后不再要求事件以 onEvent 开头，而是采用注解的方式
@Subscribe(threadMode = ThreadMode.MAIN)
public void receive(MessageEvent event){}
```

注销事件接收：

```
eventBus.unregister(this);
```

索引加速：

```
3.0 后引入了索引加速(默认不开启)的功能，即通过 apt 编译插件的方式，在代码编译的时候对注解进行索引，避免了以往通过反射造成的性能损耗。
如何使用可以参考[官方文档](http://greenrobot.org/eventbus/documentation/subscriber-index/)
```

最后，proguard 需要做一些额外处理:

```
#EventBus
 -keepclassmembers class ** {
    public void onEvent*(**);
    void onEvent*(**);
 }
```



