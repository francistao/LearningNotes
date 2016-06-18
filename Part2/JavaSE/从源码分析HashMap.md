#HashMap
---


###HashMap和Hashtable的区别：

1. Hashtable的大部分方法做了同步，HashMap没有，因此，HashMap不是线程安全的。
2. Hashtable不允许key或者value使用null值，而HashMap可以。
3. 在内部算法上，它们对key的hash算法和hash值到内存索引的映射算法不同。

###HashMap的实现原理

简单说，HashMap就是将key做hash算法，然后将hash所对应的数据映射到内存地址，直接取得key所对应的数据。在HashMap中。底层数据结构使用的是数组，所谓的内存地址即数组的下标索引。HashMap的高性能需要保证以下几点：

* hash算法必须高效
* hash值到内存地址(数组索引)的算法是快速的
* 根据内存地址(数组索引)可以直接取得对应的值

如何保证hash算法高效,hash算法有关的代码如下：

```
int hash = hash(key.hashCode());
public native int hashCode();
static int hash(int h){
	h ^= (h >>> 20) ^ (h >>> 12);
	return h ^ (h >>> 7) ^ (h >>> 4);
}
```

第一行代码是HashMap用于计算key的hash值，它前后调用了Object类的hashCode()方法和HashMap的内部函数hash()。Object类的hashCode()方法默认是native的实现，可以认为不存在性能问题。而hash()函数的实现全部基于位运算，因此，也是高效的。

当取得key的hash值后，需要通过hash值得到内存地址：

```
int i = indexFor(hash, table.length);
static int indexFor(int h, int length){
	return h & (length - 1);
}
```

indexFor()函数通过将hash值和数组长度按位与直接得到数组索引。
最后由indexFor()函数返回的数组索引直接通过数组下标便可取得对应的值，直接的内存访问速度也是相当的快，因此，可认为HashMap是高性能的。

###Hash冲突

如图3.11所示，需要存放到HashMap中的两个元素1和2，通过hash计算后，发现对应在内存中的同一个地址，如何处理？
其实HashMap的底层实现使用的是数组，但是数组内的元素并不是简单的值。而是一个Entry类的对象。因此，对HashMap结构贴切描述如图3.12所示。

![这里写图片描述](http://img.blog.csdn.net/20160509103524275)


可以看到，HashMap的内部维护着一个Entry数组，每一个Entry表项包括key、value、next和hash几项。next部分指向另外一个Entry。进一步阅读HashMap的put()方法源码，可以看到当put()操作有冲突时，新的Entry依然会被安放在对应的索引下标内，并替换原有的值。同时为了保证旧值不丢失，会将新的Entry的next指向旧值。这便实现了在一个数组索引空间内存放多个值项。因此，如图3.12所示，HashMap实际上是一个链表的数组。

```
public V put(K key, V value){
	if(key == null)
		return putForNullKey(value);
	int hash = hash(key.hashCode());
	int i = indexFor(hash, table.length);
	for(Entry<K, V> e = table[i]; e != null; e = e.next){
		Object k;
		//如果当前的key已经存在于HashMap中
		if(e.hash == hash && ((k = e.key) == key || key.equals(k)))
		{
			V oldValue = e.value;    //取得旧值
			e.value = value;
			e.recordAccess(this);
			return oldValue;     //返回旧值
		}
	}
	modCount++;
	addEntry(hash, key, value, i);    //添加当前的表项到i位置
	return null;
}
```

addEntry()方法的实现如下：

```
void addEntry(int hash, K key, V value, int bucketIndex){
	Entry<K,V> e = table[bucketIndex];
	//将新增元素放到i的位置，并让它的next指向旧的元素
	table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
	if(size++ >= threshold){
		resize(2 * table.length);
	}
}
```

基于HashMap的这种实现机制，只要hashCode和hash()方法实现的足够好，能够尽可能的减少冲突的产生，那么对HashMap的操作几乎等价于对数组的随机访问操作，具有很好的性能。但是，如果hashCode()或者hash()方法实现较差，在大量冲突产生的情况下，HashMap事实上就退化为几个链表，对HashMap的操作等价于遍历链表，此时性能很差。

###容量参数

除hashCode()的实现外，影响HashMap性能的还有它的容量参数。和ArrayList和Vector一样，这种基于数组的结构，不可避免的需要在数组空间不足时，进行扩展。而数组的重组相对而言较为耗时，因此对其作一定了解有助于优化HashMap的性能。

HashMap提供了两个可以指定初始化大小的构造函数：

```
public HashMap(int initialCapacity)
public HashMap(int initialCapacity, float loadFactor)
```

其中initialCapacity指定了HashMap的初始容量，loadFactor指定了其负载因子。初始容量即数组的大小，HashMap会使用大于等于initialCapacity并且是2的指数次幂的最小的整数作为内置数组的大小。负载因子又叫填充比，它是介于0和1之间的浮点数，它决定了HashMap在扩容之前，其内部数组的填充度。默认情况下，HashMap初始大小为16，负载因子为0.75。

**负载因子 ＝ 元素个数/内部数组总大小**

在实际使用中，负载因子也可以设置为大于1的数，但如果这样做，HashMap将必然产生大量冲突，因为这无疑是在尝试往只有10个口袋的包里放15件物品，必然有几只口袋要大于一个物件。因此，通常不会这么使用。

在HashMap内部，还维护了一个threshold变量，它始终被定义为当前数组总容量和负载因子的乘积，它表示HashMap的阈值。当HashMap的实际容量超过阈值时，HashMap便会进行扩容。因此，HashMap的实际容量超过阈值时，HashMap便会进行扩容。因此，HashMap的实际填充率不会超过负载因子。

HashMap扩容的代码如下：

```
void resize(int newCapacity){
	Entry[] oldTable = table;
	int oldCapacity= oldTable.length;
	if(oldCapacity == MAXMUM_CAPACITY){
		threhold = Integer.MAX_VALUE;
		return;
	}
	//建立新的数组
	Entry[] newTable = new Entry[newCapacity];
	//将原有数组转到新的数组中
	transfer(newTable);
	table = newTable;
	//重新设置阈值，为新的容量和负载因子的乘积
	threshold = (int)(newCapacity * loadFactory);
}
```

其中，数组迁移逻辑主要在transfer()函数中实现，该函数实现和注释如下：

```
void transfer(Entry[] newTable){
	Entry[] src = table;
	int newCapacity = newTable.length;
	//遍历数组内所有表项
	for(int j = 0; j < src.length; j++){
		Entry<K,V> e = src[j];
		//当该表项索引有值存在时，则进行迁移
		if(e != null){
			src[j] = null;
			do{
				//进行数据迁移
				Entry<K,V> next = e.next;
				//计算该表现在新数组内的索引，并放置到新的数组中
				//建立新的链表关系
				int i = indexFor(e,hash, newCapacity);
				e.next = newTable[i];
				newTable[i] = e;
				e = next;
			}while(e != null)
		}
	}
}
```

很明显，HashMap的扩容操作会遍历整个HashMap，应该尽量避免该操作发生，设置合理的初始大小和负载因子，可以有效的减少HashMap扩容的次数。


**参考书籍：《Java程序性能优化》**