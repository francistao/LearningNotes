##Thread与Runable如何实现多线程?

Java实现多线程有两种途径：继承Thread类或者实现Runnable接口。Runnable是接口

Runnable是接口，建议用接口的方式生成线程，因为接口可以实现多继承，况且Runnable只有一个run方法，很适合继承。

在使用Thread方法的时候只需继承Thread，并且new一个实例出来，调用start()方法既可以启动一个线程

[ java多线程 Thread 和Runnable](http://blog.csdn.net/luyysea/article/details/7995351)