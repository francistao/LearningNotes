##LinkedHashMap简介

LinkedHashMap是HashMap的子类，与HashMap有着同样的存储结构，但它加入了一个双向链表的头结点，将所有put到LinkedHashmap的节点一一串成了一个双向循环链表，因此它保留了节点插入的顺序，可以使节点的输出顺序与输入顺序相同。

LinkedHashMap可以用来实现LRU算法（这会在下面的源码中进行分析）。

LinkedHashMap同样是非线程安全的，只在单线程环境下使用。

##LinkedHashMap源码剖析

LinkedHashMap源码如下（加入了详细的注释）：

```
package java.util;  
import java.io.*;  
  
  
public class LinkedHashMap<K,V>  
    extends HashMap<K,V>  
    implements Map<K,V>  
{  
  
    private static final long serialVersionUID = 3801124242820219131L;  
  
    //双向循环链表的头结点，整个LinkedHashMap中只有一个header，  
    //它将哈希表中所有的Entry贯穿起来，header中不保存key-value对，只保存前后节点的引用  
    private transient Entry<K,V> header;  
  
    //双向链表中元素排序规则的标志位。  
    //accessOrder为false，表示按插入顺序排序  
    //accessOrder为true，表示按访问顺序排序  
    private final boolean accessOrder;  
  
    //调用HashMap的构造方法来构造底层的数组  
    public LinkedHashMap(int initialCapacity, float loadFactor) {  
        super(initialCapacity, loadFactor);  
        accessOrder = false;    //链表中的元素默认按照插入顺序排序  
    }  
  
    //加载因子取默认的0.75f  
    public LinkedHashMap(int initialCapacity) {  
        super(initialCapacity);  
        accessOrder = false;  
    }  
  
    //加载因子取默认的0.75f，容量取默认的16  
    public LinkedHashMap() {  
        super();  
        accessOrder = false;  
    }  
  
    //含有子Map的构造方法，同样调用HashMap的对应的构造方法  
    public LinkedHashMap(Map<? extends K, ? extends V> m) {  
        super(m);  
        accessOrder = false;  
    }  
  
    //该构造方法可以指定链表中的元素排序的规则  
    public LinkedHashMap(int initialCapacity,float loadFactor,boolean accessOrder) {  
        super(initialCapacity, loadFactor);  
        this.accessOrder = accessOrder;  
    }  
  
    //覆写父类的init()方法（HashMap中的init方法为空），  
    //该方法在父类的构造方法和Clone、readObject中在插入元素前被调用，  
    //初始化一个空的双向循环链表，头结点中不保存数据，头结点的下一个节点才开始保存数据。  
    void init() {  
        header = new Entry<K,V>(-1, null, null, null);  
        header.before = header.after = header;  
    }  
  
  
    //覆写HashMap中的transfer方法，它在父类的resize方法中被调用，  
    //扩容后，将key-value对重新映射到新的newTable中  
    //覆写该方法的目的是为了提高复制的效率，  
    //这里充分利用双向循环链表的特点进行迭代，不用对底层的数组进行for循环。  
    void transfer(HashMap.Entry[] newTable) {  
        int newCapacity = newTable.length;  
        for (Entry<K,V> e = header.after; e != header; e = e.after) {  
            int index = indexFor(e.hash, newCapacity);  
            e.next = newTable[index];  
            newTable[index] = e;  
        }  
    }  
  
  
    //覆写HashMap中的containsValue方法，  
    //覆写该方法的目的同样是为了提高查询的效率，  
    //利用双向循环链表的特点进行查询，少了对数组的外层for循环  
    public boolean containsValue(Object value) {  
        // Overridden to take advantage of faster iterator  
        if (value==null) {  
            for (Entry e = header.after; e != header; e = e.after)  
                if (e.value==null)  
                    return true;  
        } else {  
            for (Entry e = header.after; e != header; e = e.after)  
                if (value.equals(e.value))  
                    return true;  
        }  
        return false;  
    }  
  
  
    //覆写HashMap中的get方法，通过getEntry方法获取Entry对象。  
    //注意这里的recordAccess方法，  
    //如果链表中元素的排序规则是按照插入的先后顺序排序的话，该方法什么也不做，  
    //如果链表中元素的排序规则是按照访问的先后顺序排序的话，则将e移到链表的末尾处。  
    public V get(Object key) {  
        Entry<K,V> e = (Entry<K,V>)getEntry(key);  
        if (e == null)  
            return null;  
        e.recordAccess(this);  
        return e.value;  
    }  
  
    //清空HashMap，并将双向链表还原为只有头结点的空链表  
    public void clear() {  
        super.clear();  
        header.before = header.after = header;  
    }  
  
    //Enty的数据结构，多了两个指向前后节点的引用  
    private static class Entry<K,V> extends HashMap.Entry<K,V> {  
        // These fields comprise the doubly linked list used for iteration.  
        Entry<K,V> before, after;  
  
        //调用父类的构造方法  
        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {  
            super(hash, key, value, next);  
        }  
  
        //双向循环链表中，删除当前的Entry  
        private void remove() {  
            before.after = after;  
            after.before = before;  
        }  
  
        //双向循环立链表中，将当前的Entry插入到existingEntry的前面  
        private void addBefore(Entry<K,V> existingEntry) {  
            after  = existingEntry;  
            before = existingEntry.before;  
            before.after = this;  
            after.before = this;  
        }  
  
  
        //覆写HashMap中的recordAccess方法（HashMap中该方法为空），  
        //当调用父类的put方法，在发现插入的key已经存在时，会调用该方法，  
        //调用LinkedHashmap覆写的get方法时，也会调用到该方法，  
        //该方法提供了LRU算法的实现，它将最近使用的Entry放到双向循环链表的尾部，  
        //accessOrder为true时，get方法会调用recordAccess方法  
        //put方法在覆盖key-value对时也会调用recordAccess方法  
        //它们导致Entry最近使用，因此将其移到双向链表的末尾  
        void recordAccess(HashMap<K,V> m) {  
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;  
            //如果链表中元素按照访问顺序排序，则将当前访问的Entry移到双向循环链表的尾部，  
            //如果是按照插入的先后顺序排序，则不做任何事情。  
            if (lm.accessOrder) {  
                lm.modCount++;  
                //移除当前访问的Entry  
                remove();  
                //将当前访问的Entry插入到链表的尾部  
                addBefore(lm.header);  
            }  
        }  
  
        void recordRemoval(HashMap<K,V> m) {  
            remove();  
        }  
    }  
  
    //迭代器  
    private abstract class LinkedHashIterator<T> implements Iterator<T> {  
    Entry<K,V> nextEntry    = header.after;  
    Entry<K,V> lastReturned = null;  
  
    /** 
     * The modCount value that the iterator believes that the backing 
     * List should have.  If this expectation is violated, the iterator 
     * has detected concurrent modification. 
     */  
    int expectedModCount = modCount;  
  
    public boolean hasNext() {  
            return nextEntry != header;  
    }  
  
    public void remove() {  
        if (lastReturned == null)  
        throw new IllegalStateException();  
        if (modCount != expectedModCount)  
        throw new ConcurrentModificationException();  
  
            LinkedHashMap.this.remove(lastReturned.key);  
            lastReturned = null;  
            expectedModCount = modCount;  
    }  
  
    //从head的下一个节点开始迭代  
    Entry<K,V> nextEntry() {  
        if (modCount != expectedModCount)  
        throw new ConcurrentModificationException();  
            if (nextEntry == header)  
                throw new NoSuchElementException();  
  
            Entry<K,V> e = lastReturned = nextEntry;  
            nextEntry = e.after;  
            return e;  
    }  
    }  
  
    //key迭代器  
    private class KeyIterator extends LinkedHashIterator<K> {  
    public K next() { return nextEntry().getKey(); }  
    }  
  
    //value迭代器  
    private class ValueIterator extends LinkedHashIterator<V> {  
    public V next() { return nextEntry().value; }  
    }  
  
    //Entry迭代器  
    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {  
    public Map.Entry<K,V> next() { return nextEntry(); }  
    }  
  
    // These Overrides alter the behavior of superclass view iterator() methods  
    Iterator<K> newKeyIterator()   { return new KeyIterator();   }  
    Iterator<V> newValueIterator() { return new ValueIterator(); }  
    Iterator<Map.Entry<K,V>> newEntryIterator() { return new EntryIterator(); }  
  
  
    //覆写HashMap中的addEntry方法，LinkedHashmap并没有覆写HashMap中的put方法，  
    //而是覆写了put方法所调用的addEntry方法和recordAccess方法，  
    //put方法在插入的key已存在的情况下，会调用recordAccess方法，  
    //在插入的key不存在的情况下，要调用addEntry插入新的Entry  
    void addEntry(int hash, K key, V value, int bucketIndex) {  
        //创建新的Entry，并插入到LinkedHashMap中  
        createEntry(hash, key, value, bucketIndex);  
  
        //双向链表的第一个有效节点（header后的那个节点）为近期最少使用的节点  
        Entry<K,V> eldest = header.after;  
        //如果有必要，则删除掉该近期最少使用的节点，  
        //这要看对removeEldestEntry的覆写,由于默认为false，因此默认是不做任何处理的。  
        if (removeEldestEntry(eldest)) {  
            removeEntryForKey(eldest.key);  
        } else {  
            //扩容到原来的2倍  
            if (size >= threshold)  
                resize(2 * table.length);  
        }  
    }  
  
    void createEntry(int hash, K key, V value, int bucketIndex) {  
        //创建新的Entry，并将其插入到数组对应槽的单链表的头结点处，这点与HashMap中相同  
        HashMap.Entry<K,V> old = table[bucketIndex];  
        Entry<K,V> e = new Entry<K,V>(hash, key, value, old);  
        table[bucketIndex] = e;  
        //每次插入Entry时，都将其移到双向链表的尾部，  
        //这便会按照Entry插入LinkedHashMap的先后顺序来迭代元素，  
        //同时，新put进来的Entry是最近访问的Entry，把其放在链表末尾 ，符合LRU算法的实现  
        e.addBefore(header);  
        size++;  
    }  
  
    //该方法是用来被覆写的，一般如果用LinkedHashmap实现LRU算法，就要覆写该方法，  
    //比如可以将该方法覆写为如果设定的内存已满，则返回true，这样当再次向LinkedHashMap中put  
    //Entry时，在调用的addEntry方法中便会将近期最少使用的节点删除掉（header后的那个节点）。  
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {  
        return false;  
    }  
}  
```
##几点总结

关于LinkedHashMap的源码，给出以下几点比较重要的总结：

1、从源码中可以看出，LinkedHashMap中加入了一个head头结点，将所有插入到该LinkedHashMap中的Entry按照插入的先后顺序依次加入到以head为头结点的双向循环链表的尾部。

![](http://img.blog.csdn.net/20140716084631981?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbnNfY29kZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1、实际上就是HashMap和LinkedList两个集合类的存储结构的结合。在LinkedHashMapMap中，所有put进来的Entry都保存在如第一个图所示的哈希表中，但它又额外定义了一个以head为头结点的空的双向循环链表，每次put进来Entry，除了将其保存到对哈希表中对应的位置上外，还要将其插入到双向循环链表的尾部。

2、LinkedHashMap由于继承自HashMap，因此它具有HashMap的所有特性，同样允许key和value为null。

3、注意源码中的accessOrder标志位，当它false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致，这也是默认的双向链表的存储顺序；当它为true时，表示双向链表中的元素按照访问的先后顺序排列，可以看到，虽然Entry插入链表的顺序依然是按照其put到LinkedHashMap中的顺序，但put和get方法均有调用recordAccess方法（put方法在key相同，覆盖原有的Entry的情况下调用recordAccess方法），该方法判断accessOrder是否为true，如果是，则将当前访问的Entry（put进来的Entry或get出来的Entry）移到双向链表的尾部（key不相同时，put新Entry时，会调用addEntry，它会调用creatEntry，该方法同样将新插入的元素放入到双向链表的尾部，既符合插入的先后顺序，又符合访问的先后顺序，因为这时该Entry也被访问了），否则，什么也不做。

4、注意构造方法，前四个构造方法都将accessOrder设为false，说明默认是按照插入顺序排序的，而第五个构造方法可以自定义传入的accessOrder的值，因此可以指定双向循环链表中元素的排序规则，一般要用LinkedHashMap实现LRU算法，就要用该构造方法，将accessOrder置为true。

5、LinkedHashMap并没有覆写HashMap中的put方法，而是覆写了put方法中调用的addEntry方法和recordAccess方法，我们回过头来再看下HashMap的put方法：

```
// 将“key-value”添加到HashMap中      
public V put(K key, V value) {      
    // 若“key为null”，则将该键值对添加到table[0]中。      
    if (key == null)      
        return putForNullKey(value);      
    // 若“key不为null”，则计算该key的哈希值，然后将其添加到该哈希值对应的链表中。      
    int hash = hash(key.hashCode());      
    int i = indexFor(hash, table.length);      
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {      
        Object k;      
        // 若“该key”对应的键值对已经存在，则用新的value取代旧的value。然后退出！      
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {      
            V oldValue = e.value;      
            e.value = value;      
            e.recordAccess(this);      
            return oldValue;      
        }      
    }      
  
    // 若“该key”对应的键值对不存在，则将“key-value”添加到table中      
    modCount++;    
    //将key-value添加到table[i]处    
    addEntry(hash, key, value, i);      
    return null;      
}      
```

当要put进来的Entry的key在哈希表中已经在存在时，会调用recordAccess方法，当该key不存在时，则会调用addEntry方法将新的Entry插入到对应槽的单链表的头部。

我们先来看recordAccess方法：

```
//覆写HashMap中的recordAccess方法（HashMap中该方法为空），  
//当调用父类的put方法，在发现插入的key已经存在时，会调用该方法，  
//调用LinkedHashmap覆写的get方法时，也会调用到该方法，  
//该方法提供了LRU算法的实现，它将最近使用的Entry放到双向循环链表的尾部，  
//accessOrder为true时，get方法会调用recordAccess方法  
//put方法在覆盖key-value对时也会调用recordAccess方法  
//它们导致Entry最近使用，因此将其移到双向链表的末尾  
      void recordAccess(HashMap<K,V> m) {  
          LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;  
    //如果链表中元素按照访问顺序排序，则将当前访问的Entry移到双向循环链表的尾部，  
    //如果是按照插入的先后顺序排序，则不做任何事情。  
          if (lm.accessOrder) {  
              lm.modCount++;  
        //移除当前访问的Entry  
              remove();  
        //将当前访问的Entry插入到链表的尾部  
              addBefore(lm.header);  
          }  
      }  
```

该方法会判断accessOrder是否为true，如果为true，它会将当前访问的Entry（在这里指put进来的Entry）移动到双向循环链表的尾部，从而实现双向链表中的元素按照访问顺序来排序（最近访问的Entry放到链表的最后，这样多次下来，前面就是最近没有被访问的元素，在实现、LRU算法时，当双向链表中的节点数达到最大值时，将前面的元素删去即可，因为前面的元素是最近最少使用的），否则什么也不做。

再来看addEntry方法：

```
//覆写HashMap中的addEntry方法，LinkedHashmap并没有覆写HashMap中的put方法，  
//而是覆写了put方法所调用的addEntry方法和recordAccess方法，  
//put方法在插入的key已存在的情况下，会调用recordAccess方法，  
//在插入的key不存在的情况下，要调用addEntry插入新的Entry  
   void addEntry(int hash, K key, V value, int bucketIndex) {  
    //创建新的Entry，并插入到LinkedHashMap中  
       createEntry(hash, key, value, bucketIndex);  
  
       //双向链表的第一个有效节点（header后的那个节点）为近期最少使用的节点  
       Entry<K,V> eldest = header.after;  
    //如果有必要，则删除掉该近期最少使用的节点，  
    //这要看对removeEldestEntry的覆写,由于默认为false，因此默认是不做任何处理的。  
       if (removeEldestEntry(eldest)) {  
           removeEntryForKey(eldest.key);  
       } else {  
        //扩容到原来的2倍  
           if (size >= threshold)  
               resize(2 * table.length);  
       }  
   }  
  
   void createEntry(int hash, K key, V value, int bucketIndex) {  
    //创建新的Entry，并将其插入到数组对应槽的单链表的头结点处，这点与HashMap中相同  
       HashMap.Entry<K,V> old = table[bucketIndex];  
    Entry<K,V> e = new Entry<K,V>(hash, key, value, old);  
       table[bucketIndex] = e;  
    //每次插入Entry时，都将其移到双向链表的尾部，  
    //这便会按照Entry插入LinkedHashMap的先后顺序来迭代元素，  
    //同时，新put进来的Entry是最近访问的Entry，把其放在链表末尾 ，符合LRU算法的实现  
       e.addBefore(header);  
       size++;  
   }  
```

同样是将新的Entry插入到table中对应槽所对应单链表的头结点中，但可以看出，在createEntry中，同样把新put进来的Entry插入到了双向链表的尾部，从插入顺序的层面来说，新的Entry插入到双向链表的尾部，可以实现按照插入的先后顺序来迭代Entry，而从访问顺序的层面来说，新put进来的Entry又是最近访问的Entry，也应该将其放在双向链表的尾部。

上面还有个removeEldestEntry方法，该方法如下：

```
    //该方法是用来被覆写的，一般如果用LinkedHashmap实现LRU算法，就要覆写该方法，  
    //比如可以将该方法覆写为如果设定的内存已满，则返回true，这样当再次向LinkedHashMap中put  
    //Entry时，在调用的addEntry方法中便会将近期最少使用的节点删除掉（header后的那个节点）。  
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {  
        return false;  
    }  
}  
```

该方法默认返回false，我们一般在用LinkedHashMap实现LRU算法时，要覆写该方法，一般的实现是，当设定的内存（这里指节点个数）达到最大值时，返回true，这样put新的Entry（该Entry的key在哈希表中没有已经存在）时，就会调用removeEntryForKey方法，将最近最少使用的节点删除（head后面的那个节点，实际上是最近没有使用）。

6、LinkedHashMap覆写了HashMap的get方法：

```
//覆写HashMap中的get方法，通过getEntry方法获取Entry对象。  
//注意这里的recordAccess方法，  
//如果链表中元素的排序规则是按照插入的先后顺序排序的话，该方法什么也不做，  
//如果链表中元素的排序规则是按照访问的先后顺序排序的话，则将e移到链表的末尾处。  
   public V get(Object key) {  
       Entry<K,V> e = (Entry<K,V>)getEntry(key);  
       if (e == null)  
           return null;  
       e.recordAccess(this);  
       return e.value;  
   }  
```

先取得Entry，如果不为null，一样调用recordAccess方法，上面已经说得很清楚，这里不在多解释了。

7、最后说说LinkedHashMap是如何实现LRU的。首先，当accessOrder为true时，才会开启按访问顺序排序的模式，才能用来实现LRU算法。我们可以看到，无论是put方法还是get方法，都会导致目标Entry成为最近访问的Entry，因此便把该Entry加入到了双向链表的末尾（get方法通过调用recordAccess方法来实现，put方法在覆盖已有key的情况下，也是通过调用recordAccess方法来实现，在插入新的Entry时，则是通过createEntry中的addBefore方法来实现），这样便把最近使用了的Entry放入到了双向链表的后面，多次操作后，双向链表前面的Entry便是最近没有使用的，这样当节点个数满的时候，删除的最前面的Entry(head后面的那个Entry)便是最近最少使用的Entry。