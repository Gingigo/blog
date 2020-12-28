# Java 字节码

Java虚拟机不和包括java在内的任何语言绑定，它只与“Class”特定的二进制文件格式关联，Class文件中包含Java虚拟机指令集和符号表以及若干其他辅助信息。

<font color='orange'>Java跨平台的原因是JVM不跨平台</font>

## 字节码

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



























