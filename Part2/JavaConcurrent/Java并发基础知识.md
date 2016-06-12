#Java并发
---

（Executor框架和多线程基础）

**Thread与Runable如何实现多线程**

Java 5以前实现多线程有两种实现方法：一种是继承Thread类；另一种是实现Runnable接口。两种方式都要通过重写run()方法来定义线程的行为，推荐使用后者，因为Java中的继承是单继承，一个类有一个父类，如果继承了Thread类就无法再继承其他类了，显然使用Runnable接口更为灵活。

实现Runnable接口相比继承Thread类有如下优势：

1. 可以避免由于Java的单继承特性而带来的局限
2. 增强程序的健壮性，代码能够被多个程序共享，代码与数据是独立的
3. 适合多个相同程序代码的线程区处理同一资源的情况

补充：Java 5以后创建线程还有第三种方式：实现Callable接口，该接口中的call方法可以在线程执行结束时产生一个返回值，代码如下所示：

```
class MyTask implements Callable<Integer> {  
    private int upperBounds;  
      
    public MyTask(int upperBounds) {  
        this.upperBounds = upperBounds;  
    }  
      
    @Override  
    public Integer call() throws Exception {  
        int sum = 0;   
        for(int i = 1; i <= upperBounds; i++) {  
            sum += i;  
        }  
        return sum;  
    }  
      
}  
  
public class Test {  
  
    public static void main(String[] args) throws Exception {  
        List<Future<Integer>> list = new ArrayList<>();  
        ExecutorService service = Executors.newFixedThreadPool(10);  
        for(int i = 0; i < 10; i++) {  
            list.add(service.submit(new MyTask((int) (Math.random() * 100))));  
        }  
          
        int sum = 0;  
        for(Future<Integer> future : list) {  
            while(!future.isDone()) ;  
            sum += future.get();  
        }  
          
        System.out.println(sum);  
    }  
}  
```

**线程同步的方法有什么；锁，synchronized块，信号量等**



**锁的等级：方法锁、对象锁、类锁**

**生产者消费者模式的几种实现，阻塞队列实现，sync关键字实现，lock实现,reentrantLock等**

**ThreadLocal的设计理念与作用，ThreadPool用法与优势（这里在Android SDK原生的AsyncTask底层也有使用）**

**线程池的底层实现和工作原理（建议写一个雏形简版源码实现）**

**几个重要的线程api，interrupt，wait，sleep，stop等等**

**写出生产者消费者模式。**

**ThreadPool用法与优势。**

**Concurrent包里的其他东西：ArrayBlockingQueue、CountDownLatch等等。**

**wait()和sleep()的区别。**

sleep()方法是线程类（Thread）的静态方法，导致此线程暂停执行指定时间，将执行机会给其他线程，但是监控状态依然保持，到时后会自动恢复（线程回到就绪（ready）状态），因为调用sleep 不会释放对象锁。wait()是Object 类的方法，对此对象调用wait()方法导致本线程放弃对象锁(线程暂停执行)，进入等待此对象的等待锁定池，只有针对此对象发出notify 方法（或notifyAll）后本线程才进入对象锁定池准备获得对象锁进入就绪状态。



###3、IO（IO,NIO，目前okio已经被集成Android包）

**IO框架主要用到什么设计模式**

JDK的I/O包中就主要使用到了两种设计模式：Adatper模式和Decorator模式。


**NIO包有哪些结构？分别起到的作用？**


**NIO针对什么情景会比IO有更好的优化？**

**OKIO底层实现**

