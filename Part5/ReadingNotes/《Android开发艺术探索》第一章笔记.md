#《Android开发艺术探索》第一章笔记
---

注：此篇笔记只记录重难点，对于基础和详细内容请自行学习《Android开发艺术探索》。


(1) onStart和onResume的区别是onStart可见，还没有出现在前台，无法和用户进行交互。onResume获取到焦点可以和用户交互。

(2) 新Activity是透明主题时，旧Activity不会走onStop；

(3)Activity切换时，旧Activity的onPause会先执行，然后才会启动新的Activity；

(4)Activity在异常情况下被回收时，onSaveInstanceState方法会被回调，回调时机是在onStop之前，当Activity被重新创建的时候，onRestoreInstanceState方法会被回调，时序在onStart之后；

(5)Activity的LaunchMode

a.
 standard 系统默认。每次启动会重新创建新的实例，谁启动了这个Activity，这个Activity就在谁的栈里。

b.
 singleTop 栈顶复用模式。该Activity的onNewIntent方法会被回调，onCreate和onStart并不会被调用。

c.
 singleTask 栈内复用模式。只要该Activity在一个栈中存在，都不会重新创建，onNewIntent会被回调。如果不存在，系统会先寻找是否存在需要的栈，如果不存在该栈，就创建一个任务栈，然后把这个Activity放进去；如果存在，就会创建到已经存在的这个栈中。

d.
 singleInstance。具有此种模式的Activity只能单独存在于一个任务栈。

(5) 标识Activity任务栈名称的属性：TaskAffinity，默认为应用包名。

(6) IntentFilter匹配规则。

a.
 action匹配规则：要求intent中的action存在且必须和过滤规则中的其中一个相同 区分大小写；

b.
 category匹配规则：系统会默认加上一个android.intent.category.DEAFAULT，所以intent中可以不存在category，但如果存在就必须匹配其中一个；

c.
 data匹配规则：data由两部分组成，mimeType和URI，要求和action相似。如果没有指定URI，URI但默认值为content和file（schema）