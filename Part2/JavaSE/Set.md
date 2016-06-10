#Set
---
* 是一个泛型接口
* 继承了接口Collection
* 子接口：NavigableSet、SortedSet
* 子类：EnumSet、HashSet、LinkedHashSet、TreeSet、AbstractSet等
* 不允许重复元素

两个注意点
---

```
1. Set中的元素的类，必须有一个有效的equals方法。
2. 对Set的构造方法，传入的Collection对象中重复的元素会只留下一个
```