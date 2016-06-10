
####线性表，链表，哈希表是常用的数据结构，在进行Java开发时，JDK已经为我们提供了一系列相应的类来实现基本的数据结构。这些类均在java.util包中。本文试图通过简单的描述，向读者阐述各个类的作用以及如何正确使用这些类。 

# Collection<E\><A NAME="Collection"> </a>
***
* 是最基本的集合接口
* 继承的接口：Iterable
* 子接口：List、Set、Queue等

##简介
* 一个Collection代表一组Object，即Collection的元素（Elements），一些Collection允许相同的元素而另一些不行。一些能排序而另一些不行。
* Java SDK不提供直接继承自Collection的类，Java SDK提供的类都是继承自Collection的“子接口”如List和Set。

##如何遍历Collection中的每一个元素？
* 不论Collection的实际类型如何，它都支持一个iterator()的方法，该方法返回一个迭代子，使用该迭代子即可逐一访问Collection中每一个元素，代码如下：



```

	Iterator it = collection.iterator(); // 获得一个迭代子
    while(it.hasNext())　　
    {
        Object obj = it.next(); // 得到下一个元素
    }
```

## 方法

**retainAll(Collection<? extends E\> c)**：保留，交运算  
**addAll(Collection<? extends E\> c)**：添加，并运算  
**removeAll(Collection<? extends E\> c)**：移除，减运算  




