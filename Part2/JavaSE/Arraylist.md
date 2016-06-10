# ArrarList<E\><A NAME="Collection"> </a>
---
* 是一个类
* 实现的接口：List、Collectin、Iterable、Serializable、Cloneable、RandomAccess
* 子类：AttributeList、RoleList、RoleUnresolvedList

##简介
---
* 它是顺序表
* 大小可变
* 允许null元素
* 可LinkedList一样，不具备线程同步的安全性、但速度较快
* 第一次定义的时候没有指定数组的长度则长度是0，在第一次添加的时候判断如果是空则追加10。

##容量是如何变化的
---
在源码中：

```
private transient Object[] elementData;
```
用于保存对象。它会随着元素的添加而改变大小。在下面的叙述中，容量指的是elementData.length，而不是size。

>size不是容量，ArrayList对象占用的内存不是由size决定的。
>size的大小在每次调用add(E e)方法时加1。
>如果是调用ArrayList(Collection<? extends E> c)构造方法，则size的初始值为c

* 如果在初始化时，没有指定大小，则容量大小为0。
* 当大小为0时，第一次添加元素容量大小变为10。
* 之后每次添加时，会先确保容量是否够用
* 如果不够用，每次增加的容量为 newCapacity = oldCapacity + (oldCapacity >> 1)；即，每次增加为原来的1.5倍。

##和Vector的区别
---
* Vector线程安全
* ArrayList线程不安全，速度快

##和LinkedList的区别
---
* ArrayList是顺序表，LinkedList是链表
* ArrayList查找，修改方便，LinkedList添加、删除方便

##方法
---
**void ensureCapacity(int minCapacity)**
设置容量能容纳下minCapacity个元素

* 如果