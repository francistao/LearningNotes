#Fragment

##为何产生
* 同时适配手机和平板



##介绍
* Fragment拥有自己的生命周期和接受、处理用户的事件
* 可以动态的添加、替换和移除某个Fragment

##生命周期
* 必须依存于Activity


![Mou icon](http://img.blog.csdn.net/20140719225005356?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbG1qNjIzNTY1Nzkx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##Fragment比Activity多了几个额外的生命周期回调方法：
* onAttach(Activity):当Fragment和Activity发生关联时使用
* onCreateView(LayoutInflater, ViewGroup, Bundle):创建该Fragment的视图
* onActivityCreate(Bundle):当Activity的onCreate方法返回时调用
* onDestoryView():与onCreateView相对应，当该Fragment的视图被移除时调用
* onDetach():与onAttach相对应，当Fragment与Activity关联被取消时调用

###注意：除了onCreateView，其他的所有方法如果你重写了，必须调用父类对于该方法的实现

##静态的使用Fragment
1. 继承Fragment，重写onCreateView决定Fragment的布局
2. 在Activity中声明此Fragment,就和普通的View一样

##Fragment常用的API
* android.app.Fragment 主要用于定义Fragment
* android.app.FragmentManager 主要用于在Activity中操作Fragment
* android.app.FragmentTransaction 保证一些列Fragment操作的原子性，熟悉事务这个词
* 主要的操作都是FragmentTransaction的方法

<code>

	getFragmentManager() // v4中，getSupportFragmentManager
	
</code>

* 主要的操作都是FragmentTransaction的方法

<code>
	
	FragmentTransaction transaction = fm.benginTransatcion();//开启一个事务
	transaction.add() 
	往Activity中添加一个Fragment
	transaction.remove() 
	从Activity中移除一个Fragment，如果被移除的Fragment没有添加到回退栈（回退栈后面会详细说），这个Fragment实例将会被销毁。
	transaction.replace()
	使用另一个Fragment替换当前的，实际上就是remove()然后add()的合体~
	transaction.hide()
	隐藏当前的Fragment，仅仅是设为不可见，并不会销毁
	transaction.show()
	显示之前隐藏的Fragment
	detach()
	会将view从UI中移除,和remove()不同,此时fragment的状态依然由	FragmentManager维护。
	attach()
	重建view视图，附加到UI上并显示。
	transatcion.commit()//提交一个事务

</code>

##如何添加一个Fragment事务到回退栈

<code>

	FragmentTransaction.addToBackStack(String)

</code>
















