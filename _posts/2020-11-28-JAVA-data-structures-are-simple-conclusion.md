---
layout: post
title:  "JAVA数据结构简单概括"
author: "陈宇瀚"
date:   2020-11-29 19:30:00 +0800
header-img: "img/img-head/img-head-java.jpg"
categories: article
tags:
  - JAVA 
  - 数据结构
---
# JAVA数据结构
### Map
**Map**接口提供key到value的映射。一个Map中不能包含相同的key，每个key只能映射一个value。
#### HashMap
**HashMap**继承**Map**接口，由数组+单链表或者数组+红黑树实现一个key-value映射的哈希表。是非同步的，同时允许null value和null key。 

#### HashTable
**Hashtable**与**HashMap**哈希冲突少的时候一样都是数组+单链表，任何非空（non-null）的对象都可作为key或者value，其方法都带有**synchronized**关键字，是线程同步的。
#### HashMap与HashTable的不同
- **HashTable**线程安全， **HashMap**线程不安全；
- **HashMap**删除了**contains**方法；
- **HashMap**key-value允许为null;**HashTable**不允许。
#### TreeMap
**TreeMap**是一个非同步**有序**的key-value集合，它通过红黑树实现,可以返回**有序的key集合**。
## Collection
**Collection**是最基本的集合接口，一个**Collection**代表一组**Object**，**Collection**支持一个**iterator()**的方法，该方法返回一个迭代子，使用该迭代子即可逐一访问**Collection**中每一个元素。
### List
**List**是有序的**Collection**，用户能够使用索引来访问**List**中的元素，允许有相同的元素。
#### LinkedList
**双向链表**结构，适用于乱序插入、删除。获取数据时首先判断是在前半部分还是后半部分，之后分别通过队首/队尾来向后、向前遍历获取对应的数据。
#### ArrayList
**ArrayList**是动态数组，底层就是一个数组, 因此按序查找快, 乱序插入, 删除因为涉及到后面元素移位所以性能慢。
#### Vector
**Vector**是矢量队列，与**ArrayList**不同，**Vector**中的操作是线程安全的。
#### Stack
**Stack**继承自**Vector**，实现一个后进先出的堆栈。**Stack**提供5个额外的方法使得**Vector**得以被当作堆栈使用。基本的**push**和**pop**方法，还有**peek**方法得到栈顶的元素，**empty**方法测试堆栈是否为空，**search**方法检测一个元素在堆栈中的位置。**Stack**刚创建后是空栈。
### Set
**Set**是一种不包含重复的元素的**Collection**，即任意的两个元素e1和e2都有e1.equals(e2)=false，Set最多有一个null元素。
#### HashSet
**HashSet**这个类实现了**Set**集合，实际内部是使用**HashMap**的实例。内部使用**HashMap**存储时用key的位置来存储，value的地方存储一个没有内容的**Object**实例，所以可以保证没有重复的元素。*HashSet*是线程不同步的;
#### TreeSet
**TreeSet**是一个有序的集合，它的作用是提供有序的Set集合。**TreeSet**的元素支持2种排序方式：自然排序或者根据提供的**Comparator**进行排序。
