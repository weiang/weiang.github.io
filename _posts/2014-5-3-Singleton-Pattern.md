---
layout: post
title: Singleton Pattern 
description: "Ensure a class only has one instance and provide a global point of access to it."
modified: 2014-05-3
tags: [jekyll, tutorial, reading, design pattern]
comments: true
image:
  feature: texture-feature-05.jpg
  credit: Texture Lovers
  creditlink: http://texturelovers.com
---

> **Singleton Pattern** 
  Ensure a class only has one instance and provide a global point of access to it.

-----------

## 为什么需要Singleton模式
定义类的唯一目的就是可以用它作为模板来创建对象,而利用Singleton模式实现的类只创建和维护一个对象,这是不是有违定义类的初衷?其实不然,多数情况下类存在的目的是以其为模板创建该类对象的实例,而某些特定的引用场合下,如果一个类存在多个实例对象可能会对系统产生负面影响,甚至导致错误的发生.如在针对操作系统线程池的应用中,如果系统中存在多个线程池显然是错误的.当然有时系统中存在多个对象并不发生错误,但可能占用过多的系统资源导致资源利用率不高,这些都不是我们期望发生的,所以在面向对象的编程中引入了Singleton模式来解决这些问题.

--- 

## 单线程下的Singleton模式

在不考虑多线程的情况下,有一种简单直观的实现方式,如下:

    public class Singleton {
        private static Singleton uniqueInstance;
        
        private Singleton() {}
        
        public static Singleton getInstance() {
            if (uniqueInstance == null) {
                uniqueInstance = new Singleton();
            }
            return uniqueInstance;
        }
    }
    
这里有几点需要说明

1. Singleton类使用一个静态私有的变量(uniqueInstance)来引用这个唯一的实例对象.
2. 将Singleton类的构造函数声明为private来阻止其他类来创建该类的对象.
3. 通过类方法getInstance()来引用这个唯一的对象.当第一次调用这个方法时,由于该类对象还没有创建,需先创建.

---

## 多线程下的Singleton模式
  上节中的实现并不是线程安全的.所谓线程安全是指多线程模型下程序也能正确运行.那什么是多线程呢?即一个进程中存在多个共享数据的执行流同时在执行.在这种程序执行模型下,由于线程之间共享数据,这时就不得不考虑他们直接对数据的访问顺序对程序逻辑正确性的影响.带着这样的考量,我们来重新审视之前关于Singleton模式的实现.
  考虑这样一种情景,一个进程中两个线程同时调用getInstance()方法来引用唯一的实例对象,而在这之前实例对象还未被创建.存不存在一种线程执行序列创建多于一个的实例对象?不妨令两个线程A和B,在A线程执行完if条件句发现还没有实例对象被创建时,它将执行之后的创建语句.而在执行创建语句之前B线程也执行if条件句,由于这是实例对象还没被创建,所以B线程也会执行之后的创建语句.这样就创建两个实例对象了,这显然不是我们预料之中的.那么怎样解决这个问题呢?下面是三种常见的做法,需要根据具体的应用情景来选择合适的方案.
 
### 1. 将getInstance()声明为*synchronize*方法
通过将getInstance()声明为*synchronize*方法可以保证不同线程在调用getInstance()方法时只有一个可以执行，其他的线程等待执行该方法的线程完成,这时显然已经创建新的实例对象了,之后线程执行if语句时不可能为真,即不可能执行之后的创建语句.

这种实现的优点是:简单,通用.但缺点也非常明显:同步的代价是巨大的,它会大大降低了程序的运行效率(某些情况下可使程序的运行效率降低100倍).因为这种方案没有考虑只须在实例对象未创建的时候调用getInstance()需要这种同步操作,一旦实例变量创建之后,getInstance()方法的唯一作用就是返回该实例对象的引用,这时这种同步操作就显得多余而且效率低下了.所以只有当不太关心getInstance()方法的性能对程序运行效率的影响时才采用这种方法.

### 2. 尽早的创建Singleton中唯一的实例对象,而不是把这任务交给getInstance()方法来完成
这种做法是在Singleton类被载入时就由**JVM**来完成对唯一对象的实例化.具体实现如下:

    public class Singleton {
        private static Singleton uniqueInstance = new Singleton();
        
        private Singleton() {}
        
        public static Singleton getInstance() {
            return uniqueInstance;
        }
    }

这样做的好处也是显而易见的:将创建对象的任务交给**JVM**在载入对象时完成,有效的避免了多个线程竞争调用getInstance()方法创建对象现象的发生(这里可能引入其他问题,后面有讨论).但过早的创建对象也是有代价的,特别是当Singleton对象占用系统稀缺资源如线程池,cache等时.过早的创建这些对象占用系统资源可能会造成资源使用率不高的现象.

### 3. 采用双锁来减少线程在调用getInstance()方法时的等待
这种方案从某种程度上来说时针对方案一中线程调用getInstance()方法时等待过长的改进.具体实现如下:

    public class Singleton {
        private volatile static Singleton uniqueInstance;
        
        private Singleton() {}
        
        public static Singleton getInstance() {
            if (uniqueInstance == null) {
                synchronize (Singleton.class);
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
            return uniqueInstance;
        }
    }
  
  
相较方案一,getInstance()方法已不在是*synchronize*方法了,多个线程可以同时调用并执行它,只有当Singleton对象还未创建时(第一个if语句为真)其他线程才需要等待.并且一旦对象创建之后再也不会执行同步语句(第一个if语句恒为假).有效的消除了对象创建后调用getInstance()时额外的线程等待开销对程序性能的影响.但是相较方案一而言程序的复杂度稍有提高.
  
以上三种方案实现的Singleton类都是线程安全的,并且能够按照我们的预期行为.所以三种方案都可应用于具体应用中,具体采用那种当根据特定应用的特点来选择合适的方案.如为了保证应用的简洁性而效率又不是很重要时可采用方案一.当Singleton类所占资源不是系统的稀缺资源时可以采用方案二.当应用程序很关心性能和资源的使用率时,可采用较复杂的方案三.

另外,这里还有一点需要交代.上述方案二的正确性是建立在只有一个class loader的前提下的.不排除存在两个class loader同时载入Singleton类,这时系统中就同时存在两个Singleton对象了,这并不是我们想要的.  

---

## 总结
1. Singleton模式保证系统中某个类最多存在一个对象.
2. Java对Singleton模式的实现使用了private构造器,static方法和static变量.
3. 多线程应用下实现Singleton模式时要选择合适的方案来满足性能和资源的限制.
4. 当你的应用中使用了过多的Singleton模式时,你应该停下来考虑你的设计是否合理.
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
