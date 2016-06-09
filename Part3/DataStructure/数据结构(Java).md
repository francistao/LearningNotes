#数据结构和算法：
---
(推荐书籍：剑指offer、编程之美、Cracking、程序员代码面试指南，特别是这四本书上的重复题)

**数组与链表**

####数组：

创建

```
int c[] = {2,3,6,10,99};
int[] d = new int[10];
```



```
	/**
	 * 数组检索
	 * @param args
	 */
	public static void main(String[] args) {
		String name[];
		
		name = new String[5];
		name[0] = "egg";
		name[1] = "erqing";
		name[2] = "baby";
		
		for(int i = 0; i < name.length; i++){
			System.out.println(name[i]);
		}
	}
```
```
/**
	 * 插入
	 * 
	 * @param old
	 * @param value
	 * @param index
	 * @return
	 */
	public static int[] insert(int[] old, int value, int index) {  
        for (int k = old.length - 1; k > index; k--)  
            old[k] = old[k - 1];  
        old[index] = value;  
        return old;  
    }  

	/**
	 * 遍历
	 * 
	 * @param data
	 */
	public static void traverse(int data[]) {
		for (int j = 0; j < data.length; j++) {
			System.out.println(data[j] + " ");
		}
	}

	/**
	 * 删除
	 * 
	 * @param old
	 * @param index
	 * @return
	 */
	public static int[] delete(int[] old, int index) {
		for (int h = index; h < old.length - 1; h++) {
			old[h] = old[h + 1];
		}
		old[old.length - 1] = 0;
		return old;
	}
```

Tips:数组中删除和增加元素的原理：增加元素，需要将index后面的依次往后移动，然后将值插入index位置，删除则是将后面的值一次向前移动。

数组表示相同类型的一类数据的集合，下标从0开始。


####单链表：

![](http://img.my.csdn.net/uploads/201304/13/1365855052_1221.jpg)



**队列和栈，出栈与入栈。**

**链表的删除、插入、反向。**

**字符串操作。**

**Hash表的hash函数，冲突解决方法有哪些。**

**各种排序：冒泡、选择、插入、希尔、归并、快排、堆排、桶排、基数的原理、平均时间复杂度、最坏时间复杂度、空间复杂度、是否稳定。**

**快排的partition函数与归并的Merge函数。**

**对冒泡与快排的改进。**

**二分查找，与变种二分查找。**

**二叉树、B+树、AVL树、红黑树、哈夫曼树。**

**二叉树的前中后续遍历：递归与非递归写法，层序遍历算法。**

二叉树的前序遍历：



**图的BFS与DFS算法，最小生成树prim算法与最短路径Dijkstra算法。**

**KMP算法。**

**排列组合问题。**

**动态规划、贪心算法、分治算法。（一般不会问到）**

**大数据处理：类似10亿条数据找出最大的1000个数.........等等**