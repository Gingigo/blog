# Java虚拟机内存模型

## 运行时数据区域

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/01/blog_jvm_03.png)

JVM 运行时数据区分为**程序计数器、Java虚拟机栈、本地方法栈、Java堆、方法区**五大部分。



### 程序计数器

程序计数器( Program Counter Register )是线程私有的的一块内存空间，用于记录当前线程执行的字节码行号指示器。

- 线程私有（独享）
- 记录下一次执行字节码指令的位置

> 如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址;
>
> 如果正在执行的是本地（Native）方法，这个计数器值则应为空（Undefined）。



### Java 虚拟机栈

Java 虚拟机栈（ Java Virtual Machine Stack ）是线程私有的，生命周期与线程相同。其作用是方法执行的时候，生成一个栈帧，用于存储局部变量、操作数栈、动态链接、方法出口等信息。

- 线程私有；
- 用于执行方法时存储数据；
- 在执行方法时申请栈的大小是确定的，在运行方法期间大小不能改变的；
- 超出栈的大小，会报  `OutOfMemeoryError` 异常

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/01/blog_jvm_04.png)



### 本地方法栈

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别只是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的本地（Native）方法服务，线程私有。

- 线程私有；
- 服务的是本地（Native）方法。

### Java 堆

Java堆（Java Heap）是虚拟机管理内存中最大的一块，被所有线程共享。目的是为了存放对象实例和数组数据，“几乎”所有的对象实例都是在这里分配。

- 线程共享；
- JVM 管理中内存最大；
- 几乎存放所有的对象实例；
- 可以处于物理上不连续的内存空间中，但在逻辑上他应该是连续的

### 方法区

**方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。虽然《Java虚拟机规范》中把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫作“非堆”（Non-Heap），目的是与Java堆区分开来。

- 线程共享；
- 主要存储一些不变的数据和即时编译的数据；

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/01/blog_jvm_05.png)



#### 运行时常量池

**运行时常量池（Runtime Constant Pool）是方法区的一部分**。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），**用于存放编译期生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中**

运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，**Java语言并不要求常量一定只有编译期才能产生**，也就是说，并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可以将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern()方法。

既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。

- 类加载后用于存放编译期生成的各种字面量与符号引用；
- 运行期也可以存入常量，如String类的intern()方法；
- 受到申请内存限制。

### 直接内存

**直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是《Java虚拟机规范》中定义的内存区域**。但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现。

在JDK 1.4中新加入了NIO（New Input/Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，则肯定还是会受到本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间的限制，一般服务器管理员配置虚拟机参数时，会根据实际内存去设置-Xmx等参数信息，但经常忽略掉直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常。

- 非JVM 规范，但是经常使用；
- 能够提升性能，因为避免了在Java堆和Native堆中来回复制数据；
- 受到本机总内存限制，超出会抛出OutOfMemoryError异常

 **综上所述程序计数器、Java虚拟机栈、本地方法栈是线程私有的，即每个线程都拥有各自的程序计数器、Java虚拟机栈、本地方法区。并且他们的生命周期和所属的线程一样。而堆、方法区是线程共享的，在Java虚拟机中只有一个堆、一个方法栈。并在JVM启动的时候就创建，JVM停止才销毁** 

## 问题

1. 为什么会出现内存泄漏和溢出方面的问题？

   - 内存溢出 out of memory，是指程序在申请内存时，没有足够的内存空间供其使用，出现out of memory；比如申请了一个integer,但给它存了long才能存下的数，那就是内存溢出。

   - 内存泄露 memory leak，是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光

     >  https://www.cnblogs.com/Sharley/p/5285045.html 



> 参考：
>
> 1. 深入理解Java虚拟机：JVM高级特性与最佳实践（第三版）
> 2.  https://www.cnblogs.com/Sharley/p/5285045.html 
> 3.  https://www.cnblogs.com/cjsblog/p/9850300.html 
> 4.  https://blog.csdn.net/yohohaha/article/details/89378915 
> 5. https://blog.csdn.net/u011635492/article/details/81046174
> 6.  https://blog.csdn.net/truelove12358/article/details/100531911 

