#Thread和Runnable实现多线程的区别
---

Java中实现多线程有两种方法：继承Thread、实现Runnable接口，在程序开发中只要是多线程，肯定永远以实现Runnable接口为主，因为实现Runnable接口相比继承Thread类有如下优势：

1. 可以避免由于Java的单继承特性而带来的局限
2. 增强程序的健壮性，代码能够被多个线程共享，代码与数据是独立的
3. 适合多个相同程序的线程区处理同一资源的情况

首先通过Thread类实现

```
class MyThread extends Thread{  
    private int ticket = 5;  
    public void run(){  
        for (int i=0;i<10;i++)  
        {  
            if(ticket > 0){  
                System.out.println("ticket = " + ticket--);  
            }  
        }  
    }  
}  
  
public class ThreadDemo{  
    public static void main(String[] args){  
        new MyThread().start();  
        new MyThread().start();  
        new MyThread().start();  
    }  
} 
```

运行结果：

```
ticket = 5
ticket = 4
ticket = 5
ticket = 5
ticket = 4
ticket = 3
ticket = 2
ticket = 1
ticket = 4
ticket = 3
ticket = 3
ticket = 2
ticket = 1
ticket = 2
ticket = 1
```

每个线程单独卖了5张票，即独立的完成了买票的任务，但实际应用中，比如火车站售票，需要多个线程去共同完成任务，在本例中，即多个线程共同买5张票。

通过实现Runnable借口实现的多线程程序

```
class MyThread implements Runnable{  
    private int ticket = 5;  
    public void run(){  
        for (int i=0;i<10;i++)  
        {  
            if(ticket > 0){  
                System.out.println("ticket = " + ticket--);  
            }  
        }  
    }  
}  
  
public class RunnableDemo{  
    public static void main(String[] args){  
        MyThread my = new MyThread();  
        new Thread(my).start();  
        new Thread(my).start();  
        new Thread(my).start();  
    }  
} 
```

运行结果

```
ticket = 5
ticket = 2
ticket = 1
ticket = 3
ticket = 4
```

* 在第二种方法(Runnable)中，ticket输出的顺序并不是54321，这是因为线程执行的时机难以预测。ticket并不是原子操作。
* 在第一种方法中，我们new了3个Thread对象，即三个线程分别执行三个对象中的代码，因此便是三个线程去独立地完成卖票的任务；而在第二种方法中，我们同样也new了3个Thread对象，但只有一个Runnable对象，3个Thread对象共享这个Runnable对象中的代码，因此，便会出现3个线程共同完成卖票任务的结果。如果我们new出3个Runnable对象，作为参数分别传入3个Thread对象中，那么3个线程便会独立执行各自Runnable对象中的代码，即3个线程各自卖5张票。
* 在第二种方法中，由于3个Thread对象共同执行一个Runnable对象中的代码，因此可能会造成线程的不安全，比如可能ticket会输出-1（如果我们System.out....语句前加上线程休眠操作，该情况将很有可能出现），这种情况的出现是由于，一个线程在判断ticket为1>0后，还没有来得及减1，另一个线程已经将ticket减1，变为了0，那么接下来之前的线程再将ticket减1，便得到了-1。这就需要加入同步操作（即互斥锁），确保同一时刻只有一个线程在执行每次for循环中的操作。而在第一种方法中，并不需要加入同步操作，因为每个线程执行自己Thread对象中的代码，不存在多个线程共同执行同一个方法的情况。
