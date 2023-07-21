---
title: JVM
tags: JVM
---

# 类加载

### 获取ClassLoader的方法

1. 获取当前类的ClassLoader：clazz.getClassLoader()
2. 获取当前线程上下文的ClassLoader：Thread.currentThread().getContextClassLoader()
3. 获取系统的ClassLoader：ClassLoader.getSystemClassLoader()
4. 获取调用者的ClassLoader：DriverManager.getCallerClassLoader()

# 第二章

## 2.1 运行时数据区

![image-20200312171046973](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200312171046973.png)

### 2.1.1 Java堆

线程共享，存放对象实例，是JVM中最大的一块，由新生代和老年代组成，新生代又分为Eden区、from Survivor、to Survivor。

### 2.1.2 方法区

线程共享，存储已被JVM加载的类信息、常量、静态变量、JIT编译器编译后的代码等数据

会抛出OOM异常

#### 运行时常量池

存放编译期生成的各种字面量、符号引用和翻译出来的直接引用

具备动态性：编译期和运行时的常量都可以放入

### 2.1.3 程序计数器(PC)

线程私有：为了能够准确记录各线程正在执行的字节码指令地址，是当前线程所执行的字节码的行号指示器。

如果线程正在执行Java方法，那么PC记录的是正在执行的虚拟机字节码指令的地址；
如果正在执行Native方法，那么值为空

PC是唯一一个没有OutOfMemoryError的区域

### 2.1.4 虚拟机栈

线程私有，一个方法对应一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息。

JVM规范允许虚拟机栈的大小是动态或者固定不变的：

1. 固定：如果线程请求分配的栈容量超过虚拟机所允许的最大容量，会抛StackOverflowError异常
2. 动态扩展：如果扩展时无法申请到足够的内存，会抛OutOfMemoryError异常

-Xss1024K：设置栈内存大小

### 2.1.5 本地方法栈

与虚拟机栈的作用类似，为虚拟机执行本地方法服务

# 三、垃圾收集器与内存分配策略

## 3.2 对象存活判断

### 引用计数算法

每个对象都有一个引用计数器，新增一个引用时加1，引用释放时减1。实现简单，高效，但是无法解决对象之间循环引用的问题。比如：对象A和B互相引用，引用计数不为0，但是其他地方没有引用这两个对象，导致无法通知GC收集器回收它们

### 可达性分析算法

从GC Roots开始向下搜索，搜索走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时，则此对象是不可用的。

GC Roots包括：

- 方法区中静态属性和常量引用的对象
- 虚拟机栈中引用的对象
- 本地方法栈中JNI引用的对象

### 再谈引用

- 强引用类似Object o=new Object()，垃圾收集器永远不会回收被引用的对象
- 软引用用来描述一些还有用但非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常
  适用场景：缓存。内存足够时将数据存到内存，不够时释放
- 弱引用强度比软引用更弱一些，被弱引用关联的对象只能存活到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。
- 虚引用是最弱的一种引用关系。虚引用不会决定对象的生命周期，唯一目的就是能在对象被回收时收到一个系统通知

## 3.3 垃圾收集算法

### 复制算法

将可用内存分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另一块上，然后再把已使用过的内存空间一次清理掉。

这样使得每次都是对整个半区进行内存回收，内存分配时也不用考虑内存碎片等复杂情况，实现简单，运行高效。代价是将内存缩小为了原来的一半，代价太高

现在的商业虚拟机都采用复制算法回收新生代。将内存划分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。当回收时，将Eden和Survivor中还存活着的对象一次性复制到另外一块Survivor中，最后清理掉Eden和刚才用过的Survivor空间。

### 标记-清除算法

最基础的收集算法，分为"标记"和"清除"两个阶段：首先标记出所有需要回收的对象，标记完成后统一回收。

主要缺点有两个：一个是效率低；另一个是空间问题，标记清除之后会产生大量不连续的内存碎片，碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作

### 标记-整理算法

复制算法在对象存活率较高时需要进行较多的复制操作，效率会变低。如果不想浪费50%的空间，就需要额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接采用复制算法。

根据老年代的特点，提出了标记-整理算法：首先标记出所有需要回收的对象，然后让所有存活的对象都向一端移动，最后清理掉端外面的内存

### 分代收集算法

根据对象存活周期将内存划分为几块，一般把堆分为老年代和新生代，这样就可以根据各个年代的特点选择最适当的收集算法。在新生代中，每次垃圾收集时只有少量对象存活，那就选用复制算法。而老年代中因为对象存活率高、没有额外空间分配担保，就必须使用"标记-清除"或"标记-整理"算法

## 3.5 垃圾收集器

新生代都采用**复制算法**

![image-20200413180640423](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200413180640423.png)

### Serial收集器

进行GC时，必须暂停所有的工作线程。单线程新生代收集器，复制算法

### ParNew收集器

是Serial收集器的多线程版本，只有它和Serial能和CMS搭配，复制算法

### Parallel Scavenge

多线程新生代收集器，目标是达到一个可控制的吞吐量，复制算法

### Serial Old

Serial收集器的老年代版本，单线程收集器，使用"标记-整理"算法

### Parallel Old

使用"标记-整理"算法的多线程老年代收集器

### CMS

使用"标记-清除"算法的老年代收集器，目标是获取最短回收停顿时间，只能和Serial、ParNew配合工作。

-XX:+UseConcMarkSweepGC

四个步骤：

- 初始标记(STW)：标记GC Roots能直接关联的对象，速度很快
- 并发标记：是进行GC Roots Tracing的过程
- 重新标记(STW)：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分的标记记录，停顿时间一般会比初始标记稍长，但远比并发标记时间短
- 并发清除

由于整个过程中最耗时的并发标记和并发清除都可以和用户线程一起工作，所以从总体上说，CMS的内存回收过程是和用户线程一起并发执行的。

**优点**：并发收集，停顿时间短

**缺点：**

1. 对CPU资源非常敏感，并发阶段会降低吞吐量
2. 无法处理浮动垃圾，浮动垃圾是指并发清除阶段由于用户线程继续运行而产生的垃圾，这部分垃圾只能到下一次 GC 时才能进行回收。
3. 会产生大量的空间碎片，当剩余内存不足时，会出现Concurrent Mode Failure，临时CMS会采用Serial Old，性能降低

### G1

步骤：

1. 初始标记(STW)
2. 并发标记
3. 最终标记(STW)：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分的标记记录，虚拟机将这段时间对象变化记录到Remembered Set Logs里面，最终标记阶段需要将Remember Set Logs的数据合并到Remember Set中
4. 筛选回收：对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划

 优点：

1. 采用"标记-整理"算法，不会产生大量的空间碎片
2. 分代收集
3. 能够建立可预测的停顿时间模型
4. 使用Remembered Set来避免全堆扫描。G1中每个Region都有一个Remembered Set，当虚拟机发现程序在对Reference类型进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于不同的Region之中，如果是，就通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set之中。当进行内存回收时，在GC Roots的枚举范围中加入Remembered Set即可保证不进行全堆扫描也不会有遗漏

G1将整个Java堆划分为多个大小相等的独立区域，新生代和老年代已经不是物理隔离了



## 3.6 内存分配与回收策略

**对象优先在Eden分配**

- Minor GC(新生代GC)：Eden没有足够空间时触发，发生在新生代，非常频繁，速度比较快
- Full/Major GC(老年代GC)：速度比Minor GC慢10倍以上
  触发条件：
  1）调用System.gc，系统建议，不必然执行
  2）老年代、方法区空间不足
  3）经过Minor GC后进入老年代的**平均大小**大于老年代的可用空间
  4）由Eden区、From Space区向To Space区复制时，对象大小>To Space可用内存，则把该对象转存到老年代，如果老年代的可用内存<该对象大小，触发

**大对象直接进入老年代**

因为新生代采取复制算法，该策略可以避免Eden区和两个Survivor区之间发生大量的内存复制。

-XX:PretenureSizeThreshold

**长期存活的对象将进入老年代**

虚拟机给每个对象定义了一个对象年龄计数器，如果对象在Eden出生并且经过第1次Minor GC后仍然存活，且能够进入Survivor，那么将被移到Survivor中，并且对象年龄设为1，之后每经过一次Minor GC对象年龄加1，达到阈值(默认15)就会晋升为老年代。阈值可通过-XX:MaxTenuringThreshold设置

**动态判断对象年龄**

如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，那么大于等于该年龄的对象直接进入老年代

**空间分配担保**

在Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代**所有对象**总空间，如果成立，那么Minor GC可以确保安全。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败，如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升老年代对象的**平均大小**，如果大于，将尝试Minor GC，尽管有风险。如果小于或者设置值不允许，那么就进行**Full GC**

### 内存溢出

指的是程序需要申请新的内存时，没有足够大小的内存空间供其使用。常见类型：

#### 堆溢出

java.lang.OutOfMemoryError: Java heap space

无法在堆中分配对象；有无法被GC回收的对象

#### 栈溢出

JVM规范允许虚拟机栈的大小是动态或者固定不变的：

1. 固定：如果线程请求分配的栈容量超过虚拟机所允许的最大容量，会抛StackOverflowError异常
2. 动态扩展：如果扩展时无法申请到足够的内存，会抛OutOfMemoryError异常

原因：栈帧过大或线程太多

解决：如果是OOM，检查是否有死循环；如果是StackOverflowError，检查是否存在递归调用

#### 方法区溢出

java.lang.OutOfMemoryError: PermGen space 

原因：使用CGLib生成了大量的代理类；大量jsp；应用长时间运行没有重启

解决办法：通过MaxPermSize参数设置大小

#### 排查OOM

https://www.jianshu.com/p/c6e2abb9f657

IDEA将堆设置为10M，-XX:+HeapDumpOnOutOfMemoryError表示发生OOM时生成dump快照

![image-20200728152216630](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200728152216630.png)

mat导入hprof文件，使用Leak Suspects功能

<img src="https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200728154343221.png" alt="image-20200728154343221" style="zoom: 80%;" />

点击Details--->查看累积点的最短路径，累积发生在线程0xffc27cc0中，

![image-20200805171550778](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200728153426492.png)

在dominator_tree中找到该线程，可以看到累积了176207个TreeNode

![image-20200805192523964](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200805171550778.png)

点击See stacktrace，查看发生错误的代码位置

![image-20200728153426492](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200805192523964.png)

### 内存泄露

不再使用的内存没有被回收

1. 静态集合类中引用的对象，如果不使用也不能够被回收

   ```java
   static List<Object> list = new ArrayList<>();
       
       public static void main(String[] args) {
           for(int i =0; i<10; i++){
               Object object = new Object();
               list.add(object);
               object = null;
           }
           
       }
   ```

2. 各种连接，数据库、网络、IO流。只有连接被关闭，垃圾回收器才会回收

3. 不合理的作用域

4. 单例模式，如果持有外部对象的引用，那么这个对象是无法回收的

**解决**

1. 避免在循环中创建对象
2. 尽量少用静态变量
3. 大量字符串处理不要用String

# 七、类加载机制

## 概念

类加载机制是将类的class文件读到内存中，将其放到方法区中，然后在堆中创建一个java.lang.Class对象，用来封装类在方法区内的数据结构

![image-20200401083259820](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200401083259820.png)

**类的生命周期**

![image-20200331111020021](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200331111020021.png)

加载、验证、准备、初始化、卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序开始（注意是开始，不是进行或完成，因为这些阶段通常是交叉混合进行），而解析不一定：在某种情况下可以在初始化之后再开始，这是为了支持Java的运行时绑定

## 类加载的过程

### 加载

此"加载"是"类加载"过程的一个阶段，在加载阶段，虚拟机需要完成3件事情：

1. 通过类的全限定名获取此类的二进制字节流
2. 将这个字节流所代表的**静态存储结构**转化为方法区的运行时数据结构
3. 在Java堆中生成一个代表这个类的java.lang.Class对象，作为方法区中这些数据的访问入口

### 验证

验证是连接阶段的第一步，目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全

### 准备

在**方法区**为**类变量**（static）分配内存并设置初始值

要说明两点：

1. 仅为**类变量**分配内存
2. 初始值是数据类型的零值，比如int是0，long是0L，boolean是false，引用是null

### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程

符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量

直接引用是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄

### 初始化

根据程序代码初始化**类变量**，是执行类构造器\<clinit>()方法的过程

- \<clinit>()由所有类变量的赋值和静态代码块static{}中的语句合并产生的，顺序就是代码顺序，static{}只能访问定义在其之前的变量，不能访问之后的变量，但是可以赋值
- 当一个类初始化时，要求其父类全部都已经初始化过了，但是接口没有这个要求，只有在真正使用到父接口的时候才会初始化

只有对一个类进行主动引用的时候，才会触发初始化，**有且只有**以下5种情况：

1. 遇到new、getstatic、putstatic、invokestatic这4条字节码指令时，对应的Java代码场景：使用new创建对象、读取或设置一个类的静态字段、调用类的静态方法
2. 反射
3. 当初始化一个类的时候，如果父类还没有初始化，则先触发类的父类的初始化
4. 虚拟机启动时，用户需要指定一个要执行的主类(包含main方法的)，虚拟机会先初始化这个主类
5. 当使用JDK1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果是REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且句柄对应的类还没有初始化，则需要先触发其初始化

   其余的情况都是被动引用，比如子类引用父类的静态字段、通过数组定义来引用类、使用类的常量，子类不会初始化	

### 卸载

- 程序正常执行结束
- 执行了System.exit()
- 程序执行过程中遇到了异常或错误而终止
- 由于操作系统出现错误而导致JVM进程终止

## 对象创建过程

<img src="https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200426144225308.png" alt="image-20200426144225308" style="zoom:80%;" />

**内存分配**

在堆中划分一块内存分配给对象。内存根据堆是否规整，有两种方式：

- 指针碰撞：如果堆是规整的，即用过的在一边，空闲的在另一边，那么就把中间的指针指示器向空闲的内存移动一段与对象大小相等的距离
- 空闲列表：如果堆不是规整的，则需要虚拟机维护一个列表记录哪些内存是可用的，在分配的时候选择足够大的内存分配给对象，并且更新记录

**处理并发问题**

- 对分配内存空间的动作做同步处理（采用 CAS + 失败重试来保障更新操作的原子性）
- 在堆中为每个线程预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer, TLAB），在线程对应的TLAB上分配内存，用完之后再加上同步处理。通过-XX:+/-UserTLAB参数来设定虚拟机是否使用TLAB

内存分配后，虚拟机将每个对象分配到的内存初始化为0值，

**对象的访问定位**

java程序需要通过栈上的引用访问堆中的具体对象。目前主流的访问方式有句柄和直接指针

**句柄**

在堆中创建句柄池，引用指向对象的句柄地址，句柄中包含了指向对象实例数据和对象类型数据的指针

对象被移动时只会改变句柄中的实例数据指针，引用本身不需要修改

<img src="https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200413173855311.png" alt="image-20200413173855311" style="zoom: 67%;" />

**指针**

指向对象实例数据的地址，速度更快，Hotspot采用。需要考虑如何放置访问类型数据的相关信息

<img src="https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200413174129529.png" alt="image-20200413174129529" style="zoom:67%;" />

## 类加载器

从Java虚拟机的角度来说，只存在两种类加载器：一种是启动类加载器，用C++实现，是虚拟机自身的一部分；另一种是所有其他的类加载器，由Java语言实现，独立于虚拟机外部

从Java开发人员的角度来看，类加载器可以分为3类：

**启动类加载器：**加载JDK\jre\lib目录中的或被-Xbootclasspath参数所指定的路径中的，并且能被虚拟机识别的类库(所有java.开头的)。无法被Java程序直接引用

**扩展类加载器：**加载JDK\jre\lib\ext目录中的或者被java.ext.dirs系统变量所指定的路径中的所有类库。开发者可以直接使用

**应用程序类加载器：**也叫**系统类加载器**，加载用户类路径上所指定的类库，开发者可以直接使用。如果没有自定义自己的类加载器，一般情况下就是程序中默认的类加载器

![image-20200331210926533](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200331210926533.png)

上图展示的类加载器之间的层次关系称为类加载器的**双亲委派模型**。

工作过程：如果一个类加载器收到了类加载请求，它首先不会自己去尝试加载，而是把这个请求委托给父类加载器完成，依次向上，只有当父类加载器反馈自己无法完成这个加载请求时，子类才会尝试自己加载

类加载器之间的父子关系是通过组合实现的。对于任意一个类，都需要由加载它的**类加载器**和这个**类本身**一同确立在 JVM 中的唯一性，每一个类加载器，都有一个独立的类名称空间。

**好处**

- 防止内存中出现多个不同的系统类
- 保证Java程序稳定运行

重写loadClass()方法打破双亲委派模型

# 十三、线程安全与锁优化

## 锁优化

https://www.jianshu.com/p/e62fa839aa41

###  java对象头

synchronized的锁存在对象头中。占用两个字宽，如果对象是数组，占用三个字宽。32位虚拟机中，一个字宽是4个字节，32bit；64位虚拟机中，一个字宽是8个字节，也就是64bit

Class Metadata Address指向对象所属类的地址

![1563762528309](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/1563762528309.png)

​																			 对象头

![](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/20190602201640883.png)

​																	 Mark Word存储结构

**自旋锁**

让线程执行忙循环（自旋）。虽然避免了线程切换的开销，但是要占用处理器时间，所以占用时间短，效果非常好，反之只会白白浪费cpu资源。适用于锁保护的临界区很小的情况，超过一定次数就应该被挂起。-XX:+UseSpinning开启，可以使用-XX:PreBlockSpin更改次数，默认为10

**自适应自旋锁**

自旋的时间不固定，由前一次在同一个锁上的自旋时间和锁的拥有者的状态决定

**锁清除**

虚拟机JIT编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行清除

**锁粗化**

JVM检测到有一系列的对同一个对象加锁的操作，那么会把加锁同步的范围扩展到整个操作序列的外部

## 锁转化

<font color="red">非常好的锁转化图https://upload-images.jianshu.io/upload_images/4491294-e3bcefb2bacea224.png</font>

![1563765271797](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/1563765271797.png)

![img](https://upload-images.jianshu.io/upload_images/4491294-e3bcefb2bacea224.png)

**偏向锁**

目的是消除数据在无竞争情况下的同步原语，是在单线程执行代码块时使用的机制，进一步提升单线程执行代码块的性能。在多线程并发的情况下一定会转为轻量级锁或重量级锁。JDK6默认开启，-XX:-UseBiasedLocking。

右下角：如果未退出同步代码块，则升级为轻量级锁

![image-20200412151246127](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20200413073746304.png)



1. 检查Mark Word是否为可偏向状态，即锁标志位"01"，偏向锁标志位"1"
2. 如果是，就判断Mark Word中的thread ID是否是当前线程，如果是，就执行同步块；如果不是当前线程或者偏向锁标志位为"0"，就尝试通过CAS操作将当前线程ID记录到Mark Word的thread ID中，如果成功，就执行同步块
3. 如果失败，证明存在多线程竞争，当到达全局安全点，获得偏向锁的线程被挂起，如果持有锁的线程未活动状态或者已经退出同步块，则恢复到无锁状态，以允许其他线程竞争，并唤醒持有锁的线程，否则升级为轻量级锁，

**轻量级锁**

https://blog.csdn.net/qicha3705/article/details/120494362

目的是在没有多线程竞争的前提下，减少传统的重量级锁因使用操作系统互斥量而产生的性能消耗。轻量级锁的过程：

1. 在线程进入同步块时，如果此同步对象没有被锁定，则在当前线程的栈帧中建立**锁记录**(Lock Record)，用于存储锁对象Mark Word的拷贝，官方称为Displaced Mard Word。
2. 虚拟机使用CAS操作尝试将锁对象Mark Word中锁记录指针指向Lock Record。如果成功，那么这个线程就拥有了该对象的锁，并且对象的Mark Word的锁标志位变为"00"，同时Lock Record里的owner指向对象的Mark Word
3. 如果失败了，虚拟机首先会检查锁对象Mark Word是否已经指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，直接进入同步块执行，否则说明这个锁对象已经被抢占了，进入自旋执行第2步，如果还是失败，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为"10"，Mark Word中存储的就是指向重量级锁的指针，后面等待锁的线程也要进入阻塞状态

![image-20230219212926137](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20230219212926137.png)

轻量级锁的释放也是通过CAS来完成：

1. 通过CAS操作尝试把对象当前的Mark Word替换回Displaced Mark Word，如果成功，整个同步过程就完成了
2. 如果失败，说明有其他线程尝试获取锁，那么在释放锁的同时要唤醒被挂起的线程

![image-20200413073746304](https://raw.githubusercontent.com/zheyday/BlogImg/master/img/image-20200412151246127.png)

对于轻量级锁，其性能提升的依据是 **“对于绝大部分的锁，在整个生命周期内都是不会存在竞争的”**，如果打破这个依据则除了互斥的开销外，还有额外的CAS操作，**因此在有多线程竞争的情况下，轻量级锁比重量级锁更慢**。



# 常用参数

**-Xms/-Xmx**：堆初始值/堆最大值

**-Xmn10M**：新生代大小

**-Xss**：设置栈

**-XX:PermSize=64m -XX:MaxPermSize=256m**：方法区初始值和最大值

**-XX:NewRatio**：老年代和新生代的比值

例如：4表示老年代：新生代=4:1

**-XX:SurvivorRation**：设置Eden区和两个Survivor区的比

例如：8表示eden：两个Survivor=8:2

**-XX:PrintGCDetails**：打印GC日志

# 常用命令

**jps**

列出所有JVM的pid

**jstat**

监视JVM各种运行状态信息，显示类装载、内存、GC、JIT编译等运行数据

命令格式：

jstat -\<option> \<pid> [\<interval> [\<count>]]

**jmap**

生成堆转储快照

 -heap pid：查看堆内存使用

**jhat**

JVM Heap Analysis Tool堆转储快照分析工具与jmap搭配使用

**jstack**

Java堆栈跟踪工具，用于生成虚拟机当前时刻的线程快照，也就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。目的是定位线程出现长时间停顿的原因

**jinfo** 

实时查看和调整虚拟机各项参数

-flags pid：查看正在运行的java程序的扩展参数

**jcmd**

 pid VM.flags：查看JVM的启动参数




