#AIDL
---

1. 创建一个接口，再里面定义方法

```
package com.example.taidl;  
interface ICalcAIDL  
{  
    int add(int x , int y);  
    int min(int x , int y );  
} 
```

build一下gen目录下会生成ICalcAIDL.java文件

```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/dream/Downloads/android/androidProject/TAIDL/src/com/example/taidl/ICalcAIDL.aidl
 */
package com.example.taidl;
public interface ICalcAIDL extends android.os.IInterface
{
/** Local-side IPC implementation stub class. */
public static abstract class Stub extends android.os.Binder implements com.example.taidl.ICalcAIDL
{
private static final java.lang.String DESCRIPTOR = "com.example.taidl.ICalcAIDL";
/** Construct the stub at attach it to the interface. */
public Stub()
{
this.attachInterface(this, DESCRIPTOR);
}
/**
 * Cast an IBinder object into an com.example.taidl.ICalcAIDL interface,
 * generating a proxy if needed.
 */
public static com.example.taidl.ICalcAIDL asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof com.example.taidl.ICalcAIDL))) {
return ((com.example.taidl.ICalcAIDL)iin);
}
return new com.example.taidl.ICalcAIDL.Stub.Proxy(obj);
}
@Override public android.os.IBinder asBinder()
{
return this;
}
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
switch (code)
{
case INTERFACE_TRANSACTION:
{
reply.writeString(DESCRIPTOR);
return true;
}
case TRANSACTION_add:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
int _result = this.add(_arg0, _arg1);
reply.writeNoException();
reply.writeInt(_result);
return true;
}
case TRANSACTION_min:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
int _result = this.min(_arg0, _arg1);
reply.writeNoException();
reply.writeInt(_result);
return true;
}
}
return super.onTransact(code, data, reply, flags);
}
private static class Proxy implements com.example.taidl.ICalcAIDL
{
private android.os.IBinder mRemote;
Proxy(android.os.IBinder remote)
{
mRemote = remote;
}
@Override public android.os.IBinder asBinder()
{
return mRemote;
}
public java.lang.String getInterfaceDescriptor()
{
return DESCRIPTOR;
}
@Override public int add(int x, int y) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
int _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(x);
_data.writeInt(y);
mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
_reply.readException();
_result = _reply.readInt();
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public int min(int x, int y) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
int _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(x);
_data.writeInt(y);
mRemote.transact(Stub.TRANSACTION_min, _data, _reply, 0);
_reply.readException();
_result = _reply.readInt();
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
}
static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
static final int TRANSACTION_min = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
}
public int add(int x, int y) throws android.os.RemoteException;
public int min(int x, int y) throws android.os.RemoteException;
}

```

2. 新建一个Service

```
package com.example.taidl;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;

public class CalcService extends Service{

	private static final String TAG = "server";  
	
	public void onCreate()  
    {  
        Log.e(TAG, "onCreate");  
    }  
  
    public IBinder onBind(Intent t)  
    {  
        Log.e(TAG, "onBind");  
        return mBinder;  
    }  
  
    public void onDestroy()  
    {  
        Log.e(TAG, "onDestroy");  
        super.onDestroy();  
    }  
  
    public boolean onUnbind(Intent intent)  
    {  
        Log.e(TAG, "onUnbind");  
        return super.onUnbind(intent);  
    }  
  
    public void onRebind(Intent intent)  
    {  
        Log.e(TAG, "onRebind");  
        super.onRebind(intent);  
    }  
	private final ICalcAIDL.Stub mBinder = new ICalcAIDL.Stub() {
		
		@Override
		public int min(int x, int y) throws RemoteException {
			return x + y;
		}
		
		@Override
		public int add(int x, int y) throws RemoteException {
			// TODO Auto-generated method stub
			return x - y;
		}
	};
	
	

}

```

创建了一个mBinder对象，并在Service的onBind方法中返回

注册：

```
        <service android:name="com.example.taidl.CalcService">
            <intent-filter>  
               <action android:name="com.example.taidl.calc" />  
  
               <category android:name="android.intent.category.DEFAULT" />  
           </intent-filter>  
        </service>
```

我们一会会在别的应用程序中通过Intent来查找此Service；这个不需要Activity，所以我也就没写Activity，安装完成也看不到安装图标，悄悄在后台运行着。服务端编写完毕。下面开始编写客户端:

```
package com.example.tclient;

import com.example.taidl.ICalcAIDL;

import android.app.Activity;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.util.Log;
import android.view.View;
import android.widget.Toast;

public class MainActivity extends Activity {

	private ICalcAIDL mCalcAidl;

	private ServiceConnection mServiceConn = new ServiceConnection()
	{
		@Override
		public void onServiceDisconnected(ComponentName name)
		{
			Log.e("client", "onServiceDisconnected");
			mCalcAidl = null;
		}

		@Override
		public void onServiceConnected(ComponentName name, IBinder service)
		{
			Log.e("client", "onServiceConnected");
			mCalcAidl = ICalcAIDL.Stub.asInterface(service);
		}
	};

	@Override
	protected void onCreate(Bundle savedInstanceState)
	{
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

	}
	
	/**
	 * 点击BindService按钮时调用
	 * @param view
	 */
	public void bindService(View view)
	{
		Intent intent = new Intent();
		intent.setAction("com.example.taidl.calc");
		bindService(intent, mServiceConn, Context.BIND_AUTO_CREATE);
	}
	/**
	 * 点击unBindService按钮时调用
	 * @param view
	 */
	public void unbindService(View view)
	{
		unbindService(mServiceConn);
	}
	/**
	 * 点击12+12按钮时调用
	 * @param view
	 */
	public void addInvoked(View view) throws Exception
	{

		if (mCalcAidl != null)
		{
			int addRes = mCalcAidl.add(12, 12);
			Toast.makeText(this, addRes + "", Toast.LENGTH_SHORT).show();
		} else
		{
			Toast.makeText(this, "服务器被异常杀死，请重新绑定服务端", Toast.LENGTH_SHORT)
					.show();

		}

	}
	/**
	 * 点击50-12按钮时调用
	 * @param view
	 */
	public void minInvoked(View view) throws Exception
	{

		if (mCalcAidl != null)
		{
			int addRes = mCalcAidl.min(50, 12);
			Toast.makeText(this, addRes + "", Toast.LENGTH_SHORT).show();
		} else
		{
			Toast.makeText(this, "服务器未绑定或被异常杀死，请重新绑定服务端", Toast.LENGTH_SHORT)
					.show();

		}

	}
}

```

将服务端的aidl文件完整的复制过来，包名一定要一致。

##分析AIDL生成的代码

1. 服务端

```
private final ICalcAIDL.Stub mBinder = new ICalcAIDL.Stub()  
    {  
  
        @Override  
        public int add(int x, int y) throws RemoteException  
        {  
            return x + y;  
        }  
  
        @Override  
        public int min(int x, int y) throws RemoteException  
        {  
            return x - y;  
        }  
  
    };  

```

ICalcAILD.Stub来执行的，让我们来看看Stub这个类的声明：

```
public static abstract class Stub extends android.os.Binder implements com.zhy.calc.aidl.ICalcAIDL  
```

清楚的看到这个类是Binder的子类，是不是符合我们文章开通所说的服务端其实是一个Binder类的实例
接下来看它的onTransact()方法：

```
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException  
{  
switch (code)  
{  
case INTERFACE_TRANSACTION:  
{  
reply.writeString(DESCRIPTOR);  
return true;  
}  
case TRANSACTION_add:  
{  
data.enforceInterface(DESCRIPTOR);  
int _arg0;  
_arg0 = data.readInt();  
int _arg1;  
_arg1 = data.readInt();  
int _result = this.add(_arg0, _arg1);  
reply.writeNoException();  
reply.writeInt(_result);  
return true;  
}  
case TRANSACTION_min:  
{  
data.enforceInterface(DESCRIPTOR);  
int _arg0;  
_arg0 = data.readInt();  
int _arg1;  
_arg1 = data.readInt();  
int _result = this.min(_arg0, _arg1);  
reply.writeNoException();  
reply.writeInt(_result);  
return true;  
}  
}  
return super.onTransact(code, data, reply, flags);  
}  
```

文章开头也说到服务端的Binder实例会根据客户端依靠Binder驱动发来的消息，执行onTransact方法，然后由其参数决定执行服务端的代码。
可以看到onTransact有四个参数
code ， data ，replay ， flags

* code 是一个整形的唯一标识，用于区分执行哪个方法，客户端会传递此参数，告诉服务端执行哪个方法
* data客户端传递过来的参数
* replay服务器返回回去的值
* flags标明是否有返回值，0为有（双向），1为没有（单向）

我们仔细看case TRANSACTION_min中的代码

data.enforceInterface(DESCRIPTOR);

与客户端的writeInterfaceToken对用，标识远程服务的名称

```
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
```

接下来分别读取了客户端传入的两个参数

```
int _result = this.min(_arg0, _arg1);
reply.writeNoException();
reply.writeInt(_result);
```

然后执行this.min，即我们实现的min方法；返回result由reply写回。

add同理，可以看到服务端通过AIDL生成Stub的类，封装了服务端本来需要写的代码。

###客户端

客户端主要通过ServiceConnected与服务端连接

```
private ServiceConnection mServiceConn = new ServiceConnection()  
    {  
        @Override  
        public void onServiceDisconnected(ComponentName name)  
        {  
            Log.e("client", "onServiceDisconnected");  
            mCalcAidl = null;  
        }  
  
        @Override  
        public void onServiceConnected(ComponentName name, IBinder service)  
        {  
            Log.e("client", "onServiceConnected");  
            mCalcAidl = ICalcAIDL.Stub.asInterface(service);  
        }  
    };  
```

如果你比较敏锐，应该会猜到这个onServiceConnected中的IBinder实例，其实就是我们文章开通所说的Binder驱动，也是一个Binder实例
在ICalcAIDL.Stub.asInterface中最终调用了：

```
return new com.zhy.calc.aidl.ICalcAIDL.Stub.Proxy(obj);  
```
这个Proxy实例传入了我们的Binder驱动，并且封装了我们调用服务端的代码，文章开头说，客户端会通过Binder驱动的transact()方法调用服务端代码

直接看Proxy中的add方法

```
@Override public int add(int x, int y) throws android.os.RemoteException  
{  
android.os.Parcel _data = android.os.Parcel.obtain();  
android.os.Parcel _reply = android.os.Parcel.obtain();  
int _result;  
try {  
_data.writeInterfaceToken(DESCRIPTOR);  
_data.writeInt(x);  
_data.writeInt(y);  
mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);  
_reply.readException();  
_result = _reply.readInt();  
}  
finally {  
_reply.recycle();  
_data.recycle();  
}  
return _result;  
}  
```

首先声明两个Parcel对象，一个用于传递数据，一个用户接收返回的数据

```
_data.writeInterfaceToken(DESCRIPTOR);与服务器端的enforceInterfac对应
_data.writeInt(x);
_data.writeInt(y);写入需要传递的参数
mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
```

终于看到了我们的transact方法，第一个对应服务端的code,_data,_repay分别对应服务端的data，reply，0表示是双向的

```
_reply.readException();
_result = _reply.readInt();
```

最后读出我们服务端返回的数据，然后return。可以看到和服务端的onTransact基本是一行一行对应的。

我们已经通过AIDL生成的代码解释了Android Binder框架的工作原理。Service的作用其实就是为我们创建Binder驱动，即服务端与客户端连接的桥梁。
