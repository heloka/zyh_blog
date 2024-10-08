---
{"dg-publish":true,"permalink":"/02 Areas/01 Java学习/javaguide八股文/1.java基础/5.JVM/02JVM垃圾回收详解（重点）/"}
---

> - 常见面试题：
>
> - 如何判断对象是否死亡（两种方法）。
> - 简单的介绍一下强引用、软引用、弱引用、虚引用（虚引用与软引用和弱引用的区别、使用软引用能带来的好处）。
> - 如何判断一个常量是废弃常量
> - 如何判断一个类是无用的类
> - 垃圾收集有哪些算法，各自的特点？
> - HotSpot 为什么要分为新生代和老年代？
> - 常见的垃圾回收器有哪些？
> - 介绍一下 CMS,G1 收集器。
> - Minor Gc 和 Full GC 有什么不同呢？

## 堆空间的基本结构
Java的自动内存管理主要集中在对象内存的回收和分配上。其中，最核心的功能是对**堆内存**中对象的分配与回收的管理，Java 堆是垃圾收集器管理的主要区域，因此也被称作 **GC 堆（Garbage Collected Heap）**。
从垃圾回收的角度来说，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆被划分为了几个不同的区域，这样我们就可以根据各个区域的特点选择合适的垃圾收集算法。
在 JDK 7 版本及 JDK 7 版本之前，堆内存被通常分为下面三部分：
1. 新生代内存(Young Generation)
2. 老生代(Old Generation)
3. 永久代(Permanent Generation)
下图所示的 Eden 区、两个 Survivor 区 S0 和 S1 都属于新生代，中间一层属于老年代，最下面一层属于永久代。
![b4cef63524070616e83ad9a28a51c498_MD5.png](/img/user/04%20Archives/image/b4cef63524070616e83ad9a28a51c498_MD5.png)
注意， **JDK 8 版本之后 PermGen(永久) 已被 Metaspace(元空间) 取代，元空间使用的是直接内存** 。
- 相关阅读：[[02 Areas/01 Java学习/javaguide八股文/1.java基础/5.JVM/01Java内存区域详解（重点）#方法区和永久代以及元空间是什么关系呢？ 四星\|01Java内存区域详解（重点）#方法区和永久代以及元空间是什么关系呢？ 四星]]

### GC 准确的分类
针对 HotSpot VM 的实现，它里面的 GC 按照回收区域**分类分为两类**：
**部分收集 (Partial GC)：**
- **新生代收集**（Minor GC / Young GC）：只对新生代进行垃圾收集；
- **老年代收集**（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
- 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。只有G1收集器使用。
**整堆收集** (Full GC)：收集整个 Java 堆和方法区。
## 内存分配和回收原则 #四星
对象的内存分分配主要是指堆上分配（也可栈上分配），对象主要分配在新生代Eden区，如果启动了本地线程分配缓冲，则按照线程优先在TLAB上分配。少数情况下也会直接分配在老年代，分配的规则不固定，取决于垃圾回收器组合以及JVM中与内存相关参数的设置。

目前以Serial/Serial Old收集器为例。
### 对象优先在 Eden 区分配
- 大多数情况下，**对象在新生代中 Eden 区分配**。如果Eden区没有足够的空间来分配对象，将触发一次Minor GC（新生代垃圾回收）。
- 在Minor GC中，会对Eden区和Survivor区进行垃圾回收，将存活的对象移到Survivor区，并清理掉无用的对象。如果Survivor区也无法容纳存活的对象，或者对象年龄达到一定阈值，就会被晋升到老年代。执行 Minor GC 后，后面分配的对象如果能够存在 Eden 区的话，还是会在 Eden 区分配内存。

### 大对象直接进入老年代
- 大对象指的是需要大量连续内存空间的Java对象，比如很大的字符串以及数组。
- 大对象直接分配到老年代，避免在Eden和Survivor区发生大量内存复制，或者提前触发Full GC。
通过**XX:PertenureSizeThreshold** 参数，可以指定大于这个值的对象直接进入老年代。

### 长期存活的对象将进入老年代
既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代中。为了做到这一点，虚拟机给每个对象一个对象年龄（Age）计数器。
- 大部分情况，对象都会首先在 Eden 区域分配。如果对象在 Eden 出生并经过第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间（s0 或者 s1）中，并将对象年龄设为1(Eden 区->Survivor 区后对象的初始年龄变为 1)。
- Survivor区中对象每经过一次Minor GC，年龄增加1岁。
- **当对象年龄增加到一定程度**（默认为 15 岁），就会被晋升到老年代中。对象晋升老年代的年龄阈值，可通过参数-XX:MaxTenuringThreshold 设置。
### 动态的对象年龄判断
考虑道不同程序的内存使用状况，虚拟机并不是永远地要求对象的年龄必须达到了`MaxTenuringThreshold` 才能晋升老年代。
Hotspot遍历所有对象时，会按照年龄从小到大对其所占用的大小进行累积，当**累积的某个年龄大小超过了survivor区的一半时，取这个年龄和参数`-XX:MaxTenuringThreshold`中更小的一个值**，作为新的晋升年龄阈值。
**总结：** 如果Survivor空间中相同年龄的所有对象大小的总和大于 Survivor 空间的一半，那么年龄大于等于该年龄对象直接进入老年代。

### 空间分配担保
空间分配担保是为了确保在 Minor GC 之前老年代本身还有容纳新生代所有对象的剩余空间。
在发生MinorGC之前，虚拟机会先检查**老年代最大可用的连续空间是否大于新生代所有对象总空间**，如果这个条件成立，那么MinorGC可以确保是安全的。
如果不成立，则虚拟机会先查看 `-XX:HandlePromotionFailure` 参数的设置值是否允许担保失败(Handle Promotion Failure)
- 如果允许，那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小;
- 如果大于，JVM将尝试进行一次 Minor GC，尽管这次 Minor GC 是有风险的;
- 如果小于或者 `-XX: HandlePromotionFailure` 设置不允许冒险，那么将改为进行一次 Full GC。

JDK 6 Update 24 之后，规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行 Minor GC，否则将进行 Full GC。
空间分配担保的机制保证了在进行 Minor GC 时，老年代能够容纳新生代的对象，从而避免因老年代空间不足而导致的 Full GC 操作。
## 死亡对象判断方法 #四星
堆中几乎放着所有的对象实例，对堆垃圾回收前的第一步就是要判断哪些对象已经死亡（即不能再被任何途径使用的对象）。
### 引用计数法
给对象中添加一个引用计数器：
- 每当有一个地方引用它，计数器就加 1；
- 当引用失效，计数器就减 1；
- 任何时候计数器为 0 的对象就是不可能再被使用的。
**这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存，其最主要的原因是它很难解决对象之间循环引用的问题。**
![ff8c476fb187c690724e173686c42979_MD5.png](/img/user/04%20Archives/image/ff8c476fb187c690724e173686c42979_MD5.png)
所谓对象之间的相互引用问题，如下面代码所示：除了对象 `objA` 和 `objB` 相互引用着对方之外，这两个对象之间再无任何引用。但是他们因为互相引用对方，导致它们的引用计数器都不为 0，于是引用计数算法无法通知 GC 回收器回收他们。

### 可达性分析算法 #四星
这个算法的基本思想就是通过一系列的称为 **“GC Roots”** 的对象作为起点，从这些节点开始向下搜索，节点所走过的路径称为引用链，当一个对象到 GC Roots 没有任何引用链相连的话，则证明此对象是不可用的，需要被回收。
下图中的 `Object 6 ~ Object 10` 之间虽有引用关系，但它们到 GC Roots 不可达，因此为需要被回收的对象。
![04369f598101492d82b3524b44d9235f_MD5.png](/img/user/04%20Archives/image/04369f598101492d82b3524b44d9235f_MD5.png)
**哪些对象可以作为 GC Roots 呢？**
- 虚拟机栈(栈帧中的局部变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 所有被同步锁持有的对象
- JNI（Java Native Interface）引用的对象

**对象可以被回收，就代表一定会被回收吗？**
不一定，即使对象在可达性分析中被判定为不可达，也并非是立即回收的，要真正宣告一个对象死亡，至少要经历两次标记过程；

可达性分析法中不可达的对象被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行 `finalize` 方法。如果对象没有覆盖 `finalize` 方法，或 `finalize` 方法已经被虚拟机调用过，虚拟机认为没有必要执行。
被判定为需要执行的对象将会被放在一个队列中进行第二次标记，**除非这个对象与引用链上的任何一个对象建立关**联，否则这个对象将被真正回收。

> `Object` 类中的 `finalize` 方法一直被认为是一个糟糕的设计，成为了 Java 语言的负担，影响了 Java 语言的安全和 GC 的性能。JDK9 版本及后续版本中各个类中的 `finalize` 方法会被逐渐弃用移除。忘掉它的存在吧！

### 引用类型总结 #二星
无论是通过引用计数法判断对象引用数量，还是通过可达性分析法判断对象的引用链是否可达，判定对象的存活都与“引用”有关。
JDK1.2 之前，Java 中引用的定义很传统：如果 reference 类型的数据存储的数值代表的是另一块内存的起始地址，就称这块内存代表一个引用。
JDK1.2 以后，Java 对引用的概念进行了扩充，将引用分为强引用、软引用、弱引用、虚引用四种（引用强度逐渐减弱）
![3cac66a25a0fca86f6b34dbeb799ceae_MD5.png](/img/user/04%20Archives/image/3cac66a25a0fca86f6b34dbeb799ceae_MD5.png)
**1．强引用（StrongReference）**
以前我们使用的大部分引用实际上都是强引用，这是使用最普遍的引用。如果一个对象具有强引用，那就类似于**必不可少的生活用品**，垃圾回收器绝不会回收它。当内存空间不足，Java 虚拟机宁愿抛出 OutOfMemoryError 错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。
**2．软引用（SoftReference）**
如果一个对象只具有软引用，那就类似于**可有可无的生活用品**。如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。
软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中。
**3．弱引用（WeakReference）**
如果一个对象只具有弱引用，那就类似于**可有可无的生活用品**。弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。
弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。
**4．虚引用（PhantomReference）**
"虚引用"顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。
**虚引用主要用来跟踪对象被垃圾回收的活动**。
**虚引用与软引用和弱引用的一个区别在于：** 虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。**程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收**。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。
特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为**软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生**。
### 如何判断一个常量是废弃常量？ #二星
运行时常量池主要回收的是废弃的常量。那么，我们如何判断一个常量是废弃常量呢？
字符串常量池在堆, 运行时常量池在方法区（元空间）。
假如在字符串常量池中存在字符串 "abc"，如果当前没有任何 String 对象引用该字符串常量的话，就说明常量 "abc" 就是废弃常量，如果这时发生内存回收的话而且有必要的话，"abc" 就会被系统清理出常量池了。
### 如何判断一个类是无用的类？ #三星
方法区主要回收的是无用的类，那么如何判断一个类是无用的类的呢？
判定一个常量是否是“废弃常量”比较简单，假如在字符串常量池中存在字符串 "abc"，**如果当前没有任何 String 对象引用该字符串常量的话**，就说明常量 "abc" 就是废弃常量，如果这时发生内存回收的话而且有必要的话，"abc" 就会被系统清理出常量池了。
而要判定一个类是否是“无用的类”的条件则相对苛刻许多。类需要同时满足下面 3 个条件才能算是 **“无用的类”**：
- 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
- 加载该类的 `ClassLoader` 已经被回收。
- 该类对应的 `java.lang.Class` 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。
虚拟机可以对满足上述 3 个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样不使用了就会必然被回收。
## 垃圾收集算法 #五星
### 标记-清除算法
标记-清除（Mark-and-Sweep）算法分为“标记（Mark）”和“清除（Sweep）”阶段：首先标记出所有不需要回收的对象，在标记完成后统一回收掉所有没有被标记的对象。
它是最基础的收集算法，后续的算法都是对其不足进行改进得到。这种垃圾收集算法会带来两个明显的问题：
1. **效率问题**：标记和清除两个过程效率都不高。
2. **空间问题**：标记清除后会产生大量不连续的内存碎片。
![4d263d7b354335cd1a32632861132bf9_MD5.png](/img/user/04%20Archives/image/4d263d7b354335cd1a32632861132bf9_MD5.png)
整个标记-清除过程大致是这样的：
1. 当一个对象被创建时，给一个标记位，假设为 0 (false)；
2. 在标记阶段，我们将所有可达对象（或用户可以引用的对象）的标记位设置为 1 (true)；
3. 扫描阶段清除的就是标记位为 0 (false)的对象。
### 复制算法
为了解决标记-清除算法的效率和内存碎片问题，复制（Copying）收集算法出现了。它可以将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。
![ef98c74ffc1ea0fddd6f91ef63ee574e_MD5.png](/img/user/04%20Archives/image/ef98c74ffc1ea0fddd6f91ef63ee574e_MD5.png)
虽然改进了标记-清除算法，但依然存在下面这些问题：
- **可用内存变小**：可用内存缩小为原来的一半。
- **不适合老年代**：如果存活对象数量比较大，复制性能会变得很差。
### 标记-整理算法
标记-整理（Mark-and-Compact）算法是根据老年代的特点提出的一种标记算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。
![3be824b2a60f69e0f262860f5da1a519_MD5.png](/img/user/04%20Archives/image/3be824b2a60f69e0f262860f5da1a519_MD5.png)
由于多了整理这一步，因此效率也不高，适合老年代这种垃圾回收频率不是很高的场景。
### 分代收集算法
分代收集算法是一种垃圾收集的策略，根据对象的生命周期将内存分为不同的代，主要分为新生代和老年代。这种策略的核心思想是根据各个年代的特点选择合适的垃圾收集算法，以提高垃圾收集的效率。

比如在新生代中，每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。
而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。
### **延伸面试问题：** HotSpot 为什么要分为新生代和老年代？
HotSpot虚拟机之所以采用分代收集算法，是为了更好地结合不同代的对象特点，提高垃圾收集的效率。新生代的“标记-复制”算法适用于大量短生命周期的对象，而老年代的“标记-清除”或“标记-整理”算法则适用于长生命周期的对象。这样的划分和策略可以更好地满足不同对象的收集需求，提高整体的垃圾回收效率。

## 垃圾收集器 #四星 
12. 对于 JVM 的垃圾收集器你有什么了解的？【⭐⭐⭐⭐】有时候面试官会问出这种十分开放性的问题，你需要脑子里过一下你对这个大问题下的哪些知识熟悉哪些不熟悉，不熟悉的点一下就过，熟悉的展开讲。遇到这种问题主要从自己熟悉得方面切入就可以了。后来的面试还真遇到过好几次这种情况，我就答，垃圾收集器的种类有以下几种 Serial，ParNew...现在用的多的还是 CMS 和 G1，CMS 的垃圾收集流程是 xxx，G1 的垃圾收集流程是 xxx，他们特点是...就这样把**话题引到 CMS 和 G1** 了，只 CMS 和 G1 这部分和面试官讨论十几分钟完全没问题。
13. 新生代垃圾收集器有哪些？老年代垃圾收集器有哪些？哪些是单线程垃圾收集器，哪些是多线程垃圾收集器？各有什么特点？各基于哪一种垃圾收集算法？【⭐⭐⭐⭐】

**如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。**
虽然我们对各个收集器进行比较，但并非要挑选出一个最好的收集器。因为直到现在为止还没有最好的垃圾收集器出现，更加没有万能的垃圾收集器，**我们能做的就是根据具体应用场景选择适合自己的垃圾收集器**。试想一下：如果有一种四海之内、任何场景下都适用的完美收集器存在，那么我们的 HotSpot 虚拟机就不会实现那么多不同的垃圾收集器了。
JDK 默认垃圾收集器（使用 `java -XX:+PrintCommandLineFlags -version` 命令查看）：
- JDK 8：Parallel Scavenge（新生代）+ Parallel Old（老年代）
- JDK 9 ~ JDK20: G1

### Serial 收集器
- **特点：**
  - Serial是最简单的垃圾收集器，使用单线程进行垃圾收集，垃圾收集时使用"Stop The World"暂停所有工作线程。
  - 没有线程交互的开销，简单高效，适用于Client模式。
- **算法选择：**
  - **新生代**版：标记-复制算法。
  - 老年代版：标记-整理算法。
![8e4398326b508b3bb7a2fbce21e52149_MD5.png](/img/user/04%20Archives/image/8e4398326b508b3bb7a2fbce21e52149_MD5.png)

### Serial Old 收集器
Serial Old是Serial收集器的**老年代版本**，在jdk1.5之前的版本与Parallel收集器搭配使用，或者作为CMS收集器的备选方案。
![8e4398326b508b3bb7a2fbce21e52149_MD5.png](/img/user/04%20Archives/image/8e4398326b508b3bb7a2fbce21e52149_MD5.png)

### ParNew 收集器
- **特点：**
  - 相当于多线程版本的Serial收集器。
  - 适用于Server模式。
  - 除了 Serial 收集器外，只有它能与 CMS 收集器配合工作
- **算法：**
  - 属于新生代收集器，使用标记-复制算法，老年代一般配合CMS或者serial Old使用。

![891b2e1d86770c3232e9912a28dad441_MD5.png](/img/user/04%20Archives/image/891b2e1d86770c3232e9912a28dad441_MD5.png)
**并行和并发概念补充：**
- **并行（Parallel）**：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
- **并发（Concurrent）**：指用户线程与垃圾收集线程同时执行（但不一定是并行，可能会交替执行），用户程序在继续运行，而垃圾收集器运行在另一个 CPU 上。
### Parallel Scavenge 收集器
- **特点：**
  - 适用于吞吐量优先的场景，提供了很多参数供用户找到最合适的停顿时间或最大吞吐量。
  - 配备了**自适应调节策略**，能够根据当前系统的性能状况动态调整垃圾收集的参数。
  - **JDK1.8 默认使用 Parallel Scavenge + Parallel Old**。
- **算法：**
  - **新生代**版本：标记-复制算法。
  - 老年代版本：标记-整理算法。
![ad77642dc2ce02c438b9741eae830a21_MD5.png](/img/user/04%20Archives/image/ad77642dc2ce02c438b9741eae830a21_MD5.png)
#### 常见参数：
1.  **`-XX:MaxGCPauseMillis`：** 用于指定期望的最大垃圾收集停顿时间（毫秒）。Parallel Scavenge会尽量控制停顿时间在这个范围内。
2.  **`-XX:GCTimeRatio`：** 设置垃圾收集时间占总时间的比率。例如，设置为4表示垃圾收集时间占总时间的1/4。该参数与`-XX:MaxGCPauseMillis`一起用于自适应调节。
3.  **`-XX:ParallelGCThreads`：** 用于设置垃圾收集时的线程数量，即新生代垃圾收集的并行线程数。默认情况下，此值会根据系统的CPU核心数动态调整。
4.  **`-XX:MaxHeapSize`：** 用于设置JVM堆的最大大小。Parallel Scavenge收集器的堆大小会影响吞吐量和停顿时间。
5.  **`-XX:InitialHeapSize`：** 设置JVM堆的初始大小。根据实际需求调整初始堆大小可以影响应用程序启动时的性能。
6.  **`-XX:TargetSurvivorRatio`：** 用于设置期望的Survivor空间的使用率。Parallel Scavenge收集器的新生代包含两个Survivor区，该参数影响这两个区的使用率。
7.  **`-XX:MaxTenuringThreshold`：** 设置对象在Survivor区中的最大年龄。默认值为15，表示对象在Survivor区中存活的次数达到15次后，将晋升到老年代。
8.  **`-XX:NewRatio`：** 设置新生代与老年代的比例。例如，`-XX:NewRatio=2`表示新生代占堆的1/3。
9.  **`-XX:NewSize` 和 `-XX:MaxNewSize`：** 分别用于设置新生代的初始大小和最大大小。
10.  **`-XX:AdaptiveSizePolicy`：** 启用或禁用自适应调节策略。默认情况下启用，可以根据系统负载自动调整各种参数。


### Parallel Old 收集器
**Parallel Scavenge 收集器的老年代版本**。使用多线程和“标记-整理”算法。在注重吞吐量以及 CPU 资源的场合，都可以优先考虑 Parallel Scavenge 收集器和 Parallel Old 收集器。
![ad77642dc2ce02c438b9741eae830a21_MD5.png](/img/user/04%20Archives/image/ad77642dc2ce02c438b9741eae830a21_MD5.png)

### CMS 收集器 #四星
- [[04 Archives/99 未分类/杂项/深入理解JAVA虚拟机#CMS收集器\|深入理解JAVA虚拟机#CMS收集器]]
- **特点：**
  - 是一种**以最短回收停顿时间为目标**的并发收集器。适合在注重用户体验的应用上使用。
  -  是HotSpot 虚拟机上第一款真正意义上的并发收集器，**第一次实现了垃圾收集线程与用户线程**（基本上）同时工作。
基于 [[02 Areas/01 Java学习/javaguide八股文/1.java基础/5.JVM/02JVM垃圾回收详解（重点）#标记-清除算法\|“标记-清除”算法]]实现，标记过程分为四步：
- **初始标记：**  暂停其他其他线程，记录与 GC Roots 直接相连的对象，速度很快；
- **并发标记：** **与用户线程同时进行**，通过一个闭包结构记录可达对象。由于用户线程可能不断更新引用域，闭包结构无法实时包含当前所有可达对象，所以这个算法里会**跟踪记录这些发生引用更新的地方。**
	- **并发标记阶段**就是从GC Roots 的直接关联对象开始遍历整个对象图的过程，耗时较长但不需要停顿用户线程。
- **重新标记：** 用于修正并发标记期间**因为用户程序继续运行，而导致标记产生变动的那一部分对象**的标记记录，这个阶段的停顿时间较初始标记阶段稍长，但远比并发标记阶段短。（详见[[04 Archives/99 未分类/杂项/深入理解JAVA虚拟机#^e25c21\|增量更新方法]]）
- **并发清除：** 开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

其中**并发标记阶段**就是从GC Roots 的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾收集线程一起并发运行；

![3096fa807a2d40fbc789e4fa7041f02a_MD5.png](/img/user/04%20Archives/image/3096fa807a2d40fbc789e4fa7041f02a_MD5.png)
CMS主要优点是：**并发收集、低停顿**。但是它有下面三个明显的缺点：[[04 Archives/99 未分类/杂项/CMS收集器的缺点\|CMS收集器的缺点]]
- 对 CPU 资源敏感；在并发阶段虽然不会导致用户线程停顿，但是会因为占用了一部分线程使应用程序变慢。
- 无法处理浮动垃圾；在最后并发清理过程中，用户线程执行也会产生垃圾，但是这部分垃圾是在标记之后，所以只有等到下一次gc的时候清理掉。
- 它使用的“**标记-清除**”算法会产生大量的空间碎片，这给后续大对象空间的分配带来很多麻烦，往往会出现老年代还有很多剩余空间，但就是无法找到足够大的连续空间来分配当前对象，而不得不提前触发一次Full GC的情况。
	- 为了解决这个问题CMS提供了一个开关参数，用于在CMS不得不进行FullGC 时开启**内存碎片的合并整理**过程，但是内存整理的过程必须移动存活对象，无法并发，这样空间碎片问题解决了，但是停顿时间变长了。
	- 通过另外一个参数-XX：CMSFullGCsBeforeCompaction（此参数从JDK 9开始废弃），让CMS收集器在执行过若干次（数量 由参数值决定）不整理空间的Full GC之后，下一次进入Full GC前会先进行碎片整理。
### G1 收集器 #四星
G1 (Garbage-First) 收集器是一款面向服务器的垃圾收集器，主要适用于配备多颗处理器及大容量内存的机器，以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征。被认为是 JDK1.7 中 HotSpot 虚拟机的一个重要进化特征。[[04 Archives/99 未分类/杂项/G1收集器的更多相关内容 1\|G1收集器的更多相关内容 1]]
#### G1收集器的特点
1. **并行与并发：** G1 可充分利用 CPU、多核环境下的硬件优势，通过使用多个 CPU（CPU 或 CPU 核心）来缩短 Stop-The-World 停顿时间。部分其他收集器原本需要停顿 Java 线程执行的 GC 动作，G1 收集器仍然可以通过并发的方式让 java 程序继续执行。
2. **分代收集：** G1仍然采用分代的概念，会区分新生代和老年代。但与其他收集器不同，G1不再坚持固定大小以及固定数量的分代区域划分，而是将整个Java堆划分为多个大小相等的独立区域（Region），灵活管理新生代和老年代，不要求连续的分代区域。
3. **空间整合：** G1使用了独立区域（Region）概念，内存的回收是以 region 作为基本单位的。整体上采用“标记-整理”算法实现收集，从局部上则采用“标记-复制”算法。这两种算法都意味着G1运作期间不会产生内存空间碎片。
4. **可预测的停顿：** G1 的停顿时间相对于 CMS 具有更高的可预测性。除了追求低停顿外，G1 能够建立可预测的停顿时间模型，使使用者能够明确指定在一个给定时间片段内，GC 消耗的时间不得超过预定值。
	- 由于分区的原因，G1 可以只选取部分区域进行内存回收，这样缩小了回收的范围，因此对于全局停顿情况的发生也能得到较好的控制。
	- G1 跟踪各个 Region 里面的垃圾 堆积的价值大小，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region。保证了 G1 收集器在有限的时间内可以获取尽可能高的收集效率。

相较于 CMS，G1 还不具备全方位、压倒性优势。比如在用户程序运行过程中，G1 无论是为了垃圾收集产生的内存占用（Footprint）还是程序运行时的额外执行负载（overload）都要比 CMS 要高。
从经验上来说，在小内存应用上 CMS 的表现大概率会优于G1，而 G1 在大内存应用上则发挥其优势。平衡点在 6-8GB 之间。
#### G1 收集器的**运作过程**：
1. **初始标记：** 暂停其他其他线程，记录直接与 GC Roots 相连的对象。
2. **并发标记：** 与用户线程同时进行，从GC Roots开始对堆中对象进行可达性分析，找出存活对象，并记录它们所在的Region。
3. **最终标记：** 短暂暂停用户线程，修正并发标记期间由于用户程序继续运行而导致标记变动的对象标记记录。
4. **筛选回收：** 负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划。
![3b31ed86db961e2fb64e8238445e38fc_MD5.png](/img/user/04%20Archives/image/3b31ed86db961e2fb64e8238445e38fc_MD5.png)
G1 收集器在后台维护了一个优先列表，在每次根据允许的收集时间内，优先选择回收价值最大的 Region，这就是“Garbage-First”名称的由来。这种使用 Region 划分内存空间以及有优先级的区域回收方式，保证了 G1 收集器在有限时间内可以尽可能高的收集效率（把内存化整为零）。
从 JDK9 开始，G1 **垃圾收集器成为了默认的垃圾收集器**。

### ZGC 收集器
与 CMS 中的 ParNew 和 G1 类似，ZGC 也采用标记-复制算法，不过 ZGC 对该算法做了重大改进。
在 ZGC 中出现 Stop The World 的情况会更少！
Java11 的时候 ，ZGC 还在试验阶段。经过多个版本的迭代，不断的完善和修复问题，ZGC 在 Java 15 已经可以正式使用了！
不过，默认的垃圾回收器依然是 G1。你可以通过下面的参数启动 ZGC：
```bash
$ java -XX:+UseZGC className
```
关于 ZGC 收集器的详细介绍推荐阅读美团技术团队的 [新一代垃圾回收器 ZGC 的探索与实践](https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html) 这篇文章。
