##Vector简介
Vector也是基于数组实现的，是一个动态数组，其容量能自动增长。

Vector是JDK1.0引入了，它的很多实现方法都加入了同步语句，因此是线程安全的（其实也只是相对安全，有些时候还是要加入同步语句来保证线程的安全），可以用于多线程环境。

Vector没有实现Serializable接口，因此它不支持序列化，实现了Cloneable接口，能被克隆，实现了RandomAccess接口，支持快速随机访问。

##Vector源码剖析
Vector的源码如下（加入了比较详细的注释）：

```
package java.util;    
   
public class Vector<E>    
    extends AbstractList<E>    
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable    
{    
       
    // 保存Vector中数据的数组    
    protected Object[] elementData;    
   
    // 实际数据的数量    
    protected int elementCount;    
   
    // 容量增长系数    
    protected int capacityIncrement;    
   
    // Vector的序列版本号    
    private static final long serialVersionUID = -2767605614048989439L;    
   
    // Vector构造函数。默认容量是10。    
    public Vector() {    
        this(10);    
    }    
   
    // 指定Vector容量大小的构造函数    
    public Vector(int initialCapacity) {    
        this(initialCapacity, 0);    
    }    
   
    // 指定Vector"容量大小"和"增长系数"的构造函数    
    public Vector(int initialCapacity, int capacityIncrement) {    
        super();    
        if (initialCapacity < 0)    
            throw new IllegalArgumentException("Illegal Capacity: "+    
                                               initialCapacity);    
        // 新建一个数组，数组容量是initialCapacity    
        this.elementData = new Object[initialCapacity];    
        // 设置容量增长系数    
        this.capacityIncrement = capacityIncrement;    
    }    
   
    // 指定集合的Vector构造函数。    
    public Vector(Collection<? extends E> c) {    
        // 获取“集合(c)”的数组，并将其赋值给elementData    
        elementData = c.toArray();    
        // 设置数组长度    
        elementCount = elementData.length;    
        // c.toArray might (incorrectly) not return Object[] (see 6260652)    
        if (elementData.getClass() != Object[].class)    
            elementData = Arrays.copyOf(elementData, elementCount, Object[].class);    
    }    
   
    // 将数组Vector的全部元素都拷贝到数组anArray中    
    public synchronized void copyInto(Object[] anArray) {    
        System.arraycopy(elementData, 0, anArray, 0, elementCount);    
    }    
   
    // 将当前容量值设为 =实际元素个数    
    public synchronized void trimToSize() {    
        modCount++;    
        int oldCapacity = elementData.length;    
        if (elementCount < oldCapacity) {    
            elementData = Arrays.copyOf(elementData, elementCount);    
        }    
    }    
   
    // 确认“Vector容量”的帮助函数    
    private void ensureCapacityHelper(int minCapacity) {    
        int oldCapacity = elementData.length;    
        // 当Vector的容量不足以容纳当前的全部元素，增加容量大小。    
        // 若 容量增量系数>0(即capacityIncrement>0)，则将容量增大当capacityIncrement    
        // 否则，将容量增大一倍。    
        if (minCapacity > oldCapacity) {    
            Object[] oldData = elementData;    
            int newCapacity = (capacityIncrement > 0) ?    
                (oldCapacity + capacityIncrement) : (oldCapacity * 2);    
            if (newCapacity < minCapacity) {    
                newCapacity = minCapacity;    
            }    
            elementData = Arrays.copyOf(elementData, newCapacity);    
        }    
    }    
   
    // 确定Vector的容量。    
    public synchronized void ensureCapacity(int minCapacity) {    
        // 将Vector的改变统计数+1    
        modCount++;    
        ensureCapacityHelper(minCapacity);    
    }    
   
    // 设置容量值为 newSize    
    public synchronized void setSize(int newSize) {    
        modCount++;    
        if (newSize > elementCount) {    
            // 若 "newSize 大于 Vector容量"，则调整Vector的大小。    
            ensureCapacityHelper(newSize);    
        } else {    
            // 若 "newSize 小于/等于 Vector容量"，则将newSize位置开始的元素都设置为null    
            for (int i = newSize ; i < elementCount ; i++) {    
                elementData[i] = null;    
            }    
        }    
        elementCount = newSize;    
    }    
   
    // 返回“Vector的总的容量”    
    public synchronized int capacity() {    
        return elementData.length;    
    }    
   
    // 返回“Vector的实际大小”，即Vector中元素个数    
    public synchronized int size() {    
        return elementCount;    
    }    
   
    // 判断Vector是否为空    
    public synchronized boolean isEmpty() {    
        return elementCount == 0;    
    }    
   
    // 返回“Vector中全部元素对应的Enumeration”    
    public Enumeration<E> elements() {    
        // 通过匿名类实现Enumeration    
        return new Enumeration<E>() {    
            int count = 0;    
   
            // 是否存在下一个元素    
            public boolean hasMoreElements() {    
                return count < elementCount;    
            }    
   
            // 获取下一个元素    
            public E nextElement() {    
                synchronized (Vector.this) {    
                    if (count < elementCount) {    
                        return (E)elementData[count++];    
                    }    
                }    
                throw new NoSuchElementException("Vector Enumeration");    
            }    
        };    
    }    
   
    // 返回Vector中是否包含对象(o)    
    public boolean contains(Object o) {    
        return indexOf(o, 0) >= 0;    
    }    
   
   
    // 从index位置开始向后查找元素(o)。    
    // 若找到，则返回元素的索引值；否则，返回-1    
    public synchronized int indexOf(Object o, int index) {    
        if (o == null) {    
            // 若查找元素为null，则正向找出null元素，并返回它对应的序号    
            for (int i = index ; i < elementCount ; i++)    
            if (elementData[i]==null)    
                return i;    
        } else {    
            // 若查找元素不为null，则正向找出该元素，并返回它对应的序号    
            for (int i = index ; i < elementCount ; i++)    
            if (o.equals(elementData[i]))    
                return i;    
        }    
        return -1;    
    }    
   
    // 查找并返回元素(o)在Vector中的索引值    
    public int indexOf(Object o) {    
        return indexOf(o, 0);    
    }    
   
    // 从后向前查找元素(o)。并返回元素的索引    
    public synchronized int lastIndexOf(Object o) {    
        return lastIndexOf(o, elementCount-1);    
    }    
   
    // 从后向前查找元素(o)。开始位置是从前向后的第index个数；    
    // 若找到，则返回元素的“索引值”；否则，返回-1。    
    public synchronized int lastIndexOf(Object o, int index) {    
        if (index >= elementCount)    
            throw new IndexOutOfBoundsException(index + " >= "+ elementCount);    
   
        if (o == null) {    
            // 若查找元素为null，则反向找出null元素，并返回它对应的序号    
            for (int i = index; i >= 0; i--)    
            if (elementData[i]==null)    
                return i;    
        } else {    
            // 若查找元素不为null，则反向找出该元素，并返回它对应的序号    
            for (int i = index; i >= 0; i--)    
            if (o.equals(elementData[i]))    
                return i;    
        }    
        return -1;    
    }    
   
    // 返回Vector中index位置的元素。    
    // 若index月结，则抛出异常    
    public synchronized E elementAt(int index) {    
        if (index >= elementCount) {    
            throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);    
        }    
   
        return (E)elementData[index];    
    }    
   
    // 获取Vector中的第一个元素。    
    // 若失败，则抛出异常！    
    public synchronized E firstElement() {    
        if (elementCount == 0) {    
            throw new NoSuchElementException();    
        }    
        return (E)elementData[0];    
    }    
   
    // 获取Vector中的最后一个元素。    
    // 若失败，则抛出异常！    
    public synchronized E lastElement() {    
        if (elementCount == 0) {    
            throw new NoSuchElementException();    
        }    
        return (E)elementData[elementCount - 1];    
    }    
   
    // 设置index位置的元素值为obj    
    public synchronized void setElementAt(E obj, int index) {    
        if (index >= elementCount) {    
            throw new ArrayIndexOutOfBoundsException(index + " >= " +    
                                 elementCount);    
        }    
        elementData[index] = obj;    
    }    
   
    // 删除index位置的元素    
    public synchronized void removeElementAt(int index) {    
        modCount++;    
        if (index >= elementCount) {    
            throw new ArrayIndexOutOfBoundsException(index + " >= " +    
                                 elementCount);    
        } else if (index < 0) {    
            throw new ArrayIndexOutOfBoundsException(index);    
        }    
   
        int j = elementCount - index - 1;    
        if (j > 0) {    
            System.arraycopy(elementData, index + 1, elementData, index, j);    
        }    
        elementCount--;    
        elementData[elementCount] = null; /* to let gc do its work */   
    }    
   
    // 在index位置处插入元素(obj)    
    public synchronized void insertElementAt(E obj, int index) {    
        modCount++;    
        if (index > elementCount) {    
            throw new ArrayIndexOutOfBoundsException(index    
                                 + " > " + elementCount);    
        }    
        ensureCapacityHelper(elementCount + 1);    
        System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);    
        elementData[index] = obj;    
        elementCount++;    
    }    
   
    // 将“元素obj”添加到Vector末尾    
    public synchronized void addElement(E obj) {    
        modCount++;    
        ensureCapacityHelper(elementCount + 1);    
        elementData[elementCount++] = obj;    
    }    
   
    // 在Vector中查找并删除元素obj。    
    // 成功的话，返回true；否则，返回false。    
    public synchronized boolean removeElement(Object obj) {    
        modCount++;    
        int i = indexOf(obj);    
        if (i >= 0) {    
            removeElementAt(i);    
            return true;    
        }    
        return false;    
    }    
   
    // 删除Vector中的全部元素    
    public synchronized void removeAllElements() {    
        modCount++;    
        // 将Vector中的全部元素设为null    
        for (int i = 0; i < elementCount; i++)    
            elementData[i] = null;    
   
        elementCount = 0;    
    }    
   
    // 克隆函数    
    public synchronized Object clone() {    
        try {    
            Vector<E> v = (Vector<E>) super.clone();    
            // 将当前Vector的全部元素拷贝到v中    
            v.elementData = Arrays.copyOf(elementData, elementCount);    
            v.modCount = 0;    
            return v;    
        } catch (CloneNotSupportedException e) {    
            // this shouldn't happen, since we are Cloneable    
            throw new InternalError();    
        }    
    }    
   
    // 返回Object数组    
    public synchronized Object[] toArray() {    
        return Arrays.copyOf(elementData, elementCount);    
    }    
   
    // 返回Vector的模板数组。所谓模板数组，即可以将T设为任意的数据类型    
    public synchronized <T> T[] toArray(T[] a) {    
        // 若数组a的大小 < Vector的元素个数；    
        // 则新建一个T[]数组，数组大小是“Vector的元素个数”，并将“Vector”全部拷贝到新数组中    
        if (a.length < elementCount)    
            return (T[]) Arrays.copyOf(elementData, elementCount, a.getClass());    
   
        // 若数组a的大小 >= Vector的元素个数；    
        // 则将Vector的全部元素都拷贝到数组a中。    
    System.arraycopy(elementData, 0, a, 0, elementCount);    
   
        if (a.length > elementCount)    
            a[elementCount] = null;    
   
        return a;    
    }    
   
    // 获取index位置的元素    
    public synchronized E get(int index) {    
        if (index >= elementCount)    
            throw new ArrayIndexOutOfBoundsException(index);    
   
        return (E)elementData[index];    
    }    
   
    // 设置index位置的值为element。并返回index位置的原始值    
    public synchronized E set(int index, E element) {    
        if (index >= elementCount)    
            throw new ArrayIndexOutOfBoundsException(index);    
   
        Object oldValue = elementData[index];    
        elementData[index] = element;    
        return (E)oldValue;    
    }    
   
    // 将“元素e”添加到Vector最后。    
    public synchronized boolean add(E e) {    
        modCount++;    
        ensureCapacityHelper(elementCount + 1);    
        elementData[elementCount++] = e;    
        return true;    
    }    
   
    // 删除Vector中的元素o    
    public boolean remove(Object o) {    
        return removeElement(o);    
    }    
   
    // 在index位置添加元素element    
    public void add(int index, E element) {    
        insertElementAt(element, index);    
    }    
   
    // 删除index位置的元素，并返回index位置的原始值    
    public synchronized E remove(int index) {    
        modCount++;    
        if (index >= elementCount)    
            throw new ArrayIndexOutOfBoundsException(index);    
        Object oldValue = elementData[index];    
   
        int numMoved = elementCount - index - 1;    
        if (numMoved > 0)    
            System.arraycopy(elementData, index+1, elementData, index,    
                     numMoved);    
        elementData[--elementCount] = null; // Let gc do its work    
   
        return (E)oldValue;    
    }    
   
    // 清空Vector    
    public void clear() {    
        removeAllElements();    
    }    
   
    // 返回Vector是否包含集合c    
    public synchronized boolean containsAll(Collection<?> c) {    
        return super.containsAll(c);    
    }    
   
    // 将集合c添加到Vector中    
    public synchronized boolean addAll(Collection<? extends E> c) {    
        modCount++;    
        Object[] a = c.toArray();    
        int numNew = a.length;    
        ensureCapacityHelper(elementCount + numNew);    
        // 将集合c的全部元素拷贝到数组elementData中    
        System.arraycopy(a, 0, elementData, elementCount, numNew);    
        elementCount += numNew;    
        return numNew != 0;    
    }    
   
    // 删除集合c的全部元素    
    public synchronized boolean removeAll(Collection<?> c) {    
        return super.removeAll(c);    
    }    
   
    // 删除“非集合c中的元素”    
    public synchronized boolean retainAll(Collection<?> c)  {    
        return super.retainAll(c);    
    }    
   
    // 从index位置开始，将集合c添加到Vector中    
    public synchronized boolean addAll(int index, Collection<? extends E> c) {    
        modCount++;    
        if (index < 0 || index > elementCount)    
            throw new ArrayIndexOutOfBoundsException(index);    
   
        Object[] a = c.toArray();    
        int numNew = a.length;    
        ensureCapacityHelper(elementCount + numNew);    
   
        int numMoved = elementCount - index;    
        if (numMoved > 0)    
        System.arraycopy(elementData, index, elementData, index + numNew, numMoved);    
   
        System.arraycopy(a, 0, elementData, index, numNew);    
        elementCount += numNew;    
        return numNew != 0;    
    }    
   
    // 返回两个对象是否相等    
    public synchronized boolean equals(Object o) {    
        return super.equals(o);    
    }    
   
    // 计算哈希值    
    public synchronized int hashCode() {    
        return super.hashCode();    
    }    
   
    // 调用父类的toString()    
    public synchronized String toString() {    
        return super.toString();    
    }    
   
    // 获取Vector中fromIndex(包括)到toIndex(不包括)的子集    
    public synchronized List<E> subList(int fromIndex, int toIndex) {    
        return Collections.synchronizedList(super.subList(fromIndex, toIndex), this);    
    }    
   
    // 删除Vector中fromIndex到toIndex的元素    
    protected synchronized void removeRange(int fromIndex, int toIndex) {    
        modCount++;    
        int numMoved = elementCount - toIndex;    
        System.arraycopy(elementData, toIndex, elementData, fromIndex,    
                         numMoved);    
   
        // Let gc do its work    
        int newElementCount = elementCount - (toIndex-fromIndex);    
        while (elementCount != newElementCount)    
            elementData[--elementCount] = null;    
    }    
   
    // java.io.Serializable的写入函数    
    private synchronized void writeObject(java.io.ObjectOutputStream s)    
        throws java.io.IOException {    
        s.defaultWriteObject();    
    }    
}   
```

##几点总结

Vector的源码实现总体与ArrayList类似，关于Vector的源码，给出如下几点总结：

1、Vector有四个不同的构造方法。无参构造方法的容量为默认值10，仅包含容量的构造方法则将容量增长量（从源码中可以看出容量增长量的作用，第二点也会对容量增长量详细说）明置为0。

2、注意扩充容量的方法ensureCapacityHelper。与ArrayList相同，Vector在每次增加元素（可能是1个，也可能是一组）时，都要调用该方法来确保足够的容量。当容量不足以容纳当前的元素个数时，就先看构造方法中传入的容量增长量参数CapacityIncrement是否为0，如果不为0，就设置新的容量为就容量加上容量增长量，如果为0，就设置新的容量为旧的容量的2倍，如果设置后的新容量还不够，则直接新容量设置为传入的参数（也就是所需的容量），而后同样用Arrays.copyof()方法将元素拷贝到新的数组。

3、很多方法都加入了synchronized同步语句，来保证线程安全。

4、同样在查找给定元素索引值等的方法中，源码都将该元素的值分为null和不为null两种情况处理，Vector中也允许元素为null。

5、其他很多地方都与ArrayList实现大同小异，Vector现在已经基本不再使用。 