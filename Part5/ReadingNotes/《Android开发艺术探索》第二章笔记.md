##《Android开发艺术探索》第二章笔记
---

##IPC
* Inter-Process Communication的缩写。含义为进程间通信或跨进程通信，是指两个进程之间进行数据交换的过程。

##进程和线程的区别
* 按照操作系统的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。
* 进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。一个进程可以包含多个线程，因此进程和线程是包含与被包含的关系。

##多进程分为两种
* 第一种情况是一个应用因为某些原因自身需要采用多线程模式来实现。
* 另一种情况是当前应用需要向其他应用获取数据

##Android中的多进程模式
通过给四大组件指定android:process属性，我们可以开启多线程模式

* 进程名以":"开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一进程，而进程名不以":"开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。
* Android系统会为每个应用分配一个唯一的UID,具有相同UID的应用才能共享数据，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，比如data目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程中，那么除了能共享data目录、组件信息，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

##多进程模式的运行机制
*  Android为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配了一个独立的虚拟机，不同的虚拟机在不同的内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。
* 所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败。

一般来说，使用多进程会造成如下几个方面的问题：

* 静态成员和单例模式完全失效
* 线程同步机制完全失效


不管是锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象

* SharedPreference的可靠性下降

SharedPreferences不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这时因为SharedPreferences底层是通过读写XML文件来实现的，并发写显然是可能出问题的，甚至并发读写都有可能发生问题

* Application会多次创建

运行在同一个进程中的组件是属于同一个虚拟机和同一个Application的。同理，运行在不同进程中的组件是属于两个不同的虚拟机和Application的。

##IPC基础概念介绍
* Serializable

是Java所提供的一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。使用Serializable来实现序列化相当简单，只需要在类的声明中指定一个类似下面的标识即可自动实现默认的序列化过程。

```

private static final long serialVersionUID = 8711368828010083044L

```
通过Serializable方来实现对象的序列化，如下代码：
```

//序列化过程
User user = new User(0, "jake", true);
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cache.txt"));
out.writeObject(user);
out.close();

//反序列化过程
ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt"));
User newUser = (User)in.readObject();
in.close();

```

原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID相同时才能够正常的被反序列化。serialVersionUID的详细工作机制是这样的：序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也可能是其他的中介），当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的类的版本和当前类的版本是相同的，这个时候可以成功反序列化，否则就说明当前类和序列化的类相比发生了某些变换

给serialVersionUID制定为1L或者采用Eclipse根据当前类结构去生成的hash值，这两者并没有本质区别。

*  静态成员变量属于类不属于对象，所以不会参与序列化过程
*  其次用transient关键字标记的成员变量不参与序列化过程


##Parcelable接口
Parcelable也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过Intent的Binder传递
Parcelable的方法说明：

| 方法        | 功能           | 标记位  |
| ------------- |:-------------:| -----:|
| createFromParcel(Parcel in)      | 从序列化的对象中创建原始对象 |  |
| newArray[int size]      | 创建指定长度的原始对象数组      |    |
| User(Parcel in) |  从序列化的对象中创建原始对象      |     |
| write ToParcel(Parcel out, int flags) | 将当前对象写入序列化结构中，其中flags标识有两种值0或1（参见右侧标记位）。为1时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况都为0      |    PARCELABLE_WRITE_RETURN_VALUE |
| describeContents | 返回当前对象的内容描述。如果含有文件描述符，返回1（参见右侧标记位），否则返回0，几乎所有的情况都返回0      |    CONTENTS_FILE_DESCRIPTOR |


* 系统已经为我们提供了许多实现了Parcelable接口的类，它们都是可以直接序列化的，比如Intent、Bundle、Bitmap等，同时List和Map也可以序列化，前提是它们里面的每个元素都是可序列化的。

##如何选取
Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列化需要大量I/O操作。而Parceleble是Android中的序列化方式，因此更适合在Android平台上，缺点是麻烦，但是效率高，这是Android推荐的序列化方式，所以我们要首选Parcelable。Parcelable主要用在内存序列化上，通过Parcelable将对象序列化到存储设备中或者将对象序列化之后通过网络传输，但是过程稍显复杂，因此在这两种情况下建议大家使用Serializable。


##Binder
* 继承了IBinder接口
* Binder是一种跨进程通信方式
* 是ServiceManager连接各种Manager（ActivityManager,WindowManager等）和相应ManagerService的桥梁
* 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务器会返回一个包含了服务器端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者是数据，这里的服务包含了普通服务和基于AIDL的服务

aidl工具根据aidl文件自动生成的java接口的解析：首先，它声明了几个接口方法，同时还声明了几个整型的id用于标识这些方法，id用于标识在transact过程中客户端所请求的到底是哪个方法；接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub内部的代理类Proxy来完成。
所以，这个接口的核心就是它的内部类Stub和Stub内部的代理类Proxy。 下面分析其中的方法：

* asInterface(android.os.IBinder obj):用于将服务器端的Binder对象转化成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端是在同一进程中，那么这个方法返回的是服务端的Stub对象本身，否则返回的是系统封装的Stub.Proxy对象。
* asBinder:返回当前Binder对象
* onTransact：这个方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。
这个方法的原型是public Boolean onTransact(int code, Parcelable data, Parcelable reply, int flags)
服务端通过code可以知道客户端请求的目标方法，接着从data中取出所需的参数，然后执行目标方法，执行完毕之后，将结果写入到reply中。如果此方法返回false，说明客户端的请求失败，利用这个特性可以做权限验证(即验证是否有权限调用该服务)。
* Proxy#[Method]：代理类中的接口方法，这些方法运行在客户端，当客户端远程调用此方法时，它的内部实现是：首先创建该方法所需要的参数，然后把方法的参数信息写入到_data中，接着调用transact方法来发起RPC请求，同时当前线程挂起；然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，最后返回_reply中的数据。

######首先，当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以如果一个远程方法是很耗时的，那么不能在UI线程发起此远程请求；其次，由于服务端的Binder方法运行在Binder的线程池中，所以不管Binder是否耗时都应该采用同步的方式去实现，因为它已经运行在一个线程中了。

###Binder两种重要的方法
1. linkToDeath
2. unlinkToDeath
Binder运行在服务端，如果由于某种服务端异常终止了的话会导致客户端的远程调用失败、所以Binder提供了两个配对的方法linkToDeath和unlinkToDeath，通过linkToDeath方法可以给Binder设置一个死亡代理，当Binder死亡的时候客户端就会收到通知，然后就可以重新发起连接从而恢复连接了。
###如何给Binder设置死亡代理
1、声明一个DeathRecipient对象、DeathRecipient是一个接口，其内部只有一个方法bindDied，实现这个方法就可以在Binder死亡的时候收到通知了。

```

private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        if (mRemoteBookManager == null) return;
        mRemoteBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
        mRemoteBookManager = null;
        // TODO:这里重新绑定远程Service
    }
};

```

2、在客户端绑定远程服务成功之后，给binder设置死亡代理

```

mRemoteBookManager.asBinder().linkToDeath(mDeathRecipient, 0);

```

##Android的IPC方式

1、 使用Bundle

Bundle实现了Parcelable接口，Activity、Service和Receiver都支持在Intent中传递Bundle数据

2、 使用文件共享

这种方式简单，适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读写的问题，SharedPreferences是一个特例，虽然它也是文件的一种，但是由于系统对它的读写有一定的缓存策略，即在内存中会有一份SharedPreferences文件的缓存，因此在多进程模式下、系统对它的读写就变的不可靠，当面对高并发读写访问的时候，有很大几率会丢失，因此，不建议在进程间通信中使用SharedPreferences。

3、 使用Messenger

Messenger是一种轻量级的IPC方案，它的底层实现就是AIDL。Messenger是以串行的方式处理请求的，即服务端只能一个个处理，不存在并发执行的情形。

4、 使用AIDL

大致流程：首先建一个Service和一个AIDL接口，接着创建一个类继承自AIDL接口中的Stub类中的抽象方法，在Service的onBind方法中返回这个类的对象，然后客户端就可以绑定服务端Service，建立连接后就可以访问远程服务端的方法了。
1.AIDL支持的数据类型：基本数据类型、String和CharSequence、ArrayList、HashMap、Parcelable以及AIDL；
2.某些类即使和AIDL文件在同一个包中也要显式import进来；
3.AIDL中除了基本数据类，其他类型的参数都要标上方向：in、out或者inout；
4.AIDL接口中支持方法，不支持声明静态变量；
5.为了方便AIDL的开发，建议把所有和AIDL相关的类和文件全部放入同一个包中，这样做的好处是，当客户端是另一个应用的时候，可以直接把整个包复制到客户端工程中。
6.RemoteCallbackList是系统专门提供的用于删除跨进程Listener的接口。RemoteCallbackList是一个泛型，支持管理任意的AIDL接口，因为所有的AIDL接口都继承自IInterface接口。

5、使用ContentProvider

1.ContentProvider主要以表格的形式来组织数据，并且可以包含多个表；
2.ContentProvider还支持文件数据，比如图片、视频等，系统提供的MediaStore就是文件类型的ContentProvider；
3.ContentProvider对底层的数据存储方式没有任何要求，可以是SQLite、文件，甚至是内存中的一个对象都行；
4.要观察ContentProvider中的数据变化情况，可以通过ContentResolver的registerContentObserver方法来注册观察者；

6、使用Socket

套接字，分为流式套接字和用户数据报套接字两种，分别对应于网络的传输控制层中TCP和UDP协议。

* TCP协议是面向连接的协议，提供稳定的双向通信功能，TCP连接的建立需要经过"三次握手"才能完成，为了提供稳定的数据传输功能，其本身提供了超时重传功能，因此具有很高的稳定性
* UDP是无连接的，提供不稳定的单向通信功能，当然UDP也可以实现双向通信功能，在性能上，UDP具有更好的效率，其缺点是不保证数据能够正确传输，尤其是在网络拥塞的情况下。

##Binder连接池
* 当项目规模很大的时候，创建很多个Service是不对的做法，因为service是系统资源，太多的service会使得应用看起来很重，所以最好是将所有的AIDL放在同一个Service中去管理。整个工作机制是：每个业务模块创建自己的AIDL接口并实现此接口，这个时候不同业务模块之间是不能有耦合的，所有实现细节我们要单独开来，然后向服务端提供自己的唯一标识和其对应的Binder对象；对于服务端来说，只需要一个Service，服务端提供一个queryBinder接口，这个接口能够根据业务模块的特征来返回相应的Binder对象给它们，不同的业务模块拿到所需的Binder对象后就可以进行远程方法调用了。
Binder连接池的主要作用就是将每个业务模块的Binder请求统一转发到远程Service去执行，从而避免了重复创建Service的过程。
* 建议在AIDL开发工作中引入BinderPool机制。

##选用合适的IPC方式
![Mou icon](http://hujiaweibujidao.github.io/images/androidart_ipc.png)






















