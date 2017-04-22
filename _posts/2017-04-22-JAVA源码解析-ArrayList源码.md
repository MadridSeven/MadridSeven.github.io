---
layout: post
title: "JAVA源码解析-ArrayList源码"
date: 2017-04-22
description: "JAVA，ArrayList源码"
tag: JAVA源码解析

---

### 概述

&emsp;&emsp;ArrayList 是基于数组实现的，是一个动态数组，其容量能自动增长。ArrayList不是线程同步的，在多线程下可以考虑使用 Collections.synchronizedList 方法将该列表“包装”起来。这最好在创建时完成，以防止意外对列表进行不同步的访问：代码如下：
```Java
List list = Collections.synchronizedList(new ArrayList(...));
```

### 继承的父类及实现的接口

&emsp;&emsp;ArrayList 继承了 AbstractList 类，此类提供了 List 接口的骨干实现，继承此类的子类适合用于“随机访问”数据存储（如数组），Vector 也是此类的子类。与 AbstractList 类对应的类是 AbstractSequentialList 类，继承该类的子类适合用于“连续访问”数据存储（如链接列表），代表的子类如 LinkedList 。

&emsp;&emsp;ArrayList 实现了 List 接口，List 接口通常表示一个列表（数组、队列、链表、栈等），其中的元素可以重复，代表的实现类有 ArrayList、LinkedList、Stack,、Vector。<br/>
&emsp;&emsp;ArrayList 实现了 RandomAccess 接口，该接口为标记接口，用来表明其支持快速随机访问。<br/>
&emsp;&emsp;ArrayList 实现了 Cloneable 接口，以指示 Object.clone() 方法可以合法地对该类实例进行按字段复制。<br/>
&emsp;&emsp;ArrayList 实现了 Serializable 接口，因此它支持序列化，能够通过序列化传输。



### 源码分析

&emsp;&emsp;本文所涉及的源码的 jdk 版本为1.8，我并没有将所有的源码拷贝过来，省略了部分不常用的方法，源码中都加入了比较详细的注释。

```Java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    // 序列版本号
    private static final long serialVersionUID = 8683452581122892189L;
    //默认容量
    private static final int DEFAULT_CAPACITY = 10;
    //常量表示一个无参的空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
    //常量表是一个带参的空数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    //ArrayList基于该数组实现
    transient Object[] elementData;
    //当前的实际元素个数
    private int size;

    //用初始容量作为参数的构造方法
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            //初始容量大于0,实例化数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            //初始化等于0，将空数组赋给elementData
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    //无参的构造方法
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    //以Collection集合作为参数的构造方法
    public ArrayList(Collection<? extends E> c) {
        ////返回包含c所有元素的数组
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                //复制指定数组，使elementData具有指定长度
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            //c中没有元素
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

    //将当前容量值设为当前实际元素大小
    public void trimToSize()
    {
        modCount++;
        if (size < elementData.length)
        {
            elementData = (size == 0)? EMPTY_ELEMENTDATA:Arrays.copyOf(elementData, size);
        }
    }

    //将集合的capacit增加minCapacity
    public void ensureCapacity(int minCapacity)
    {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)?0:DEFAULT_CAPACITY;
        if (minCapacity > minExpand)
        {
            ensureExplicitCapacity(minCapacity);
        }
    }

    private void ensureCapacityInternal(int minCapacity)
    {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity)
    {
        modCount++;
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    private void grow(int minCapacity)
    {
        int oldCapacity = elementData.length;
　　　　　//注意此处扩充capacity的方式是将其向右移位再加上原来的数，实际上是扩充了1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity)
    {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    //返回ArrayList的大小
    public int size()
    {
        return size;
    }

    //判断ArrayList是否为空
    public boolean isEmpty() {
        return size == 0;
    }

    //判断ArrayList中是否包含Object(o)
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    //正向查找，返回ArrayList中元素Object(o)的索引位置
    public int indexOf(Object o)
    {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        }
        else
        {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    //逆向查找，返回返回ArrayList中元素Object(o)的索引位置
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

    //返回此 ArrayList实例的浅拷贝。
    public Object clone()
    {
        try
        {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        }
        catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }

    //返回一个包含ArrayList中所有元素的数组
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }

    @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }

    //返回至指定索引的值
    public E get(int index)
    {
        rangeCheck(index); //检查给定的索引值是否越界
        return elementData(index);
    }

    //将指定索引上的值替换为新值，并返回旧值
    public E set(int index, E element)   
    {
        rangeCheck(index);
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

    //将指定的元素添加到此列表的尾部
    public boolean add(E e)
    {
        ensureCapacityInternal(size + 1);  
        elementData[size++] = e;
        return true;
    }

    // 将element添加到ArrayList的指定位置   
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacityInternal(size + 1);

        //从指定源数组中复制一个数组，复制从指定的位置开始，到目标数组的指定位置结束。
        //arraycopy(被复制的数组, 从第几个元素开始复制, 要复制到的数组, 从第几个元素开始粘贴, 一共需要复制的元素个数)
        //即在数组elementData从index位置开始，复制到index+1位置,共复制size-index个元素
        System.arraycopy(elementData, index, elementData, index + 1,size - index);
        elementData[index] = element;
        size++;
    }

    //删除ArrayList指定位置的元素  
    public E remove(int index)
    {
        rangeCheck(index);
        modCount++;
        E oldValue = elementData(index);
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
        elementData[--size] = null; //将原数组最后一个位置置为null
        return oldValue;
    }

    //移除ArrayList中首次出现的指定元素(如果存在)。
    public boolean remove(Object o) {
        if (o == null)
        {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null)
                {
                    fastRemove(index);
                    return true;
                }
        }
        else
        {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index]))
                {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    //快速删除指定位置的元素
    private void fastRemove(int index)
    {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        elementData[--size] = null;
    }

    //清空ArrayList，将全部的元素设为null
    public void clear()
    {
        modCount++;
        for (int i = 0; i < size; i++)
            elementData[i] = null;
        size = 0;
    }

    //按照c的迭代器所返回的元素顺序，将c中的所有元素添加到此列表的尾部
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

    //从指定位置index开始，将指定c中的所有元素插入到此列表中
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        int numMoved = size - index;
        if (numMoved > 0)
            //先将ArrayList中从index开始的numMoved个元素移动到起始位置为index+numNew的后面去
            System.arraycopy(elementData, index, elementData, index + numNew, numMoved);
        //再将c中的numNew个元素复制到起始位置为index的存储空间中去
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

    //删除fromIndex到toIndex之间的全部元素
    protected void removeRange(int fromIndex, int toIndex)
    {
        modCount++;
        //numMoved为删除索引后面的元素个数
        int numMoved = size - toIndex;  
        //将删除索引后面的元素复制到以fromIndex为起始位置的存储空间中去
        System.arraycopy(elementData, toIndex, elementData, fromIndex,numMoved);
        int newSize = size - (toIndex-fromIndex);
        //将ArrayList后面(toIndex-fromIndex)个元素置为null
        for (int i = newSize; i < size; i++)
        {
            elementData[i] = null;
        }
        size = newSize;
    }

    //检查索引是否越界
    private void rangeCheck(int index)
    {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    private void rangeCheckForAdd(int index)
    {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private String outOfBoundsMsg(int index) {
        return "Index: "+index+", Size: "+size;
    }

    //删除ArrayList中包含在c中的元素
    public boolean removeAll(Collection<?> c)
    {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    //删除ArrayList中除包含在c中的元素，和removeAll相反
    public boolean retainAll(Collection<?> c)
    {
        Objects.requireNonNull(c);  //检查指定对象是否为空
        return batchRemove(c, true);
    }

    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try
        {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)  //判断c中是否有elementData[r]元素

                    elementData[w++] = elementData[r];
        }
        finally
        {
            if (r != size)
            {
                System.arraycopy(elementData, r, elementData, w, size - r);
                w += size - r;
            }
            if (w != size)
            {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }

    //将ArrayList的“容量，所有的元素值”都写入到输出流中
    private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException
    {
        int expectedModCount = modCount;
        s.defaultWriteObject();
        //写入数组大小
        s.writeInt(size);
        //写入所有数组的元素
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    //先将ArrayList的“大小”读出，然后将“所有的元素值”读出
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        s.defaultReadObject();
        s.readInt(); // ignored
        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);
            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
}
```

### 总结

&emsp;&emsp;关于 ArrayList 的源码和使用，给出几点比较重要的总结：

&emsp;&emsp;注意扩充容量的方法ensureCapacity。ArrayList在每次增加元素（可能是1个，也可能是一组）时，都要调用该方法来确保足够的容量。当容量不足以容纳当前的元素个数时，就设置新的容量为旧的容量的1.5倍，如果设置后的新容量还不够，则直接新容量设置为传入的参数（也就是所需的容量），而后用Arrays.copyof()方法将元素拷贝到新的数组（详见下面的第3点）。从中可以看出，当容量不够时，每次增加元素，都要将原来的元素拷贝到一个新的数组中，非常之耗时，<font color="red">也因此建议在事先能确定元素数量的情况下，才使用ArrayList，否则建议使用LinkedList。</font>

&emsp;&emsp;ArrayList的实现中大量地调用了Arrays.copyof()和System.arraycopy()方法。我们有必要对这两个方法的实现做下深入的了解。</br>
&emsp;&emsp;先来看Arrays.copyof()。它有很多个重载的方法，但实现思路都是一样的，我们来看泛型版本的源码：
```Java
public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
}
```
&emsp;&emsp;很明显调用了另一个copyof方法，该方法有三个参数，最后一个参数指明要转换的数据的类型，其源码如下：
```Java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
}
```
&emsp;&emsp;这里可以很明显地看出，该方法实际上是在其内部又创建了一个长度为newlength的数组，调用System.arraycopy()方法，将原来数组中的元素复制到了新的数组中。

&emsp;&emsp;下面来看System.arraycopy()方法。该方法被标记了native，调用了系统的C/C++代码，在JDK中是看不到的，但在openJDK中可以看到其源码。该函数实际上最终调用了C语言的memmove()函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在复制大量数组元素时用该方法，以取得更高的效率。

&emsp;&emsp;<font color="red">ArrayList基于数组实现，可以通过下标索引直接查找到指定位置的元素，因此查找效率高，但每次插入或删除元素，就要大量地移动元素，插入删除元素的效率低。</font>

----------
<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>

----------



<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>
