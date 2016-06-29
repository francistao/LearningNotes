#Fragment

##为何产生
* 同时适配手机和平板、UI和逻辑的共享。

##介绍
* Fragment也会被加入回退栈中。
* Fragment拥有自己的生命周期和接受、处理用户的事件
* 可以动态的添加、替换和移除某个Fragment

##生命周期
* 必须依存于Activity

![Mou icon](http://img.blog.csdn.net/20140719225005356?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbG1qNjIzNTY1Nzkx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* Fragment依附于Activity的生命状态
* ![Mou icon](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/FlowchartDiagram.jpg?raw=true)

生命周期中那么多方法，懵逼了的话我们就一起来看一下每一个生命周期方法的含义吧。

##Fragment生命周期方法含义：
* `public void onAttach(Context context)`
	* onAttach方法会在Fragment于窗口关联后立刻调用。从该方法开始，就可以通过Fragment.getActivity方法获取与Fragment关联的窗口对象，但因为Fragment的控件未初始化，所以不能够操作控件。
	
* `public void onCreate(Bundle savedInstanceState)`
	* 在调用完onAttach执行完之后立刻调用onCreate方法，可以在Bundle对象中获取一些在Activity中传过来的数据。通常会在该方法中读取保存的状态，获取或初始化一些数据。在该方法中不要进行耗时操作，不然窗口不会显示。
	
* `public View onCreateView(LayoutInflater inflater, ViewGroup container,Bundle savedInstanceState)`
	* 该方法是Fragment很重要的一个生命周期方法，因为会在该方法中创建在Fragment显示的View，其中inflater是用来装载布局文件的，container是`<fragment>`标签的父标签对应对象，savedInstanceState参数可以获取Fragment保存的状态，如果未保存那么就为null。
	
* `public void onViewCreated(View view,Bundle savedInstanceState)`
	* Android在创建完Fragment中的View对象之后，会立刻回调该方法。其种view参数就是onCreateView中返回的view，而bundle对象用于一般用途。
	
* `public void onActivityCreated(Bundle savedInstanceState)`
	* 在Activity的onCreate方法执行完之后，Android系统会立刻调用该方法，表示窗口已经初始化完成，从这一个时候开始，就可以在Fragment中使用getActivity().findViewById(Id);来操控Activity中的view了。
	
* `public void onStart()`
	* 这个没啥可讲的，但有一个细节需要知道，当系统调用该方法的时候，fragment已经显示在ui上，但还不能进行互动，因为onResume方法还没执行完。
	
* `public void onResume()`
	* 该方法为fragment从创建到显示Android系统调用的最后一个生命周期方法，调用完该方法时候，fragment就可以与用户互动了。

* `public void onPause()`
	* fragment由活跃状态变成非活跃状态执行的第一个回调方法，通常可以在这个方法中保存一些需要临时暂停的工作。如保存音乐播放进度，然后在onResume中恢复音乐播放进度。

* `public void onStop()`
	* 当onStop返回的时候，fragment将从屏幕上消失。

* `public void onDestoryView()`
	* 该方法的调用意味着在 `onCreateView` 中创建的视图都将被移除。

* `public void onDestroy()`
	* Android在Fragment不再使用时会调用该方法，要注意的是~这时Fragment还和Activity藕断丝连！并且可以获得Fragment对象，但无法对获得的Fragment进行任何操作（呵~呵呵~我已经不听你的了）。

* `public void onDetach()`
	* 为Fragment生命周期中的最后一个方法，当该方法执行完后，Fragment与Activity不再有关联(分手！我们分手！！(╯‵□′)╯︵┻━┻)。

##Fragment比Activity多了几个额外的生命周期回调方法：
* onAttach(Activity):当Fragment和Activity发生关联时使用
* onCreateView(LayoutInflater, ViewGroup, Bundle):创建该Fragment的视图
* onActivityCreate(Bundle):当Activity的onCreate方法返回时调用
* onDestoryView():与onCreateView相对应，当该Fragment的视图被移除时调用
* onDetach():与onAttach相对应，当Fragment与Activity关联被取消时调用

###注意：除了onCreateView，其他的所有方法如果你重写了，必须调用父类对于该方法的实现

##Fragment与Activity之间的交互
* Fragment与Activity之间的交互可以通过`Fragment.setArguments(Bundle args)`以及`Fragment.getArguments()`来实现。

##Fragment状态的持久化。
由于Activity会经常性的发生配置变化，所以依附它的Fragment就有需要将其状态保存起来问题。下面有两个常用的方法去将Fragment的状态持久化。

* 方法一：
	* 可以通过`protected void onSaveInstanceState(Bundle outState)`,`protected void onRestoreInstanceState(Bundle savedInstanceState)` 状态保存和恢复的方法将状态持久化。
* 方法二(更方便,让Android自动帮我们保存Fragment状态)：
	* 我们只需要将Fragment在Activity中作为一个变量整个保存，只要保存了Fragment，那么Fragment的状态就得到保存了，所以呢.....
		* `FragmentManager.putFragment(Bundle bundle, String key, Fragment fragment)` 是在Activity中保存Fragment的方法。
		* `FragmentManager.getFragment(Bundle bundle, String key)` 是在Activity中获取所保存的Frament的方法。
	* 很显然，key就传入Fragment的id，fragment就是你要保存状态的fragment，但，我们注意到上面的两个方法，第一个参数都是Bundle，这就意味着*FragmentManager*是通过Bundle去保存Fragment的。但是，这个方法仅仅能够保存Fragment中的控件状态，比如说EditText中用户已经输入的文字（*注意！在这里，控件需要设置一个id，否则Android将不会为我们保存控件的状态*），而Fragment中需要持久化的变量依然会丢失，但依然有解决办法，就是利用方法一！
	* 下面给出状态持久化的事例代码：
		```		
			 /** Activity中的代码 **/
			 FragmentB fragmentB;

		    @Override
		    protected void onCreate(@Nullable Bundle savedInstanceState) {
		        super.onCreate(savedInstanceState);
		        setContentView(R.layout.fragment_activity);
		        if( savedInstanceState != null ){
		            fragmentB = (FragmentB) getSupportFragmentManager().getFragment(savedInstanceState,"fragmentB");
		        }
		        init();
		    }

		    @Override
		    protected void onSaveInstanceState(Bundle outState) {
		        if( fragmentB != null ){
		            getSupportFragmentManager().putFragment(outState,"fragmentB",fragmentB);
		        }

		        super.onSaveInstanceState(outState);
		    }

		    /** Fragment中保存变量的代码 **/

		    @Nullable
		    @Override
		    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
		        AppLog.e("onCreateView");
		        if ( null != savedInstanceState ){
		            String savedString = savedInstanceState.getString("string");
		            //得到保存下来的string
		        }
		        View root = inflater.inflate(R.layout.fragment_a,null);
		        return root;
		    }

			@Override
		    public void onSaveInstanceState(Bundle outState) {
		        outState.putString("string","anAngryAnt");
		        super.onSaveInstanceState(outState);
		    }

		```

##静态的使用Fragment
1. 继承Fragment，重写onCreateView决定Fragment的布局
2. 在Activity中声明此Fragment,就和普通的View一样

##Fragment常用的API
* android.support.v4.app.Fragment 主要用于定义Fragment
* android.support.v4.app.FragmentManager 主要用于在Activity中操作Fragment，可以使用FragmentManager.findFragmenById，FragmentManager.findFragmentByTag等方法去找到一个Fragment
* android.support.v4.app.FragmentTransaction 保证一些列Fragment操作的原子性，熟悉事务这个词
* 主要的操作都是FragmentTransaction的方法
(一般我们为了向下兼容，都使用support.v4包里面的Fragment)
<code>

	getFragmentManager() // Fragment若使用的是support.v4包中的，那就使用getSupportFragmentManager代替
	
</code>

* 主要的操作都是FragmentTransaction的方法

```
	
	FragmentTransaction transaction = fm.benginTransatcion();//开启一个事务
	transaction.add() 
	//往Activity中添加一个Fragment

	transaction.remove() 
	//从Activity中移除一个Fragment，如果被移除的Fragment没有添加到回退栈（回退栈后面会详细说），这个Fragment实例将会被销毁。

	transaction.replace()
	//使用另一个Fragment替换当前的，实际上就是remove()然后add()的合体~

	transaction.hide()
	//隐藏当前的Fragment，仅仅是设为不可见，并不会销毁

	transaction.show()
	//显示之前隐藏的Fragment

	detach()
	//当fragment被加入到回退栈的时候，该方法与*remove()*的作用是相同的，
	//反之，该方法只是将fragment从视图中移除，
	//之后仍然可以通过*attach()*方法重新使用fragment，
	//而调用了*remove()*方法之后，
	//不仅将Fragment从视图中移除，fragment还将不再可用。

	attach()
	//重建view视图，附加到UI上并显示。

	transatcion.commit()
	//提交一个事务

```

##管理Fragment回退栈
* 跟踪回退栈状态
	* 我们通过实现*``OnBackStackChangedListener``*接口来实现回退栈状态跟踪，具体如下
 	```
 	public class XXX implements FragmentManager.OnBackStackChangedListener 

 	/** 实现接口所要实现的方法 **/

	@Override
    public void onBackStackChanged() {
        //do whatevery you want
    }

    /** 设置回退栈监听接口 **／
    getSupportFragmentManager().addOnBackStackChangedListener(this);


 	```
		
* 管理回退栈
	* ``FragmentTransaction.addToBackStack(String)`` *--将一个刚刚添加的Fragment加入到回退栈中*
	* ``getSupportFragmentManager().getBackStackEntryCount()`` *－获取回退栈中实体数量*
	* ``getSupportFragmentManager().popBackStack(String name, int flags)`` *－根据name立刻弹出栈顶的fragment*
	* ``getSupportFragmentManager().popBackStack(int id, int flags)`` *－根据id立刻弹出栈顶的fragment*















