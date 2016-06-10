# List<E\>
***
* 是一个接口
* 继承的接口：Collection
* 间接继承的接口：Iterable
* 实现类：ArrayList、LinkedList、Vector等

##简介
* List是有序的**Collection**
	- 可以对每个元素的插入位置进行精准控制
	- 可以根据索引访问元素
* 允许重复元素
* 有自己的迭代器 ListIterator
* 如果元素包含自身，equals()和hashCode()不再是良定义的

## 方法
**boolean add(E e)**: 添加到末尾  
**void add(int index, E e)**: 添加到指定位置  

**E set(int index, E e)**: 设置指定位置的元素,返回一个E  
**get(int index)**: 获得指定位置的元素  

**Iterator iterator()**   
**ListIterator listIterator()**  
**ListIterator listIterator(int index)**  

**int indexOf(E e)**  
**int lastIndexOf(E e)**

**List<E> subList(int fromIndex, int toIndex)**

##子类介绍
1. ArrayList是一个可改变大小的数组，当更多的元素加入到ArrayList中时，其大小将会动态的增长。内部的元素可以直接通过get与set方法进行访问，因为ArrayList本质上就是一个数组
2. LinkedList 是一个双链表,在添加和删除元素时具有比ArrayList更好的性能.但在get与set方面弱于ArrayList.当然,这些对比都是指数据量很大或者操作很频繁的情况下的对比,如果数据和运算量很小,那么对比将失去意义。
LinkedList还实现了Queue接口，该接口比List提供了更多的方法，包括offer(),peek(),poll等
3. Vector 和ArrayList类似,但属于强同步类。如果你的程序本身是线程安全的(thread-safe,没有在多个线程之间共享同一个集合/对象),那么使用ArrayList是更好的选择。
Vector和ArrayList在更多元素添加进来时会请求更大的空间。Vector每次请求其大小的双倍空间，而ArrayList每次对size增长50%.





	