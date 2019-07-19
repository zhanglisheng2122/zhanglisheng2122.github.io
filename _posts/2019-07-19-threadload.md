---
layout: post
title: Java并发编程笔记      
categories: JAVA并发编程
description: ThreadLocal辩析
keywords: Markdown
---


# ThreadLocal的使用

>ThreadLocal,即线程变量，是一个以ThradLocal对象为键，任意对象为值的存储结构。这个结构被附带在线程上，也就是说一个线程可以根据一个ThreadLocal对象查询到绑定在这个线程上的一个值。  


ThreadLocal类接口很简单，只有4个方法，我们先来了解一下：

```java
void set(Object value) 
```
设置当前线程的线程局部变量的值。


```java
public Object get()
```
该方法返回当前线程所对应的线程局部变量。



```java
public void remove()
```
将当前线程局部变量的值删除，目的是为了减少内存的占用。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。


```java
protected Object initialValue()
```
返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次。ThreadLocal中的缺省实现直接返回一个null。


## 实现解析
![avatar](/images/blog/2019-07-19-threadlocal_01.png)

>ThreadLocal.java 


```java
public T get() {
    Thread t = Thread.currentThread(); 
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

 
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

>Thread.java  
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```


>上面先取到当前线程，然后调用getMap方法获取对应的ThreadLocalMap,ThreadLocalMap是ThreadLocal的静态内部类，然后Thread类中有一个这样类型的成员，所以getMap是直接返回Thread的成员。  
>看下ThreadLocal的内部类ThreadLocalMap源码  

```java
static class ThreadLocalMap {
      
    static class Entry extends WeakReference<ThreadLocal<?>> {
       
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

  
    private static final int INITIAL_CAPACITY = 16;

   
    private Entry[] table;
	...
```

>可以看到有个Entry内部静态类，它继承了WeakReference,记录了两个信息，一个是ThreadLocal的类型，一个是Object的值。
