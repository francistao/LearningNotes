#Builder模式
---

##模式介绍
---

###模式的定义

将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

###模式的使用场景

1. 相同的方法，不同的执行顺序，产生不同的事件结果时；
2. 多个部件或零件，都可以装配到一个对象中，但是产生的运行结果又不相同时；
3. 产品类非常复杂，或者产品类中的调用顺序不同产生了不同的效能，这个时候使用建造者模式非常合适；


###Android源码中的模式实现

在Android源码中，我们最常用到的Builder模式就是AlertDialog.Builder， 使用该Builder来构建复杂的AlertDialog对象。简单示例如下 :

```
//显示基本的AlertDialog  
    private void showDialog(Context context) {  
        AlertDialog.Builder builder = new AlertDialog.Builder(context);  
        builder.setIcon(R.drawable.icon);  
        builder.setTitle("Title");  
        builder.setMessage("Message");  
        builder.setPositiveButton("Button1",  
                new DialogInterface.OnClickListener() {  
                    public void onClick(DialogInterface dialog, int whichButton) {  
                        setTitle("点击了对话框上的Button1");  
                    }  
                });  
        builder.setNeutralButton("Button2",  
                new DialogInterface.OnClickListener() {  
                    public void onClick(DialogInterface dialog, int whichButton) {  
                        setTitle("点击了对话框上的Button2");  
                    }  
                });  
        builder.setNegativeButton("Button3",  
                new DialogInterface.OnClickListener() {  
                    public void onClick(DialogInterface dialog, int whichButton) {  
                        setTitle("点击了对话框上的Button3");  
                    }  
                });  
        builder.create().show();  // 构建AlertDialog， 并且显示
    } 
```

##优点与缺点
---

###优点

* 良好的封装性， 使用建造者模式可以使客户端不必知道产品内部组成的细节；
* 建造者独立，容易扩展；
* 在对象创建过程中会使用到系统中的一些其它对象，这些对象在产品对象的创建过程中不易得到。

###缺点

* 会产生多余的Builder对象以及Director对象，消耗内存；
* 对象的构建过程暴露。