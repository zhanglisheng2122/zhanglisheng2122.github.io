---
layout: post
title: ThreadLocal辩析   
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
>回顾我们的get方法，其实就是拿到每个线程独有的ThreadLocalMap
然后再用ThreadLocal的当前实例，拿到Map中的相应的Entry，然后就可以拿到相应的值返回出去。当然，如果Map为空，还会先进行map的创建，初始化等工作。

## 引发的内存泄漏分析
在讨论这个话题之前，我们先了解几个概念  

>**强引用**就是指在程序代码之中普遍存在的，类似“Object obj=new Object（）”这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象实例。    

>**软引用**是用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象实例列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK 1.2之后，提供了SoftReference类来实现软引用。  

>**弱引用**也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象实例只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象实例。在JDK 1.2之后，提供了WeakReference类来实现弱引用。  

>**虚引用**也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象实例是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象实例被收集器回收时收到一个系统通知。在JDK 1.2之后，提供了PhantomReference类来实现虚引用。 
    



根据我们前面对ThreadLocal的分析，我们可以知道每个Thread 维护一个 ThreadLocalMap，这个映射表的 key 是 ThreadLocal实例本身，value 是真正需要存储的 Object，也就是说 ThreadLocal 本身并不存储值，它只是作为一个 key 来让线程从 ThreadLocalMap 获取 value。仔细观察ThreadLocalMap，这个map是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。  

因此使用了ThreadLocal后，引用链如图所示  
![avatar](/images/blog/2019-07-19-threadlocal_02.png)

图中的虚线表示弱引用。  
这样，当把threadlocal变量设置为null以后，没有任何强引用指向threadlocal实例，所以threadlocal将会被gc回收，这样一来，ThreadLocalMap中就会出现key为null的Entry,就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value，而这块value永远不会被访问到了，所以存在着内存泄露。  

>只有当前thread结束以后，current thread就不会存在栈中，强引用断开，Current Thread、Map value将全部被GC回收。最好的做法是不在需要使用ThreadLocal变量后，都调用它的remove()方法，清除数据。

其实考察ThreadLocal的实现，我们可以看见，无论是get()、set()在某些时候，调用了expungeStaleEntry方法用来清除Entry中Key为null的Value，但是这是不及时的，也不是每次都会执行的，所以一些情况下还是会发生内存泄露。只有remove()方法中显式调用了expungeStaleEntry方法。  


#总结
JVM利用设置ThreadLocalMap的Key为弱引用，来避免内存泄露。  
JVM利用调用remove、get、set方法的时候，回收弱引用。  
当ThreadLocal存储很多Key为null的Entry的时候，而不再去调用remove、get、set方法，那么将导致内存泄漏。  
使用线程池+ ThreadLocal时要小心，因为这种情况下，线程是一直在不断的重复运行的，从而也就造成了value可能造成累积的情况。