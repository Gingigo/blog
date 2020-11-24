# 02、synchronized 和 volatile

- 关键字<font color='orange'>synchronized</font> 可用来保障原子性、可见性和有序性。
- 非线程安全问题会在多个线程对同一个对象中的<font color='orange'>实例变量</font> 进行并发访问时发生，产生的后果就是“脏读”。
-  线程安全是指获得实例变量的值是经过同步处理的，不会出现脏读的现象。
-  原子性， 对共享内存的操作必须是要么全部执行直到执行结束，且中间过程不能被任何外部因素打断，要么就不执行 。
-  可见性，多线程操作共享内存时，执行结果能够及时的同步到共享内存，确保其他线程对此结果及时可见。
-  有序性， 程序的执行顺序按照代码顺序执行，在单线程环境下，程序的执行都是有序的，但是在多线程环境下，JMM 为了性能优化，编译器和处理器会对指令进行重排，程序的执行会变成无序。

## synchonized

- 方法内部变量，线程安全。非线程安全问题存在于<font color='orange'>实例变量</font>中，对于方法内部的私有变量，则不存在非线程安全问题，结果是“线程安全”的。
- 在多线程环境下，多个线程同时操作操作实例变量，可能会出现线程不安全。如果多个线程共同访问一个对象中的实例变量，则有可能出现非线程安全问题。

### 如何解决线程不安全问题

synchonized， 线程进入用synchronized声明的方法时就上锁，方法执行完成后自动解锁 ， 之后下一个线程才会进入用synchronized声明的方法里，不解锁其他线程执行不了用synchronized声明的方法。

### 原理

- 显式同步 ： 同步代码块代表着显式同步，指的是有明确的<font color='orange'>monitorenter</font>和<font color='orange'>monitorexit</font>指令；
- 隐式同步： 同步方法代表着隐式同步，同步方法是由方法调用指令读取运行时常量池中方法的<font color='orange'>ACC_SYNCHRONIZED</font>标志来隐式实现的 。

```java
public class Test {
    synchronized public static void testMethod() {
    }
    public static void main(String[] args) throws InterruptedException {
        testMethod();
    }
}
```

执行：

```shell
> javac Test.java
> javap -c -v Test.class
```

得到关键字节码：

```class
  public static synchronized void testMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 5: 0
```

在反编译的字节码指令中，对public synchronized void myMethod()方法使用了flag标记ACC_SYNCHRONIZED，说明此方法是同步的。 

```java
public class Test2 {
    public void myMethod() {
        synchronized (this) {
            int age = 100;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Test2 test = new Test2();
        test.myMethod();
    }
}
```

执行：

```shell
> javac Test2.java
> javap -c -v Test2.class
```

得到关键字节码：

```class
 public void myMethod();           
   descriptor: ()V                 
   flags: ACC_PUBLIC               
   Code:                           
     stack=2, locals=4, args_size=1
        0: aload_0                 
        1: dup                     
        2: astore_1                
        3: monitorenter            
        4: bipush        100       
        6: istore_2                
        7: aload_1                 
        8: monitorexit             
        9: goto          17        
       12: astore_3                
       13: aload_1                 
       14: monitorexit             
       15: aload_3                 
       16: athrow                  
       17: return                  
     Exception table:              
        from    to  target type    
            4     9    12   any    
           12    15    12   any    
```

字节码中使用monitorenter和monitorexit指令进行同步处理。 

### sychronized 用法

1. 实例方法（也就是普通方法）加上关键字
2. 静态方法加上关键字
3. 方法中使用同步代码块

### sychronized 的范围

需要先理解的是<font color='orange'> 在方法声明处添加synchronized并不是锁方法，而是锁当前类的对象 </font>

- 实例方法（也就是普通方法）加上关键字 
  - 锁住的就是这个对象实例；
    - 类 A 有对象 a1，其中有一个实例方法 f() 加了 sychronized 关键字 ,线程 T1 访问 f(),就代表锁住了对象 a1。
  - 访问加 synchronized 关键字的<font color='orange'>实例方法</font>都受阻，但是可以访问其他方法和类方法。
    - 类 A 有对象 a1，其中有一个实例方法 f() 加了 sychronized 关键字 ，f1() ,线程 T1 访问 f(),就代表锁住了对象 a1,但是线程 T2 可以正常访问其他未加 sychronized 的方法或者类方法。
  - 多个对象实例多个锁；
    - 类 A 有对象 a1 和 a2 ,线程 T1 锁住了 a1 ,线程 T2 锁住了 a2 ,线程 T1 和 T2 互不干扰。
- 静态方法(类方法)加上关键字
  - 锁住的就是这个类
    - 类 A ，其中有一个static f() 加了 sychronized 关键字 ,线程 T1 访问 f(),就代表锁住了类 A 。
  - 只对加了sychronized 的<font color='orange'>类方法</font>互斥
    - 类 A ，其中有一个static f() 加了 sychronized 关键字和方法 static  f1() ,线程 T1 访问 f(),就代表锁住了类 A ,但是线程 T2 可以正常范围**未加关键字的类方法f1()**。
    - 类 A ，其中有一个static f() 加了 sychronized 关键字和方法 f1() ,线程 T1 访问 f(),就代表锁住了类 A ,但是线程 T2 可以正常范围**实例方法f1()**。
    - 类 A ，其中有一个static f() 加了 sychronized 关键字和方法 f1() ,线程 T1 访问 f(),就代表锁住了类 A ,但是线程 T2 可以正常范围**加了 sychronized的 实例方法f1()**。

- 同步代码块
  - 锁住的是 sychronized(锁对象)，括号里面的对象。可以是 对象或者this、xx.class，锁住的范围和括号里面的有关系。
  - 如果锁住的是对象 this，访问该对象加了sychronized的实例方法会阻塞，普通方法和类方法都不会阻塞。
  - 如果锁住的是 Class 对象，访问该类，加了sychronized的类方法会阻塞，实例方法和不加sychronized的方法不会阻塞
  - 同步代码块能够更小的控制同步区域，效果会更好。

总结：

1. 加载方法上的 sychronized 锁住的是对象，实例方法 ->锁住-> 实例对象,类方法  ->锁住-> 类对象（Class）。
2. 实例对象、类对象锁资源之间是互不干扰的，即 类方法的 sychronized 同和实例方法的 sychronized  不互斥。
3. 同步代码块锁住的是,sychronized(锁对象)，括号里面的对象。

### 特性

- sychronized 是可重入锁
  - 可重入锁“是指自己可以再次获取自己的内部锁。锁重入支持继承的环境。
- 出现异常，自动释放锁资源
  - 锁住的资源对象如果抛出异常，则会释放锁资源

## volatile

- 可见性： B线程能马上看到A线程更改的数据。 
-  禁止代码重排序。

**可见性**

- 加了 volatile 修饰的变量，数据的更改能被各个线程察觉。主要的原因是线程会强制读取写到公共堆栈中，这样所有线程公用一个地址中的数据，以实现可见性。

**禁止代码重排序**

- 加了volatile 修饰的变量，就像代码中加了屏障，屏障之前的代码不可以被重排序优化到屏障之后。

  例子：

```
A变量的操作
B变量的操作
volatile Z变量的操作
C变量的操作
D变量的操作
```

那么会有4种情况发生：

1. A、B可以重排序。
2. C、D可以重排序。
3. A、B不可以重排到Z的后面。
4. C、D不可以重排到Z的前面。

 这就是屏障的作用，关键字synchronized具有同样的特性。关键字synchronized之前的代码不可以重排到synchronized之后； 之后的代码不可以重排到synchronized之前。

## 场景

- **当想实现一个变量的值被更改时，让其他线程能取到最新的值时，就要对变量使用volatile。**
-  **当多个线程对同一个对象中的同一个实例变量进行操作时，为了避免出现非线程安全问题，就要使用synchronized。** 











