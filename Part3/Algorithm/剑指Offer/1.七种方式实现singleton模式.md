#面试题2.七种方式实现Singleton模式
---
```

public class Test {

	/**
	 * 单例模式，懒汉式，线程安全
	 */
	public static class Singleton {
		private final static Singleton INSTANCE = new Singleton();

		private Singleton() {

		}

		public static Singleton getInstance() {
			return INSTANCE;
		}
	}

	/**
	 * 单例模式，懒汉式，线程不安全
	 */
	public static class Singleton2 {
		private static Singleton2 instance = null;

		private Singleton2() {

		}

		public static Singleton2 getInstance() {
			if (instance == null) {
				instance = new Singleton2();
			}
			return instance;
		}
	}

	/**
	 * 单例模式，饿汉式，线程安全，多线程环境下效率不高
	 */
	public static class Singleton3 {
		private static Singleton3 instance = null;

		private Singleton3() {

		}

		public static synchronized Singleton3 getInstance() {
			if (instance == null) {
				instance = new Singleton3();
			}
			return instance;
		}
	}

	/**
	 * 单例模式，懒汉式，变种，线程安全
	 */
	public static class Singleton4 {
		private static Singleton4 instance = null;

		static {
			instance = new Singleton4();
		}

		private Singleton4() {

		}

		public static Singleton4 getInstance() {
			return instance;
		}
	}

	/**
	 * 单例模式，使用静态内部类，线程安全（推荐）
	 */
	public static class Singleton5 {
		private final static class SingletonHolder {
			private static final Singleton5 INSTANCE = new Singleton5();
		}

		private static Singleton5 getInstance() {
			return SingletonHolder.INSTANCE;
		}
	}

	/**
	 * 静态内部类，使用枚举方式，线程安全（推荐）
	 */
	public enum Singleton6 {
		INSTANCE;
		public void whateverMethod() {

		}
	}

	/**
	 * 静态内部类，使用双重校验锁，线程安全（推荐）
	 */
	public static class Singleton7 {
		private volatile static Singleton7 instance = null;

		private Singleton7() {

		}

		public static Singleton7 getInstance() {
			if (instance == null) {
				synchronized (Singleton7.class) {
					if (instance == null) {
						instance = new Singleton7();
					}
				}
			}
			return instance;
		}
	}

}

```
