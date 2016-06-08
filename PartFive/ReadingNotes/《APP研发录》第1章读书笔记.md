#《APP研发录》读书笔记
---
##第一章
###1.1重新规划Android项目结构

重新规划Android项目的目录结构，分两步走：

1. **建立AndroidLab类库，将与业务无关的逻辑转移到AndroidLab，AndroidLab至少包括五大部分：包名+ acticity,cache,net,ui,utils 。**

	* activity包里面存放的是与业务无关的Activity基类。
	* net包里存放的是网络底层封装。
	* cacahe包里面存放的是缓存数据的图片和图片的相关处理。
	* ui包中存放的是自定义控件
	* utils包中存放的是各种与类无关的功用方法。	
2. **将主项目的类分门别类的进行划分，放置在各种包中。**


	
	* activity:我们按照模块继续划分，将不同模块的Activity划分到不同的包中。
	* adapter:所有的适配器都放在一起
	* entity:所有的实体都放在一起
	* db:SQLite相关逻辑的封装
	* engine:将业务相关的类都放在一起
	* ui:将自定义控件都放在这个包里
	* utils:将所有的公用方法都放在这里
	* interfaces:真正意义上的接口，命名以I作为开头
	* listener:基于Listener的接口，命名以On作为开头
	
###1.2为Activity定义新的生命周期

可以把onCreate方法拆成三个子方法

* initVariables:初始化变量，包括Intent带的数据和Activity内的变量
* initViews:加载layout布局文件，初始化控件，为控件挂上事件方法
* loadData:调用MobileAPI获取数据

###1.3统一事件编程模型

只要在一个团队内部达成了协议，决定使用某种事件编程方式，所有开发人员就要按照同样的方式编写代码。

###1.4实体化编程

####1.4.1在网络请求中使用实体

一些开发人员不使用实体化编程，在获取MobileAPI网络请求返回的JSON数据时，使用JSONObject或者JSONArray来承载数据，然后把返回的数据当作一个字典，根据键取出响应的值。介绍fastJSON和GSON这种实体化编程的方式.

* **使用fastJSON如下**：

```
WeatherEntity weatherEntity = JSON.parseObject(content,
						WeatherEntity.class);
				WeatherInfo weatherInfo = weatherEntity.getWeatherInfo();
				if (weatherInfo != null) {
					tvCity.setText(weatherInfo.getCity());
					tvCityId.setText(weatherInfo.getCityid());
				}
```

* **使用GSON如下：**：


```
Gson gson = new Gson();
				WeatherEntity weatherEntity = gson.fromJson(content,
						WeatherEntity.class);
				WeatherInfo weatherInfo = weatherEntity.getWeatherInfo();
				if (weatherInfo != null) {
					tvCity.setText(weatherInfo.getCity());
					tvCityId.setText(weatherInfo.getCityid());
				}
```

####1.4.3在页面跳转中实现实体

在一个页面中，数据的来源有两种：
1. 调用MobileAPI获取JSON数据
2. 从上一个页面传递过来

Activity之间的数据传递一个偷懒的办法是设置一个全局变量，在来源页设置全局变量，在目标页接收全局变量。以下是来源页MainActivity的代码：

```
Intent intent = new Intent(MainActivity.this,
						LoginActivity.class);
				intent.putExtra(AppConstants.Email, "jianqiang.bao@qq.com");
								
				CinemaBean cinema = new CinemaBean();
				cinema.setCinemaId("1");
				cinema.setCinemaName("星美");

				//使用全局变量的方式传递参数
				GlobalVariables.Cinema = cinema;
				startActivity(intent);
```


以下是目标页LoginActivity的代码：

```
// 使用全局变量的方式传值
		CinemaBean cinema = GlobalVariables.Cinema;
		if (cinema != null) {
			cinemaName = cinema.getCinemaName();
		} else {
			cinemaName = "";
		}
```

**不建议使用全局变量。App一旦被切换到后台，当手机内存不足的时候，就会回收这些全局变量，从而当App再次切换回前台时，再继续使用全局变量，就会因为它们为空而崩溃。如果必须使用全局变量，就一定要把它们序列化到本地。这样即使全局变量为空，也能从本地文件中恢复。**

我们使用Intent在页面间来传递数据实体的机制。

首先，在MainActivity中：

```
Intent intent = new Intent(MainActivity.this,
						LoginNewActivity.class);
				intent.putExtra(AppConstants.Email, "jianqiang.bao@qq.com");
				
				CinemaBean cinema = new CinemaBean();
				cinema.setCinemaId("1");
				cinema.setCinemaName("星美");
				
				//使用intent上挂可序列化实体的方式传递参数
				intent.putExtra(AppConstants.Cinema, cinema);

				startActivity(intent);
```
其次，目标页LoginActivity要这样写：

```
CinemaBean cinema = (CinemaBean)getIntent()
				.getSerializableExtra(AppConstants.Cinema);
		if (cinema != null) {
			cinemaName = cinema.getCinemaName();
		} else {
			cinemaName = "";
		}
```

这里的CinemaBean要实现Serializable接口，支持序列化：

```
public class CinemaBean implements Serializable {
	private static final long serialVersionUID = 1L;

	private String cinemaId;
	private String cinemaName;

	public CinemaBean() {

	}

	public String getCinemaId() {
		return cinemaId;
	}

	public void setCinemaId(String cinemaId) {
		this.cinemaId = cinemaId;
	}

	public String getCinemaName() {
		return cinemaName;
	}

	public void setCinemaName(String cinemaName) {
		this.cinemaName = cinemaName;
	}

}

```

###1.5Adapter模版
不对Adapter的写法进行规范，就会写出以下的Adapter

* 很多开发人员都喜欢将Adapter内嵌在Activity中，一般会使用SimpleAdapter
* 由于没有使用实体，所以一般会把一个字典作为构造函数的参数注入到Adapter中

希望Adapter只有一个编码风格，这样发现了问题也很容易排查。于是要求所有的Adapter都继承自BaseAdapter，从构造函数注入List<自定义实体>这样的数据集合，从而完成ListView的填充工作：

```
public class CinemaAdapter extends BaseAdapter {
	private final ArrayList<CinemaBean> cinemaList;
	private final AppBaseActivity context;

	public CinemaAdapter(ArrayList<CinemaBean> cinemaList,
			AppBaseActivity context) {
		this.cinemaList = cinemaList;
		this.context = context;
	}

	public int getCount() {
		return cinemaList.size();
	}

	public CinemaBean getItem(final int position) {
		return cinemaList.get(position);
	}

	public long getItemId(final int position) {

		return position;
	}
```

对于每个自定义的Adapter，都要实现以下4个方法：

* getCount()
* getItem()
* getItemId()
* getView()

此外，还要内置一个Holder嵌套类，用于存放ListView中每一行中的控件。ViewHolder的存在，可以避免频繁创建同一个列表项，从而极大的节省内存，如下：

```
class Holder {
		TextView tvCinemaName;
		TextView tvCinemaId;
	}
```

当有很多列表数据时，快速滑动列表会变的很卡，其实就是因为没有ViewHolder机制导致的，正确的写法如下：

```
public View getView(final int position, View convertView,
			final ViewGroup parent) {
		final Holder holder;
		if (convertView == null) {
			holder = new Holder();
			convertView = context.getLayoutInflater().inflate(
					R.layout.item_cinemalist, null);
			holder.tvCinemaName = (TextView) convertView
					.findViewById(R.id.tvCinemaName);
			holder.tvCinemaId = (TextView) convertView
					.findViewById(R.id.tvCinemaId);
			convertView.setTag(holder);
		} else {
			holder = (Holder) convertView.getTag();
		}

		CinemaBean cinema = cinemaList.get(position);
		holder.tvCinemaName.setText(cinema.getCinemaName());
		holder.tvCinemaId.setText(cinema.getCinemaId());
		return convertView;
	}
```

在Activity中，在使用Adapter的地方，我们按照下面的方式把列表数据传递过去：

```
@Override
	protected void initViews(Bundle savedInstanceState) {
		setContentView(R.layout.activity_listdemo);		

		lvCinemaList = (ListView) findViewById(R.id.lvCinemalist);
		
		CinemaAdapter adapter = new CinemaAdapter(
				cinemaList, ListDemoActivity.this);
		lvCinemaList.setAdapter(adapter);
		lvCinemaList.setOnItemClickListener(
				new AdapterView.OnItemClickListener() {
					@Override
					public void onItemClick(AdapterView<?> parent, View view,
							int position, long id) {
						//do something
					}
				});
	}
```

###1.6类型安全转换函数

在每天统计线上崩溃的时候，我们发现因为类型转换不正确导致的崩溃占了很大的比例。于是去检查程序中的所有类型转换，发现主要集中在两个地方：Object类型的对象，substring函数。

*  对于一个Object类型的对象，我们对其使用字符串操作函数toString，当其为null时就会崩溃。

	```
	int result = Integer.valueOf(obj.toString());
	```
	一旦obj这个对象为空，那么上面这行代码就会直接崩溃，这里的obj，一般是从JSON数据中取出来的，对于MobileAPI返回的JSON数据，我们无法保证其永远不为空。
	
	比较好的做法是编写一个类型安全转换函数convertToInt，实现如下：
	
	```
	public final static int convertToInt(Object value, int defaultValue) {
		if (value == null || "".equals(value.toString().trim())) {
			return defaultValue;
		}
		try {
			return Integer.valueOf(value.toString());
		} catch (Exception e) {
			try {
				return Double.valueOf(value.toString()).intValue();
			} catch (Exception e1) {
				return defaultValue;
			}
		}
	}
	```
	将这个方法放到Utils类下面，每当要把一个Object对象转化成整型时，都使用该方法，就不会崩溃
	
	```
	int result = Utils.convertToInt(obj, 0);
	```
* 如果长度不够，执行substring函数就会崩溃。Java的substring有两个参数：start和end。

	```
	String cityName = "T";
	String firstLetter = cityName.substring(1, 2);
	```
对于第一个参数设为0一般没有问题，设置为大于0如上代码就会崩溃，使用substring函数的时候，都要判断start和end两个参数是否越界了，应该这样写：
	
	```
	String cityName = "T";
	String firstLetter = "";
	if(cityName.length() > 1)
	{
		cityName.substring(1, 2);
	}
	```


**对于MobileAPI返回的数据**

* 首先，不能让App直接崩溃，应该在解析JSON数据的外面包一层try...catch...语句，讲截获到的异常在catch中进行处理，比如发送错误日志给服务器。
* 其次，需要分级对待，例如：
	* 对于那些不需要加工就能直接展示的数据，我们不担心，即使为空页面也就是不显示而已，不会引起逻辑的问题。
	* 对于那些很重要的数据，比如涉及支付的金额不能为空的逻辑，这时候就应该弹出提示框提示用户当前服务不可用，并停止接下来的操作。