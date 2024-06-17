# JVM

![知识导图](https://img2.imgtp.com/2024/05/31/g1KRE74o.png)

## JVM常规


### **对象的创建**

Java 对象的创建过程我建议最好是能默写出来，并且要掌握每一步在做什么。

*Step1:类加载检查*

虚拟机遇到一条 new 指令时，首先将去检查

1. 这个指令的参数是否能在常量池中定位到这个类的符号引用，
2. 检查这个符号引用代表的类是否已被加载过、解析和初始化过。

如果没有，那必须先执行相应的类加载过程。

*Step2:分配内存*

在**类加载检查**通过后，接下来虚拟机将为新生对象**分配内存**。对象所需的内存大小在类加载完成后便可确定，为对象分配空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。**分配方式**有 **“指针碰撞”** 和 **“空闲列表”** 两种，**选择哪种分配方式由 Java 堆是否规整决定，而 Java 堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定**。

**内存分配的两种方式** （补充内容，需要掌握）：

- 指针碰撞：
    - 适用场合：堆内存规整（即没有内存碎片）的情况下。
    - 原理：用过的内存全部整合到一边，没有用过的内存放在另一边，中间有一个分界指针，只需要向着没用过的内存方向将该指针移动对象内存大小位置即可。
    - 使用该分配方式的 GC 收集器：Serial, ParNew
- 空闲列表：
    - 适用场合：堆内存不规整的情况下。
    - 原理：虚拟机会维护一个列表，该列表中会记录哪些内存块是可用的，在分配的时候，找一块儿足够大的内存块儿来划分给对象实例，最后更新列表记录。
    - 使用该分配方式的 GC 收集器：CMS

选择以上两种方式中的哪一种，取决于 Java 堆内存是否规整。而 Java 堆内存是否规整，取决于 GC 收集器的算法是"标记-清除"，还是"标记-整理"（也称作"标记-压缩"），值得注意的是，复制算法内存也是规整的。

**内存分配并发问题（补充内容，需要掌握）**

在创建对象的时候有一个很重要的问题，就是线程安全，因为在实际开发过程中，创建对象是很频繁的事情，作为虚拟机来说，必须要保证线程是安全的，通常来讲，虚拟机采用两种方式来保证线程安全：

- **CAS+失败重试：** CAS 是乐观锁的一种实现方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。**虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性。**
- **TLAB：** 为每一个线程预先在 Eden 区分配一块儿内存，JVM 在给线程中的对象分配内存时，首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用上述的 CAS 进行内存分配

*Step3:初始化零值* 

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

*Step4:设置对象头*

初始化零值完成之后，**虚拟机要对对象进行必要的设置**，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。 **这些信息存放在对象头中。** 另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

*Step5:执行 `init` 方法* 

在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从 Java 程序的视角来看，对象创建才刚开始，`<init>` 方法还没有执行，所有的字段都还为零。所以一般来说，执行 new 指令之后会接着执行 `<init>` 方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。

## 内存分布

![](https://img2.imgtp.com/2024/05/31/5BFdHBIw.png)

![](https://img2.imgtp.com/2024/05/31/0ddrpkWp.png)

### 线程共享：堆内存

Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就**是存放对象实例**，几乎所有的对象实例以及**数组都**在这里分配内存。

在 JDK 7 版本及 JDK 7 版本之前，堆内存被通常分为下面三部分：

1. 新生代内存(Young Generation)
2. 老年代(Old Generation)
3. 永久代(Permanent Generation)

**JDK 8 版本之后 PermGen(永久代) 已被 Metaspace(元空间) 取代，元空间使用的是本地内存**

![](https://img2.imgtp.com/2024/05/31/RLkvAx6g.png)

 ![](https://img2.imgtp.com/2024/05/31/68iVG1AY.png)

- **功能**：存储对象实例的区域，也是 Java 内存管理的主要区域。几乎所有的对象都在这里分配内存。
- **特点**：堆内存是所有线程共享的区域，堆区在 JVM 启动时创建。根据垃圾收集机制，堆区通常分为年轻代（Young Generation）和老年代（Old Generation）。
    - **年轻代（Young Generation）**：进一步分为 Eden 区、Survivor 0 区和 Survivor 1 区，主要存放新创建的对象，垃圾回收频率较高。
    - **老年代（Old Generation）**：存放经过多次垃圾回收后仍然存活的对象，垃圾回收频率较低。

### 线程共享：直接内存

方法区 (Method Area)：



- **功能**：方法区（也称为永久代（Permanent Generation）在 JDK 1.7 及以前）存储类结构、常量、静态变量和即时编译器（JIT）编译后的代码等数据。
- **特点**：方法区也是所有线程共享的区域。从 JDK 1.8 开始，永久代被移除，取而代之的是元空间（Metaspace）。元空间在本地内存中，而不是在 JVM 内存中。

### 线程私有：栈

 **1. 虚拟机栈 (JVM Stack)**

![](https://img2.imgtp.com/2024/05/31/Kd8pQQ9G.png)

**功能**：虚拟机栈是线程私有的。它存储的是 Java 方法执行的栈帧（Stack Frame），来存储方法的的变量表、操作数栈、动态链接方法、返回值、返回地址等信息。

**特点**：局部变量表存储方法参数和局部变量，这些数据在编译期确定。

 **2. 本地方法栈 (Native Method Stack)**

**功能**：为虚拟机使用到的 Native 方法服务。它主要用于<u>执行（Native）方法。</u>

**特点**：本地方法栈也是线程私有的。它可以使用 C 或 C++ 等本地代码实现。

 **3. 程序计数器 (Program Counter Register)**

- **功能**：
- 发生线程切换。这时，每个线程就需要一个属于自己的计数器来记录下一条要运行的指令。如果执行的是JAVA方法，计数器记录正在执行的java字节码地址，如果执行的是native方法，则计数器为空
- **特点**：程序计数器是线程私有的，每个线程都有一个独立的程序计数器。

 

### 堆栈区别

 *1、 功能不同* 

栈内存用来存储局部变量和方法调用，而堆内存用来存储Java中的对象。无论是成员变量，局部变量，还是类变量，它们指向的对象都存储在堆内存中。

*2、 共享性不同* 

栈内存是线程私有的。 堆内存是所有线程共有的。

 *3、 异常错误不同* 

如果栈内存或者堆内存不足都会抛出异常。 

栈空间不足：`java.lang.StackOverFlowError`。 

堆空间不足：`java.lang.OutOfMemoryError`。 

 *4、 空间大小* 

栈的空间大小远远小于堆的。

## 内存分配和原则



### **1. 年轻代（Young Generation）**

***Eden区***

Eden区是年轻代（Young Generation）中的一部分，主要用于新创建的对象。大多数新对象首先被分配到Eden区。Eden区相对较小，因为它假设大部分对象很快就会变得不可达，即“短命”的对象。

***Survivor区***

一旦Eden区满，JVM会执行一次**Minor GC**（也称为Young GC），将Eden区中仍然存活的对象复制到两个Survivor区之一（通常称为From或To）。这两个Survivor区轮流充当源区和目标区，以实现对象在两者之间的复制。每次Minor GC后，源区的存活对象被复制到目标区，然后源区被清空。这个过程称为**复制算法**，它简化了内存管理和碎片问题。

### **2. 老年代（Tenured Generation）**

经过几次Minor GC后仍然存活的对象会被晋升到**老年代**。老年代存放长期存活的对象，这里的垃圾回收频率较低，因为大多数对象是持久的。老年代的垃圾回收通常涉及`Major GC`或`Full GC`，使用**标记-清除**或**标记-整理**算法，这两种算法比复制算法更为复杂和耗时。

### **3. 永久代到元空间（PermGen到Metaspace）**

在Java 8之前，类的元数据（如类的结构信息）存储在**永久代（Permanent Generation, PermGen）中。永久代也是垃圾回收的目标之一，但由于其大小固定，容易出现PermGen OutOfMemoryError**。从Java 8开始，永久代被**元空间（Metaspace）**取代，元空间使用本地内存而不是堆内存，这意味着它的大小不再受JVM堆大小的限制。

- **Metaspace**存储类的元数据，包括类的结构信息。元空间的大小可以动态扩展，直到操作系统无法分配更多内存为止。元空间的垃圾回收主要是针对那些不再被任何类加载器引用的类元数据进行回收，这通常在类加载器自身被垃圾回收时发生。

- Metaspace是干什么的
  
    Metaspace是Java虚拟机（JVM）中用于存储类元数据的内存区域，它在Java 8及之后的版本中引入，用来替代先前的永久代（Permanent Generation）。元数据包括类的结构信息（如类名、方法、字段等）、方法元数据（如字节码、常量池等）以及其他用于描述类和接口所需的运行时数据。
    
    在Java 8之前的版本中，类元数据存储在永久代中，永久代是堆内存的一部分，其大小是固定的，这有时会导致诸如`java.lang.OutOfMemoryError: PermGen space`这样的问题，特别是在类加载频繁的应用场景下。Metaspace的引入解决了这些问题，主要有以下几个关键点：
    
    1. **动态分配与回收**：Metaspace位于本地内存中，而非JVM堆内存，这意味着它的大小不再受JVM堆大小的直接限制，可以更灵活地进行动态扩展。当类和类加载器不再被使用时，它们所占用的Metaspace空间可以被回收，从而减少内存泄漏的风险。
    2. **减少内存限制冲突**：由于Metaspace不占用堆内存，减少了堆内存与永久代之间因大小配置不当而引起的内存限制问题。
    3. **提高性能**：Metaspace的管理去除了永久代中与垃圾回收相关的复杂性，减少了Full GC的发生概率，提高了应用程序的性能。Metaspace的回收不依赖于传统的垃圾回收过程，而是与类加载器的生命周期紧密相关，当类加载器被卸载时，其所加载的类的元数据也会被回收。
    4. **配置灵活性**：用户可以通过JVM参数（如`XX:MetaspaceSize`初始化大小、`XX:MaxMetaspaceSize`最大大小）来控制Metaspace的大小，从而更好地适应不同的应用需求。
    
    尽管Metaspace的引入极大地改善了类元数据的管理，但如果不加以适当控制，Metaspace的大小仍有可能无限制地增长，最终耗尽系统可用的本地内存，导致`java.lang.OutOfMemoryError: Metaspace`错误。因此，监控和合理配置Metaspace对于维护Java应用的稳定运行至关重要。
    

**总结：**

针对 HotSpot VM 的实现，它里面的 GC 其实准确分类只有两大种：

部分收集 (Partial GC)：

- 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
- 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
- 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。

整堆收集 (Full GC)：收集整个 Java 堆和方法区

## FullGC发生

除直接调用`System.gc`外，触发`Full GC`执行的情况有如下四种。

### 1 旧生代空间不足

旧生代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误： java.lang.OutOfMemoryError: Java heap space 为避免以上两种状况引起的FullGC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

### 2 PermanetGeneration空间满

`PermanetGeneration`中存放的为一些class的信息等，当系统中

要加载的类、反射的类和调用的方法较多时,`PermanetGeneration`可能会被占满，在未配置为采用`CMS GC`的情况下会执行`Full GC`。

如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息： `java.lang.OutOfMemoryError`: `PermGen space` 为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

### 3 CMS GC时出现`promotion failed`和`concurrent mode failure`

对于采用CMS进行旧生代GC的程序而言，尤其要注意GC日志中是否有promotion failed和concurrent mode failure两种状况，当这两种状况出现时可能会触发Full GC。 promotionfailed是在进行Minor GC时，survivor space放不下、对象只能放入旧生代，而此时旧生代也放不下造成的；concurrent mode failure是在执行CMS GC的过程中同时有对象要放入旧生代，而此时旧生代空间不足造成的。 应对措施为：增大survivorspace、旧生代空间或调低触发并发GC的比率，但在JDK 5.0+、6.0+的版本中有可能会由于JDK的bug29导致CMS在remark完毕后很久才触发sweeping动作。对于这种状况，可通过设置`-XX:CMSMaxAbortablePrecleanTime=5`（单位为ms）来避免。

### 4. 统计得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间

这是一个较为复杂的触发情况，Hotspot为了避免由于新生代对象晋升到旧生代导致旧生代空间不足的现象，在进行MinorGC时，做了一个判断，如果之前统计所得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间，那么就直接触发Full GC。 例如程序第一次触发MinorGC后，有6MB的对象晋升到旧生代，那么当下一次Minor GC发生时，首先检查旧生代的剩余空间是否大于6MB，如果小于6MB，则执行Full GC。 当新生代采用PSGC时，方式稍有不同，PS GC是在Minor GC后也会检查，例如上面的例子中第一次Minor GC后，PS GC会检查此时旧生代的剩余空间是否大于6MB，如小于，则触发对旧生代的回收。 除了以上4种状况外，对于使用RMI来进行RPC或管理的Sun JDK应用而言，默认情况下会一小时执行一次Full GC。可通过在启动时通过

`-java-Dsun.rmi.dgc.client.gcInterval=3600000`来设置Full GC执行的间隔时间或通过`-XX:+DisableExplicitGC`来禁止RMI调用System.gc。 

## 死亡对象判断

### 1.引用对象计数法

原理：

- 每当有一个地方引用它，计数器就加 1；
- 当引用失效，计数器就减 1；
- 任何时候计数器为 0 的对象就是不可能再被使用的。

这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存，其最主要的原因是它很<u>难解决对象之间循环引用</u>的问题。

![](https://img2.imgtp.com/2024/05/31/jC2tgaS7.png)

如上图，无法计数清零

### **2.可达性分析算法**

到 GC Roots 不可达，因此为需要被回收的对象。下图中的 `Object 6 ~ Object 10` 之间虽有引用关系，无法到GC Roots

![](https://img2.imgtp.com/2024/05/31/DvHZqPWQ.png)

- **哪些对象可以作为 GC Roots 呢？**
    - 虚拟机栈(栈帧中的局部变量表)中引用的对象
    - 本地方法栈(Native 方法)中引用的对象
    - 方法区中类静态属性引用的对象
    - 方法区中常量引用的对象
    - 所有被同步锁持有的对象
    - JNI（Java Native Interface）引用的对象

## 引用四种类型

![Java 引用类型总结](https://oss.javaguide.cn/github/javaguide/java/jvm/java-reference-type.png)

**强饮用**

垃圾回收器绝不会回收它。当内存空间不足，Java 虚拟机宁愿抛出 OutOfMemoryError 错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。

创建一个对象 `Person person = new Person();` 这里的 `person` 就是一个强引用。

**软引用**

软引用的对象在内存充足时会被保留，但如果内存变得紧张，垃圾回收器就会回收这些对象来释放内存，以避免OutOfMemoryError。

**弱引用（Weak Reference）**

弱引用的对象在垃圾回收时一定会被回收，哪怕内存还很充裕。它用来标记那些不是必须保留的对象。

虚引用

更像是一个通知机制，告诉你可以执行某些操作了，但它本身对对象的生命周期没有任何影响。

## 死亡常量

  [**如何判断一个类是无用的类？**](https://javaguide.cn/java/jvm/jvm-garbage-collection.html#%E5%A6%82%E4%BD%95%E5%88%A4%E6%96%AD%E4%B8%80%E4%B8%AA%E7%B1%BB%E6%98%AF%E6%97%A0%E7%94%A8%E7%9A%84%E7%B1%BB)

  方法区主要回收的是无用的类，那么如何判断一个类是无用的类的呢？

  判定一个常量是否是“废弃常量”比较简单，而要判定一个类是否是“无用的类”的条件则相对苛刻许多。类需要同时满足下面 3 个条件才能算是 **“无用的类”**：

  该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。

 加载该类的 `ClassLoader` 已经被回收。

  该类对应的 `java.lang.Class` 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

  虚拟机可以对满足上述 3 个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样不使用了就会必然被回收。

## 垃圾整理算法

### **标记-清除算法**

标记-清除（Mark-and-Sweep）算法分为“标记（Mark）”和“清除（Sweep）”阶段：首先标记出所有不需要回收的对象，在标记完成后统一回收掉所有没有被标记的对象。

1. **效率问题**：标记和清除两个过程效率都不高。
2. **空间问题**：标记清除后会产生大量不连续的内存碎片。

![](https://img2.imgtp.com/2024/05/31/gGzVifZu.png)

### 复制算法

内存分为大小相同的两块 。 存活的对象复制到另一块去，然后再把使用的空间到另一块一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

- **可用内存变小**：可用内存缩小为原来的一半。
- **不适合老年代**：如果存活对象数量比较大，复制性能会变得很差。

![](https://img2.imgtp.com/2024/05/31/iMlTTAsu.png)

### 标记-整理算法 

“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接覆盖边界以外的内存。

### 不同代不同算法

1. **新生代（Young Generation）**：
    - **复制算法（Copying Algorithm）**：新生代通常采用复制算法进行垃圾回收。这种算法将新生代分为两个相同大小的区域：Eden区和两个Survivor区（一般是From区和To区）。对象首先被分配到Eden区，当Eden区满时，会触发Minor GC（新生代垃圾回收）。Minor GC 会将存活的对象复制到其中一个Survivor区，并清理掉无用对象。经过多次Minor GC 后，存活的对象会被复制到另一个Survivor区或者晋升到老年代。
2. **老年代（Old Generation）**：
    - **标记-清除算法（Mark-Sweep Algorithm）标记-整理算法（Mark-Compact Algorithm）或者分代收集算法（Generational Collection）**：老年代一般使用标记-清除算法进行垃圾回收。这种算法分为标记和清除两个阶段。首先，标记阶段会标记所有活跃的对象，然后在清除阶段将未标记的对象清除掉，释放内存空间。但是，标记-清除算法有可能会导致内存碎片化，影响后续对象的分配。因此，一些垃圾回收器会结合其他算法来优化老年代的垃圾回收，如标记-整理算法（Mark-Compact Algorithm）或者分代收集算法（Generational Collection）。
3. **永久代/元空间（Permanent Generation/Metaspace）**：
    - **标记-整理算法（Mark-Compact Algorithm）**：永久代（在Java 8 之前）或者元空间（在Java 8 及之后）通常使用标记-整理算法进行垃圾回收。这种算法会标记所有活跃的对象，然后将它们整理到一端，清理掉未标记的对象，并释放内存空间。
> 如果说垃圾回收算法是理论，那么这篇大概就是算法的实现
> 

JDK 默认垃圾收集器（使用 `java -XX:+PrintCommandLineFlags -version` 命令查看）：

- JDK 8：Parallel Scavenge（新生代）+ Parallel Old（老年代）
- JDK 9 ~ JDK20: G1



## 垃圾搜集器

### Serial 收集器

> 单线程，STW，client
> 

Serial（串行）收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一个单线程收集器了。它的 **“单线程”** 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ "`Stop The World`" ），直到它收集结束。

<u>新生代采用标记-复制算法，老年代采用标记-整理算法。</u>

![](https://img2.imgtp.com/2024/05/31/8lLti94P.png)

 Serial 收集器有没有优于其他垃圾收集器的地方呢？当然有，它简单而高效（与其他收集器的单线程相比）。Serial 收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。

运行在 Client 模式下的虚拟机来说是个不错的选择。

 

### `ParNew` 收集器

> 多线程，server

`ParNew` 收集器其实就是 Serial 收集器的多线程版本，除了使用<u>多线程进行垃圾收集</u>外，其余行为（控制参数、收集算法、回收策略等等）和 Serial 收集器完全一样。

<u>新生代采用标记-复制算法，老年代采用标记-整理算法。</u>

![](https://img2.imgtp.com/2024/05/31/dOmemKjz.png)

运行在 **Server 模式**下的虚拟机的首要选择，除了 Serial 收集器外，只有它能与 CMS 收集器（真正意义上的并发收集器，后面会介绍到）配合工作。

### Parallel Scavenge 收集器

关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。 Parallel Scavenge 收集器提供了很多参数供用户找到最合适的停顿时间或最大吞吐量，如果对于收集器运作不太了解，手工优化存在困难的时候，使用 Parallel Scavenge 收集器配合自适应调节策略，把内存管理优化交给虚拟机去完成也是一个不错的选择。

<u>新生代采用标记-复制算法，老年代采用标记-整理算法。</u>

 ![](https://img2.imgtp.com/2024/05/31/RuiDCQpX.png)

### CMS 收集器 

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用。

CMS（Concurrent Mark Sweep）收集器是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

从名字中的`Mark Sweep`这两个词可以看出，CMS 收集器是一种 **“标记-清除”算法**实现的，它的运作过程相比于前面几种垃圾收集器来说更加复杂一些。整个过程分为四个步骤：

- **初始标记：** 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；
- **并发标记：** 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
- **重新标记：** 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短
- **并发清除：** 开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

 ![](https://img2.imgtp.com/2024/05/31/YLpOvBo5.png)



从它的名字就可以看出它是一款优秀的垃圾收集器，主要优点：**并发收集、低停顿**。但是它有下面三个明显的缺点：

- **对 CPU 资源敏感；**
- **无法处理浮动垃圾；**
- **它使用的回收算法-“标记-清除”算法会导致收集结束时会有大量空间碎片产生。**

CMS 垃圾回收器在 Java 9 中已经被标记为过时(deprecated)，并在 Java 14 中被移除。

### `G1` 收集器

> server

<u>从 JDK9 开始，G1 垃圾收集器成为了默认的垃圾收集器。</u>

G1 (Garbage-First) 是一款<u>面向服务器</u>的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征.

- 每次只清理一部分,而不是清理全部的增量式清理,以保证停顿时间不会过长

- 其取消了年轻代与老年代的物理划分,但仍属于分代收集器,算法将堆分为若干个逻辑区域(region),一部分用作年轻代,一部分用作老年代,还有用来存储巨型对象的分区. 

- 同CMS相同,会遍历所有对象,标记引用情况,清除对象后会对区域进行复制移动,以整合碎片空间.

- 年轻代回收: 并行复制采用复制算法,并行收集,会StopTheWorld.

  老年代回收: 会对年轻代一并回收

  

G1 收集器的运作大致分为以下几个步骤：

- **初始标记**:完成堆root对象的标记,会StopTheWorld
- **并发标记**: 并发标记 GC线程和应用线程并发执行
- **最终标记**:完成三色标记周期,会StopTheWorld
- **筛选回收**:会优先对可回收空间加大的区域进行回收

![](https://img2.imgtp.com/2024/05/31/fNi16T4c.png)



G1 收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字 Garbage-First 的由来) 。这种使用 Region 划分内存空间以及有优先级的区域回收方式，保证了 G1 收集器在有限时间内可以尽可能高的收集效率（把内存化整为零）。



### *ZGC*算法

前面提供的高效垃圾回收算法,针对大堆内存设计,可以处理TB级别的堆,可以做到10ms以下的回收停

顿时间.

![](https://img2.imgtp.com/2024/05/31/7mGAWtvT.png)

1.着色指针

2.读屏障

3.并发处理

4.基于region

5.内存压缩(整理) 

roots标记：标记root对象,会StopTheWorld. 并发标记：利用读屏障与应用线程一起运行标记,可能

会发生StopTheWorld. 清除会清理标记为不可用的对象. roots重定位：是对存活的对象进行移动,以

腾出大块内存空间,减少碎片产生.重定位最开始会StopTheWorld,却决于重定位集与对象总活动集的

比例. 并发重定位与并发标记类似. 