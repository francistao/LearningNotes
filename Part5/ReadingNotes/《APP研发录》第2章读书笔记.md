#APP研发录第二章笔记
---

* 抛弃AsyncTask，自定义一套网络底层的封装框架。
* 设计一套App缓存策略。
* 设计一套MockService的机制，在没有MobileAPI的时候，也能假装获取到了网络返回的数据。
* 封装了用户Cookie的逻辑。

##2.1 网络底层封装

很多公司和团队都是用AsyncTask来封装网络底层，因为这个类非常好用，内部封装了很多好用的方法，但缺点是可扩展性不高。

对于网络请求，我们一般定义为GET和POST即可，GET为请求数据，POST为修改数据（增删改）。

###1.Request格式

所有的MobileAPI都可以写作http://www.xxx.com/aaaa.api的形式。

* 对于GET，我们可以写作：http://www.xxx.com/aaaa.api?k1=va&k2=v2的形式，也就是说，把key-value这样的键值对存放在URL上。之所以这样设计，是为了更方便地定义数据缓存。我们尽量使GET的参数都是string、int这样的简单类型。
* 对于POST，我们将key-value这样的键值对存放在Form表单中，进行提交。POST经常会提交大量数据，所以有些键值对要定义成集合或者复杂的自定义实体，这时我们就需要将这样的值转换为JSON字符串进行提交，由App传递到MobileAPI后，再将JSON字符串转换为对应的实体。

###2.Response格式

我们一般使用JSON作为MobileAPI返回的结果。最规范的JSON数据返回格式如下。
JSON数据格式1：

```
{ 
  "isError" : true,
    "errorType" : 1,
      "errorMessage" : "网络异常",
        "result" : ""
}
```

JSON数据格式2：

```
{ 
  "isError" : false,
    "errorType" : 0,
      "errorMessage" : "",
        "result" : {
          "cinemaId" : 1,
            "cinemaName" : "星美"
        }
} 
```

这里，isError是调用MobileAPI成功与否，errorType是错误类型（如果成功则为0），errorMessage是错误消息（如果成功则为空），result是成功请求返回的数据结果（如果失败则返回空）。

既然所有的JSON都返回isError、errorType、errorMessage、result这4个字段，我们不妨定义一个Response实体类，作为所有JSON实体的最外层，代码如下所示：

```
public class Response
{
  private boolean error;
  private int errorType;       // 1为Cookie失效
  private String errorMessage;
  private String result;
  public boolean hasError() {
    return error;
  }
  public void setError(boolean hasError) {
    this.error = hasError;
  }
  public String getErrorMessage() {
    return errorMessage;
  }
  public void setErrorMessage(String errorMessage) {
    this.errorMessage = errorMessage;
  }
  public String getResult() {
    return result;
  }
  public void setResult(String result) {
    this.result = result;
  }
  public int getErrorType() {
      return errorType;
  }
  public void setErrorType(int errorType) {
    this.errorType = errorType;
  }
} 
```

如果成功返回了数据，数据会存放在result字段中，映射为Response实体的result属性。

上面的JSON数据返回的是一笔影院数据，如果返回的result是很多影院的数据集合，那么就要把result解析为相应的实体集合，如下所示：

```
{ 
  "isError" : false,
    "errorType" : 0,
      "errorMessage" : "",
        "result" : [
          {"cinemaId" : 1, "cinemaName" : "星美"},
          {"cinemaId" : 2, "cinemaName" : "万达"}
        ]
} 
```

####2.1.2 AsyncTask的使用和缺点

对AsyncTask的封装属于网络底层的技术，所以AsyncTask应该封装在AndroidLib类库中，而不是具体的项目里。
对网络异常的分类，也就是Response类中的errorType字段，分析如下：

* 一种是请求发送到MobileAPI，MobileAPI执行过程中发现的异常，这时候要自定义错误类型，也就是errorType，比如说1是Cookie过期，2是第三方支付平台不能连接，等等，这些已知的错误都是大于0的整数，因接口不同而各自定义不同。
* 另一种是在App访问MobileAPI接口时发生的异常，有可能App自身网络不稳定，有可能因为网络传输不好导致返回了空值，这些异常情况我们都标记为负数。

基于上述分析，AsyncTask的doInBackground方法复写为：

```
@Override
protected Response doInBackground(String… url) {
  return getResponseFromURL(url[0]);
}
private Response getResponseFromURL(String url) {
  Response response = new Response();
  HttpGet get = new HttpGet(url);
  String strResponse = null;
  try {
    HttpParams httpParameters = new BasicHttpParams();
    HttpConnectionParams.setConnectionTimeout(httpParameters, 8000);
    HttpClient httpClient = new DefaultHttpClient(httpParameters);
    HttpResponse httpResponse = httpClient.execute(get);
    if (httpResponse.getStatusLine().getStatusCode() 
        == HttpStatus.SC_OK) {
      strResponse = EntityUtils.toString(httpResponse.getEntity());
    }
  } catch (Exception e) {
    response.setErrorType(-1);
    response.setError(true);
    response.setErrorMessage(e.getMessage());
  }
  if (strResponse == null) {
      response.setErrorType(-1);
    response.setError(true);
    response.setErrorMessage("网络异常, 返回空值");
  } else {
    strResponse = "{'isError':false,'errorType':0,'errorMessage':'',
    'result':{'city':'北京','cityid':'101010100','temp':'17',
      'WD':'西南风','WS':'2级','SD':'54%','WSE':'2','time':'23:15',
        'isRadar':'1','Radar':'JC_RADAR_AZ9010_JB',
          'njd':'暂无实况','qy':'1016'}}";
          response = JSON.parseObject(strResponse, Response.class);
}
return response;
} 
```

相应的，在AsyncTask的onPostExecute方法中，我们要对错误类型进行分类，从而进一步回调：

```
public abstract class RequestAsyncTask 
  extends AsyncTask<String, Void, Response> {
    public abstract void onSuccess(String content);
    public abstract void onFail(String errorMessage);
    @Override
      protected void onPreExecute() {
    }
    @Override
      protected void onPostExecute(Response response) {
      if(response.hasError()) {
        onFail(response.getErrorMessage());
      } else {
        onSuccess(response.getResult());
      }
    } 
```


目前我们只定义了onSuccess和onFail两个回调函数，将网络返回值简单地分为成功与失败两种情况。

在相应的Activity页面，调用AysncTask如下所示：

```
protected void loadData() {
  String url = "http://www.weather.com.cn/data/sk/101010100.html";
  RequestAsyncTask task = new RequestAsyncTask() {
    @Override
    public void onSuccess(String content) {
      // 第2种写法, 基于fastJSON
      WeatherEntity weatherEntity = JSON.parseObject(content,
                                                     WeatherEntity.class);
      WeatherInfo weatherInfo = weatherEntity.getWeatherInfo();
      if (weatherInfo != null) {
        tvCity.setText(weatherInfo.getCity());
        tvCityId.setText(weatherInfo.getCityid());
      }
    }
     @Override
     public void onFail(String errorMessage) {
       new AlertDialog.Builder(WeatherByFastJsonActivity.this)
         .setTitle("出错啦").setMessage(errorMessage)
         .setPositiveButton("确定", null).show();
     }
  };
  task.execute(url);
} 
```

网上关于如何使用AsyncTask的文章不胜枚举，大家都在欣赏它的优点，却忽略了它的致命缺点，那就是不能灵活控制其内部的线程池。

线程池里面的每个线程存放的都是MobileAPI的调用请求，而AsyncTask中又没有暴露出取消这些请求的方法，也就是我们熟知的CancelRequest方法，所以，一旦从A页面跳转到B页面，那么在A页面发起的MobileAPI请求，如果还没有返回，并不会被取消。

对于一款频繁调用MobileAPI的应用类App而言，最严重的情况发生在首页到二级页面的跳转，因为在首页会调用十几个MobileAPI接口，视网络情况而定，如果是WiFi，应该很快就能请求到数据，不会产生积压，但如果是3G或者2G，那么请求就会花费很长时间，而我们在这期间就跳转到二级页面，而这个二级页面也会调用MobileAPI接口，那么将得不到任何结果，因为首页的请求还在排队处理中，之前的那十几个MobileAPI接口的数据还都遥遥无期在线程池里排队呢，就更不要说当前页面这个请求了。

如果你不信，我们可以做个试验。记录每次MobileAPI请求发起和接收数据的时间点，你会看到，在迅速进入二级页面后，首页的十几个MobileAPI请求只有发起时间并没有返回时间，说明它们还在处理过程中，都被堵塞了。

####2.1.3 使用原生的ThreadPoolExcutor+Runnable+Handler

既然AsyncTask有诸多问题，那么退而求其次，使用ThreadPoolExecutor+Runnable+Handler的原生方式，对网络底层进行封装。

建议大家下载源码感受一些，源码下载地址：

[《App研发录》 源码](http://www.cnblogs.com/Jax/p/4656789.html)

####2.1.4　网络底层的一些优化工作

接下来将完善这个框架，修复其中的一些瑕疵，如onFail的统一处理机制、UrlConfigManager的优化、ProgressBar的处理等。

**1.onFail的统一处理机制**

如果访问MobileAPI请求失败，我们一般希望只是在App上简单地弹出一个提示框，告诉用户网络有异常。

也就是说，对于每个在Activity中声明的RequestCallback实例而言，尽管每个onSuccess方法的处理逻辑各不相同，但每个onFail方法都是一样的逻辑和代码，如下所示：

```
weatherCallback = new RequestCallback() {
  @Override
  public void onSuccess(String content) {
    WeatherInfo weatherInfo = JSON.parseObject(content,
                                               WeatherInfo.class);
    if (weatherInfo != null) {
      tvCity.setText(weatherInfo.getCity());
      tvCityId.setText(weatherInfo.getCityid());
    }
  }
   @Override
   public void onFail(String errorMessage) {
     new AlertDialog.Builder(WeatherByFastJsonActivity.this)
       .setTitle("出错啦").setMessage(errorMessage)
       .setPositiveButton("确定", null).show();
   }
}; 
```

我不希望每次都编写同样的onFail方法，这会使程序很臃肿。于是在AppBaseActivity中写一个自定义类AbstractRequestCallback，如下所示：

```
public abstract class AppBaseActivity extends BaseActivity {
  public abstract class AbstractRequestCallback 
    implements RequestCallback {
    public abstract void onSuccess(String content);
    public void onFail(String errorMessage) {
      new AlertDialog.Builder(AppBaseActivity.this)
        .setTitle("出错啦").setMessage(errorMessage)
        .setPositiveButton("确定", null).show();
    }
      }
} 
```

那么我们的weatherRequestCallback的实例化就可以改写如下，可以看到，不再需要重写onFail方法：

```
weatherCallback = new AbstractRequestCallback() {
  @Override
  public void onSuccess(String content) {
    WeatherInfo weatherInfo = JSON.parseObject(content,
                                               WeatherInfo.class);
    if (weatherInfo != null) {
      tvCity.setText(weatherInfo.getCity());
      tvCityId.setText(weatherInfo.getCityid());
    }
  }
  }; 
```

当然，如果有些MobileAPI接口在返回错误时需要App特殊处理，比如重启App或者啥都不做，我们只需要在实例化AbstractRequestCallback时，重写onFail方法即可，如下所示。重写的onFail方法是一个空方法，表示出错时啥都不做：

```
weatherCallback = new AbstractRequestCallback() {
  @Override
  public void onSuccess(String content) {
    WeatherInfo weatherInfo = JSON.parseObject(content,
                                               WeatherInfo.class);
    if (weatherInfo != null) {
      tvCity.setText(weatherInfo.getCity());
      tvCityId.setText(weatherInfo.getCityid());
    }
      }
   @Override
   public void onFail(String errorMessage) {
     // 重启App或者啥都不做
   }
}; 
```

**2.UrlConfigManager的优化**

在UrlConfigManager的实现上，我们采取的策略是每发起一次MobileAPI请求，都会读取url.xml文件，把符合这次MobileAPI接口调用的参数取出来。

在一个大量调用MobileAPI的App中，这样的设计会造成频繁读xml文件，性能很差。于是我们对其进行改造，在App启动时，一次性将url.xml文件都读取到内存，把所有的UrlData实体保存在一个集合中，然后每次调用MobileAPI接口，直接从内存的这个集合中查找。考虑到内存中的数据会被回收，所以上述这个集合一旦为空，我们要从url.xml中再次读取。

基于上述方案，我们对UrlConfigManager的findUrl方法进行改造：

```
public static URLData findURL(final Activity activity, 
                              final String findKey) {
  // 如果urlList还没有数据(第一次)
  // 或者被回收了, 那么(重新)加载xml
  if (urlList == null || urlList.isEmpty())
    fetchUrlDataFromXml(activity);
  for (URLData data : urlList) {
    if (findKey.equals(data.getKey())) {
      return data;
    }
  }
  return null;
} 
```

其中，fetchUrlDataFromXml方法就不多说了，它的工作就是把xml的数据都搬到内存集合urlList中。

**3.不是每个请求都需要回调的**

有些时候，我们调用一个MobileAPI接口，并不需要知道调用成功与否以及返回结果是什么，比如向MobileAPI发送打点统计数据。那就是说，我们不需要回调函数了，那么代码可以写为：

```
void loadAPIData3() {
  ArrayList<RequestParameter> params 
    = new ArrayList<RequestParameter>();
  RequestParameter rp1 = 
    new RequestParameter("cityId", "111");
  RequestParameter rp2 = 
    new RequestParameter("cityName", "Beijing");
  params.add(rp1);
  params.add(rp2);
  RemoteService.getInstance()
    .invoke(this, "getWeatherInfo", params, null);
} 
```

我们将空的RequestCallback传给HttpRequest，那么在HttpRequest处理请求返回的结果时，就需要添加HttpRequest是否为空的判断，不为空，才会处理返回结果；否则，发起MobileAPI请求后什么都不做。

有以下两个地方需要修改：

1）处理请求时：

```
response = httpClient.execute(request);
if ((requestCallback != null)) {
  // 获取状态
  final int statusCode = 
    response.getStatusLine().getStatusCode();
  if (statusCode == HttpStatus.SC_OK) {
    final ByteArrayOutputStream content = 
      new ByteArrayOutputStream(); 
```

2）遇到异常，是否要回调onFail方法：

```
public void handleNetworkError(final String errorMsg) {
  if ((requestCallback != null)) {
    handler.post(new Runnable() {
      @Override
        public void run() {
        HttpRequest.this.requestCallback
          .onFail(errorMsg);
      }
    });
  }
} 
```

**4.ProgressBar的处理**

在调用MobileAPI的时候，会显示进度条ProgressBar，直到返回结果到onSuccess或onFail回调方法，ProgressBar才会消失。

由于App要保持风格统一，所以所有页面的ProgressBar应该长得一样。那么我们就可以将其定义在AppBaseActivity中，如下所示：

```
public abstract class AppBaseActivity extends BaseActivity {
  protected ProgressDialog dlg;
  public abstract class AbstractRequestCallback 
    implements RequestCallback {
    public abstract void onSuccess(String content);
    public void onFail(String errorMessage) {
      dlg.dismiss();
          public void onFail(String errorMessage) {
      dlg.dismiss();
      new AlertDialog.Builder(AppBaseActivity.this)
        .setTitle("出错啦").setMessage(errorMessage)
        .setPositiveButton("确定", null).show();
    }
  }
} 
```

在使用的时候，在开始调用MobileAPI的地方，执行show方法；在onSuccess和onFail方法的开始，执行dismiss方法：

```
@Override
protected void loadData() {
  dlg = Utils.createProgressDialog(this,
                                   this.getString(R.string.str_loading));
  dlg.show();
  loadAPIData1();
}
void loadAPIData1() {
  weatherCallback = new AbstractRequestCallback() {
    @Override
    public void onSuccess(String content) {
      dlg.dismiss();
      WeatherInfo weatherInfo = JSON.parseObject(content,
                                                 WeatherInfo.class);
      if (weatherInfo != null) { 
```

不要把Dialog的show方法和dismiss方法封装到网络底层。网络底层的调用经常是在子线程执行的，子线程是不能操作Dialog、Toast和控件的。

###2.2　App数据缓存设计

如果以为上一节内容就是网络底层框架的全部，那就错了。那只是网络底层框架的最核心的功能，我们还有很多高级功能没有介绍。在接下来的几节中，我将陆续介绍到这些高级功能。本节先介绍App本地的缓存策略。

####2.2.1　数据缓存策略

对于任何一款应用类App，如果访问MobileAPI的速度和牛车一样慢，那么就是失败之作。不要在WiFi下测试速度，那是自欺欺人，要把App放在2G或3G网络环境下进行测试，才能得到大部分用户的真实数据。

访问MobileAPI，主要慢在一来一回的传输速度上，对于服务器的处理速度，不需要担心太多，大多数服务器逻辑原本就是支持网站端的，现在只是在外面包了一层，返回给App而已。

既然时间主要花在了数据传输上，那么我们就要想一些应对的措施。比如说，减少MobileAPI的调用次数。对于一个App页面，它一次性可能需要3部分数据，分别从3个MobileAPI接口获取，那么我们就可以做一个新的MobileAPI接口，将这3部分数据都获取到，然后一次性返回。

减少调用次数只是若干解决方案中的一种，更极端的做法是，App调用一次MobileAPI接口后，在一个时间段内不再调用，仍然使用上次调用接口获取到的数据，这些数据保存在App上，我们称为App缓存，这个时间段我们称为App缓存时间。

App缓存只能针对于MobileAPI中GET类型的接口，对于POST不适用。因为GET是获取数据，而POST是修改数据。

此外，即使是GET类型的接口，对于那些即时性很低的、不怎么改变的数据，比如获取商品的描述，缓存时间可以设置得比较长，比如5~10分钟，对于那些即时性比较高、频繁变动的数据，比如商品价格，缓存时间就会比较短，甚至不能进行缓存。

即使对于同一个需要做App缓存的MobileAPI，参数不同，缓存也是不同的。比如GetWeather.api这个MobileAPI接口，它有一个参数也就是时间date，对于date=2014-9-8和date=2014-9-9，它们就分别对应两个缓存，不能存在一起。

接下来要说的是，App缓存存在哪里，以及以什么方式进行存放。由于缓存数据比较大，所以我们将其存在SD卡上，而不是内存中。这样的话，App缓存策略就仅限于那些有SD卡的手机用户了。

我们可以将xxx.apik1=va&k2=v2这样的URL格式作为key，存放App缓存数据。需要注意的是，我们要对k1、k2这些key进行排序，这样才能唯一，否则对于如下URL：

```
xxxx.apik1=v1&k2=v2
xxxx.apik2=v2&k1=v1 
```

就会被认为是两个不同的key存放在缓存中，但其实它们是一样的。
对上面的介绍总结如下：

1）对于App而言，它是感受不到取的是缓存数据还是调用MobileAPI。具体工作由网络底层完成。

2）在url.xml中为每一个MobileAPI接口配置缓存时间Expired。对于post，一律设置为0，因为post不需要缓存。

3）在HttpRequest类中的run方法中，改动3个地方：

a）写一个排序算法sortKeys，对URL中的key进行排序。

b）将newUrl作为key，检查缓存中是否有数据，有则直接返回；否则，继续调用MobileAPI接口。

```
// 如果这个get的API有缓存时间(大于0)
if (urlData.getExpires() > 0) {
  final String content = CacheManager.getInstance()
    .getFileCache(newUrl);
  if (content != null) {
    handler.post(new Runnable() {
      @Override
        public void run() {
        requestCallback.onSuccess(content);
      }
    });
    return;
  }
} 
```

c）MobileAPI接口返回数据后，将数据存入缓存。

```
final Response responseInJson = JSON.parseObject(
  strResponse, Response.class);
if (responseInJson.hasError()) {
  handleNetworkError(responseInJson.getErrorMessage());
} else {
  // 把成功获取到的数据记录到缓存
  if (urlData.getNetType().equals(REQUEST_GET)
      && urlData.getExpires() > 0) {
    CacheManager.getInstance().putFileCache(newUrl,
                                            responseInJson.getResult(),
                                            urlData.getExpires());
  }
  handler.post(new Runnable() {
    @Override
      public void run() {
      requestCallback.onSuccess(responseInJson
                                .getResult());
    }
  });
} 
```

4）CacheManager用于操作读写缓存数据，并判断缓存数据是否过期。缓存中存放的实体就是CacheItem。

5）在App项目中，创建YoungHeartApplication这个Application级别的类，在程序启动时，初始化缓存的目录，如果不存在则创建之。

```
public class YoungHeartApplication extends Application {
  @Override
    public void onCreate() {
    super.onCreate(); 
    CacheManager.getInstance().initCacheDir();
  }
} 
```

####2.2.2　强制更新

不光是App端需要记录缓存数据，在MobileAPI的很多接口，其实也需要一样的设计。

如果对于某个接口的数据，MobileAPI缓存了5分钟，App缓存了3分钟，那么最极端的情况是，用户在8分钟内是看不到数据更新的。因此，我们需要在页面上提供一个强制更新的按钮。

我们可以让RemoteService多暴露一个boolean类型的参数，用于判断是否要遵守App端缓存策略，如果是，则在从url.xml中取出UrlData实体后，将其expired强制设置为0，这样就不会执行缓存策略了。

RemoteService的改动如下：

```
public void invoke(final BaseActivity activity,
                   final String apiKey,
                   final List<RequestParameter> params,
                   final RequestCallback callBack) {
  invoke(activity, apiKey, params, callBack, false);
}
public void invoke(final BaseActivity activity,
                   final String apiKey,
                   final List<RequestParameter> params,
                   final RequestCallback callBack,
                   final boolean forceUpdate) {
  final URLData urlData = 
    UrlConfigManager.findURL(activity, apiKey);
  if(forceUpdate) {
    // 如果强制更新, 那么就把过期时间强制设置为0
    urlData.setExpires(0); 
  }
  HttpRequest request = 
    activity.getRequestManager().createRequest(
    urlData, params, callBack);
  DefaultThreadPool.getInstance().execute(request);
} 
```

那么在调用的时候，只需要加一个参数就好：

```
RemoteService.getInstance().invoke(
  this, "getWeatherInfo", params,
  weatherCallback); 
```

###2.3　MockService

设计App端MockService包括如下几个关键点：

1）对需要Mock数据的MobileAPI接口，通过在url.xml中配置Node节点MockClass属性，来指定要使用那个Mock子类生成的数据：

```
<Node
    Key="getWeatherInfo"
    Expires="300"
    NetType="get"
    MockClass="com.youngheart.mockdata.MockWeatherInfo"
    Url="http://www.weather.com.cn/data/sk/101010100.html" /> 
```

这里将使用com.mockdata.mockdata包下的MockWeatherInfo子类来解析。

2）我使用了反射工厂来设计MockService。MockService类是基类，它有一个抽象方法getJsonData，用于返回手动生成的Mock数据。

```
public abstract class MockService {
  public abstract String getJsonData();
} 
```

每个要Mock数据的MobileAPI接口，都对应一个继承自MockService的子类，都要实现各自的getJsonData方法，返回不同的JSON数据。

比如在上述url.xml中声明的MockWeatherInfo，它对应的类实现如下：

```
public class MockWeatherInfo extends MockService {
  @Override
  public String getJsonData() {
    WeatherInfo weather = new WeatherInfo();
    weather.setCity("Beijing");
    weather.setCityid("10000");
    Response response = getSuccessResponse();
    response.setResult(JSON.toJSONString(weather));
    return JSON.toJSONString(response);
  }
  } 
```

3）接下来介绍如何实现反射机制。

主要的改造工作在RemoteService类的invoke方法中，根据是否在url.xml中指定了MockClass值来决定，是调用线上MobileAPI还是从本地MockService直接取假数据。

如果MockClass有值，就把这个值反射为一个具体的类，比如MockWeatherInfo，然后调用它的getJsonData方法。

```
public void invoke(final BaseActivity activity, 
                   final String apiKey,
                   final List<RequestParameter> params, 
                   final RequestCallback callBack) {
  final URLData urlData = UrlConfigManager.findURL(activity, apiKey);
  if (urlData.getMockClass() != null) {
    try {
      MockService mockService = (MockService) Class.forName(
        urlData.getMockClass()).newInstance();
      String strResponse = mockService.getJsonData();
      final Response responseInJson = 
        JSON.parseObject(strResponse, Response.class);
      if (callBack != null) {
        if (responseInJson.hasError()) {
          callBack.onFail(responseInJson.getErrorMessage());
        } else {
          callBack.onSuccess(responseInJson.getResult());
        }
      }
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    } catch (InstantiationException e) {
      e.printStackTrace();
    } catch (IllegalAccessException e) {
      e.printStackTrace();
    }
  } else {
    HttpRequest request = 
      activity.getRequestManager().createRequest(
      urlData, params, callBack);
    DefaultThreadPool.getInstance().execute(request);
  }
} 
```

###2.4　用户登录

####2.4.1　登录成功后的各种场景

首先，贯穿App的，应该有一个User全局变量，在每次登录成功后，会将其isLogin属性设置为true，在退出登录后，则将该属性设置为false。这个User全局变量要支持序列化到本地的功能，这样数据才不会因内存回收而丢失。

其次，登录分为3种情形：

情形1：点击登录按钮，进入登录页面LoginActivity，登录成功后，直接进入个人中心PersonCenterActivity。这种情况最直截了当，一路执行startActivity(intent)就能达到目的。

情形2：在页面A，想要跳转到页面B，并携带一些参数，却发现没有登录，于是先跳转到登录页，登录成功后，再跳转到B页面，同时仍然带着那些参数。

这就主要是setResult(intent,resultCode)发挥作用的时候了，Activity的回调机制这时候派上了用场，如下所示：

```
btnLogin2.setOnClickListener(new OnClickListener(){
  @Override
  public void onClick(View v) {
  if(User.getInstance().isLogin()) {
      gotoNewsActivity();
    } else {
      Intent intent = new Intent(LoginMainActivity.this, 
                                 LoginActivity.class);
      intent.putExtra(AppConstants.NeedCallback, true);
      startActivityForResult(intent, 
                             LOGIN_REDIRECT_OUTSIDE);
    }
  }
}); 
```

情形3：在页面A，执行某个操作，却发现没有登录，于是跳转到登录页，登录成功后，再回到页面A，继续执行该操作。

处理方式同于情形2，也是使用setResult来完成回调。

```
btnLogin3.setOnClickListener(new OnClickListener(){
  @Override
  public void onClick(View v) {
    if(User.getInstance().isLogin()) {
      changeText();
    } else {
      Intent intent = new Intent(LoginMainActivity.this, 
                                 LoginActivity.class);
      intent.putExtra(AppConstants.NeedCallback, true);
      startActivityForResult(intent, 
                             LOGIN_REDIRECT_INSIDE);
    }
  }
}); 
```

无论是上述哪种情形，登录页面LoginActivity只有一个，所以要把上面的三个逻辑整合在一起，如下所示：

```
RequestCallback loginCallback = new AbstractRequestCallback() {
@Override
  public void onSuccess(String content) {
    UserInfo userInfo = JSON.parseObject(content,
                                         UserInfo.class);
    if (userInfo != null) {
      User.getInstance().reset();
      User.getInstance().setLoginName(userInfo.getLoginName());
      User.getInstance().setScore(userInfo.getScore());
      User.getInstance().setUserName(userInfo.getUserName());
      User.getInstance().setLoginStatus(true);
      User.getInstance().save();
    }
    if(needCallback) {
      setResult(Activity.RESULT_OK);
      finish();
    } else {
      Intent intent = new Intent(LoginActivity.this, 
                                 PersonCenterActivity.class);
      startActivity(intent);
    }
  }
  }; 
```

整合的关键在于从上个页面传过来needCallback变量，它决定了是否要回到上个页面。

另一方面，我们看到，在登录成功后，我们会把用户信息存储到User这个全局变量并序列化到本地，这是因为各个模块都有可能使用到用户的信息。其中LoginStatus是关键，接下来的篇幅将着重谈论这个属性。

最后在LoginMainActivity中的onActivityResult回调函数，它负责处理登录后的事情，如下所示：

```
@Override
  protected void onActivityResult(int requestCode, 
                                  int resultCode, Intent data) {
  if (resultCode != Activity.RESULT_OK) {
    return;
  }
  switch (requestCode) {
    case LOGIN_REDIRECT_OUTSIDE:
    gotoNewsActivity();
    break;
    case LOGIN_REDIRECT_INSIDE:
    changeText();
    break;
    default:
    break;
  }
} 
```

我们看到，对于情形2，当用户在LoginMainActivity点击按钮想跳转到NewsActivity，如果已经登录，就直接跳转过去；否则，先到LoginActivity登录，然后回调LoginMain-Activity的onActivityResult，仍然跳转到NewsActivity。

####2.4.2　自动登录

所谓自动登录，就是登录成功后，重启App后用户仍然是登录状态。

最直接的方法是，登录成功后，本地保存用户名和密码。重启App后，检查本地是否有保存用户名和密码，如果有，则将用户名和密码传入到登录接口，模拟用户登录的行为。

但这样就有安全风险了，分析如下：

* 本地保存用户密码，这种的敏感信息容易被人窃取。要么是在本地文件中看到这些信息，要么是侦听App的网络请求，获取到请求的数据。
* 所以本地保存密码时，一定要进行加密。对称加密是不可靠的，因为很难确保App的源代码不外泄，所以别有用心的人还是可以根据源码中的对称加密算法，反向把密码推算出来。只有不对称加密才是安全的。
* 那么登录之后呢？市面上大多数App的逻辑都有问题，它们会在本地保存一个isLogin的全局变量，登录成功后设置为true。接下来涉及用户相关的MobileAPI，只有在这个值为true时才能调用，它们会把UserId传递给服务器。
* 服务器的解决方案通常也很简陋，它没有任何安全机制，包括用户信息相关的MobileAPI接口，只要接口调用参数中有UserId，它就会去把相关的数据取出并返回。
* 一种补救措施是，每次调用用户相关的MobileAPI接口时，都需要把UserId和加密后的密码一起传递。而服务器需要对那些用户相关的MobileAPI接口加上安全验证机制，每次请求都检查用户名和密码是否正确。我们要求密码是经过哈希散列算法不对称加密过的，是无法还原的。服务器的验证工作是根据传过来的UserId从数据库中取出相应的密码，然后进行比对。注意，数据库中存放的密码是在注册的时候经过哈希散列算法加密过的。
* 本地保存用户名和密码的另一个问题是，每次用户启动App，登录页都会一闪而过，因为它要模拟用户登录的行为：假装输入用户名和密码，然后假装点击登录按钮。这样做用户体验很不好倒是其次，关键是这种做法有个无法自圆其说的硬伤——出于安全考虑，我们要修改登录接口，使其除了接收用户名和密码这两个参数外，还必须接收验证码，也就是动态口令.

我们知道，验证码必须是手动输入的，否则就失去了它存在的意义。但是当前这种自动登录的做法，我们只知道用户名和密码，而不知道每次生成的验证码是什么，所以就不能自动登录了。是时候该抛弃这种每次启动就进行一次登录的机制了，其实Web在这一点已经做得很成熟了，那就是Cookie机制。

也有的人管Cookie叫Token，这是用户身份的唯一性标志。

首先，App在登录成功后，会从服务器获取到一个Cookie，这个Cookie存放在Http-Response的header中，如下所示：

```
Set-Cookie: customer=huangxp; path=/foo; domain=.ibm.com; 
expires= Wednesday, 19-OCT-05 23:12:40 GMT; [secure] 
```

我们将其取出来，不用关心它是什么，只要把它存放在本地文件中即可。

我们需要修改App的网络底层，也就是HttpRequest类，分以下几步：

1）每次发起MobileAPI请求时，都要把本地保存的Cookie取出来，放到HttpRequest的header中。还是那句话，不用管Cookie是什么，也不管Cookie是否有值，都应如下操作：

```
// 添加Cookie到请求头中
addCookie();
// 发送请求
response = httpClient.execute(request); 
```

2）每次接收MobileAPI的相应结果时，都把HttpResponse的header里面的Cookie取出来，覆盖本地保存的Cookie。不用管Cookie有值与否，如下所示：

```
if (urlData.getNetType().equals(REQUEST_GET)
    && urlData.getExpires() > 0) {
  CacheManager.getInstance().putFileCache(newUrl,
                                          responseInJson.getResult(),
                                                                                    urlData.getExpires());
}
handler.post(new Runnable() {
  @Override
  public void run() {
    requestCallback.onSuccess(responseInJson
                              .getResult());
  }
});
// 保存Cookie
saveCookie(); 
```

以下是addCookie和saveCookie方法的实现：

```
public void addCookie() {
  List<SerializableCookie> cookieList = null;
  Object cookieObj = BaseUtils.restoreObject(cookiePath);
  if (cookieObj != null) {
    cookieList = (ArrayList<SerializableCookie>) cookieObj;
  }
  if ((cookieList != null) && (cookieList.size() > 0)) {
    final BasicCookieStore cs = new BasicCookieStore();
    cs.addCookies(cookieList.toArray(new Cookie[] {}));
    httpClient.setCookieStore(cs);
  } else {
    httpClient.setCookieStore(null);
  }
}
public synchronized void saveCookie() {
  // 获取本次访问的cookie
  final List<Cookie> cookies = 
    httpClient.getCookieStore().getCookies();
  // 将普通cookie转换为可序列化的cookie
  List<SerializableCookie> serializableCookies = null;
  if ((cookies != null) && (cookies.size() > 0)) {
    serializableCookies = new ArrayList<SerializableCookie>();
    for (final Cookie c : cookies) {
      serializableCookies.add(new SerializableCookie(c));
    }
        }
  }
  BaseUtils.saveObject(cookiePath, serializableCookies);
} 
```

而服务器的相应操作，对于来自App的请求：

3）如果是用户信息相关的，则判断HttpRequest中Cookie是否有效，如果有效，就去执行后续的逻辑并返回结果；否则，返回Cookie过期失效的错误信息。

4）如果是用户无关的，则不需要检查HttpRequest中Cookie，直接执行下面的逻辑即可。

此外，还需要注意几个地方，都是些琐碎的工作：

* 用户注销功能，要把本地保存的Cookie清空。App判断用户是否登录的标志，就是Cookie是否为空。
* 用户注册功能，一般在注册成功后，都会拿着用户名和密码再调用一次登录接口，这就又和验证码功能冲突了，解决方案是注册成功后直接跳转到登录页面，让用户手动再输入一次。这是从产品层面来解决问题。另一种解决方案是，注册成功后进入个人中心页面，不需要再登录一次，而是把注册和登录接口绑在一起。
* 对于Cookie过期，App应该跳转到登录页面，让用户手动进行登录。这里有一个比较有挑战性的工作，就是登录成功后，应该返回手动登录之前的那个页面。我们在下一节再细说这个技术。

####2.4.3　Cookie过期的统一处理

Cookie不是一直有效的，到了一定时间就会失效。

Cookie过期的表现是，当访问MobileAPI某个接口的时候，就不会返回数据了，代之以Cookie过期的错误消息，这时要统一处理。

我们要求MobileAPI在遇到这种情况时，直接返回以下内容的JSON，其中，errorType固定为1：

```
{
  "isError" : true,
    "errorType" : 1,
      "errorMessage" : "Cookie失效, 请重新登录",
        "result" : ""
} 
```

为此我们修改AndroidLib，使之支持Cookie失效的场景。

1）在RequestCallback中增加一种onCookieExpired回调方法，如下所示：

```
public interface RequestCallback
{
  public void onSuccess(String content);
  public void onFail(String errorMessage);
  public void onCookieExpired();
} 
```

2）在网络底层对JSON返回结果进行解析，如果发现是属于Cookie过期的错误类型，就直接回调onCookieExpired方法，如下所示：

```
final Response responseInJson = JSON.parseObject(
  strResponse, Response.class);
if (responseInJson.hasError()) {
  if(responseInJson.getErrorType() == 1) {
    handler.post(new Runnable() {
      @Override
        public void run() {
        requestCallback.onCookieExpired();
      }
    });
  } else {
    handleNetworkError(responseInJson.getErrorMessage());
  } 
```

我们模拟一种场景，在CookieExpiredActivity页面，访问天气预报这个MobileAPI接口，如果Cookie失效，则弹出对话框，通知用户“Cookie过期，请重新登录”，点击确定按钮，将跳转到登录页。登录成功后，将回到上一个页面，即CookieExpiredActivity。

由于所有页面处理Cookie过期的逻辑都是相同的，所以我们将其封装到基类AppBaseActivity中，放在和onFail方法平级的位置：

```
public void onCookieExpired() {
  dlg.dismiss();
  new AlertDialog.Builder(AppBaseActivity.this)
  .setTitle("出错啦")
  .setMessage("Cookie过期, 请重新登录")
  .setPositiveButton("确定",
                     new DialogInterface.OnClickListener() {
                       @Override
                       public void onClick(DialogInterface dialog,
                                                                  int which) {
                         Intent intent = new Intent(
                           AppBaseActivity.this,
                           LoginActivity.class);
                         intent.putExtra(AppConstants.NeedCallback,
                                         true);
                         startActivity(intent);
                       }
                     }).show();
} 
```

####2.4.4　防止黑客刷库


* MobileAPI在发现有同一IP短时间内频繁访问某一个MobileAPI接口时，就直接返回一段HTML5，要求用户输入验证码。
* App在接收到这段代码时，就在页面上显示一个浮层，里面一个WebView，显示这个要求用户输入验证码的HTML5。


###2.5　HTTP头中的奥妙

####2.5.1　HTTP请求

HTTP请求分为HTTPRequest和HTTPResponse两种。但无论哪种请求，都由header和body两部分组成。

**1.HTTP Body**

Body部分就是存放数据的地方，回顾一下我们在HTTPRequest类中封装的网络请求：

1）对于get形式的HTTPRequest，要发送的数据都以键值对的形式存放在URL上，比如aaa.apik1=va&k2=va。它的Body是空的，如下所示：

```
if (urlData.getNetType().equals(REQUEST_GET)) {
  // 添加参数
  final StringBuffer paramBuffer = new StringBuffer();
  if ((parameter != null) && (parameter.size() > 0)) {
    // 这里要对key进行排序
    sortKeys();
    for (final RequestParameter p : parameter) {
      if (paramBuffer.length() == 0) {
        paramBuffer.append(p.getName() + "="
                           + BaseUtils.UrlEncodeUnicode(p.getValue()));
      } else {
        paramBuffer.append("&" + p.getName() + "="
                           + BaseUtils.UrlEncodeUnicode(p.getValue()));
      }
    }
    newUrl = url + "?" + paramBuffer.toString();
  } else {
    newUrl = url;
  }
  request = new HttpGet(newUrl);
} 
```
2）对于post形式的HTTPRequest，要发送的数据都存在Body里面，也是以键值对的形式，所以代码编写与get情形完全不同，如下所示：

```
else if (urlData.getNetType().equals(REQUEST_POST)) {
  request = new HttpPost(url);
  // 添加参数
  if ((parameter != null) && (parameter.size() > 0)) {
    final List<BasicNameValuePair> list = 
      new ArrayList<BasicNameValuePair>();
    for (final RequestParameter p : parameter) {
      list.add(new BasicNameValuePair(
        p.getName(), p.getValue()));
    }
    ((HttpPost) request).setEntity(
      new UrlEncodedFormEntity(list, HTTP.UTF_8));
  }
} 
```

**2.HTTP Header**

与Body相比，HTTP header就丰富的多了。它由很多键值对（key-value）组成，其中有些key是标准的，兼容于各大浏览器，比如：

* accept
* accept-language
* referrer
* user-agent
* accept-encoding

此外，我们还可以在MobileAPI端自定义一些键值对，然后要求App在调用MobileAPI时把这些信息传递过来。比如MobileAPI可以定义一个check-value这样的key，然后要求App将AppId（同一公司的不同App编号）、ClientType（Android还是iPhone、iPad）这些值拼接在一起经过MD5加密后，作为这个key的值传递给MobileAPI，然后由MobileAPI再去分析这些数据。
对于App开发人员而言，只要按照MobileAPI的要求，把这些key所需要的值拼接成HTTPRequest头正确传递过去即可。如下所示：

```
void setHttpHeaders(final HttpUriRequest httpMessage)
{
  headers.clear();
  headers.put(FrameConstants.ACCEPT_CHARSET, "UTF-8,*");
  headers.put(FrameConstants.USER_AGENT, 
              "Young Heart Android App ");
  if ((httpMessage != null) && (headers != null))
  {
    for (final Entry<String, String> entry : headers.entrySet())
    {
      if (entry.getKey()!=null)
      {
        httpMessage.addHeader(entry.getKey(), entry.getValue());
      }
    }
  }
} 
```

我们在组装Cookie之前调用setHttpHeaders方法：

```
// 添加必要的头信息
setHttpHeaders(request);
// 添加Cookie到请求头中
addCookie();
// 发送请求
response = httpClient.execute(request); 
```

而在返回数据时，也可以从HTTP Response头中把所需要的数据解析出来。Android SDK将其封装成了若干方法以供调用。我们在下面的章节将会看到。

前面我们介绍过Cookie，其实也是HTTP头的一部分。它的作用我们已经见识过了。下面将讨论HTTP头中的另几个重要字段。

####2.5.2　时间校准

接下来要介绍的是HTTP Response头中另一重要属性：Date，这个属性中记录了MobileAPI当时的服务器时间。

为什么说这个属性很重要呢？App开发人员经常遇到的一个bug就是，App显示的时间不准，经常会因为时区问题前后差几个小时，而接到用户的投诉。

为了解决这个问题，要从MobileAPI和App同时做一些工作。MobileAPI永远使用UTC时间。包括入参和返回值，都不要使用Date格式，而是减去UTC时间1970年1月1日的差值，这是一个long类型的长整数。

在App端比较麻烦。这里我们只讨论中国，比如国内航班时间、电影上映时间等等，那么我们把MobileAPI返回的long型时间转换为GMT8时区的时间就万事大吉了——只需要额外加8个小时。无论使用的人身在哪个时区，他们看到的都应该是一个时间，也就是GMT8的时间。

由于App本地时间会不准，比如前后差十几分钟，又比如设置了GMT9的时区，这样在取本地时间的时候，就会差一个小时。遇到这种情况，就要依赖于HTTP Response头的Date属性了。

每调用一次MobileAPI，就取出HTTP Response头的Date值，转换为GMT时间后，再减去本地取出的时间，得到一个差值delta。这个值可能是因为手机时间不准而差出来的那十几分钟，也可能是因为时区不同导致的1个小时差值。我们将这个delta值保存下来。那么每当取本地当前时间的时候，再额外加上这个delta差值，就得到了服务器GMT8的时间，就做到了任何人看到的时间是一样的。

因为App会频繁调用MobileAPI，所以这个delta值也会频繁更新，不用担心长期不调用MobileAPI而导致的这个delta值不太准的问题。

接下来我们修改AndroidLib框架，以支持上述的这些功能。

1）首先，在HTTPRequest类提供一个用于更新本地时间和服务器时间差值的方法updateDeltaBetweenServerAndClientTime，如下所示，由于我们在这里补上了UTC和GMT8相差的那8个小时，所以App其他地方不再需要考虑时差的问题，如下所示：

```
void updateDeltaBetweenServerAndClientTime() {
  if (response != null) {
    final Header header = response.getLastHeader("Date");
    if (header != null) {
      final String strServerDate = header.getValue();
      try {
        if ((strServerDate != null) && !strServerDate.equals("")) {
          final SimpleDateFormat sdf = new SimpleDateFormat(
            "EEE, d MMM yyyy HH:mm:ss z", Locale.ENGLISH);
          TimeZone.setDefault(TimeZone.getTimeZone("GMT+8"));
          Date serverDateUAT = sdf.parse(strServerDate);
          deltaBetweenServerAndClientTime = serverDateUAT
            .getTime()
            + 8 * 60 * 60 * 1000
            - System.currentTimeMillis();
        }
      } catch (java.text.ParseException e) {
        e.printStackTrace();
      }
    }
  }
} 
```

我们会在发起MobileAPI网络请求得到响应结果后，执行该方法，更新这个差值：

```
// 发送请求
response = httpClient.execute(request);
// 获取状态
final int statusCode = response.getStatusLine().getStatusCode();
// 设置回调函数, 但如果requestCallback, 说明不需要回调, 不需要知道返回结果
if ((requestCallback != null)) {
  if (statusCode == HttpStatus.SC_OK) {
    // 更新服务器时间和本地时间的差值
    updateDeltaBetweenServerAndClientTime(); 
```

因为我们的App会频繁的调用MobileAPI，所以为了避免频繁读写文件，我们没有将deltaBetweenServerAndClientTime存到本地文件，而是放在了内存中，当作一个全局变量来使用。

2）我们把这个deltaBetweenServerAndClientTime方法暴露出来，供外界调用：

```
public static Date getServerTime() {
  return new Date(System.currentTimeMillis()
                  + deltaBetweenServerAndClientTime);
} 
```

现在我们就可以模拟一个场景了。我把手机的时间改成任意一个值，然后再进入到WeatherByFastJsonActivity页面，因为页面加载的时候会调用MobileAPI获取天气的接口，所以本地会保存一个deltaBetweenServerAndClientTime差值。点击WeatherByFastJsonActivity页面上的“获取服务器时间”按钮，会因为我调用了AndroidLib中封装好的getServerTime方法，而弹出GMT8的当前时间：

```
btnShowTime.setOnClickListener(new View.OnClickListener() {
                               @Override
                               public void onClick(View v) {
  String strCurrentTime = Utils.getServerTime().toString();
  new AlertDialog.Builder(WeatherByFastJsonActivity.this)
    .setTitle("当前时间是：").setMessage(strCurrentTime)
    .setPositiveButton("确定", null).show();
}
}); 
```

####2.5.3　开启gzip压缩

HTTP协议上的gzip编码是一种用来改进Web应用程序性能的技术。大流量的Web站点常常使用gzip压缩技术来减少传输量的大小，减少传输量大小有两个明显的好处，一是可以减少存储空间，二是通过网络传输时，可以减少传输的时间。

使用gzip的流程如下：

1）在App发起请求时，在HTTPRequest头中，添加要求支持gzip的key-value，这里的key是Accept-Encoding，value是gzip。如下所示，我们需要修改setHttpHeaders方法：

```
void setHttpHeaders(final HttpUriRequest httpMessage) {
  headers.clear();
  headers.put(FrameConstants.ACCEPT_CHARSET, "UTF-8,*");
  headers.put(FrameConstants.USER_AGENT, "Young Heart Android App ");
  headers.put(FrameConstants.ACCEPT_ENCODING, "gzip");
  if ((httpMessage != null) && (headers != null)) {
    for (final Entry<String, String> entry : headers.entrySet()) {
      if (entry.getKey() != null) {
        httpMessage.addHeader(entry.getKey(), entry.getValue());
      }
    }
  }
} 
```

2）MobileAPI的逻辑是，检查HTTP请求头中的Accept-Encoding是否有gzip值，如果有，就会执行gzip压缩。

如果执行了gzip压缩，那么在返回值也就是HTTPResponse的头中，有一个content-encoding字段，会带有gzip的值；否则，就没有这个值。

3）App检查HTTPResponse头中的content-encoding字段是否包含gzip值，这个值的有无，导致了App解析HTTPResponse的姿势不同，如下所示值，这个值的有无，导致了App解析HTTPResponse的姿势不同，如下所示（以下代码参见HTTPRequest这个类）：

```
String strResponse = "";
if ((response.getEntity().getContentEncoding() != null)
    && (response.getEntity().getContentEncoding()
        .getValue() != null)) {
  if (response.getEntity().getContentEncoding()
      .getValue().contains("gzip")) {
    final InputStream in = response.getEntity()
      .getContent();
    final InputStream is = new GZIPInputStream(in);
                                               strResponse = HttpRequest.inputStreamToString(is);
    is.close();
  } else {
    response.getEntity().writeTo(content);
    strResponse = new String(content.toByteArray()).trim();
  }
} else {
  response.getEntity().writeTo(content);
  strResponse = new String(content.toByteArray()).trim();
} 
```