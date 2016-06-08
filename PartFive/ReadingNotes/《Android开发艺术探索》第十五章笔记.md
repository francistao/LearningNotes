#Android性能优化
---
Android不可能无限制的使用内存和CPU资源，过多的使用内存会导致内存溢出，即OOM。而过多的使用CPU资源，一般是指做大量的耗时任务，会导致手机变的卡顿甚至出现程序无法响应的情况，即ANR。

###15.1.1布局优化

1、如何进行布局优化？

* 首先删除布局中无用的控件和层级
* 其次有选择的使用性能较低的ViewGroup。
* 布局优化的另一种手段是采用<include>标签、<merge>标签、ViewStub。<include>标签主要用于布局重用，<merge>标签一般和<include>配合使用，它可以降低减少布局的层级，而ViewStub则提供了按需加载的功能，当需要时才会将ViewStub中的布局加载到内存，提高了程序的初始化效率。

2、<include>标签只支持android:layout_开头的属性，android:id属性例外。

3、ViewStub继承了View，它非常轻量级且宽/高都是0，因此它本身并不参与任何的布局和绘制过程。ViewStub的意义在于按需加载所需的布局文件，在实际开发中，有很多布局文件在正常情况下不会显示，比如网络异常时的界面，这个时候就没有必要在整个界面初始化的时候将其加载起来，通过ViewStub就可以做到在使用的时候再加载，提高了程序初始化的性能。

如下所示，android:id是ViewStub的id，而android:inflatedId是布局的根元素的id。

```
<ViewStub android:id="@+id/xxx"
  android:inflatedId="@+id/yyy"
  android:layout="@layout/zzz"
  ...
</ViewStub>
```

需要加载ViewStub中的布局的时候，可以按照如下两种方式进行：

```
((ViewStub)findViewById(R.id.xx)).setVisibility(View.VISIBLE);
```

或者

```
View importPanel = ((ViewStub)findViewById(R.id.stub_import)).inflate();
```

###15.1.1绘制优化

绘制优化是指View的onDraw方法要避免执行大量的操作：

* 在onDraw中不要创建新的局部对象，这是因为onDraw方法可能会被频繁调用，这样就会在一瞬间产生大量的临时对象，这不仅占用了过多的内存而且还会导致系统更加频繁的gc，降低了程序的执行效率。
* onDraw方法中不要指定耗时任务，也不能执行成千上万次的循环操作，View的绘制帧率保证60fps是最佳的，这就要求每帧的绘制时间不超过16ms，虽然程序很难保证16ms这个时间，但是尽量降低onDraw方法的复杂度总是切实有效的。


###15.1.3内存泄漏优化

可能导致内存泄漏的场景很多，例如静态变量、单例模式、属性动画、AsyncTask、Handler等等


###15.1.4响应速度优化和ANR日志分析

1. ANR出现的情况：Activity如果5秒内没有响应屏幕触摸事件或者键盘输入事件就会ANR。而BroadcastReceiver如果10s没有执行完操作也会出现ANR。
2. 当一个进程发生了ANR之后，系统会在/data/anr目录下创建一个文件traces.txt，通过分析这个文件就能定位ANR的原因。

###15.1.5ListView和Bitmap优化

1. ListView优化：采用ViewHolder并避免在getView方法中执行耗时操作；根据列表的滑动状态来绘制任务的执行效率；可以尝试开启硬件加速期来使ListView的滑动更加流畅。
2. Bitmap优化：根据需要对图片进行采样，主要是通过BitmapFactory.Options来根据需要对图片进行采样，采样主要用到了BitmapFactory.Options的inSampleSize参数。

###15.1.6线程优化

1. 采用线程池，避免程序中存在大量的Thread。线程池可以重用内部的线程，从而避免了线程的创建和销毁所带来的性能开销，同时线程池还能有效的控制线程池的最大并发数，避免大量的线程因互相抢占系统资源从而导致阻塞现象的发生。

###15.1.7一些性能优化建议

* 避免 创建过多的对象
* 不要过多的使用枚举，枚举占用的内存空间要比整形大
* 常量请用static final来修饰
* 使用一些Android特有的数据结构，比如SparseArray和Pair等，它们都具有更好的性能
* 适当使用软引用和弱引用
* 采用内存缓存和磁盘缓存
* 尽量采用静态内部类，这样可以避免潜在的由于内部类而导致的内存泄漏


###15.2内存泄漏分析之MAT工具

MAT是功能强大的内存分析工具，主要有Histograms和Dominator Tree等功能


###15.3提高程序的可维护性

1. 命名要规范，要能正确地传达出变量或者方法的含义，少用缩写，关于变量的前缀可以参考Android源码的命名方式，比如私有成员以m开头，静态成员以s开头，常量则全部用大写字母表示，等等。
2. 代码的排版上需要留出合理的空白来区分不同的代码块，其中同类变量的声明要放在一起，两类变量之间要留出一行空白作为区分。
3. 合理的命名风格，仅在非常关键的代码添加注释。


