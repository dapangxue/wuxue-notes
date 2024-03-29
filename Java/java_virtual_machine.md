# JVM虚拟机

[TOC]

## 一、虚拟机概况

<img src="./img/JavaVirtualMachine.PNG">

1. 程序计数器（Program Counter Register）
2. 虚拟机栈（Virtual Machine Stack）
3. 本地方法栈 (Native Method Stack)
4. 堆（Heap）
5. 方法区（Method Area）

#### 1.程序计数器

线程私有，当前程序执行字节码的行号指示器，不会抛出错误。线程切换后可以恢复到原来的位置。

#### 2.虚拟机栈

线程私有，方法的调用和调用完成涉及到栈帧的入栈到出栈的过程。栈帧存储了`局部变量表`、`操作栈`、`动态链接`、`方法出口`等信息。局部变量表所占的内存类编译期可知，内存在编译期间完成分配。会抛出`StackOverFlowError`和`OutOfMemoryError`异常。

可以通过-Xss来指定每个虚拟机栈所占用的内存，在JDK1.4中默认为256K，JDK1.5+默认为1M。

> java -Xss2M 当前运行的类

该区域可能会抛出如下异常：

+ 线程请求的栈深度超过最大值，会抛出StackOverFlowError异常。
+ 栈进行动态扩展时如果无法申请到足够内存，会抛出OutOfMemoryError异常。

#### 3.本地方法栈

线程私有，用于执行本地方法，即Java内部的native方法（C、C++、汇编语言），会抛出`StackOverFlowError`和`OutOfMemoryError`异常。

#### 4.堆

线程公有，所有线程共享的一块区域。用于存放对象实例（包括数组），会抛出OutOfMemoryError异常。

所有的对象都在这里分配内存，是Garbage Collection的重要区域。

现在的垃圾收集器基本都是采用分代收集算法，针对处于不同年龄的区域执行不同的垃圾回收算法。将堆分为了`新生代`和`老年代`。

Java对象的分代年龄存储在对象头的Mark Word中。Java堆可以处于物理内存上不连续的内存空间中，只要逻辑上连续即可。

可以通过-Xms和-Xmx来设定堆内存的初始值和最大值。

#### 5.方法区

线程公有，存储已经被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码，会抛出OutOfMemoryError异常。

较早的永久代和现在的元空间都是方法区的实现方式。JDK1.7中原本属于永久代的`字符串常量池`和`静态变量`的存储移动到了堆中，JDK1.8中永久代已经完全删除，而是采用了Meta Space，运行时常量池和静态变量都存储到了堆中，Meta Space存储类的元数据，元空间占用的是本地内存。

类的元数据包含了类的层级信息，方法数据和方法信息（如字节码，栈和变量大小），运行时常量池，已确定的符号引用和虚方法表。

HotSpot虚拟机JDK1.7以前采用永久代实现方法区不是一个好的主意，默认的永久代存储空间很容易引起OOM异常，32位机器默认的永久代大小为64M，64位机器则为85M。InfoQ上一篇文章指出(参考文献1)，永久代的垃圾回收和老年代的垃圾回收是绑定的，一旦其中一个区域满了，就会触发Full GC。

将永久代移除后，类元数据信息保存在Meta Space中，元空间不占用JVM的内存，它的最大分配空间就是系统可用的内存空间，元空间的最大容量可以设置，如果不设置的话，JVM会根据类的元数据大小动态增加元空间的容量。

## 二、对象的访问定位

对象的访问定位有两种方法：

+ 使用句柄
+ 直接指针

使用句柄相当于一次二转手，也就是要先访问句柄，通过句柄访问到对象的实例数据和对象的类型数据，比如静态变量。

使用直接指针，可以减少一次定位的时间开销，先访问堆中的对象实例数据，再访问方法区中的对象类型数据。HotSpot虚拟机使用的是这种方法，直接指针的访问的一个优点就是速度更快，节省了一次指针定位的开销。

## 三、垃圾回收

### 判断对象已死

判断对象是否已经死亡有两种方法，引用计数法和GC Roots的可达性分析。

引用计数法的缺点就是无法解决相互循环引用的问题。出色的内存数据库Redis就是采用了引用计数法来判定对象是否已经死亡的。

Java采用了基于GC Roots的可达性分析，通过分析多个GC Roots的可达到的节点，判断出垃圾对象。

可以作为GC Roots的对象主要有：

+ 虚拟机栈中引用的对象
+ 方法区中静态变量引用的对象
+ 方法区中常量引用的对象
+ 本地方法栈中引用的对象

### 何时被回收

常见的引用有强引用、软引用、弱引用和虚引用。

+ 强引用在代码中是普遍存在的，常见的Object object = new Object()就是强引用，只要强引用存在，那么垃圾收集器永远不会回收。
+ 软引用用于描述那些还有用，但是非必须的对象，需要继承SoftReference类来实现软引用，在系统发生内存溢出的异常之前，虚拟机会对这些对象进行回收。
+ 弱引用引用的对象只会存活到下次垃圾回收之前，当垃圾收集器工作时，无论当前内存是否足够，这一类对象都会被回收。
+ 虚引用（幽灵引用/幻影引用），最弱的一种引用关系，虚引用的对象完全不会影响对象的回收时机，只是在回收时得到一个通知而已。

### 垃圾收集算法

常见的垃圾收集算法有：复制算法、标记-整理、标记-清除和分代收集。新生代的对象存活时间不长，主要用复制算法。老年代主要采用标记-清除和标记-整理两种算法。

标记-清除算法：首先对垃圾对象进行标记，标记完成后统一回收所有被标记的对象。标记-清除方法的缺点就是会产生不连续的内存碎片，以后如果需要分配大对象，很难找到连续的内存空间，可能会提前触发下一次的垃圾回收。

标记-整理算法：和标记-清除算法一样，首先对所有的垃圾对象进行标记，然后让存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

复制算法：将新生代划分为一个eden区和两个survivor区，回收时将一个eden区和一个survivor区的对象移动到另一个survivor区域内，如果这个survivor区域的内存不够使用，采用分配担保机制，让一部分对象提前进入老年代。

分代收集算法其实依赖了上述的新生代和老年代算法，在对象头的Mark Word中保存了当前对象的年龄，当年龄达到一定程度，就可以晋升为老年代。

### 垃圾收集器

垃圾收集器一般都需要划分为针对新生代的收集器和针对老年代的收集器。当然还存在一些跨新生代和老年代的收集器。

一般的垃圾收集器都有Stop the World的特性，也就是暂停工作线程，频繁的这样停顿是不好的。

新生代的垃圾收集器有Serial、ParNew、Parallel Scavenge。它们都是Stop the World的收集器，不一样的是Serial是单线程的新生代垃圾收集器，ParNew是多线程的新生代垃圾收集器，Parallel Scavenge是保证了吞吐量的收集器。

对应的老年代的收集器Serial Old、Parallel Old、CMS收集器，除了CMS是并发收集器外，其他的收集器都是Stop the World的垃圾收集器。

CMS收集器是一款真正意义上的并发收集器，它将老年代的垃圾收集划分为四个阶段：初始标记、并发标记、重新标记和并发清除。初始标记仅仅只标记GC Roots可以直接到达的对象，并发标记就是GC Roots继续真正tracing的过程，重新标记主要是修正并发标记期间，用户程序运作导致标记产生变动的那一部分对象的标记记录。

CMS收集器有一些缺点：

+ 并发收集器对CPU资源太敏感，比如在并发标记和并发清除阶段会占用CPU，也就是用于工作的CPU个数会减少，用户可能会感觉到速度变慢。
+ CMS收集器无法处理浮动垃圾，并发清理时，用户的工作线程还在运行，这一阶段产生的垃圾只能等到下次回收，这部分垃圾叫做浮动垃圾。
+ 由于CMS垃圾收集器基于标记-清除算法，会产生内存碎片。

还有一个特殊的垃圾收集器就是G1收集器，它横跨了新生代和老年代，它将新生代和老年代划分为一个个Region，内部维护了一个优先队列，每次收集都会选择性价比最高的Region进行收集。G1收集器采用的垃圾收集算法有复制和标记-整理算法。从整体来看是基于标记-整理算法实现的收集器，从局部来看是基于复制算法实现的。G1相对于CMS收集器的另一大优势就是可预测的停顿。

在Java11中，引入的最新的垃圾收集器Z Garbage Collector是一款可伸缩、低延迟、并发性能好的垃圾收集器。它采用了着色指针和读屏障这两项新技术。

### JVM内存常见问题

常见的内存问题就是内存泄露和内存溢出。

内存溢出指的是程序在内存申请时，没有足够的内存空间分配给它。

内存泄露指的是程序在申请内存后，无法释放已申请的内存空间，内存泄露最终会导致OOM。

出现内存泄露的情况有：
+ 长生命周期的对象持有短生命周期的对象的引用很可能发生内存泄露。
+ 集合中的内存泄露，比如ArrayList,HashMap。
+ ThreadLocal使用不当

#### 内存泄露的代码示例

```JAVA
public class Simple {
    Object object;
    public void method() {
        object = new Object();
        // other code
    }
}
```

在上面的示例中，假设创建的Object对象只有在method方法内使用，如果Simple的实例存活时间很长，但是开发者希望在method方法结束后，Object对象被回收，在这样的情况下，是不会被回收的。如果这使用在一个接口中，每分钟接口的调用量是10000，那么这个对象不断创建后，很容易引起OOM的问题。

有一种解决方式如下,及时的解除掉引用，便于垃圾的回收。

```JAVA
public class Simple {
    Object object;
    public void method() {
        object = new Object();
        // use Object
        object = null;
        // other code
    }
}
```

使用集合类也会引起内存泄露，代码如下

```JAVA
public class Simple{

    private static List<Object> list = new ArrayList<>();

    public static void method() {
        Object object = new Object();
        list.add(object);
        object = null;
    }

}
```

在上面示例中，如果仅仅只是释放引用本身，那么list仍然在引用该对象，所以对象不会被回收，如果需要被回收，可以将对象remove掉，或者list = null进行回收。

## 四、类加载

类加载的过程分为加载、连接、初始化、使用、卸载这几个阶段，也就是类的生命周期。

### 加载

在.java文件编译后，生成了.class是一个字节流文件，加载的过程，就是通过类的全限定名获取这个类的二进制字节流文件，将这个文件的字节流转化为方法区的运行时数据结构（方法区存储了虚拟机加载的类信息），在内存中生成这个类的Class对象，作为方法区这个类的各种数据的访问入口。

其实在加载的过程不一定需要读取.class文件，其他地方也可以读，可以从ZIP包中读取，用于反射的运行时计算生成（动态代理）。

### 验证

验证包含了文件格式的验证

+ 是否以魔数0XCAFFBABE开头
+ 主次版本号是否在当前虚拟机处理范围之内
+ 常量池是否包含不被支持的常量类型

2. 元数据的验证

+ 这个类是否有父类，类的父类是否继承了不被允许继承的类
+ 类是否实现了抽象类或者是接口中要求实现的方法

3. 字节码验证

+ 数据类型是否正确，是否把long型的数字赋给了int型
+ 引用变量的赋值是否正确，是否把父类的实例赋给了子类的引用变量

4. 符号引用验证

+ 通过字符串描述的全限定名是否找到对应的类
+ 访问控制符控制的变量或方法是否能被当前类访问

### 准备

主要是给类变量分配内存并设置类变量的初始值（零值）。

``` JAVA
private static int value = 123;
```

首先在准备阶段赋给变量value的初始值为0，而不是123。

如果定义的是一个常量，那么会直接给该常量赋值对应的值。

```JAVA
private static final int value = 123;
```

### 解析

将符号引用转为直接引用的过程。

### 初始化

类变量赋真实的值，在初始化阶段会执行\<clinit>()方法，\<clinit>()方法是由编译器自动收集类中所有的类变量的赋值动作和静态语句块中的语句合并产生的。

虚拟机保证了在执行子类\<clinit>()之前，父类的\<clinit>()已经执行完毕。

## 参考文献

1. [Java 永久代去哪儿了](https://www.infoq.cn/article/Java-PERMGEN-Removed/)
2. [ThreadLocal内存泄漏](https://www.cnblogs.com/aspirant/p/8991010.html)
