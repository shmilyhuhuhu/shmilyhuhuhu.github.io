---
layout:     post
title:      ArrayList扩容机制
subtitle:   集合扩容
date:       2020-06-15
author:     shmily
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - 集合ArrayList
    - java
    - 面试
---


# ArrayList扩容机制
## 一、先从ArrayList的接口和父类说起

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

继承父类 *AbstractList<E>*

实现的接口有 *List<E>* 链表接口、*RandomAccess* 随机访问接口、*Cloneable* 克隆接口、*java.io.Serializable* 序列化接口

## 二、从构造函数说起

```
   /**
     * 默认初始容量大小
     */
    private static final int DEFAULT_CAPACITY = 10;
    
    /**
     * 默认初始容量的一个空数组
     */
	
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     *默认构造函数，使用初始容量10构造一个空列表(无参数构造)
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    /**
     * 带初始容量参数的构造函数。（用户自己指定容量）
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {//初始容量大于0
            //创建initialCapacity大小的数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {//初始容量等于0
            //创建空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {//初始容量小于0，抛出异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }


   /**
    *构造包含指定collection元素的列表，这些元素利用该集合的迭代器按顺序返回
    *如果指定的集合为null，throws NullPointerException。 
    */
     public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
*用无参的构造方法创建ArrayList时，实际上初始化的是一个空数组。当真正对数组进行add添加元素操作时，才会真正的分配容量。即向数组中添加第一个元素时，数组容量扩为10*

## 三、一步一步分析ArrayList扩容机制

以无参构造函数创建ArrayList为例

### 1、先看add方法
```
    /**
     * 将指定的元素追加到此列表的末尾。 
     */
    public boolean add(E e) {
   //添加元素之前，先调用ensureCapacityInternal方法,为了得到最小扩容size
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //这里看到ArrayList添加元素的实质就相当于为数组赋值
        elementData[size++] = e;
        return true;
    }
```
### 2、再看ensureCapacityInternal()方法

可以看到我们在add方法里面，首先调用了ensureCapacityInternal方法


```
   //得到最小扩容量
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
              // 获取默认的容量和传入参数的较大值
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```
当要add进第一个元素的时候，minCapacity=1，在Math.max()方法比较之后，minCapacity为10。（*所以在默认的情况，在添加第一个元素的时候ArrayList被扩容成长度为10的一个数组*）

### ensureExplicitCapacity()方法
如果调用ensureCapacityInternal()方法就一定会进到ensureExplicitCapacity()方法中。接下来看一下这个方法的源码：

```
  //判断是否需要扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            //调用grow方法进行扩容，调用此方法代表已经开始扩容了
            grow(minCapacity);
    }
```
整个grow()的过程如下：
 
 * 当我们要add进第一个元素到ArrayList时，elementData为0（因为还是一个空的list），因为执行了ensureCapacityInternal()方法，所以minCapacity此时会变成10.此时，minCapacity-elementData.length>0成立，所以会进入grow(minCapacity)方法
 * 当添加第二元素的时候，minCapacity为2，此时的elementData.length在添加完第一个元素的时候已经变成10了，此时minCapacity-elementData.length>0不成立，所以并不会进入到grow(minCapacity)方法。
 * 添加第3，4····到第十个都不会满足条件去执行grow(minCapacity)方法，此时的数组容量一直为10，直到添加第11个元素，minCapacity(为11)比elementData.length(为10)要大。进入grow方法进行扩容。

### grow()方法
这个才是ArrayList的核心扩容方法

```
    /**
     * 要分配的最大数组大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * ArrayList扩容的核心方法。
     */
    private void grow(int minCapacity) {
        // oldCapacity为旧容量，newCapacity为新容量
        int oldCapacity = elementData.length;
        //将oldCapacity 右移一位，其效果相当于oldCapacity /2，
        //我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍，
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量，
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
       // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，
       //如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 MAX_ARRAY_SIZE 即为 `Integer.MAX_VALUE - 8`。
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
int newCapacity = oldCapacity + (oldCapacity >> 1);所以ArrayList每次扩容之后容量都会变成原来的1.5倍左右，（oldCapacity为偶数就是1.5倍，奇数的话就不是准确的1.5倍，奇数会丢掉小数）


*(位移运算)：>>1 右移一位相当于除2，右移n位相当于除以 2 的 n 次方。这里 oldCapacity 明显右移了1位所以相当于oldCapacity /2。对于大数据的2进制运算,位移运算符比那些普通运算符的运算要快很多,因为程序仅仅移动一下而已,不去计算,这样提高了效率,节省了资源*

（为什么位移运算比普通运算符效率高？：从效率上看，使用移位指令有更高的效率，因为移位指令占2个机器周期，而乘除法指令占4个机器周期。从硬件上看，移位对硬件更容易实现，所以会用移位，移一位就乘2,这种乘法当然考虑移位了。）

所以扩容的基本过程如下：

* 当add第一个元素时，oldCapacity为0，比较滞后第一个if判断成立，newCapacity = minCapacity(为10)。但是第二个if判断不会成立，即newCapacity不比MAX_ARRAY_SIZE大不会进入hugeCapacity方法，数组容量为10，add方法中return true，然后size增加1.
* 当add第11个元素进入grow方法时，newCapacity为15，比minCapacity(为11)大，第一个if判断不成立。新容量没有大于数组最大size，不会进入hugeCapacity方法。数组容量扩为15，add方法中return true，size增为11.


### hugeCapacity()方法
从上面 grow() 方法源码我们知道： 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) hugeCapacity() 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，如果minCapacity大于最大容量，则新容量则为Integer.MAX_VALUE，否则，新容量大小则为 MAX_ARRAY_SIZE 即为 Integer.MAX_VALUE - 8。

```
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        //对minCapacity和MAX_ARRAY_SIZE进行比较
        //若minCapacity大，将Integer.MAX_VALUE作为新数组的大小
        //若MAX_ARRAY_SIZE大，将MAX_ARRAY_SIZE作为新数组的大小
        //MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

## 四、system.arraycopy()和Arrays.copyOf()方法
在arrayList中大量调用了这个两个方法，比如：我们上面讲的扩容操作以及add(int index,E element)都会调用到该方法

### 4.1 system.arraycopy()方法

```
    /**
     * 在此列表中的指定位置插入指定的元素。 
     *先调用 rangeCheckForAdd 对index进行界限检查；然后调用 ensureCapacityInternal 方法保证capacity足够大；
     *再将从index开始之后的所有成员后移一个位置；将element插入index位置；最后size加1。
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //arraycopy()方法实现数组自己复制自己
        //elementData:源数组;index:源数组中的起始位置;elementData：目标数组；index + 1：目标数组中的起始位置； size - index：要复制的数组元素的数量；
        System.arraycopy(elementData, index, elementData, index + 1, size - index);
        elementData[index] = element;
        size++;
    }
```

### 4.2 Arrays.copyOf()方法

```
   /**
     以正确的顺序返回一个包含此列表中所有元素的数组（从第一个到最后一个元素）; 返回的数组的运行时类型是指定数组的运行时类型。 
     */
    public Object[] toArray() {
    //elementData：要复制的数组；size：要复制的长度
        return Arrays.copyOf(elementData, size);
    }
```

### 4.3 两者的联系和区别
联系：

看两者源代码可以发现 copyOf() 内部实际调用了 System.arraycopy() 方法

区别：

arraycopy() 需要目标数组，将原数组拷贝到你自己定义的数组里或者原数组，而且可以选择拷贝的起点和长度以及放入新数组中的位置 copyOf() 是系统自动在内部新建一个数组，并返回该数组。

## 五 ensureCapacity方法
源码如下：

```
    /**
    如有必要，增加此 ArrayList 实例的容量，以确保它至少可以容纳由minimum capacity参数指定的元素数。
     *
     * @param   minCapacity   所需的最小容量
     */
    public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
```

最好在 add 大量元素之前用 ensureCapacity 方法，以减少增量重新分配的次数

下面的方法进行测试：
```
public class EnsureCapacityTest {
	public static void main(String[] args) {
		ArrayList<Object> list = new ArrayList<Object>();
		final int N = 10000000;
		long startTime = System.currentTimeMillis();
		for (int i = 0; i < N; i++) {
			list.add(i);
		}
		long endTime = System.currentTimeMillis();
		System.out.println("使用ensureCapacity方法前："+(endTime - startTime));

	}
}
```






















