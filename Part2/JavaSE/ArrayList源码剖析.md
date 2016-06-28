##ArrayList简介

ArrayList是基于数组实现的，是一个动态数组，其容量能自动增长，类似于C语言中的动态申请内存，动态增长内存。

ArrayList不是线程安全的，只能在单线程环境下，多线程环境下可以考虑用collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类，也可以使用concurrent并发包下的CopyOnWriteArrayList类。

 ArrayList实现了Serializable接口，因此它支持序列化，能够通过序列化传输，实现了RandomAccess接口，支持快速随机访问，实际上就是通过下标序号进行快速访问，实现了Cloneable接口，能被克隆。

ArrayList源码剖析

ArrayList的源码如下（加入了比较详细的注释）：

```
package java.util;    
   
public class ArrayList<E> extends AbstractList<E>    
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable    
{    
    // 序列版本号    
    private static final long serialVersionUID = 8683452581122892189L;    
   
    // ArrayList基于该数组实现，用该数组保存数据   
    private transient Object[] elementData;    
   
    // ArrayList中实际数据的数量    
    private int size;    
   
    // ArrayList带容量大小的构造函数。    
    public ArrayList(int initialCapacity) {    
        super();    
        if (initialCapacity < 0)    
            throw new IllegalArgumentException("Illegal Capacity: "+    
                                               initialCapacity);    
        // 新建一个数组    
        this.elementData = new Object[initialCapacity];    
    }    
   
    // ArrayList无参构造函数。默认容量是10。    
    public ArrayList() {    
        this(10);    
    }    
   
    // 创建一个包含collection的ArrayList    
    public ArrayList(Collection<? extends E> c) {    
        elementData = c.toArray();    
        size = elementData.length;    
        if (elementData.getClass() != Object[].class)    
            elementData = Arrays.copyOf(elementData, size, Object[].class);    
    }    
   
   
    // 将当前容量值设为实际元素个数    
    public void trimToSize() {    
        modCount++;    
        int oldCapacity = elementData.length;    
        if (size < oldCapacity) {    
            elementData = Arrays.copyOf(elementData, size);    
        }    
    }    
   
   
    // 确定ArrarList的容量。    
    // 若ArrayList的容量不足以容纳当前的全部元素，设置 新的容量=“(原始容量x3)/2 + 1”    
    public void ensureCapacity(int minCapacity) {    
        // 将“修改统计数”+1，该变量主要是用来实现fail-fast机制的    
        modCount++;    
        int oldCapacity = elementData.length;    
        // 若当前容量不足以容纳当前的元素个数，设置 新的容量=“(原始容量x3)/2 + 1”    
        if (minCapacity > oldCapacity) {    
            Object oldData[] = elementData;    
            int newCapacity = (oldCapacity * 3)/2 + 1;    
            //如果还不够，则直接将minCapacity设置为当前容量  
            if (newCapacity < minCapacity)    
                newCapacity = minCapacity;    
            elementData = Arrays.copyOf(elementData, newCapacity);    
        }    
    }    
   
    // 添加元素e    
    public boolean add(E e) {    
        // 确定ArrayList的容量大小    
        ensureCapacity(size + 1);  // Increments modCount!!    
        // 添加e到ArrayList中    
        elementData[size++] = e;    
        return true;    
    }    
   
    // 返回ArrayList的实际大小    
    public int size() {    
        return size;    
    }    
   
    // ArrayList是否包含Object(o)    
    public boolean contains(Object o) {    
        return indexOf(o) >= 0;    
    }    
   
    //返回ArrayList是否为空    
    public boolean isEmpty() {    
        return size == 0;    
    }    
   
    // 正向查找，返回元素的索引值    
    public int indexOf(Object o) {    
        if (o == null) {    
            for (int i = 0; i < size; i++)    
            if (elementData[i]==null)    
                return i;    
            } else {    
                for (int i = 0; i < size; i++)    
                if (o.equals(elementData[i]))    
                    return i;    
            }    
            return -1;    
        }    
   
        // 反向查找，返回元素的索引值    
        public int lastIndexOf(Object o) {    
        if (o == null) {    
            for (int i = size-1; i >= 0; i--)    
            if (elementData[i]==null)    
                return i;    
        } else {    
            for (int i = size-1; i >= 0; i--)    
            if (o.equals(elementData[i]))    
                return i;    
        }    
        return -1;    
    }    
   
    // 反向查找(从数组末尾向开始查找)，返回元素(o)的索引值    
    public int lastIndexOf(Object o) {    
        if (o == null) {    
            for (int i = size-1; i >= 0; i--)    
            if (elementData[i]==null)    
                return i;    
        } else {    
            for (int i = size-1; i >= 0; i--)    
            if (o.equals(elementData[i]))    
                return i;    
        }    
        return -1;    
    }    
     
   
    // 返回ArrayList的Object数组    
    public Object[] toArray() {    
        return Arrays.copyOf(elementData, size);    
    }    
   
    // 返回ArrayList元素组成的数组  
    public <T> T[] toArray(T[] a) {    
        // 若数组a的大小 < ArrayList的元素个数；    
        // 则新建一个T[]数组，数组大小是“ArrayList的元素个数”，并将“ArrayList”全部拷贝到新数组中    
        if (a.length < size)    
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());    
   
        // 若数组a的大小 >= ArrayList的元素个数；    
        // 则将ArrayList的全部元素都拷贝到数组a中。    
        System.arraycopy(elementData, 0, a, 0, size);    
        if (a.length > size)    
            a[size] = null;    
        return a;    
    }    
   
    // 获取index位置的元素值    
    public E get(int index) {    
        RangeCheck(index);    
   
        return (E) elementData[index];    
    }    
   
    // 设置index位置的值为element    
    public E set(int index, E element) {    
        RangeCheck(index);    
   
        E oldValue = (E) elementData[index];    
        elementData[index] = element;    
        return oldValue;    
    }    
   
    // 将e添加到ArrayList中    
    public boolean add(E e) {    
        ensureCapacity(size + 1);  // Increments modCount!!    
        elementData[size++] = e;    
        return true;    
    }    
   
    // 将e添加到ArrayList的指定位置    
    public void add(int index, E element) {    
        if (index > size || index < 0)    
            throw new IndexOutOfBoundsException(    
            "Index: "+index+", Size: "+size);    
   
        ensureCapacity(size+1);  // Increments modCount!!    
        System.arraycopy(elementData, index, elementData, index + 1,    
             size - index);    
        elementData[index] = element;    
        size++;    
    }    
   
    // 删除ArrayList指定位置的元素    
    public E remove(int index) {    
        RangeCheck(index);    
   
        modCount++;    
        E oldValue = (E) elementData[index];    
   
        int numMoved = size - index - 1;    
        if (numMoved > 0)    
            System.arraycopy(elementData, index+1, elementData, index,    
                 numMoved);    
        elementData[--size] = null; // Let gc do its work    
   
        return oldValue;    
    }    
   
    // 删除ArrayList的指定元素    
    public boolean remove(Object o) {    
        if (o == null) {    
                for (int index = 0; index < size; index++)    
            if (elementData[index] == null) {    
                fastRemove(index);    
                return true;    
            }    
        } else {    
            for (int index = 0; index < size; index++)    
            if (o.equals(elementData[index])) {    
                fastRemove(index);    
                return true;    
            }    
        }    
        return false;    
    }    
   
   
    // 快速删除第index个元素    
    private void fastRemove(int index) {    
        modCount++;    
        int numMoved = size - index - 1;    
        // 从"index+1"开始，用后面的元素替换前面的元素。    
        if (numMoved > 0)    
            System.arraycopy(elementData, index+1, elementData, index,    
                             numMoved);    
        // 将最后一个元素设为null    
        elementData[--size] = null; // Let gc do its work    
    }    
   
    // 删除元素    
    public boolean remove(Object o) {    
        if (o == null) {    
            for (int index = 0; index < size; index++)    
            if (elementData[index] == null) {    
                fastRemove(index);    
            return true;    
            }    
        } else {    
            // 便利ArrayList，找到“元素o”，则删除，并返回true。    
            for (int index = 0; index < size; index++)    
            if (o.equals(elementData[index])) {    
                fastRemove(index);    
            return true;    
            }    
        }    
        return false;    
    }    
   
    // 清空ArrayList，将全部的元素设为null    
    public void clear() {    
        modCount++;    
   
        for (int i = 0; i < size; i++)    
            elementData[i] = null;    
   
        size = 0;    
    }    
   
    // 将集合c追加到ArrayList中    
    public boolean addAll(Collection<? extends E> c) {    
        Object[] a = c.toArray();    
        int numNew = a.length;    
        ensureCapacity(size + numNew);  // Increments modCount    
        System.arraycopy(a, 0, elementData, size, numNew);    
        size += numNew;    
        return numNew != 0;    
    }    
   
    // 从index位置开始，将集合c添加到ArrayList    
    public boolean addAll(int index, Collection<? extends E> c) {    
        if (index > size || index < 0)    
            throw new IndexOutOfBoundsException(    
            "Index: " + index + ", Size: " + size);    
   
        Object[] a = c.toArray();    
        int numNew = a.length;    
        ensureCapacity(size + numNew);  // Increments modCount    
   
        int numMoved = size - index;    
        if (numMoved > 0)    
            System.arraycopy(elementData, index, elementData, index + numNew,    
                 numMoved);    
   
        System.arraycopy(a, 0, elementData, index, numNew);    
        size += numNew;    
        return numNew != 0;    
    }    
   
    // 删除fromIndex到toIndex之间的全部元素。    
    protected void removeRange(int fromIndex, int toIndex) {    
    modCount++;    
    int numMoved = size - toIndex;    
        System.arraycopy(elementData, toIndex, elementData, fromIndex,    
                         numMoved);    
   
    // Let gc do its work    
    int newSize = size - (toIndex-fromIndex);    
    while (size != newSize)    
        elementData[--size] = null;    
    }    
   
    private void RangeCheck(int index) {    
    if (index >= size)    
        throw new IndexOutOfBoundsException(    
        "Index: "+index+", Size: "+size);    
    }    
   
   
    // 克隆函数    
    public Object clone() {    
        try {    
            ArrayList<E> v = (ArrayList<E>) super.clone();    
            // 将当前ArrayList的全部元素拷贝到v中    
            v.elementData = Arrays.copyOf(elementData, size);    
            v.modCount = 0;    
            return v;    
        } catch (CloneNotSupportedException e) {    
            // this shouldn't happen, since we are Cloneable    
            throw new InternalError();    
        }    
    }    
   
   
    // java.io.Serializable的写入函数    
    // 将ArrayList的“容量，所有的元素值”都写入到输出流中    
    private void writeObject(java.io.ObjectOutputStream s)    
        throws java.io.IOException{    
    // Write out element count, and any hidden stuff    
    int expectedModCount = modCount;    
    s.defaultWriteObject();    
   
        // 写入“数组的容量”    
        s.writeInt(elementData.length);    
   
    // 写入“数组的每一个元素”    
    for (int i=0; i<size; i++)    
            s.writeObject(elementData[i]);    
   
    if (modCount != expectedModCount) {    
            throw new ConcurrentModificationException();    
        }    
   
    }    
   
   
    // java.io.Serializable的读取函数：根据写入方式读出    
    // 先将ArrayList的“容量”读出，然后将“所有的元素值”读出    
    private void readObject(java.io.ObjectInputStream s)    
        throws java.io.IOException, ClassNotFoundException {    
        // Read in size, and any hidden stuff    
        s.defaultReadObject();    
   
        // 从输入流中读取ArrayList的“容量”    
        int arrayLength = s.readInt();    
        Object[] a = elementData = new Object[arrayLength];    
   
        // 从输入流中将“所有的元素值”读出    
        for (int i=0; i<size; i++)    
            a[i] = s.readObject();    
    }    
}  
```

##几点总结

关于ArrayList的源码，给出几点比较重要的总结：

1. 注意其三个不同的构造方法。无参构造方法构造的ArrayList的容量默认为10，带有Collection参数的构造方法，将Collection转化为数组赋给ArrayList的实现数组elementData。
2. 注意扩充容量的方法ensureCapacity。ArrayList在每次增加元素（可能是1个，也可能是一组）时，都要调用该方法来确保足够的容量。当容量不足以容纳当前的元素个数时，就设置新的容量为旧的容量的1.5倍加1，如果设置后的新容量还不够，则直接新容量设置为传入的参数（也就是所需的容量），而后用Arrays.copyof()方法将元素拷贝到新的数组（详见下面的第3点）。从中可以看出，当容量不够时，每次增加元素，都要将原来的元素拷贝到一个新的数组中，非常之耗时，也因此建议在事先能确定元素数量的情况下，才使用ArrayList，否则建议使用LinkedList。
3. ArrayList的实现中大量地调用了Arrays.copyof()和System.arraycopy()方法。我们有必要对这两个方法的实现做下深入的了解。

首先来看Arrays.copyof()方法。它有很多个重载的方法，但实现思路都是一样的，我们来看泛型版本的源码：

```
public static <T> T[] copyOf(T[] original, int newLength) {  
    return (T[]) copyOf(original, newLength, original.getClass());  
}  
```

很明显调用了另一个copyof方法，该方法有三个参数，最后一个参数指明要转换的数据的类型，其源码如下：

```
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {  
    T[] copy = ((Object)newType == (Object)Object[].class)  
        ? (T[]) new Object[newLength]  
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);  
    System.arraycopy(original, 0, copy, 0,  
                     Math.min(original.length, newLength));  
    return copy;  
}  
```

这里可以很明显地看出，该方法实际上是在其内部又创建了一个长度为newlength的数组，调用System.arraycopy()方法，将原来数组中的元素复制到了新的数组中。

下面来看System.arraycopy()方法。该方法被标记了native，调用了系统的C/C++代码，在JDK中是看不到的，但在openJDK中可以看到其源码。该函数实际上最终调用了C语言的memmove()函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在复制大量数组元素时用该方法，以取得更高的效率。
4. 注意ArrayList的两个转化为静态数组的toArray方法。

第一个，Object[] toArray()方法。该方法有可能会抛出java.lang.ClassCastException异常，如果直接用向下转型的方法，将整个ArrayList集合转变为指定类型的Array数组，便会抛出该异常，而如果转化为Array数组时不向下转型，而是将每个元素向下转型，则不会抛出该异常，显然对数组中的元素一个个进行向下转型，效率不高，且不太方便。

第二个，<T> T[] toArray(T[] a)方法。该方法可以直接将ArrayList转换得到的Array进行整体向下转型（转型其实是在该方法的源码中实现的），且从该方法的源码中可以看出，参数a的大小不足时，内部会调用Arrays.copyOf方法，该方法内部创建一个新的数组返回，因此对该方法的常用形式如下：

```
public static Integer[] vectorToArray2(ArrayList<Integer> v) {    
    Integer[] newText = (Integer[])v.toArray(new Integer[0]);    
    return newText;    
}    
```


5.ArrayList基于数组实现，可以通过下标索引直接查找到指定位置的元素，因此查找效率高，但每次插入或删除元素，就要大量地移动元素，插入删除元素的效率低。

6.在查找给定元素索引值等的方法中，源码都将该元素的值分为null和不为null两种情况处理，ArrayList中允许元素为null。