#《Android开发艺术探索》第三章笔记
---
##View的基础知识

* 什么是View

View是Android中所有控件的基类，View是一种界面层的控件的一种抽象，它代表了一个控件，在Android设计中，ViewGroup也继承了View，这就意味着View本身就可以是单个控件也可以是多个控件组成的一组控件，通过这种关系就形成了View树的结构。

* View的位置参数

view的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom，其中top是左上角纵坐标，left是左上角横坐标，right是右下角横坐标，bottom是右下角纵坐标
View的宽高和坐标的关系：

width = right - left;

height = bottom - top;

如何得到这四个参数：

Left = getLeft();

Right = getRight();

Top = getTop();

Bottom = getBottom();

从Android 3.0开始，view增加了x、y、translationX、translationY四个参数，这几个参数也是相对于父容器的坐标。x和y是左上角的坐标，而translationX和translationY是view左上角相对于父容器的偏移量，默认值都是0。

x = left + translationX

y = top + translationY

* MotionEvent和TouchSlop

MotionEvent:

在手指触摸屏幕后所产生的一系列事件中，典型的时间类型有：

1、ACTION_DOWN-手指刚接触屏幕

2、ACTION_MOVE-手指在屏幕上移动

3、ACTION_UP-手机从屏幕上松开的一瞬间

正常情况下，一次手指触摸屏幕的行为会触发一系列点击事件，考虑如下几种情况：

1、点击屏幕后离开松开，事件序列为 DOWN -> UP

2、点击屏幕滑动一会再松开，事件序列为DOWN->MOVE->...->UP

通过MotionEvent对象我们可以得到点击事件发生的x和y坐标，getX/getY返回的是相对于当前View左上角的x和y坐标，getRawX和getRawY是相对于手机屏幕左上角的x和y坐标。

TouchSlop:

TouchSlope是系统所能识别出的可以被认为是滑动的最小距离，获取方式是ViewConfiguration.get(getContext()).getScaledTouchSlope()。

* VelocityTracker、GestureDetector和Scroller

1、 VelocityTracker：用于追踪手指在滑动过程中的速度，包括水平和垂直方向上的速度。

VelocityTracker的使用方式：



	//初始化
	VelocityTracker mVelocityTracker = VelocityTracker.obtain();

	//在onTouchEvent方法中
	mVelocityTracker.addMovement(event);

	//获取速度
	mVelocityTracker.computeCurrentVelocity(1000);

	float xVelocity = mVelocityTracker.getXVelocity();
	//重置和回收

	mVelocityTracker.clear(); //一般在MotionEvent.ACTION_UP的时候调用

	mVelocityTracker.recycle(); //一般在onDetachedFromWindow中调用
	
	





速度的计算公式：

速度 = （终点位置 - 起点位置） / 时间段

速度可能为负值，例如当手指从屏幕右边往左边滑动的时候。此外，速度是单位时间内移动的像素数，单位时间不一定是1秒钟，可以使用方法computeCurrentVelocity(xxx)指定单位时间是多少，单位是ms。例如通过computeCurrentVelocity(1000)来获取速度，手指在1s中滑动了100个像素，那么速度是100，即100(像素/1000ms)。如果computeCurrentVelocity(100)来获取速度，在100ms内手指只是滑动了10个像素，那么速度是10，即10(像素/100ms)。

当不需要的时候，需要调用clear方法来重置并回收内存



	velocityTracker.clear();
	velocityTracker.recycler();
	
	


2、GestureDetector

手势检测，用于辅助检测用户的点击、滑动、长按、双击等行为。
 
在日常开发中，比较常用的有:onSingleTapUp(单击)、onFling(快速滑动)、onScroll（拖动）、onLongPress(长按)、onDoubleTap(双击)，建议：如果只是监听滑动相关的事件在onTouchEvent中实现；如果要监听双击这种行为的话，那么就使用GestureDetector。

3、Scroller

弹性滑动对象，用于实现View的弹性滑动。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能共同完成这个功能。


##View的滑动

通过三种方式可以实现View的滑动

* 第一种是通过View本身提供的scrollTo/scrollBy方法来实现滑动
* 第二种是通过动画给View施加平移效果来实现滑动
* 通过改变View的LayoutParams使得View重新布局从而实现滑动

1、使用scrollTo/scrollBy
scrollTo和scrollBy方法只能改变view内容的位置而不能改变view在布局中的位置。 scrollBy是基于当前位置的相对滑动，而scrollTo是基于所传参数的绝对滑动。通过View的getScrollX和getScrollY方法可以得到滑动的距离。

2、使用动画
使用动画来移动view主要是操作view的translationX和translationY属性，既可以使用传统的view动画，也可以使用属性动画，使用后者需要考虑兼容性问题，如果要兼容Android3.0一下版本系统的话推荐使用nineoldandroids。使用动画还存在一个交互问题：在android3.0以前的系统上，view动画和属性动画，新位置均无法触发点击事件，同时，老位置仍然可以触发单击事件。从3.0开始，属性动画的单击事件触发位置为移动后的位置，view动画仍然在原位置。

3、改变布局参数
通过改变LayoutParams的方式去实现View的滑动是一种灵活的方法。

4、各种滑动方式的对比

* scrollTo/scrollBy:操作简单，适合对View内容的滑动
* 动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果
* 改变布局参数：操作稍微复杂，适用于有交互的View

动画兼容库nineoldandroids中的ViewHelper类提供了很多的get/set方法来为属性动画服务，例如setTranslationX和setTranslationY方法，这些方法是没有版本要求的。

##弹性滑动

1、使用Scroller
Scroller的工作原理：Scroller本身并不能实现view的滑动，它需要配合view的computeScroll方法才能完成弹性滑动的效果，它不断地让view重绘，而每一次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔Scroller就可以得出view的当前的滑动位置，知道了滑动位置就可以通过scrollTo方法来完成view的滑动。就这样，view的每一次重绘都会导致view进行小幅度的滑动，而多次的小幅度滑动就组成了弹性滑动，这就是Scroller的工作原理。

2、通过动画
采用这种方法除了能完成弹性滑动以外，还可以实现其他动画效果，我们完全可以在onAnimationUpdate方法中加上我们想要的其他操作。

3、使用延时策略
使用延时策略来实现弹性滑动，它的核心思想是通过发送一系列延时消息从而达到一种渐进式的效果，具体来说可以使用Handler的sendEmptyMessageDelayed(xxx)或view的postDelayed方法，也可以使用线程的sleep方法。

##View的事件分发机制

1、事件分发机制的三个重要方法

* public boolean dispatchTouchEvent(MotionEvent ev)

用来进行事件的分发。如果事件能够传递给当前的View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

* public boolean onInterceptTouchEvent(MotionEvent event)

在上述方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

* public boolean onTouchEvent(MotionEvent event)

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前的事件，如果不消耗，则在同一个事件序列中，当前View无法再次接受到事件。

这三个方法的关系可以用如下伪代码表示：


	public boolean dispatchTouchEvent(MotionEvent event)
	{
		boolean consume = false;
		if(onInterceptTouchEvent(ev))
		{
			consume = onTouchEvent(ev);
		}
		else
		{
			consume = child.dispatchTouchEvent(ev);
		}
		return consume;
	}
	


我们可以大致了解点击事件的传递规则：对于一个根ViewGroup来说，点击事件产生后，首先会传递给它，这时它的dispatchTouchEvent会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true就表示它要拦截当前事件，接着事件就会交给这个ViewGroup处理，即它的onTouchEvent方法就会被调用；如果这个ViewGroup的onInterceptTouchEvent方法返回false就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被最终处理。

####OnTouchListener的优先级比onTouchEvent要高

如果给一个view设置了OnTouchListener，那么OnTouchListener中的onTouch方法会被回调。这时事件如何处理还要看onTouch的返回值，如果返回false，那么当前view的onTouchEvent方法会被调用；如果返回true，那么onTouchEvent方法将不会被调用。
在onTouchEvent方法中，如果当前view设置了OnClickListener，那么它的onClick方法会被调用，所以OnClickListener的优先级最低。

当点击一个事件产生后，它的传递过程遵循如顺序，Activity->Window->View

如果一个View的onTouchEvent方法返回false，那么它的父容器的onTouchEvent方法将会被调用，依次类推，如果所有的元素都不处理这个事件，那么这个事件将会最终传递给Activity处理（调用Activity的onTouchEvent方法）

关于事件传递的机制，给出一些结论：

* 同一个事件序列是以down事件开始，中间含有数量不定的move事件，最终以up事件结束
* 正常情况下，一个事件序列只能被一个View拦截且消耗。一旦一个元素拦截了某次事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理
* 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件，那么同一事件序列的其他事情都不会再交给它来处理，并且事件将重新交给它的父容器去处理（调用父容器的onTouchEvent方法）；如果它消耗ACTION_DOWN事件，但是不消耗其他类型事件，那么这个点击事件会消失，父容器的onTouchEvent方法不会被调用，当前view依然可以收到后续的事件，但是这些事件最后都会传递给Activity处理。
* ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouchEvent方法默认返回false，View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会调用。
* View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable和longClickable同时为false）。View的longClickable属性默认都为false，clickable要分情况，比如Button的clickable属性默认为true，而TextView的clickable属性默认为false。
* View的enable属性不影响onTouchEvent的默认返回值，哪怕一个View是disable状态的，只要它的clickable或者longClickable有一个为true，那么它的onTouchEvent就返回true
* 事件传递过程总是先传递给父元素，然后再由父元素分发给子view，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外，即当面对ACTION_DOWN事件时，ViewGroup总是会调用自己的onInterceptTouchEvent方法来询问自己是否要拦截事件。

##View的滑动冲突

1、常见的滑动冲突场景

* 外部滑动方向与内部滑动方向不一致，比如ViewPager中包含ListView
* 外部滑动方向与内部滑动方向一致
* 上面两种情况的嵌套

2、滑动冲突的处理规则

可以根据滑动距离和水平方向形成的夹角；或者根据水平和竖直方向滑动的距离差；或者两个方向上的速度差等。

3、滑动冲突的解决方式

* 外部拦截法

点击事件都经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截，该方法需要重写父容器的onInterceptTouchEvent方法，在内部做相应的拦截即可，伪代码如下：

	public boolean onInterceptTouchEvent(MotionEvent event) {
    	boolean intercepted = false;
    	int x = (int) event.getX();
   		int y = (int) event.getY();

    	switch (event.getAction()) {
    	case MotionEvent.ACTION_DOWN: {
    	    intercepted = false;
    	    break;
    	}
    	case MotionEvent.ACTION_MOVE: {
     	   int deltaX = x - mLastXIntercept;
     	   int deltaY = y - mLastYIntercept;
     	   if (父容器需要拦截当前点击事件的条件，例如：Math.abs(deltaX) > Math.abs(deltaY)) {
       	     intercepted = true;
       	 } else {
        	    intercepted = false;
      	  }
       	 break;
    	}
    	case MotionEvent.ACTION_UP: {
     	   intercepted = false;
        	break;
    	}	
    	default:
        	break;
    	}

    	mLastXIntercept = x;
    	mLastYIntercept = y;

    	return intercepted;
	}

* 内部拦截法

父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就由父容器进行处理，这种方法和Android中的事件分发机制不一样，需要配合requestDisallowInterceptTouchEvent方法才能正常工作。

	public boolean dispatchTouchEvent(MotionEvent event) {
    	int x = (int) event.getX();
    	int y = (int) event.getY();

    	switch (event.getAction()) {
    	case MotionEvent.ACTION_DOWN: {]
    	    getParent().requestDisallowInterceptTouchEvent(true);
        	break;
    	}
    	case MotionEvent.ACTION_MOVE: {
        	int deltaX = x - mLastX;
        	int deltaY = y - mLastY;
        	if (当前view需要拦截当前点击事件的条件，例如：	Math.abs(deltaX) > Math.abs(deltaY)) {
            		getParent().requestDisallowInterceptTouchEvent(false);
        	}
        	break;
    	}
    	case MotionEvent.ACTION_UP: {
        	break;
    	}
    	default:
        	break;
    	}

    	mLastX = x;
    	mLastY = y;
    	return super.dispatchTouchEvent(event);
	}
	
	














