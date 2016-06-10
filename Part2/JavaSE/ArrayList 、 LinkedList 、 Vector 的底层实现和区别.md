#Java基础之集合List-ArrayList、LinkedList、Vector的底层实现和区别



1. ArrayList底层实际是采用数组实现的（并且该数组的类型是Object类型的）
2. 如果jdk6,采用Array.copyOf()方法来生成一个新的数组，如果是jdk5，采用的是System.arraycopy()方法（当添加的数据量大于数组的长度的时候）
3. List list = new ArrayList()时，底层会生成一个长度为10的数组来存放对象
4. ArrayList、Vector底部都是采用数组实现的
5. 对于ArrayList，方法都不是同步的，对于Vector，大部分public方法都是同步的
6. LinkedList采用双向循环列表
7. 对于ArrayList，查询速度很快，增加和删除（非最后一个节点）操作非常慢（本质上由数组的特性决定的）
8. 对于LinkedList，查询速度非常慢，增加和删除操作非常快（本质上是由双向循环列表决定的）

参考博客：

[http://blog.csdn.net/sundenskyqq/article/details/27630179](http://blog.csdn.net/sundenskyqq/article/details/27630179)
