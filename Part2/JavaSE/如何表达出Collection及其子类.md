#如何介绍数据结构

1. 是泛型？接口？类？
2. 是个什么？
3. 相比父类的特点？
4. 相比于其他类型的特点？
5. 自己的特殊方法？
6. 子类？


#Collection
---
1. 是一个泛型的接口
2. 继承了超级接口Iterable
3. 每个Collection的对象包含了一组对象
4. 所有的实现类都有两个构造方法，一个是无参构造方法，第二个是用另外一个Collection对象作为构造方法的参数
5. 遍历Collection使用Iterator迭代器实现
6. retainAll(collection),AddAll(),removeAll(c)分别对应了集合的交并差运算
7. 没有具体的直接实现，但提供了更具体的子接口，如Set、List等


#List
---
1. 是一个接口，继承了接口Collection
2. List是有序的Collection，能够精确控制插入、获取的位置
3. 和Set接口的最大区别是，List允许重复值，Set不能
4. 它的直接实现类有ArrayList，LinkedList，Vector等
5. List有自己的迭代器ListIterator，可以通过这个迭代器进行逆序的迭代，以及用迭代器设置元素的值

#ArrayList
---
1. ArrayList实现了Collection接口
2. ArrayList是一个顺序表。大小可变。
3. ArrayList相比LinkedList在查找和修改元素上比较快，但是在添加和删除上比LinkedList慢
4. ArrayList相比Vector是线程不安全的


#LinkedList
---
1. LinkedList泛型接口
2. 链表
3. 由于实现了Deque接口，所以它还是一个双端队列