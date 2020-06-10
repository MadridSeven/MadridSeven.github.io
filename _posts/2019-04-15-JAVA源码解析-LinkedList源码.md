---
layout: post
title: "JAVA源码解析-LinkedList源码"
date: 2019-04-15
description: "JAVA，LinkedList源码"
tag: JAVA源码解析

---


### 概述

&emsp;&emsp;LinkedList 是 list 接口的链表实现，它是基于双向循环链表实现的，这一点在源码中
很容易就能看出来。除了可以当做链表来操作外，它还可以当做栈、队列和双端队列来使用。

&emsp;&emsp;LinkedList 和 ArrayList 一样，它不是线程安全的，在多线程下可以考虑使用 Collections.synchronizedList
方法将该列表“包装”起来。这最好在创建时完成，以防止意外对列表进行不同步的访问：代码如下：

```Java
List list = Collections.synchronizedList(new LinkedList(...));
```

### 继承的父类及实现的接口

&emsp;&emsp;LinkedList 继承了 AbstractSequentialList 类，此类提供了 List 接口的骨干实现，
，继承该类的子类适合用于“连续访问”数据存储（如链表）。与此类对应的有 AbstractList 类，继承此
类的子类适合用于“随机访问”数据存储（如数组），代表的子类有 ArrayList 和 Vector 。

&emsp;&emsp;LinkedList 实现了 List 接口，List 接口通常表示一个列表（数组、队列、链表、栈等），
其中的元素可以重复，代表的实现类有 ArrayList、LinkedList、Stack,、Vector。<br>
&emsp;&emsp;LinkedList 实现了 Deque 接口，表示该类可以当做双端队列来使用。队列通常不允许null元素，
因为我们通常调用poll方法是否返回null来判断队列是否为空，但是LinkedList是允许null元素的，所以，
在使用LinkedList最为队列的实现时，不应该将null元素插入队列；<br>
&emsp;&emsp;LinkedList 实现了 Cloneable 接口，以指示 Object.clone() 方法可以合法地
对该类实例进行按字段复制。<br>
&emsp;&emsp;LinkedList 实现了 Serializable 接口，因此它支持序列化，能够通过序列化传输。

### 源码分析

&emsp;&emsp;本文所涉及的源码的 jdk 版本为1.8，我并没有将所有的源码拷贝过来，省略了部分不
常用的方法，源码中都加入了比较详细的注释。我对代码方法的顺序略微进行了调整，比如把节点类型
Node 的定义放在了前面，这是为了方便大家阅读。

&emsp;&emsp;注：LinedList 的头元素为空，我在代码的注释里提到的头元素或头节点，均指的是当前
链表的第一个非空元素或第一个以非空元素作为当前值的节点。

```Java

public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    //当前的实际元素个数
    transient int size = 0;
    //节点的定义
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

    //头节点
    transient Node<E> first;
    //尾节点
    transient Node<E> last;

    //无参的构造方法
    public LinkedList() {
    }
    //以Collection集合作为参数的构造方法
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    //从表头插入一个元素（内部方法）
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;

        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }

    //从表尾插入一个元素（只能同类或同包下使用）
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

    //在非空节点succ之前插入元素e。（只能同类或同包下使用）
    void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

    //在源码中作为删除头节点的方法（内部方法）
    private E unlinkFirst(Node<E> f) {
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }

    //在源码中作为删除尾节点的方法（内部方法）
    private E unlinkLast(Node<E> l) {
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }

    //删除节点x。（只能同类或同包下使用）
    //该方法将在文后总结中重点分析
    E unlink(Node<E> x) {
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
        /*
          如果前驱节点为空，则当前节点就是头元素，（节点为空和节点的值为空是两个不同的概念）
          删除该节点就把头元素指向该节点的后继节点，再将该节点置为空即可
        */
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }

    //返回头节点的当前元素值
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

    //返回尾节点的当前元素值
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

    //删除头节点
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

    //删除尾节点
    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }

    //从表头插入元素e
    public void addFirst(E e) {
        linkFirst(e);
    }

    //从表尾插入元素e
    public void addLast(E e) {
        linkLast(e);
    }

    //如果包含元素o则返回true
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    /*
    返回此列表中首次出现的指定元素的索引，如果此列表中不包含该元素，则返回 -1
    */
    public int indexOf(Object o) {
        //从头节点开始向后遍历整个链表进行查找
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }

    /*
    返回此列表中最后出现的指定元素的索引，如果此列表中不包含该元素，则返回 -1
    */
    public int lastIndexOf(Object o) {
        //从表尾节点开始向前遍历整个链表进行查找
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }

    //返回链表中当前元素的个数
    public int size() {
        return size;
    }

    //添加一个元素（添加到表尾）
    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    //找到元素o并删除
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    //将collection中的元素添加到表尾
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    //将collection中的元素从指定位置开始加入到表中
    public boolean addAll(int index, Collection<? extends E> c) {
        //index >= 0 && index <= size
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;
        //succ保存的是index位置的元素
        Node<E> pred, succ;
        //index == size 则为从表尾插入
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            //当index != size 的时候， 那么index就一定出现在之前链表中
            //此处的调用的node方法，下面源码也已经给出，
            //node方法的主要作用就是判断index所处位置是在之前链表的上半部分还是下半部分，
            //在上半部分就从第一个元素开始循环，循环到index位置时返回元素,
            //如果是在后半部分，那么就从最后一个元素往前循环，循环到index位置时返回元素
            succ = node(index);
            //得到index所在元素的上一个元素引用
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            //构造Node对象， 此时Node对象持有对前一个元素以及当前元素的引用
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                //如果前一个元素为null, 那么说明之前不存在链表，此元素将设置为链表的第一个元素
                first = newNode;
            else
                //如果已存在链表，那么就从之前链表的index位置开始插入
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

    //清空该链表
    public void clear() {
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }

    //返回链表中指定位置的元素
    public E get(int index) {
        //index >= 0 && index < size
        checkElementIndex(index);
        return node(index).item;
    }

    //将此链表中指定位置的元素替换为指定的元素，返回该位置原元素
    public E set(int index, E element) {
        //index >= 0 && index < size
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }

    //在此链表中指定的位置插入指定的元素
    public void add(int index, E element) {
        //index >= 0 && index <= size
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    //删除指定位置的节点
    public E remove(int index) {
          //index >= 0 && index < size
        checkElementIndex(index);
        return unlink(node(index));
    }

    //上面已经把这方法解释了一遍，这儿就不多说了，贴出来就为了让大家看得直观
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

}



```

&emsp;&emsp;另外，LinkedList 还为实现队列和栈的操作提供了如 offer(E e)、peek()、poll()、pop()、push()等方法，这些方法其实就是调用 LinkedList 的其他方法来达到队列和栈的效果，这里就不列出来了。

### 总结

&emsp;&emsp;关于 LinkedList 的源码和使用，给出几点比较重要的总结：

&emsp;&emsp;1.在查找和删除某元素时，源码中都划分为该元素为null和不为null两种情况来处理，LinkedList中允许元素为null。

&emsp;&emsp;2.LinkedList是基于链表实现的，因此不存在容量不足的问题，所以这里没有扩容的方法。

&emsp;&emsp;3.要理解 LinkedList 中删除元素的方法unlink，举个形象的例子：李四左手牵着张三，右手牵着王五， 现在我们要删除李四， 那么只需要直接将张三的手牵向王五。

&emsp;&emsp;4. LinkedList 基于双向链表实现， 因此具有链表 插入快、 索引慢的特性，与 ArrayList 正好相反。

&emsp;&emsp;5. LinkedList 实现了栈和队列的操作方法，因此也可以作为栈、队列和双端队列来使用。


----------
<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>

----------



<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>
