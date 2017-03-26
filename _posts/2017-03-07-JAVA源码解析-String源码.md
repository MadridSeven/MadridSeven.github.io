---
layout: post
title: "JAVA源码解析-String源码"
date: 2017-03-07 
description: "JAVA，String源码"
tag: JAVA源码解析
 
---
&emsp;&emsp;开学已经一周了，刚开学也是忙的不亦乐乎，总想开个好头嘛。眼看这就大三下学期了，表示压力山大啊，感觉再过上个不到半年我就成个社会人了，想着都怕。今天在知乎上闲逛，发现有初学者问关于String的问题，看了些解答之后突然很想看看String的源码，毕竟String类在平时的开发中用的相当相当多，所以就开始写这一专题《JAVA源码解析》，在String之后还会和大家一起阅读JAVA集合类的源码。当然这并不表示我的《JAVA基础进阶》就不写了，以后还会继续更新。←（开篇得有一段废话这是惯例）


----------

## 一、String类
&emsp;&emsp;String 类代表字符串。Java 程序中的所有字符串字面值（如 "abc" ）都作为此类的实例来实现的。首先String类是被final所修饰的，所以不允许被继承和修改，String类实现了Serializable、Comparable、CharSequence这三个接口，Serializable接口使得String可序列化；Comparable为String提供了比较器，使其可进行排序；CharSequence接口有length()，charAt(int index)，subSequence(int start,int end)方法。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence
```
## 二、String属性
&emsp;&emsp;String声明了4个变量如下图：

![这里写图片描述](http://img.blog.csdn.net/20170306221038410?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTWFkcmlkQmFp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

&emsp;&emsp;String类中包含一个不可变的char数组用来存放字符串（在Java中String类其实就是对字符数组的封装），一个int型的变量hash用来存放计算后的哈希值。serialVersionUID变量提供序列化的ID，serialPersistentFields变量声明了一个可序列化的字段。


```java
	private final char value[];
	//一个不可变的char数组用来存放字符串
	private final char value[];
	//一个int型的变量hash用来存放计算后的哈希值
    private int hash; // Default to 0
	//提供序列化的ID
    private static final long serialVersionUID = -6849794470754667710L;
	//声明了一个可序列化的字段
    private static final ObjectStreamField[] serialPersistentFields =
        new ObjectStreamField[0];
```
## 三、String构造方法
&emsp;&emsp;String类型共有五个常用的构造器，其代码如下。

```java
	//不带参数的构造函数
	public String() {
        this.value = "".value;
    }

	//参数为String类型的构造函数
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }

	//参数为字符数组的构造函数
    public String(char value[]) {
        this.value = Arrays.copyOf(value, value.length);
    }

	//参数为字符数组的构造函数，offset为复制的其实位置，count为要复制的字符个数
    public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= value.length) {
                this.value = "".value;
                return;
            }
        }
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }

	/*整形数组作为参数的构造函数，分配一个新的 String，它包含 Unicode 代码点数组参数一个子数组的字符。offset 参数是该子数组第一个代码点的索引，count 参数指定子数组的长度。将该子数组的内容转换为 char；后续对 int 数组的修改不会影响新创建的字符串。*/
    public String(int[] codePoints, int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= codePoints.length) {
                this.value = "".value;
                return;
            }
        }
        if (offset > codePoints.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }

        final int end = offset + count;

        
        int n = count;
        for (int i = offset; i < end; i++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                continue;
            else if (Character.isValidCodePoint(c))
                n++;
            else throw new IllegalArgumentException(Integer.toString(c));
        }

        
        final char[] v = new char[n];

        for (int i = offset, j = 0; i < end; i++, j++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                v[j] = (char)c;
            else
                Character.toSurrogates(c, v, j++);
        }

        this.value = v;
    }
```
## 四、String常用方法
&emsp;&emsp;String类提供了许多对字符串进行操作的方法，很多方法都在平时的开发中经常使用，我们一起来看看一些常用的方法是如何实现的。

&emsp;&emsp;getChars方法是将一个String字符串，按照给定的参数复制到目标字符数组的方法。其中传入4个参数：int类型的srcBegin为字符串中要复制的第一个字符的索引；int类型的srcEnd为字符串中要复制的最后一个字符之后的索引（要复制的最后一个字符位于索引 srcEnd-1 处）；char类型的数组dst[]为目标数组；int类型的desBegin为目标数组中的起始偏移量。

```java
    public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
        if (srcBegin < 0) {
            throw new StringIndexOutOfBoundsException(srcBegin);
        }
        if (srcEnd > value.length) {
            throw new StringIndexOutOfBoundsException(srcEnd);
        }
        if (srcBegin > srcEnd) {
            throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
        }
        System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
    }
```

&emsp;&emsp;equals方法用来将此字符串与指定的对象比较。当且仅当该参数不为 null，并且是与此对象表示相同字符序列的 String 对象时，结果才为 true。源代码如下：

```java
    public boolean equals(Object anObject) {
	    //如果引用的是同一个对象，返回真
        if (this == anObject) {
            return true;
        }
        //如果不是String类型的数据，返回假
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            //如果char数组长度不相等，返回假
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                //从后往前单个字符判断，如果有不相等，返回假
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                //每个字符都相等，返回真
                return true;
            }
        }
        return false;
    }
```
&emsp;&emsp;这段代码写的很好，说不出来的好，有一种美感。（JAVA的所有源代码其实都是(∩_∩)）

&emsp;&emsp;startsWith方法用来测试此字符串从指定索引开始的子字符串是否以指定前缀开始，该方法有两个参数，分别是String类型的prefix表示前缀，int类型的toffset表示在当前字符串中开始查找的位置。以下是源代码：

```java
    public boolean startsWith(String prefix, int toffset) {
        char ta[] = value;
        int to = toffset;
        char pa[] = prefix.value;
        int po = 0;
        int pc = prefix.value.length;
        /*如果开始查找的位置小于0或大于当前字符串长度与指定前缀长度的差值，则返回false*/
        if ((toffset < 0) || (toffset > value.length - pc)) {
            return false;
        }
        //从此字符串的指定索引开始比较是否与指定前缀相等
        while (--pc >= 0) {
            if (ta[to++] != pa[po++]) {
	            //不相等返回false
                return false;
            }
        }
        //相等返回true
        return true;
    }
```

&emsp;&emsp;concat方法用来将指定字符串连接到此字符串的结尾，如果参数字符串的长度为 0，则返回此 String 对象。否则，创建一个新的 String 对象，用来表示由此 String 对象表示的字符序列和参数字符串表示的字符序列连接而成的字符序列。以下是源代码：

```java
    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
```

&emsp;&emsp;trim方法返回字符串的副本，忽略前导空白和尾部空白。这在开发中也是很常用的方法，源代码如下：

```java
public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */

    //找到字符串前段没有空格的位置
    while ((st < len) && (val[st] <= ' ')) {
        st++;
    }
    //找到字符串末尾没有空格的位置
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    //如果前后都没有出现空格，返回字符串本身
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
```


&emsp;&emsp;subString方法返回一个新字符串，它是此字符串的一个子字符串。该子字符串从指定的 beginIndex 处开始，直到索引 endIndex - 1 处的字符。因此，该子字符串的长度为 endIndex-beginIndex。该方法包含两个int类型的参数，分别是： beginIndex - 起始索引（包括），endIndex - 结束索引（不包括）。源码如下：

```java
    public String substring(int beginIndex, int endIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        if (endIndex > value.length) {
            throw new StringIndexOutOfBoundsException(endIndex);
        }
        int subLen = endIndex - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        /*如果起始索引为0并且结束索引为此字符串的长度则返回此字符串，否则创建新的字符串*/
        return ((beginIndex == 0) && (endIndex == value.length)) ? this
                : new String(value, beginIndex, subLen);
    }
```

<font color="green">注：我挑了一些常用的方法展现出来，有些方法没有一一列举大家可以自行查看jdk1.7的源码即可。</font>


----------


<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>


----------


<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>