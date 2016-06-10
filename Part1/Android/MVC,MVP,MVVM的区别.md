#MVC,MVP,MVVM的区别
---


#MVC
软件可以分为三部分

* 视图（View）:用户界面
* 控制器（Controller）:业务逻辑
* 模型（Model）:数据保存

各部分之间的通信方式如下：

1. View传送指令到Controller
2. Controller完成业务逻辑后，要求Model改变状态
3. Model将新的数据发送到View，用户得到反馈

Tips：所有的通信都是单向的。

#互动模式
接受用户指令时，MVC可以分为两种方式。一种是通过View接受指令，传递给Controller。

另一种是直接通过Controller接受指令


#MVP

MVP模式将Controller改名为Presenter，同时改变了通信方向。

1. 各部分之间的通信，都是双向的
2. View和Model不发生联系，都通过Presenter传递
3. View非常薄，不部署任何业务逻辑，称为"被动视图"(Passive View)，即没有任何主动性，而Presenter非常厚，所有逻辑都部署在那里。


#MVVM

MVVM模式将Presenter改名为ViewModel，基本上与MVP模式完全一致。

唯一的区别是，它采用双向绑定(data-binding)：View的变动，自动反映在ViewModel，反之亦然。





