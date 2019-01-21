> 好的文章是能把各个知识点，通过逻辑关系串连起来，让人豁然开朗的同时又记忆深刻。
>

导读：对象除了生死之外，还有其他状态吗？对象真正的死亡，难道只经历一次简单的判定？如何在垂死的边缘“拯救”一个将死对象？判断对象的生死存活都有那些算法？本文带你一起找到这些答案。

在正式开始之前，我们先来了解一下垃圾回收。

## GC介绍

**GC：**Garbage Collection，中文翻译为垃圾回收。

**GC的历史**

GC有着很长的历史了，最初的GC算法发布于1960年（已经快有60年的历史了），Lisp之父John McCarthy发布的，他是一名非常有名的黑客，也是人工智能之父，同时也是GC之父。

**为什么要学习GC？**

1、排查内存溢出和内存泄露的问题。

2、系统调优，处理更高的并发瓶颈。

**GC的作用**

1、找到内存空间的垃圾。

2、回收垃圾。

## 对象生死判断算法

垃圾回收的第一步就是判断对象是否存活，只有“死去”的对象，才会被垃圾回收器所收回。

#### 引用计数器算法

引用计算器判断对象是否存活的算法是这样的：给每一个对象设置一个引用计数器，每当有一个地方引用这个对象的时候，计数器就加1，与之相反，每当引用失效的时候就减1。

**优点：**实现简单、性能高。

**缺点：**增减处理频繁消耗cpu计算、计数器占用很多位浪费空间、最重要的缺点是无法解决循环引用的问题。

因为引用计数器算法很难解决循环引用的问题，所以主流的Java虚拟机都没有使用引用计数器算法来管理内存。

来看一段循环引用的代码：

```java
public class ReferenceDemo {
    public Object instance = null;
    private static final int _1Mb = 1024 * 1024;
    private byte[] bigSize = new byte[10 * _1Mb]; // 申请内存
    public static void main(String[] args) {
        System.out.println(String.format("开始：%d M",Runtime.getRuntime().freeMemory() / (1024 * 1024)));
        ReferenceDemo referenceDemo = new ReferenceDemo();
        ReferenceDemo referenceDemo2 = new ReferenceDemo();
        referenceDemo.instance = referenceDemo2;
        referenceDemo2.instance = referenceDemo;
        System.out.println(String.format("运行：%d M",Runtime.getRuntime().freeMemory() / (1024 * 1024)));
        referenceDemo = null;
        referenceDemo2 = null;
        System.gc(); // 手动触发垃圾回收
        System.out.println(String.format("结束：%d M",Runtime.getRuntime().freeMemory() / (1024 * 1024)));
    }
}
```

运行的结果：

> 开始：117 M
>
> 运行中：96 M
>
> 结束：119 M

从结果可以看出，虚拟机并没有因为相互引用就不回收它们，也侧面说明了虚拟机并不是使用引用计数器实现的。

#### 可达性分析算法

在主流的语言的主流实现中，比如Java、C#、甚至是古老的Lisp都是使用的可达性分析算法来判断对象是否存活的。

这个算法的核心思路就是通过一些列的“GC Roots”对象作为起始点，从这些对象开始往下搜索，搜索所经过的路径称之为“引用链”。

当一个对象到GC Roots没有任何引用链相连的时候，证明此对象是可以被回收的。如下图所示：

![可达性分析算法](http://icdn.apigo.cn/blog/gccalc-001.png)

**在Java中，可作为GC Roots对象的列表：**

- Java虚拟机栈中的引用对象。
- 本地方法栈中JNI（既一般说的Native方法）引用的对象。
- 方法区中类静态常量的引用对象。
- 方法区中常量的引用对象。

## 对象生死与引用的关系

从上面的两种算法来看，不管是引用计数法还是可达性分析算法都与对象的“引用”有关，这说明：对象的引用决定了对象的生死。那对象的引用都有那些呢？

在JDK1.2之前，引用的定义很传统：如果reference类型的数据中存储的数值代表的是另一块内存的起始地址，就称这块内存代表着一块引用。

这样的定义很纯粹，但是也很狭隘，这种情况下一个对象要么被引用，要么没引用，对于介于两者之间的对象显得无能为力。

JDK1.2之后对引用进行了扩充，将引用分为：

- 强引用（Strong Reference）

- 软引用（Soft Reference）

- 弱引用（Weak Reference）

- 虚引用（Phantom Reference）

这也就是文章开头第一个问题的答案，对象不是非生即死的，当空间还足够时，还可以保留这些对象，如果空间不足时，再抛弃这些对象。很多缓存功能的实现也符合这样的场景。

强引用、软引用、弱引用、虚引用，这4种引用的强度是依次递减的。

**强引用：**在代码中普遍存在的，类似“Object obj = new Object()”这类引用，只要强引用还在，垃圾收集器永远不会回收掉被引用的对象。

**软引用：**是一种相对强引用弱化一些的引用，可以让对象豁免一些垃圾收集，只有当jvm认为内存不足时，才会去试图回收软引用指向的对象。jvm会确保在抛出OutOfMemoryError之前，清理软引用指向的对象。

**弱引用：**非必需对象，但它的强度比软引用更弱，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。

**虚引用：**也称为幽灵引用或幻影引用，是最弱的一种引用关系，无法通过虚引用来获取一个对象实例，为对象设置虚引用的目的只有一个，就是当着个对象被收集器回收时收到一条系统通知。

## 死亡标记与拯救

在可达性算法中不可达的对象，并不是“非死不可”的，要真正宣告一个对象死亡，至少要经历两次标记的过程。

如果对象在进行可达性分析之后，没有与GC Roots相连接的引用链，它会被第一次标记，并进行筛选，筛选的条件是此对象是否有必要执行finalize()方法。

**执行finalize()方法的两个条件：**

1、重写了finalize()方法。

2、finalize()方法之前没被调用过，因为对象的finalize()方法只能被执行一次。

如果满足以上两个条件，这个对象将会放置在F-Queue的队列之中，并在稍后由一个虚拟机自建的、低优先级Finalizer线程来执行它。

**对象的“自我拯救”**

finalize()方法是对象脱离死亡命运最后的机会，如果对象在finalize()方法中重新与引用链上的任何一个对象建立关联即可，比如把自己（this关键字）赋值给某个类变量或对象的成员变量。

来看具体的实现代码：

```java
public class FinalizeDemo {
    public static FinalizeDemo Hook = null;
    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("执行finalize方法");
        FinalizeDemo.Hook = this;
    }
    public static void main(String[] args) throws InterruptedException {
        Hook = new FinalizeDemo();
        // 第一次拯救
        Hook = null;
        System.gc();
        Thread.sleep(500); // 等待finalize执行
        if (Hook != null) {
            System.out.println("我还活着");
        } else {
            System.out.println("我已经死了");
        }
        // 第二次，代码完全一样
        Hook = null;
        System.gc();
        Thread.sleep(500); // 等待finalize执行
        if (Hook != null) {
            System.out.println("我还活着");
        } else {
            System.out.println("我已经死了");
        }
    }
}
```

执行的结果：

> 执行finalize方法
>
> 我还活着
>
> 我已经死了

从结果可以看出，任何对象的finalize()方法都只会被系统调用一次。

**不建议使用finalize()方法来拯救对象**，原因如下：

1、对象的finalize()只能执行一次。

2、它的运行代价高昂。

3、不确定性大。

4、无法保证各个对象的调用顺序。

## 参考

《深入理解Java虚拟机》

《垃圾回收的算法与实现》

>  ※ 为写好一篇技术文章，背后是读了两本书的“艰辛”。写作不易，请多支持!!!

## 最后

关注公众号，发送“gc”关键字，领取《垃圾回收的算法与实现》学习资料。

![](http://icdn.apigo.cn/myinfo/gognzhonghao.jpg)

![](http://icdn.apigo.cn/myinfo/wchat-pay.png)
