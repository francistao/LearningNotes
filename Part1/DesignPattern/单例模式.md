#单例模式
---

###定义
>保证一个类仅有一个实例，并提供一个访问它的全局访问点。

>Singleton:负责创建Singleton类自己的唯一实例，并提供一个getInstance的方法，让外部来访问这个类的唯一实例。


* 饿汉式：
	```
	private static Singleton uniqueInstance = new Singleton();
	```
* 懒汉式
	```
	private static Singleton uniqueInstance = null;
	```

###功能
单例模式是用来保证这个类在运行期间只会被创建一个类实例，另外，单例模式还提供了一个全局唯一访问这个类实例的访问点，就是getInstance方法。

###范围
Java里面实现的单例是一个虚拟机的范围。因为装载类的功能是虚拟机的，所以一个虚拟机在通过自己的ClassLoader装载饿汉式实现单例类的时候就会创建一个类的实例。

懒汉式单例有延迟加载和缓存的思想

###优缺点
* 懒汉式是典型的时间换空间
* 饿汉式是典型的空间换时间

---
* 不加同步的懒汉式是线程不安全的。比如，有两个线程，一个是线程A，一个是线程B，它们同时调用getInstance方法，就可能导致并发问题。

* 饿汉式是线程安全的，因为虚拟机保证只会装载一次，在装载类的时候是不会发生并发的。

---

如何实现懒汉式的线程安全？

加上synchronized即可

```
public static synchronized Singleton getInstance(){}
```

但这样会降低整个访问的速度，而且每次都要判断。可以用双重检查加锁。

双重加锁机制，指的是：并不是每次进入getInstance方法都需要同步，而是先不同步，进入方法过后，先检查实例是否存在，如果不存在才进入下面的同步块，这是第一重检查。进入同步块后，再次检查实例是否存在，如果不存在，就在同步的情况下创建一个实例。这是第二重检查。

双重加锁机制的实现会使用一个关键字volatile，它的意思是：被volatile修饰的变量的值，将不会被本地线程缓存，所有对该变量的读写都是直接操作共享内存，从而确保多个线程能正确的处理该变量。

```

/**
 * 双重检查加锁的单例模式
 * @author dream
 *
 */
public class Singleton {

	/**
	 * 对保存实例的变量添加volitile的修饰
	 */
	private volatile static Singleton instance = null;
	private Singleton(){
		
	}
	
	public static Singleton getInstance(){
		//先检查实例是否存在，如果不存在才进入下面的同步块
		if(instance == null){
			//同步块，线程安全的创建实例
			synchronized (Singleton.class) {
				//再次检查实例是否存在，如果不存在才真正的创建实例
				instance = new Singleton();
			}
		}
		return instance;
	}
	
}

```

###一种更好的单例实现方式

```
public class Singleton {

	/**
	 * 类级的内部类，也就是静态类的成员式内部类，该内部类的实例与外部类的实例
	 * 没有绑定关系，而且只有被调用时才会装载，从而实现了延迟加载
	 * @author dream
	 *
	 */
	private static class SingletonHolder{
		/**
		 * 静态初始化器，由JVM来保证线程安全
		 */
		private static final Singleton instance = new Singleton();
	}
	
	/**
	 * 私有化构造方法
	 */
	private Singleton(){
		
	}
	
	public static Singleton getInstance(){
		return SingletonHolder.instance;
	}
}

```

根据《高效Java第二版》中的说法，单元素的枚举类型已经成为实现Singleton的最佳方法。

```
package example6;

/**
 * 使用枚举来实现单例模式的示例
 * @author dream
 *
 */
public class Singleton {

	/**
	 * 定义一个枚举的元素，它就代表了Singleton的一个实例
	 */
	uniqueInstance;
	
	/**
	 * 示意方法，单例可以有自己的操作
	 */
	public void singletonOperation(){
		//功能树立
	}
}

```
---

###本质
控制实例数量

###何时选用单例模式
当需要控制一个类的实例只能有一个，而且客户只能从一个全局访问点访问它时，可以选用单例模式，这些功能恰好是单例模式要解决的问题。

