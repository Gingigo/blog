# JVM 助记符

## 查看字节码

### javap

javap是jdk自带的反解析工具。它的作用就是根据class字节码文件，反解析出当前类对应的code区（汇编指令）、本地变量表、异常表和代码行偏移量映射表、常量池等等信息。

#### 用法

```shell
javap <options> <classes>
```

```shell
-help  --help  -?        	输出此用法消息
 -version                 	版本信息，其实是当前javap所在jdk的版本信息，不是class在哪个jdk下生成的。
 -v  -verbose             	输出附加信息（包括行号、本地变量表，反汇编等详细信息）
 -l                         输出行号和本地变量表
 -public                    仅显示公共类和成员
 -protected               	显示受保护的/公共类和成员
 -package                 	显示程序包/受保护的/公共类 和成员 (默认)
 -p  -private             	显示所有类和成员
 -c                       	对代码进行反汇编
 -s                       	输出内部类型签名
 -sysinfo                 	显示正在处理的类的系统信息 (路径, 大小, 日期, MD5 散列)
 -constants               	显示静态最终常量
 -classpath <path>        	指定查找用户类文件的位置
 -bootclasspath <path>    	覆盖引导类文件的位置
```

一般常用的是-v -l -c三个选项。

`javap -v classxx`，不仅会输出行号、本地变量表信息、反编译汇编代码，还会输出当前类用到的常量池等信息

` javap -l` 会输出行号和本地变量表信息

`javap -c` 会对当前class字节码进行反编译生成汇编代码

## 助记符查询

- [官方字节码字典](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)

## 常用助记符

- `ldc`：表示将int、float或者String类型的常量值从常量池中推送至栈顶
- `bipush`：表示将单字节（-128-127）的常量值推送到栈顶
- `sipush`：表示将一个短整型值（-32768-32369）推送至栈顶
- `iconst_1`：表示将int型的1推送至栈顶（iconst_m1到iconst_5）
- `anewarray`：表示创建一个引用类型（如类、接口）的数组，并将其引用值压入栈顶
- `newarray`：表示创建一个指定原始类型（int boolean float double）的数组，并将其引用值压入栈顶



