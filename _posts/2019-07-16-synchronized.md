---
layout: post
title: synchronized内置锁   
categories: JAVA并发编程
description: synchronized内置锁   
keywords: Markdown
---

# synchronized内置锁
>Java支持多个线程同时访问一个对象或者对象的成员变量，关键字synchronized可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性，又称为内置锁机制。


## 对象锁和类锁
>Java中的每一个对象都可以做为锁。具体变现为以下3种形式  
>对于普通同步方法，锁是当前实例对象  
>对于静态同步方法，锁是当前类的Class对象  
>对于同步方法块，锁是Synchronized括号里配置的对象    

	
用在同步块上
```java
public void incCount(){
    synchronized (obj){
        count++;
    }
}

用在方法上
public synchronized void incCount2(){
        count++;
}

用在同步块上，但是锁的是当前类的对象实例
public void incCount3(){
    synchronized (this){
        count++;
    }
}

public static synchronized void synClass(){
    System.out.println(Thread.currentThread().getName()
            +"synClass going...");
    SleepTools.second(1);
    System.out.println(Thread.currentThread().getName()
            +"synClass end");
}
```

## Synchronized实现原理
当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常必须释放锁。
JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束和异常处，JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个monitor与之关联，当一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权。

简单来说
>JVM就是根据该标示符来实现方法的同步的：当方法被调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将>先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。


### Java对象头
>synchronized使用的锁是存放在Java对象头里面  

![avatar](/images/blog/2019-07-16-Synchronized_01.png)
>具体位置是对象头里面的MarkWord，MarkWord里默认数据是存储对象的HashCode等信息  

![avatar](/images/blog/2019-07-16-Synchronized_02.png)


>但是会随着对象的运行改变而发生变化，不同的锁状态对应着不同的记录存储方式

![avatar](/images/blog/2019-07-16-Synchronized_03.png)


## 锁升级与对比
>JavaSE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，锁一共有4种状态，级别从低到高依次是：无状态锁、偏向锁、轻量级锁、重量级锁，这几个锁状态会随着竞争情况逐渐升级，但不能降级，目的是为了提高获得锁和释放锁的效率。  

### 偏向锁
>它会偏向于第一个访问锁的线程，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要触发同步的，减少加锁／解锁的一些CAS操作（比如等待队列的一些CAS操作），这种情况下，就会给线程加一个偏向锁。 如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁。它通过消除资源无竞争情况下的同步原语，进一步提高了程序的运行性能。  

![avatar](/images/blog/2019-07-16-Synchronized_05.png)

### 轻量级锁  
>轻量级锁是由偏向锁升级来的，偏向锁运行在一个线程进入同步块的情况下，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁。  

#### 轻量级锁的加锁过程：  
>在代码进入同步块之前，JVM首先将在当前线程的栈帧中建立一个名为锁记录的空间，并将对象头的Mark Word复制到锁记录中，官方称之为 Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针，如果成功，当前线程获取到锁，如果失败表示其他线程竞争锁，当前线程通过自旋来获取锁。

#### 轻量级锁的解锁过程：
>轻量锁解锁时，会使用原子的CAS操作将Displaced Mark Word替换回对象头，如果成功表示没有发生竞争，如果失败，表示当前存在锁竞争，锁就会膨胀成重量级锁。  

![avatar](/images/blog/2019-07-16-Synchronized_04.png)

## 锁的优缺点对比

![avatar](/images/blog/2019-07-16-Synchronized_06.png)

## JDK对锁的更多优化措施
### 逃逸分析
>如果证明一个对象不会逃逸方法外或者线程外，则可针对此变量进行优化：
同步消除synchronization Elimination，如果一个对象不会逃逸出线程，则对此变量的同步措施可消除。
### 锁消除和粗化
>锁消除：虚拟机的运行时编译器在运行时如果检测到一些要求同步的代码上不可能发生共享数据竞争，则会去掉这些锁。  
锁粗化：将临近的代码块用同一个锁合并起来。
消除无意义的锁获取和释放，可以提高程序运行性能。


## 可能会问到的面试题

### 死锁与活锁的区别，死锁与饥饿的区别？  
**死锁：**是指两个或两个以上的进程（或线程）在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。 
产生死锁的必要条件： 
互斥条件：所谓互斥就是进程在某一时间内独占资源。
请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。 
不剥夺条件:进程已获得资源，在末使用完之前，不能强行剥夺。 
循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。  
**活锁：**任务或者执行者没有被阻塞，由于某些条件没有满足，导致一直重复尝试，失败，尝试，失败。
活锁和死锁的区别在于，处于活锁的实体是在不断的改变状态，所谓的“活”， 而处于死锁的实体表现为等待；活锁有可能自行解开，死锁则不能。  
**饥饿：**一个或者多个线程因为种种原因无法获得所需要的资源，导致一直无法执行的状态。  

### synchronized底层实现原理
synchronized (this)原理：涉及两条指令：monitorenter，monitorexit；再说同步方法，从同步方法反编译的结果来看，方法的同步并没有通过指令monitorenter和monitorexit来实现，相对于普通方法，其常量池中多了ACC_SYNCHRONIZED标示符。  

JVM就是根据该标示符来实现方法的同步的：当方法被调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。  
注意，这个问题可能会接着追问，java对象头信息，偏向锁，轻量锁，重量级锁及其他们相互间转化。