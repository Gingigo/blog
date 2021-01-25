# Java 字节码

Java虚拟机不和包括java在内的任何语言绑定，它只与“Class”特定的二进制文件格式关联，Class文件中包含Java虚拟机指令集和符号表以及若干其他辅助信息。

<font color='orange'>Java跨平台的原因是JVM不跨平台</font>

## 查看

```java
public class ByteCodeTest1 {
    private int a = 1;

    public int getA() {
        return a;
    }

    public void setA(int a) {
        this.a = a;
    }
}
```

### `javap` 

- 执行 `javap ByteCodeTest1.class`

```java
Compiled from "ByteCodeTest1.java"
public class com.gin.jvm.bytecode.ByteCodeTest1 {
  public com.gin.jvm.bytecode.ByteCodeTest1();
  public int getA();
  public void setA(int);
}
```

- 执行 `javap -c ByteCodeTest1.class`

```java
Compiled from "ByteCodeTest1.java"
public class com.gin.jvm.bytecode.ByteCodeTest1 {
  public com.gin.jvm.bytecode.ByteCodeTest1();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: iconst_1
       6: putfield      #2                  // Field a:I
       9: return

  public int getA();
    Code:
       0: aload_0
       1: getfield      #2                  // Field a:I
       4: ireturn

  public void setA(int);
    Code:
       0: aload_0
       1: iload_1
       2: putfield      #2                  // Field a:I
       5: return
}
```

- 执行 `javap -c ByteCodeTest1.class`

```java
Classfile /E:/DevelopProject/Gin/intentional-learning/shengsiyuan-jvm/out/production/classes/com/gin/jvm/bytecode/ByteCodeTest1.class
  Last modified 2020-12-27; size 505 bytes
  MD5 checksum 67b2106ba18003b2a24da3d7e4959c1d
  Compiled from "ByteCodeTest1.java"
public class com.gin.jvm.bytecode.ByteCodeTest1
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#21         // com/gin/jvm/bytecode/ByteCodeTest1.a:I
   #3 = Class              #22            // com/gin/jvm/bytecode/ByteCodeTest1
   #4 = Class              #23            // java/lang/Object
   #5 = Utf8               a
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/gin/jvm/bytecode/ByteCodeTest1;
  #14 = Utf8               getA
  #15 = Utf8               ()I
  #16 = Utf8               setA
  #17 = Utf8               (I)V
  #18 = Utf8               SourceFile
  #19 = Utf8               ByteCodeTest1.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = NameAndType        #5:#6          // a:I
  #22 = Utf8               com/gin/jvm/bytecode/ByteCodeTest1
  #23 = Utf8               java/lang/Object
{
  public com.gin.jvm.bytecode.ByteCodeTest1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_1
         6: putfield      #2                  // Field a:I
         9: return
      LineNumberTable:
        line 3: 0
        line 4: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  this   Lcom/gin/jvm/bytecode/ByteCodeTest1;

  public int getA();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field a:I
         4: ireturn
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/gin/jvm/bytecode/ByteCodeTest1;

  public void setA(int);
    descriptor: (I)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: iload_1
         2: putfield      #2                  // Field a:I
         5: return
      LineNumberTable:
        line 11: 0
        line 12: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/gin/jvm/bytecode/ByteCodeTest1;
            0       6     1     a   I
}
SourceFile: "ByteCodeTest1.java"
```

### 十六进制文件

工具：

1. IDEA
2. [BinEd](https://plugins.jetbrains.com/plugin/9339-bined--binary-hexadecimal-editor)
3. [jclasslib](https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer)

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/05-01.png)

## 结构

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/Snipaste_2021-01-04_14-20-33.png)

1. 魔数：所有的`.class`文件的前四个字节都是魔数，魔数值为固定值：`0XCAFEBABE`(<font color='orange'>咖啡宝贝</font>)

2. 版本号：魔数后面4个字节是版本信息，前两个字节表示minor version（次版本号），后两个字节表示major version（主版本号），十六进制34=十进制52。所以该文件的版本号为1.8.0。低版本的编译器编译的字节码可以在高版本的JVM下运行，反过来则不行。

3. 常量池（constant pool）：版本号之后的就是常量池入口，一个java类定义的很多信息都是由常量池来维护和描述的，可以将常量池看作是class文件的资源仓库，包括java类定义的方法和变量信息，常量池中主要存储两类常量：字面量和符号引用。字面量如文本字符串、java中生命的final常量值等，符号引用如类和接口的全局限定名，字段的名称和描述符，方法的名称和描述符等。

   1. 常量池的整体结构：Java类对应的常量池主要由常量池数量和常量池数组两部分共同构成，常量池数量紧跟在主版本号后面，占据两个字节，而常量池数组在常量池数量之后。

   2. 常量池数组与一般数组不同的是，常量池数组中元素的类型、结构都是不同的，长度当然也就不同，但是每一种元素的第一个数据都是一个u1类型标志位，占据一个字节，JVM在解析常量池时，就会根据这个u1类型的来获取对应的元素的具体类型。

   3.  常量池数组中元素的个数=常量池数-1,（其中0暂时不使用）。目的是满足某些常量池索引值的数据在特定的情况下需要表达不引用任何常量池的含义。根本原因在于索引为0也是一个常量，它是JVM的保留常量，它不位于常量表中。<font color='orange'>这个常量就对应null</font>，所以常量池的索引从1而非0开始。

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/pool.png)
   
4. 在JVM规范中，每个变量/字段都有描述信息，主要的作用是描述字段的数据类型，方法的参数列表（包括数量、类型和顺序）与返回值。

   - 根据描述符规则，基本数据类型和代表无返回值的void类型都用一个大写字符来表示，而对象类型使用字符L+对象的全限定名称来表示。为了压缩字节码文件的体积，对于基本数据类型，JVM都只使用一个大写字母来表示。如下所示:B - byte，C - char，D - double，F - float，I - int，J -l ong，S -short，Z - boolean，V - void，L-对象类型，如Ljava/lang/String;
     对于数组类型来说，每一个维度使用一个前置的[ 来表示，如int[]表示为[I ，String [][]被记录为[[Ljava/lang/String;

5. 用描述符描述方法的时候，用先参数列表后返回值的方式来描述。参数列表按照参数的严格顺序放在一组（）之内，如方法`String getNameByID(int id ,String name)`   表示为： `(I,Ljava/lang/String;)Ljava/lang/String;`

### 字节码数据结构

1. 字节数据直接量：这是基本的数据类型。共细分为u1、u2、u4、u8四种，分别代表连续的1个字节、2个字节、4个字节、8个字节组成的整体数据。
2. 表/数组：表是由多个基本数据或其他表，按照既定顺序组成的大的数据集合。表是有结构的，它的结构体：组成表的成分所在的位置和顺序都是已经严格定义好的。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/re.png)

- `Access Falgs`：访问标志信息包括了该class文件是类还是接口，是否被定义成public，是否是abstract，如果是类，是否被定义成final。`access_flags`,如果为`0x0021`是`0x0020`和`0x0001`的并集,表示ACC_PUBLIC和ACC_SUPER

- `Fields`,字段表用于描述类和接口中声明的变量。这里的字段包含了类级别变量和实例变量，但是不包括方法内部声明的局部变量。

  ​	![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/fieled.png)

- `methods`方法表,方法的属性结构：方法中的每个属性都是一个attribute_info结构:

  - JVM预定义了部分attribute，但是编译器自己也可以实现自己的attribute写入class文件里，供运行时使用；
  - 不同的attribute通过attribute_name_index来区分。

- attribute_info格式:

  ```
  attribute_info{
  u2 attribute_name_index;
  u4 attribute_length;
  u1 info[attribute_length]
  }
  ```

- attribute_name_index值为code，则为Code结构:Code的作用是保存该方法的结构，所对应的的字节码

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/dss.png)  
  
  - attribute_length：表示attribute所包含的字节数，不包含attribute_name_index和attribute_length
  - max_stacks：表示这个方法运行的任何时刻所能达到的操作数栈的最大深度
  - max_locals：表示方法执行期间创建的局部变量的数目，包含用来表示传入的参数的局部变量
  - code_length：表示该方法所包含的字节码的字节数以及具体的指令码。具体的字节码是指该方法被调用时，虚拟机所执行的字节码
  - exception_table：存放处理异常的信息，每个exception_table表，是由start_pc、end_pc、hangder_pc、catch_type组成
  - start_pc、end_pc：表示在code数组中从start_pc到end_pc（包含start_pc，不包含end_pc）的指令抛出的异常会由这个表项来处理
  - hangder_pc：表示处理异常的代码的开始处。
  - catch_type：表示会被处理的异常类型，它指向常量池中的一个异常类。当catch_type=0时，表示处理所有的异常。
  
- 附加其他属性（LineNumbeTable_attribute）：这个属性表示code数组中，字节码与java代码行数之间的关系，可以在调试的时候定位代码执行的行数

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/rew.png)

- LocalVariableTable ：结构类似于 LineNumbeTable_attribute,<font color='orange'>对于Java中的任何一个非静态方法，至少会有一个局部变量，就是this。</font>

#### synchronized

- 显式

  ```java
  public void test(String str) {
          synchronized (this) {//给当前对象上锁
              System.out.println("Hello World");
          }
  }
  ```

  字节码：

  ```class
   0 aload_0
   1 dup
   2 astore_2
   3 monitorenter
   4 getstatic #10 <java/lang/System.out>
   7 ldc #11 <Hello World>
   9 invokevirtual #12 <java/io/PrintStream.println>
  12 aload_2
  13 monitorexit
  14 goto 22 (+8)
  17 astore_3
  18 aload_2
  19 monitorexit
  20 aload_3
  21 athrow
  22 return
  ```

- 隐式

  ```java
  public static synchronized void test() {
  }
  ```

  字节码:访问标识

  ```class
  0x0029 [public static synchronized]
  ```

#### 构造方法

- `<init>`,实例初始化的方法，如果一个类中有 n 个 构造器，就有 n 个 `<init>`方法对应，每个`<init>`方法都会初始化实例变量的值和自身局部变量的值。
- `<clinit>`，静态变量和静态代码快初始化的地方，一个类中最多只有一个`clinit` 方法

#### this

- 对于Java类中的每一个实例方法（非static方法），其在编译后所生成的字节码当中，方法参数的数量总是会比源代码中方法参数的数量多一个（this）
- 它位于方法的第一个参数位置处；这样我们就可以在Java的实例方法中使用 this 来访问当前对象的属性以及其他方法。
- 这个操作式在编译期间完成的，即由 javac 编译器在编译的时候将对 this 的访问转化为对于一个普通实例方法参数的访问；
- 接下来在运行期间由JVM在调用实例方法时,自动向实例方法传入this参数.所以,在实例方法的局部变量表中,至少会有一个指向当前对象的局部变量

#### 异常表

Java 字节码对于异常的处理方式：

- 统一采用异常表的方式来对异常进行处理；
- 在 jdk1.4.2之前的版本中，并不是使用异常表的方式对异常进行处理的，而是采用特定的指令方式；
- 当异常处理存在finally语句块时，现代化的JVM采取的处理方式是将finally语句内的字节码拼接到每个catch语句块后面。也是就说，程序中存在多少个 catch，就存在多少个 finally 块的内容



## 问题

1、递归如何计算栈的深度   

2、静态分派和动态分派











