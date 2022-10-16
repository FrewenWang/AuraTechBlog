---
title: 创建型模式-单例模式（Singleton Pattern）
date: 2019-06-13 17:00:35
tags: [设计模式]
---

文章参考：[https://blog.csdn.net/zhengzhb/article/details/7331354](：https://blog.csdn.net/zhengzhb/article/details/7331354)

### 设计模式简介
对于系统中的某些类来说，只有一个实例很重要，例如，一个系统中可以存在多个打印任务，但是只能有一个打印机，只能有一个正在工作的任务。一个系统只能有一个窗口管理器或文件系统；一个系统只能有一个计时工具或ID（序号）生成器。

如何保证一个类只有一个实例并且这个实例易于被访问呢？定义一个全局变量可以确保对象随时都可以被访问，但不能防止我们实例化多个对象。一个更好的解决办法是让类自身负责保存它的唯一实例。这个类可以保证没有其他实例被创建，并且它可以提供一个访问该实例的方法。这就是单例模式的必要性。

###  设计模式定义
单例模式(Singleton Pattern)确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例，这个类称为单例类，它提供全局访问的方法。

单例模式是一种创建型模式，指某个类采用Singleton模式，则在这个类被创建后，只可能产生一个实例供外部访问，并且提供一个全局的访问点。单例模式的要点有三个：一是某个类只能有一个实例；二是它必须自行创建这个实例；三是它必须自行向整个系统提供这个实例。单例模式是一种对象创建型模式。单例模式又名单件模式或单态模式。

 核心知识点如下：
 
- (1) 将采用单例设计模式的类的构造方法私有化（采用private修饰）。
- (2) 在其内部产生该类的实例化对象，并将其封装成private static类型。
- (3) 定义一个静态方法返回该类的实例。

### UML图设计

![image](https://raw.githubusercontent.com/FrewenWong/PicUploader/master/20190804172909-singletonuml.png)

<!--![image](http://note.youdao.com/yws/res/37552/BF1447069BDA4353B90B55D2316660DE)-->


### 单例模式的优缺点

优点：
　
- 1、提供了对唯一实例，并且自行实例化并向整个系统提供这个实例；
- 2、Java中频繁创建和销毁类对象都会占用一部分系统资源，使用单例模式可以提高性能；
- 3、允许可变数量的实例；
 
缺点：
- 
- 1、单例模式中，没有抽象层，所以对于单例类的扩展并不方便；
- 2、单例类的职责过重，在一定程度上违背了“单一职责原则”；
- 3、滥用单例将带来一些负面问题，如为了节省资源将数据库连接池对象设计为的单例类，可能会导致共享连接池对象的程序过多而出现连接池溢出；如果实例化的对象长时间不被利用，系统会认为是垃圾而被回收，这将导致对象状态的丢失；

### 单例模式的类型

#### 1.饿汉式单例.

这种方式基于classloader机制避免了多线程的同步问题，单例对象在类装载时就实例化。所以饿汉式是用空间来换时间，解决了线程同步的问题。

优点是：写起来比较简单，而且不存在多线程同步问题，避免了synchronized所造成的性能问题；

缺点是：当类SingletonTest被加载的时候，会初始化static的instance，静态变量被创建并分配内存空间，从这以后，这个static的instance对象便一直占着这段内存（即便你还没有用到这个实例），当类被卸载时，静态变量被摧毁，并释放所占有的内存，因此在某些特定条件下会耗费内存。

```
private static Singleton INSTANCE = new Singleton();
　private Singleton() {}
　public static Singleton getInstance() {
     return INSTANCE;   
  }
```
#### 2.懒汉式单例。

懒汉式主要是用时间来换空间。主要当单例在使用的时候才会检查是否初始化对象。在多线程情况下，同时调用入口方法获取实例时可能无法正常工作，所以懒汉式单例是非线程安全的。

优点是：写起来比较简单，当类Singleton被加载的时候，静态变量static的instance未被创建并分配内存空间，当getInstance方法第一次被调用时，初始化instance变量，并分配内存，因此在某些特定条件下会节约了内存；

缺点是：并发环境下很可能出现多个Singleton实例。

```
 //懒汉单例，线程不安全，当启动多线程，同时获取多个实例时会报错
   private static Singleton INSTANCE = null;
   private Singleton() {}
   public static Singleton getInstance() {
        if(INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
   }  
```
当然，懒汉式也可以解决非线程安全的问题。那就是给getInstance()方法加上synchronized锁。

优点是：使用synchronized关键字避免多线程访问时，出现多个Singleton实例。

缺点是：同步方法频繁调用时，效率略低。

```
private static Singleton INSTANCE = null;
   private Singleton() {}
   public static synchronized Singleton getInstance() {
        if(INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
   }  
```

#### 3.Double Check Lock(DCL 双重锁定)实现单例
我们把上面的懒汉式进行优化修改一下。

```
//懒汉单例，线程不安全，当启动多线程，同时获取多个实例时会报错
   private static Singleton INSTANCE = null;
   private Singleton() {}
   public static Singleton getInstance() {
        if(INSTANCE == null) {
            synchronized(Singleton.class){
                if(null ==INSTANCE){
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
   }  
```
 我们来着重分析一下getInstance方法.我们可以看到getInstance方法中对instance进行了两次判空。第一层判空主要是为了避免不必要的多次同步。第二层判空则是为了在null的情况才创造实例对象。这是什么意思呢？是不是有点摸不着头脑，我们就来着重分析一下。
 
 假设线程A执行到mInstance = new Singleton()语句，在这里看起来是一句代码，其实他并不是一个原子操作。这句代码最终会被编译成多条汇编指令，他大致做了三件事情：
 
- （一）给Singleton的实例分配内存
- （二）调用Singleton的构造方法，初始化成员字段
- （三）将mInstance的对象指向分配的内存空间（此时mInstance就不是null了）。

但是，由于Java编译器允许处理器乱序执行。以及JDK1.5之前JMM（Java内存模型）中cache、寄存器到主存回写顺序的规定。上面的第二步和第三步的顺序是不固定的。所以执行属性有可能是1-2-3或者1-3-2。如果是后者，3执行完，但是2未执行。被切换到线程b上。这时候mInstance因为已经在线程b上执行了第三步，所以mInstance已经不为空了。所以线程b执行取走了mInstance对象（但是，此时Singleton的对象并没有初始化）所以这个时候使用就会出错了。这就是DCL失效问题。

在JDK1.5以后。SUN官方已经注意到这个问题了。调整了JVM。集体化了Volatile关键字。如果是JDK1.5之后的版本。则只需要向下面来声明单例对象：

```
private volatile static Singleton INSTANCE = null;
```
这样就可以保证每次取用INSTANCE对象的时候，都是从主内存中取对象。这样使用DCL就可以保证对象单例。但是，当时使用volatile或多或少也会影响到性能。但是考虑到程序的正确性。牺牲掉这点性能是值得的。

DCL的方法优点：资源利用率高，第一次执行getInstance()时候单例对象才会被实例化，效率高。

缺点是第一次加载的时候反应有点慢

#### 4.静态内部类实现单例
静态内部类实现单例跟饿汉模式有点类似，只是在类装载的时候，SingletonHolder并没有被主动使用，只有显式调用getInstance方法是，才会调用装载SingletoHolder类，从而实例化instance，既实现了线程安全，又避免了同步带来的性能影响。


```
public class OptimusNetWork {

    private final OptimusNetClient mHttpClient;

    private OptimusNetWork() {
        mHttpClient = new OptimusNetClient();
    }

    // 我们定义一个静态内部类，来作为单例对象的持有者
    private static class OptimusNetWorkHolder {
        private static final OptimusNetWork INSTANCE = new OptimusNetWork();
    }
    // 所有只有调用getInstance的时候，才会调用OptimusNetWorkHolder。从而实例化OptimusNetWork
    public final static OptimusNetWork getInstance() {
        return OptimusNetWorkHolder.INSTANCE;
    }
}
```

#### 5.枚举实现的单例

有没有个更简单的实现单例的方式。

目前最为安全的实现单例的方法是通过内部静态enum的方法来实现，因为JVM会保证enum不能被反射并且构造器方法只执行一次。


```
public class EnumSingleton(){
    //EnumSingleton 私有构造函数 
	private EnumSingleton(){}
	
    public static EnumSingleton getInstance(){
    	SingletonEnum.INSTANCE.getInstance();
    }
    private static enum SingletonEnum() {
          INSTANCE;
          public void doSomething(){
              System.out.println("do sth");
          }
     }
}    
```

####  6.使用容器实现单例

这个就比较不常用了。所以我们暂时就先不介绍他了。

### 问题思考：单例模式与垃圾回收

我们先来思考一个问题：当一个单例的对象长久不用时，会不会被jvm的垃圾收集机制回收??

 首先说一下为什么会产生这一疑问，笔者本人再此之前从来没有考虑过垃圾回收对单例模式的影响，直到去年读了一本书，《设计模式之禅》秦小波著。在书中提到在j2ee应用中，jvm垃圾回收机制会把长久不用的单例类对象当作垃圾，并在cpu空闲的时候对其进行回收。之前读过的几本设计模式的书，包括《java与模式》，书中都没有提到jvm垃圾回收机制对单例的影响。并且在工作过程中，也没有过单例对象被回收的经历，加上工作中很多前辈曾经告诫过笔者：尽量不要声明太多的静态属性，因为这些静态属性被加载后不会被释放。因此对jvm垃圾收集会回收单例对象这一说法持怀疑态度。渐渐地，发现在同事中和网上的技术人员中，对这一问题也基本上是鲜明的对立两派。那么到底jvm会不会回收长久不用的单例对象呢。
 
 所以到底会不会回收。我们特地写了一段代码来进行验证。
 
```
class Singleton {
	private byte[] a = new byte[6*1024*1024];
	private static Singleton singleton = new Singleton();
	private Singleton(){}
	
	public static Singleton getInstance(){
		return singleton;
	}
}
 
class Obj {
	private byte[] a = new byte[3*1024*1024];
}
 
public class Client{
	public static void main(String[] args) throws Exception{
		Singleton.getInstance();
		while(true){
			new Obj();
		}
	}
}
```

本段程序的目的是模拟j2ee容器，首先实例化单例类，这个单例类占6M内存，然后程序进入死循环，不断的创建对象，逼迫jvm进行垃圾回收，然后观察垃圾收集信息，如果进行垃圾收集后，内存仍然大于6M，则说明垃圾回收不会回收单例对象。运行本程序使用的虚拟机是hotspot虚拟机，也就是我们使用的最多的java官方提供的虚拟机，俗称jdk，版本是jdk1.6.0_12 运行时vm arguments参数为：-verbose:gc -Xms20M -Xmx20M，意思是每次jvm进行垃圾回收时显示内存信息，jvm的内存设为固定20M。

```
运行结果：

……

[Full GC 18566K->6278K(20352K), 0.0101066 secs]

[GC 18567K->18566K(20352K), 0.0001978 secs]

[Full GC 18566K->6278K(20352K), 0.0088229 secs]

……    
```
 从运行结果中可以看到总有6M空间没有被收集。因此，笔者认为，至少在hotspot虚拟机中，垃圾回收是不会回收单例对象的。

后来查阅了一些相关的资料，hotspot虚拟机的垃圾收集算法使用根搜索算法。这个算法的基本思路是：对任何“活”的对象，一定能最终追溯到其存活在堆栈或静态存储区之中的引用。通过一系列名为根（GC Roots）的引用作为起点，从这些根开始搜索，经过一系列的路径，如果可以到达java堆中的对象，那么这个对象就是“活”的，是不可回收的。可以作为根的对象有：

- 1、虚拟机栈（栈桢中的本地变量表）中的引用的对象。
- 2、方法区中的类静态属性引用的对象。
- 3、方法区中的常量引用的对象。
- 4、本地方法栈中JNI的引用的对象。

方法区是jvm的一块内存区域，用来存放类相关的信息。很明显，java中单例模式创建的对象被自己类中的静态属性所引用，符合第二条，因此，单例对象不会被jvm垃圾收集。

虽然jvm堆中的单例对象不会被垃圾收集，但是单例类本身如果长时间不用会不会被收集呢？因为jvm对方法区也是有垃圾收集机制的。如果单例类被收集，那么堆中的对象就会失去到根的路径，必然会被垃圾收集掉。jvm卸载类的判定条件如下：

- 该类所有的实例都已经被回收，也就是java堆中不存在该类的任何实例。
- 加载该类的ClassLoader已经被回收。
- 该类对应的java.lang.Class对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法。

只有三个条件都满足，jvm才会在垃圾收集的时候卸载类。显然，单例的类不满足条件一，因此单例类也不会被卸载。也就是说，只要单例类中的静态引用指向jvm堆中的单例对象，那么单例类和单例对象都不会被垃圾收集，依据根搜索算法，对象是否会被垃圾收集与未被使用时间长短无关，仅仅在于这个对象是不是“活”的。假如一个对象长久未使用而被回收，那么收集算法应该是最近最长未使用算法，最近最长未使用算法一般用在操作系统的内外存交换中，如果用在虚拟机垃圾回收中，岂不是太不安全了？以上是笔者的观点。

因此笔者的观点是：**在hotspot虚拟机1.6版本中，除非人为地断开单例中静态引用到单例对象的联接，否则jvm垃圾收集器是不会回收单例对象的。**