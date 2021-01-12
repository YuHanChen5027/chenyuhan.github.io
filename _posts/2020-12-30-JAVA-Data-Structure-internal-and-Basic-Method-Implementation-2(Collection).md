---
layout: post
title:  "JAVA数据结构内部及基础方法实现二（Collection）"
author: "陈宇瀚"
date:   2020-12-30 16:35:00 +0800
header-img: "img/img-head/img-head-java.jpg"
categories: article
tags:
  - JAVA 
  - 数据结构
---

JAVA有几种常用的数据结构，主要是继承**Collection**和**Map**这两个主要接口的数据实现类

在jdk1.7和jdk1.8中，实现会有些许不同，之后会在注解中添加两版本区别
下面分别介绍几个常用的数据结构(按照继承的接口分为两类)，以下代码截取自**基于JAVA8的android SDK 28**
# Collection
**Collection**是最基本的集合接口，一个**Collection**代表一组**Object**，即**Collection**的元素**Elements**。一些**Collection**允许相同的元素而另一些不行。一些能排序而另一些不行。**Java SDK**不提供直接继承自**Collection**的类，**Java SDK**提供的类都是继承自**Collection的**“子接口”如**List**和**Set**。

论**Collection**的实际类型如何，它都支持一个**iterator**()的方法，该方法返回一个**迭代子**，使用该迭代子即可逐一访问**Collection**中每一个元素。典型的用法如下：
```java
Iterator it = collection.iterator(); // 获得一个迭代子
　　　　while(it.hasNext()) {
　　　　　　Object obj = it.next(); // 得到下一个元素
　　　　}
```
由**Collection**接口派生的两个接口是**List**和**Set**。
## List
**List**是有序的**Collection**，使用此接口能够精确的控制每个元素插入的位置。用户能够使用索引（元素在**List**中的位置，类似于数组下标）来访问**List**中的元素，这类似于**Java**的数组，与之后介绍的**Set**不一样，**List**允许有相同的元素。实现**List**接口的常用类有**LinkedList**，**ArrayList**，**Vector**和**Stack**。

### LinkedList（JAVA7/8中基本没有改动）
双向链表结构，适用于乱序插入、删除。但指定序列操作性能不如**ArrayList**。

**LinkedList**的父类接口，以及内部有几个主要的变量，如下：
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
    
transient int size = 0;//链表尺寸
transient Node<E> first;//链表第一个节点指针
transient Node<E> last;//链表末尾节点指针

##注transient关键字：将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会被序列化。

```
*注：实现了Cloneable接口，即实现clone()函数。代表能被克隆。*
数据类**Node**的结构
```java
  private static class Node<E> {
        E item;//节点数据
        Node<E> next;//当前节点下一个节点
        Node<E> prev;//当前节点上一个节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
#### 构造方法
```java
 public LinkedList() {
 }
 //构造一个包含指定元素的列表
 public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
}
    
public boolean addAll(Collection<? extends E> c) {
        //可参考下方的addAll解析，就是在size(此时size = 0)后加入指定元素的列表的数据
        return addAll(size, c);
}
```

以下是使用时的常用方法实现代码以及分析:

*注：modCount是父类AbstractList中定义的一个int型的属性记录了ArrayList结构性变化的次数。List中add()、remove()、addAll()、removeRange()及clear()这些方法每调用一次，modCount的值就加1。*

#### add()/addLast(E)/add(int,E)
**add(E)/addLast(E)**:将指定的元素添加到列表的末尾。
```java
public boolean add(E e) {
        linkLast(e);
        return true;
    }
public void addLast(E e) {
        linkLast(e);
    }
    
    void linkLast(E e) {
        final Node<E> l = last;//获取末尾节点
        final Node<E> newNode = new Node<>(l, e, null);//将传入的数据封装成 Node
        last = newNode; 
        if (l == null)//判断是否存在末尾节点
            first = newNode;//不存在则说明链表为空，直接将节点赋值为 newNode
        else
            l.next = newNode;//否则将末尾节点的next赋值为 newNode
        size++;
        modCount++;
    }
```
**add(int,E)**:在列表的指定位置插入指定的元素。
```java
 public void add(int index, E element) {
        //判断index下标是否在链表范围内(0=<index<=size),超出则抛出IndexOutOfBoundsException 
        checkPositionIndex(index);

        if (index == size)
            linkLast(element); //此时相当于插入末尾，与add(E)方法一致
        else
            linkBefore(element, node(index));//重点看这个方法，插入链表中间
    }
    
     /**
     * 在节点index前插入新节点
     */
    void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;//获得index位置节点的前置节点
        //创建新节点，前置节点为index位置节点的前置节点，后置节点为index位置节点
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;//改变index位置节点的前置节点为新节点
        if (pred == null) //判断index位置节点的前置节点是否为空
            first = newNode;    //为空代表插在链表头
        else
            pred.next = newNode;    //不为空将节点的后置节点设置为新节点
        size++;
        modCount++;
        
    }
```
#### remove(int)
**remove(int)**：删除列表中指定位置的元素。
```java
 public E remove(int index) {
        /**判断index下标是否在链表索引范围内(0=<index<size),超出则抛出IndexOutOfBoundsException**/
        checkElementIndex(index);
        return unlink(node(index));//node(index)获取到了index下表的Node
    }
    
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
        //判断前置节点是否为null。
        if (prev == null) {
            first = next;//为空则代表是第一个节点，将后置节点设置为首位节点
        } else {
            //不为空则代表是中间节点，将前置节点的后置节点设置为自己的后直节点，断开自己与前置节点的链接。
            prev.next = next 
            x.prev = null;
        }
        //同样的方式判断后置
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }
            
        x.item = null;//当前节点的数据置null
        size--;
        modCount++;
        return element; 
    }
```
#### get(int)
**get(int)**:Returns the element at the specified position in this list.
```java
public E get(int index){
        /**判断index下标是否在链表索引范围内(0=<index<size),超出则抛出IndexOutOfBoundsException**/
        checkElementIndex(index);
        return node(index).item;
    }
    
    /**
    *   返回指定元素索引处的(非空)节点。
    */
    Node<E> node(int index) {
        /**size >> 1相当于size/2**/
        if (index < (size >> 1)) { 
          /**如果获取index小于size/2，则从首位节点开始循环获取**/
            Node<E> x = first;  
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
          /**如果获取index大于size/2，则从末尾节点开始循环获取**/
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
#### set(int,E)
**set(int,E)**:将列表中指定位置的元素替换为指定的元素。
```java
  public E set(int index, E element) {
        /**判断index下标是否在链表索引范围内(0=<index<size),超出则抛出IndexOutOfBoundsException**/
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```
#### addAll(Collection<? extends E>)/addAll(int , Collection<? extends E>)
**addAll(Collection<? extends E>)/addAll(int , Collection<? extends E>)**
：将指定集合中的所有元素添加到末尾/index节点之后
```java
   //本质上是调用同个方法
   public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);//传入size,直接在末尾插入
    }
    
    public boolean addAll(int index, Collection<? extends E> c) {
        //判断插入位置index是否超出范围
        checkPositionIndex(index);
        //将集合转成数组
        Object[] a = c.toArray();
        int numNew = a.length;
        //集合数为0，等于没有数据
        if (numNew == 0)
            return false;

        //predpred是插入节点集合的前置节点
        //succ是插入位置节点，用于之后判断是中间插入还是末尾插入
        Node<E> pred, succ;
        if (index == size) {
            //index等于链表尺寸，代表是末尾插入
            succ = null;
            pred = last;
        } else {
            //中间插入，保留插入位置的节点
            succ = node(index); 
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null) 
                //前置节点为null,相当于插入链表头
                first = newNode;
            else
                //不为空，则从插入位置之后插入
                pred.next = newNode;
            pred = newNode;
        }
        //插入完成后，判断插入位置succ
        if (succ == null) {
            //succ为null,代表末尾插入
            last = pred;
        } else {
            //succ不为null,代表中间插入，设置插入全部数据后的最后一个数据节点后置节点为succ
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```
#### clear()
**clear()**:从列表中删除所有元素。
```java
public void clear() {
        //将链表中间的节点内容都设置为null
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        //将首位节点和末尾节点设置为null，尺寸设置为0
        first = last = null;
        size = 0;
        modCount++;
    }
```
还有一些其他方法，但都是基于以上方法的逻辑，就不介绍了。

### ArrayList（以下代码基于JAVA8）
**ArrayList**是动态数组，底层就是一个数组, 因此按序查找快, 乱序插入, 删除因为涉及到后面元素移位所以性能慢。
首先需要介绍一个**ArrayList**方法内常用的一个方法**Arrays.copyOf(T[],int)**，作用是复制数组，在**ArrayList**初始化，扩容时都会用到，代码如下
```java
 //第一个参数是原始数组，第二个是返回数组的长度
 public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
 }
 ////第一个参数是原始数组，第二个是返回数组的长度，第三个返回数组的类
 public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        //判断返回数组的类是否是Object,是的话就创建一个Object数组，不是的话就创建一个特定类的数组
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        //将original从第0个下标开始的Math.min(original.length, newLength)的数据拷贝至copy数组的第0个下标
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
 }

```
**ArrayList**的父类接口，以及内部有几个主要的变量，如下：
```java
 public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
    
 private static final int DEFAULT_CAPACITY = 10;//默认初始容量
 private static final Object[] EMPTY_ELEMENTDATA = {};//用于空实例的共享空数组实例。
 private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
transient Object[] elementData;//存储数组列表元素的数组缓冲区。
private int size;//实际元素个数

 private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;//要分配的数组的最大大小，超出的话可能会导致OutOfMemoryError
```
**EMPTY_ELEMENTDATA**和**DEFAULTCAPACITY_EMPTY_ELEMENTDATA**都是空数组，区别主要是为了判断是使用了哪种构造函数，无参构造创建时使用的是**DEFAULTCAPACITY_EMPTY_ELEMENTDATA**，有参构造创建时，当**initialCapacity == 0**使用的是**EMPTY_ELEMENTDATA**。以便确认如何扩容(下面会分析)。

*注：RandmoAccess接口，即提供了随机访问功能。*
### 构造方法
**构造方法**:**ArrayList**构造方法分为三个：
```java
//置为DEFAULTCAPACITY_EMPTY_ELEMENTDATA的空列表，之后会在第一次添加元素时扩容成10容量。
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
/**
*   按照集合的迭代器返回元素的顺序构造一个包含指定集合元素的列表。
*/
 public ArrayList(Collection<? extends E> c) {
        //将集合转成数组直接赋值给elementData
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
- 空参构造方法：将**elementData**设置为**DEFAULTCAPACITY_EMPTY_ELEMENTDATA**，最后会初始化成一个容量为10的空数组；
- 带**int**参数的构造方法：将**elementData**实例化成传入的初始容量大小的数组，若传入的值为0，则将**elementData**设置为**EMPTY_ELEMENTDATA**；
- 带**Collection**参数的构造方法：按照传入的集合构造一个包含指定元素的列表，r将集合转成数组直接赋值给**elementData**，将元素个数赋值给**size**，若元素个数不为0，最后将**elementData**转换成Object[]类的数组，若元素个数为0，则将**elementData**置为**EMPTY_ELEMENTDATA**；

总的来说：
- 无参构造:**elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA**;
- 有参构造:
- - **initialCapacity==0, elementData = EMPTY_ELEMENTDATA**        
- - **initialCapacity > 0, elementData = new Object[initialCapacity];** 

以下是使用时的常用方法实现代码以及分析:
#### add(E)/add(int,E)
**add(E)**:将指定的元素添加到列表的末尾，调用过程如下。
```java
 public boolean add(E e) {
        //判断增加一个元素后的容量是否超出当前数组容量，超过扩容，不超过则不变
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //将当前数组最后一个数据后的位置赋值为e
        elementData[size++] = e;
        return true;
 }
    
    private void ensureCapacityInternal(int minCapacity) {
        //判断当前elementData是否等于DEFAULTCAPACITY_EMPTY_ELEMENTDATA
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            //将所需大小设置成DEFAULT_CAPACITY(10)和minCapacity中比较大的值
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
    
        private void ensureExplicitCapacity(int minCapacity) {
            modCount++;
        
            // 如果所需的最小容量大于当前elementData的长度
            if (minCapacity - elementData.length > 0)
            grow(minCapacity);
        }
        
    //根据minCapacity(需要的最小容量)扩容
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;//获得当前数组长度
        int newCapacity = oldCapacity + (oldCapacity >> 1);//新容量为原容量的1.5倍
        if (newCapacity - minCapacity < 0) 
            //如果新容量仍小于所需最小容量，直接将新容量置为需要的最小容量
            newCapacity = minCapacity;  
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            //判断当前容量是否超出可分配的最大容量，超出则设置新容量为
            //Integer.MAX_VALUE，保证数组容量不超过Integer.MAX_VALUE
            newCapacity = hugeCapacity(minCapacity);
            
        //根据新容量创建新的数组赋值给elementData，此时minCapacity通常接近size
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        //Integer.MAX_VALUE = MAX_ARRAY_SIZE + 8;
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
**add(int,E)**:在列表中的指定位置插入指定的元素。将当前位于该位置的元素(如果有的话)和随后的元素向右移动(下标加1)。
```java
 public void add(int index, E element) {
        //插入位置超过当前数据尺寸或者插入位置小于0
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
        
        //判断插入数据后的数据是否超出数组范围，超出则扩容，不超出则不变
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //将elementData数组从index以及之后的数据拷贝至index+1的位置
        //相当于腾出index这个下标的位置，用于存放插入的数据element
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```
#### remove(int)
**remove(int):**:删除列表中指定位置的元素。将所有后续元素向左移动(从它们的下标减去1)。
```java
 public E remove(int index) {
        //判断是否超出范围
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        //获得删除位置的数据
        E oldValue = (E) elementData[index];
        
        //删除后需要移动的数据数量。
        //例共10个数据，删除第5个数据(下标index = 4)，
        //那么需要移动的数据是第五个数据后的数据。也就是10- 4 -1 =5 个数据
        int numMoved = size - index - 1;
        
        //需要移动的数据大于0
        if (numMoved > 0)
            //将elementData数组的第index+1位置开始的numMoved个数据复制到index(即数据往前移一位)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
                            
        //最后一个数据置为null，同时size-1; 
        elementData[--size] = null; // clear to let GC do its work
        //返回删除的数据
        return oldValue;
    }
```
#### get(E)
**get(E)**:返回列表中指定位置的元素。
```java
 public E get(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
        
        return (E) elementData[index];
    }
```
#### set(int,E)
**set(int,E)**:用指定的元素替换列表中指定位置的元素。
```java
public E set(int index, E element) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        E oldValue = (E) elementData[index];
        elementData[index] = element;
        return oldValue;
    }
```
#### addAll(Collection<? extends E>)/addAll(int , Collection<? extends E>)
**addAll(Collection<? extends E>)**:将指定集合中的所有元素追加到末尾
```java
 public boolean addAll(Collection<? extends E> c) {
        //将集合转成数组，并获取数组长度
        Object[] a = c.toArray();
        int numNew = a.length;
        
        //判断追加后元素是否会超出数组范围，超出则扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //将追加数组a从0开始numNew(所有)数据拷贝至elementData的size下标后
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        //返回追加数组数据是否有数据
        return numNew != 0;
    }
```
**addAll(int , Collection<? extends E>)**:将指定集合中的所有元素插入到此列表中，从指定位置开始。
```java
public boolean addAll(int index, Collection<? extends E> c) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
        //将集合转成数组，并获取数组长度
        Object[] a = c.toArray();
        int numNew = a.length;
        //判断插入后元素是否会超出数组范围，超出则扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount
        
        //插入数据后需要移动的数据量
        int numMoved = size - index;
        if (numMoved > 0)
            //将elementData数组的第index位置开始的numMoved个数据复制到index+numNew(即数据往后移numNew位)，腾出numNew个位置存放插入元素
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
        //将a数组的数据复制至elementData刚才腾出的位置
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```
#### clear()
**clear()**:从列表中删除所有元素。
```java
 public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```
**ArrayList**内部还有一个用于分页查看的**SubList**的内部类，在**subList(int,int)**内部会调用，作用是返回一个List集合的其中一部分视图。本质也是对**elementData**这个数组进行操作，这里就不展开。

*注：在无参初始化ArrayList时，Java7是直接初始化10大小的数组，而JAVA8是初始化空数组，在第一次扩容时才按照10进行扩容*
### Vector
**Vector**是矢量队列，与**ArrayList**不同，**Vector**中的操作是**线程安全**的。
**Vector**的父类接口，以及内部有几个主要的变量，如下：
```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
    
    protected Object[] elementData;//存储矢量组件的数组缓冲区。
    protected int elementCount;//实际元素个数
    protected int capacityIncrement;//capacityIncrement是每次Vector容量增加时的增量值，如果<=0，则每次扩容时容量翻倍，否则就扩容后容量就只是容量+增量。
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;//要分配的数组的最大大小，超出的话可能会导致OutOfMemoryError
```
#### 构造方法
**构造方法**:**Vector**构造方法分为三个：
```java
 public Vector() {
        //初始化容量为10
        this(10);
    }
    
 public Vector(int initialCapacity) {
        //初始化增量为0
        this(initialCapacity, 0);
    }
    
 public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }
    
 public Vector(Collection<? extends E> c) {
        elementData = c.toArray();
        elementCount = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
    }
```
前三个构造方法本质是调用一个方法：初始化一个特定容量的数组，初始化特定增量值，具体的初始化参数看源码；

带**Collection**参数的构造方法：按照传入的集合构造一个包含指定元素的列表，r将集合转成数组直接赋值给**elementData**，将元素个数赋值给**elementCount**，最后将**elementData**转换成Object[]类的数组；
#### add(E)/add(int,E)
**add(E)**:将指定的元素添加到Vector的末尾。**synchronized**方法，线程安全。
```java
  public synchronized boolean add(E e) {
        modCount++;
        //容量判断，容量不够则扩容，不然则不变
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
    
  private void ensureCapacityHelper(int minCapacity) {
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
  private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //如果设定的capacityIncrement增量大于0，则新容量为旧容量+增量，否则为双倍旧容量
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
        //判断新容量是否大于所需的最小容量，小于则新容量改为minCapacity
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //判断新容量是否超出
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            //判断当前容量是否超出可分配的最大容量，超出则设置新容量为
            //Integer.MAX_VALUE，保证数组容量不超过Integer.MAX_VALUE
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
  private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
**add(int,E)**：在Vector的指定位置插入指定元素。该方法内部调用的方法有**synchronized**，所以也是线程安全的。
```java
 public void add(int index, E element) {
        insertElementAt(element, index);
    }
 /**
  * 在指定的{@code index}处将指定的对象作为这个向量中的组件插入。
  *这个向量中的每个索引值大于或等于指定的{@code index}的组件向上移动，使其索引值比之前的值大1。
  */
 public synchronized void insertElementAt(E obj, int index) {
        modCount++;
        //判断插入位置是否超出范围
        if (index > elementCount) {
            throw new ArrayIndexOutOfBoundsException(index
                                                     + " > " + elementCount);
        }
        //容量判断，容量不够则扩容，不然则不变
        ensureCapacityHelper(elementCount + 1);
        //腾出数组中index下标位置
        System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
        //将obj插入这个位置
        elementData[index] = obj;
        elementCount++;
    }
```
#### remove(int)
**remove(int)**:移除Vector中指定位置的元素。线程安全
```java
 public synchronized E remove(int index) {
        modCount++;
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);
        E oldValue = elementData(index);
        
        //numMoved需要移动的数据数量
        int numMoved = elementCount - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--elementCount] = null; // Let gc do its work

        return oldValue;
    }

```
#### get(E)
**get(E)**:返回该Vector中指定位置的元素。线程安全
```java
 public synchronized E get(int index) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        return elementData(index);
    }
```
#### set(int,E)
**set(int,E)**:用指定的元素替换Vector中指定位置的元素。
```java
 public synchronized E set(int index, E element) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
#### addAll(Collection<? extends E>)/addAll(int , Collection<? extends E>)
**addAll(Collection<? extends E>)**：将指定集合中的所有元素追加到末尾，线程安全
```java
 public synchronized boolean addAll(Collection<? extends E> c) {
        modCount++;
        Object[] a = c.toArray();
        int numNew = a.length;
        //判断容量
        ensureCapacityHelper(elementCount + numNew);
        //拷贝数据
        System.arraycopy(a, 0, elementData, elementCount, numNew);
        elementCount += numNew;
        return numNew != 0;
    }
```
**addAll(int , Collection<? extends E>)**:将指定集合中的所有元素插入到Vector的指定位置。线程安全
```java
  public synchronized boolean addAll(int index, Collection<? extends E> c) {
        modCount++;
        if (index < 0 || index > elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        //判断容量
        ensureCapacityHelper(elementCount + numNew);
        //计算需要移动的元素数量
        int numMoved = elementCount - index;
        if (numMoved > 0)
            //移动数据，腾出numNew个位置
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
        //拷贝数据至腾出的位置
        System.arraycopy(a, 0, elementData, index, numNew);
        elementCount += numNew;
        return numNew != 0;
    }
```
#### clear();
**clear()**:从Vector中删除所有元素。线程安全
```java
 public void clear() {
        removeAllElements();
    }
    
 public synchronized void removeAllElements() {
        modCount++;
        // Let gc do its work
        for (int i = 0; i < elementCount; i++)
            elementData[i] = null;

        elementCount = 0;
    }
```
从代码可以得知，其实**Vector**就是线程安全的**ArrayList**，方法内的代码逻辑基本一致，就是多增加了**synchronized**关键字。同时因为**Vector**是同步的，当一个**Iterator**被创建而且正在被使用，另一个线程改变了**Vector**的状态（例如，添加或删除了一些元素），这时调用**Iterator**的方法时将抛出**ConcurrentModificationException**，因此必须捕获该异常。
### Stack
**Stack**继承自**Vector**，实现一个**后进先出**的堆栈。**Stack**提供5个额外的方法使得**Vector**得以被当作堆栈使用。基本的**push**和**pop**方法，还有**peek**方法得到栈顶的元素，**empty**方法测试堆栈是否为空，**search**方法检测一个元素在堆栈中的位置。**Stack**刚创建后是空栈。
#### 构造方法
```java
 //创建一个空堆栈。
 public Stack() {
    }
```
#### push(E)
**push(E)**:将E推到堆栈的顶部.
```java
 public E push(E item) {
        addElement(item);

        return item;
    }
 //Vector的方法
 public synchronized void addElement(E obj) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = obj;
    }
```
#### peek()
**peek()**:查看堆栈顶部的对象，而不将其从堆栈中移除。
```java
 public synchronized E peek() {
        int     len = size();

        if (len == 0)
            throw new EmptyStackException();
        return elementAt(len - 1);
    }
```
#### pop()
**pop()**:删除堆栈顶部的对象，并将该对象作为函数的值返回。
```java
     public synchronized E pop() {
        E       obj;
        int     len = size();

        obj = peek();
        removeElementAt(len - 1);

        return obj;
    }
    
    //Vector中的方法，移除指定位置的元素，并将之后的数据向前移动一位
    public synchronized void removeElementAt(int index) {
        modCount++;
        if (index >= elementCount) {
            throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                     elementCount);
        }
        else if (index < 0) {
            throw new ArrayIndexOutOfBoundsException(index);
        }
        int j = elementCount - index - 1;
        if (j > 0) {
            System.arraycopy(elementData, index + 1, elementData, index, j);
        }
        elementCount--;
        elementData[elementCount] = null; /* to let gc do its work */
    }
    
```
#### empty()
**empty()**:查看此堆栈是否为空。本质是看数据size是否为0
```java
 public boolean empty() {
        return size() == 0;
    }
```
#### search(Object)
**search(Object)**:返回对象在堆栈中的基于1(即栈顶)的位置。
```java
public synchronized int search(Object o) {
        int i = lastIndexOf(o);
        
        
        if (i >= 0) {
            return size() - i;
        }
        return -1;
    }
//Vector中的方法,返回Vector中指定元素最后一次出现的索引，如果vector中不包含该元素，则返回-1。
public synchronized int lastIndexOf(Object o) {
        return lastIndexOf(o, elementCount-1);
    }
//Vector中的方法,从index往回搜索值等于o的元素，返回Vector中指定元素最后一次出现的索引，如果没有找到该元素则返回-1。
public synchronized int lastIndexOf(Object o, int index) {
        if (index >= elementCount)
            throw new IndexOutOfBoundsException(index + " >= "+ elementCount);

        if (o == null) {
            for (int i = index; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = index; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

## Set
**Set**是一种不包含重复的元素的**Collection**，即任意的两个元素e1和e2都有e1.equals(e2)=false，**Set**最多有一个**null**元素。
### HashSet
**HashSet**这个类实现了**Set**集合，实际内部是使用**HashMap**的实例。**HashSet**中对重复元素的理解：和通常意义上的理解不太一样！
两个元素（对象）的**hashCode**返回值相同，并且**equals**返回值为**true**时（或者**地址**相同时），才称这两个元素是相同的。
**HashSet**的父类接口，以及内部有几个主要的变量，如下：
```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
    
    private transient HashMap<E,Object> map;
    //HashMap每个键值对应的value
    private static final Object PRESENT = new Object();
```
可以看到**HashSet**内部其实是由**HashMap**组成，**HashMap**是一种存储键值对的哈希表.
#### 构造方法
```java
 //构造一个新的空集合;支持HashMap实例具有默认初始容量(16)和负载系数(0.75)。
 public HashSet() {
        map = new HashMap<>();
    }
 //构造一个包含指定元素的的新集合;创建的HashMap实例具有默认初始容量(16和c.size()/.75f(后者约为c.size()的1.3倍)两者中的最大值)和负载系数(0.75)。
 public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
 //构造一个新的空集合;内部构造指定初始容量的HashMap实例
 public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
    
 //构造一个新的空集合;内部构造指定初始容量和指定负载系数的HashMap实例
 public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

 //构造一个新的、空的链接哈希集。(这个包的私有构造函数仅由LinkedHashSet使用。)后备HashMap实例是具有指定初始容量和指定负载系数的LinkedHashMap。 
 HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```
#### add(E)/add(int,E)
**add(E)**:如果E还不存在的话，将指定的元素添加到这个集合中。如果存在，则值不变直接返回false;
```java
  public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```
代码可以看出
#### remove(int):如果指定的元素存在，则从集合中移除该元素。返回集合中是否包含此元素。
```java
   public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
```
#### contains(Object)
**contains(Object)**:返回**HashSet**中是否包含此元素
```java
  public boolean contains(Object o) {
        return map.containsKey(o);
    }
```
#### clear()
**clear()**:从集合中删除所有元素。
```java
 public void clear() {
        map.clear();
    }
```
#### 总结
**HashSet**基本所有方法都是直接调用**HashMap**的方法，为了保证数据不重复，采用了存储时用**key**存储的方法，再**HashMap**中,**key**是不重复的，所以**HashSet**就不会有存储重复的数据了。
*HashSet是线程不同步的*
### TreeSet
**TreeSet**是一个有序的集合，它的作用是提供有序的Set集合。**TreeSet**的元素支持2种排序方式：自然排序或者根据提供的Comparator进行排序。
