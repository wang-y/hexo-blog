---
title: Java 面试坑点记录(1)
date: 2017-07-24 20:36:59
categories: 面试
tags: 
    - java
    - 面试
---

# 关于HashMap

### 特性
1. HashMap可以接受null键值和值，而Hashtable则不能;
2. HashMap是非synchronized;
3. HashMap很快;
4. HashMap储存的是键值对

### 坑点

#### HashMap的工作原理吗？
HashMap是基于hashing的原理，我们使用put(key, value)存储对象到HashMap中，使用get(key)从HashMap中获取对象。
当我们给put()方法传递键和值时，我们先对键调用hashCode()方法，返回的hashCode用于找到bucket位置来储存Entry对象。

#### 当两个对象的hashcode相同会发生什么？
因为hashcode相同，所以它们的bucket位置相同，‘碰撞’会发生。
因为HashMap使用链表存储对象，这个Entry(包含有键值对的Map.Entry对象)会存储在链表中。

#### 如果两个键的hashcode相同，你如何获取值对象？
当我们调用get()方法，HashMap会使用键对象的hashcode找到bucket位置，调用keys.equals()方法去找到链表中正确的节点，最终找到要找的值对象。

#### 如果HashMap的大小超过了负载因子(load factor)定义的容量，怎么办？
默认的负载因子大小为0.75，当一个map填满了75%的bucket时候，和其它集合类(如ArrayList等)一样，将会创建原来HashMap大小的两倍的bucket数组，来重新调整map的大小，并将原来的对象放入新的bucket数组中。这个过程叫作rehashing，因为它调用hash方法找到新的bucket位置。


#### 你了解重新调整HashMap大小存在什么问题吗？
当重新调整HashMap大小的时候，存在条件竞争:因为如果两个线程都发现HashMap需要重新调整大小了，它们会同时试着调整大小。在调整大小的过程中，存储在链表中的元素的次序会反过来，因为移动到新的bucket位置的时候，HashMap并不会将元素放在链表的尾部，而是放在头部，这是为了避免尾部遍历(tail traversing)。

PS:如果条件竞争发生了，那么就死循环了。这个时候，你可以质问面试官，为什么这么奇怪，要在多线程的环境下使用HashMap呢？：）

### 减少碰撞的发生，提高效率的方法
使用不可变的、声明作final的对象，并且采用合适的equals()和hashCode()方法;
使用String，Interger这样的wrapper类作为键是非常好的选择。

### 为什么String, Interger这样的wrapper类适合作为键？ 
String, Interger这样的wrapper类作为HashMap的键是再适合不过了，而且String最为常用。因为String是不可变的，也是final的，而且已经重写了equals()和hashCode()方法了。其他的wrapper类也有这个特点。不可变性是必要的，因为为了要计算hashCode()，就要防止键值改变，如果键值在放入时和获取时返回不同的hashcode的话，那么就不能从HashMap中找到你想要的对象。不可变性还有其他的优点如线程安全。如果你可以仅仅通过将某个field声明成final就能保证hashCode是不变的，那么请这么做吧。因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的。如果两个不相等的对象返回不同的hashcode的话，那么碰撞的几率就会小些，这样就能提高HashMap的性能。

### 我们可以使用自定义的对象作为键吗？
 这是前一个问题的延伸。当然你可能使用任何对象作为键，只要它遵守了equals()和hashCode()方法的定义规则，并且当对象插入到Map中之后将不会再改变了。如果这个自定义对象时不可变的，那么它已经满足了作为键的条件，因为当它创建之后就已经不能改变了。

## 总结

### HashMap的工作原理

HashMap基于hashing原理，我们通过put()和get()方法储存和获取对象。当我们将键值对传递给put()方法时，它调用键对象的hashCode()方法来计算hashcode，让后找到bucket位置来储存值对象。当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。HashMap使用链表来解决碰撞问题，当发生碰撞了，对象将会储存在链表的下一个节点中。 HashMap在每个链表节点中储存键值对对象。

当两个不同的键对象的hashcode相同时会发生什么？ 它们会储存在同一个bucket位置的链表中。键对象的equals()方法用来找到键值对

# ConcurrentHashMap
JDK 1.5 以后加入。

## JDK1.8以前的原理 

就是把Map分成了N个Segment，put和get的时候，都是现根据key.hashCode()算出放到哪个Segment中

