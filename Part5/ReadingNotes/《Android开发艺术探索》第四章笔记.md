##View的工作原理
---

###4.1 初识ViewRoot和DecorView

1、ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立连接。

2、View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure、layout、draw三个过程才能最终将一个View绘制出来，其中measure用来测量View的宽和高，layout用来确定View在父容器的放置位置，而draw则负责将View绘制在屏幕上。

3、performTraversals会依次调用performMeasure、performLayout、performDraw三个方法，这三个方法分别完成顶级View的measure、layout和draw这三大流程，其中performMeasure会调用measure方法，在measure方法中又会调用onMeasure方法，在onMeasure方法中对所有的子元素进行measure过程，这个时候measure流程就会从父容器传递到子元素中了，这样就完成了一次measure过程。接着子元素就会重复父容器的measure过程，如此反复就完成了整个View树的遍历。

4、measure过程中决定了View的宽/高，Measure完成以后，可以通过getMeasureWidth和getMeasureHeight方法来获取到View测量后的宽/高，在几乎所有的情况下它都等同于View最终的宽高。layout决定了View的四个顶点的坐标和View的实际的宽高，通过getWidth和getHeight方法可以获得最终的宽高。draw过程决定了View的显示。

5、DecorView其实是一个FrameLayout，其中包含了一个竖直方向的LinearLayout，上面是标题栏，下面是内容栏（id为android.R.id.content）。View层的事件都先经过DecorView，然后才传给我们的View。

###理解MeasureSpec

1、MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的内存分配，为了方便操作，其提供了打包和解包方法。SpecMode和SpecSize也是一个int值，一组SpecMode和SpecSize可以打包为一个MeasureSpec，而一个MeasureSpec可以通过解包的形式来得出其原始的SpecMode和SpecSize。

SpecMode有三类，每一类都表示特殊的含义：

1. UNSPECIFIED   父容器不对View有任何的限制，要多大给多大，这种情况下一般用于系统内部，表示一种测量的状态。
2. EXACTLY   父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值，它对应于LayoutParams中的match_parent和具体的数值这两种模式
3. AT_MOST   父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。它对应于LayoutParams中的wrap_content

2、MeasureSpec和LayoutParams的对应关系

在View测量的时候系统会将LayoutParams在父容器的约束下转换成对应的MeasureSpec，然后再根据这个MeasureSpec来确定View测量后的宽高。

MeasureSpec不是唯一由LayoutParams决定的，LayoutParams需要和父容器一起才能决定View的MeasureSpec，从而进一步确定View的宽高。对于DecorView，它的MeasureSpec由窗口的尺寸和其自身的LayoutParams来决定；对于普通View，它的MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定

3、当view采用固定宽高时，不管父容器的MeasureSpec是什么，view的MeasureSpec都是精确模式，并且大小是LayoutParams中的大小。
当view的宽高是match_parent时，如果父容器的模式是精确模式，那么view也是精确模式，并且大小是父容器的剩余空间；如果父容器是最大模式，那么view也是最大模式，并且大小是不会超过父容器的剩余空间。
当view的宽高是wrap_content时，不管父容器的模式是精确模式还是最大模式，view的模式总是最大模式，并且大小不超过父容器的剩余空间。

###4.3 View的工作流程

1、View的measure过程和Activity的生命周期方法不是同步执行的，因此无法保证Activity执行了onCreate、onStart、onResume时某个View已经测量完毕了。如果View还没有测量完毕，那么获得的宽和高都是0。下面是四种解决该问题的方法：

* Activity/View#onWindowsChanged方法

onWindowFocusChanged方法表示View已经初始化完毕了，宽高已经准备好了，这个时候去获取是没问题的。这个方法会被调用多次，当Activity继续执行或者暂停执行的时候，这个方法都会被调用。

* View.post(runnable)

通过post将一个Runnable投递到消息队列的尾部，然后等待Looper调用此runnable的时候，View也已经初始化好了。

* ViewTreeObsever

使用ViewTreeObserver的众多回调方法可以完成这个功能，比如使用onGlobalLayoutListener接口，当View树的状态发生改变或者View树内部的View的可见性发生改变时，onGlobalLayout方法将被回调。伴随着View树的变化，这个方法也会被多次调用。

* view.measure(int widthMeasureSpec, int heightMeasureSpec)

通过手动对View进行measure来得到View的宽高，这个要根据View的LayoutParams来处理：

match_parent:无法measure出具体的宽高

wrap_content:如下measure，设置最大值

```
int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1 << 30) - 1, MeasureSpec.AT_MOST);

int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1 << 30) - 1, MeasureSpec.AT_MOST);

view.measure(widthMeasureSpec, heightMeasureSpec);
```



精确值：例如100px

```

int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100, MeasureSpec.EXACTLY);

int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100, MeasureSpec.EXACTLY);

view.measure(widthMeasureSpec, heightMeasureSpec);
```


2、在View的默认实现中，View的测量宽高和最终宽高时相等的，只不过测量宽高形成于measure过程，而最终宽高形成于layout过程。

3、draw过程大概有下面几步：

1. 绘制背景：background.draw(canvas);
2. 绘制自己：onDraw();
3. 绘制children：dispatchDraw;
4. 绘制装饰：onDrawScrollBars

###4.4自定义View

作者将自定义View分为以下4类：

1. 继承view重写onDraw方法
2. 继承ViewGroup派生特殊的Layout
3. 继承特定的View(比如TextView)
4. 继承特殊的ViewGroup(比如LinearLayout)

自定义View须知：

1. 让View支持wrap_content
2. 如果有必要，让你的View支持padding
3. 尽量不要在View中使用Handler，没必要
4. View中如果有线程或者动画，需要及时停止，参考View#onDetachedFromWindow
5. View带有滑动嵌套情形时，需要处理好滑动冲突











