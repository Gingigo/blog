# lambda

### 什么是 lambda 表达式？

java 8 最大的特性就是引入 Lambda 表达式，lambda 表达式是一个可传递的代码块， 可以在以后执行一次或多次。

下面举个例子说明一下

我们输入一个学生看他·的身高是否超过178？

#### 普通版

新建 `Student` 实体类

```java
public class Student {
    private int id;
    private String name;
    private int height;

    public Student(int id, String name, int height) {
        this.id = id;
        this.name = name;
        this.height = height;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getHeight() {
        return height;
    }

    public void setHeight(int height) {
        this.height = height;
    }
    
    public static String getClassName(){
        return "高一(3)班";
    }
    
    public  String said(String str){
        return str+", 我要成为海贼王";
    }
}
```

创建`IPredicate` 类：

```java
import java.util.function.Predicate;

public class IPredicate implements Predicate<Integer> {
    @Override
    public boolean test(Integer integer) {
        return integer > 178;
    }
}
```

测试类代码：

```java
public class LambdaTest01 {
    public static void main(String[] args) {
        IPredicate predicate = new IPredicate();
        Student student = new Student(1, "tom", 180);
        System.out.println(
            student.getName() + "身高大于178吗？" 
            + predicate.test(student.getHeight())
        );
    }
}
```

测试结果：

```java
tom身高大于178吗？true
```

#### Lambda版

测试代码类：

```java
public class LambdaTest01 {
    public static void main(String[] args) {
		//IPredicate predicate = new IPredicate();
        //lambda 表达式
        Predicate<Integer> predicate = x -> x > 178;
        Student student = new Student(1, "tom", 180);
        System.out.println(
            student.getName() + "身高大于178吗？"
            + predicate.test(student.getHeight())
        );
    }
}
```

测试结果：

```shell
tom身高大于178吗？true
```

结果是一样的。

我们现在知道了Lambda表达式比普通的实现方式更加简单，而且代码更少，而且少了一个文件，主要归功于<font color='orange'>lambda 表达式是一个可传递的代码块</font>。

#### 如何定义个 lambda 表达式

**语法**

lambda 表达式的语法格式如下：

```java
(parameters) -> expression
或
(parameters) ->{ statements; }
```

以下是lambda表达式的重要特征:

- **可选类型声明：**不需要声明参数类型，编译器可以统一识别参数值。
- **可选的参数圆括号：**一个参数无需定义圆括号，但多个参数需要定义圆括号。
- **可选的大括号：**如果主体包含了一个语句，就不需要使用大括号。
- **可选的返回关键字：**如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指定明表达式返回了一个数值。

例如：

第一种：通用型

```java
(参数类型 参数1,参数类型 参数2)->{
    //对参数1和参数2操作
    //如果无需返回不用返回
    return null;
};

//如：计算x-y的值

(int x,int y)->{
    return x-y
}

//或者 不需要返回值,直接输出
(int x,int y)->{
    System.out.println(x-y);
}
```

第二种：主体是语句型（能用一句语句执行）

```java
(参数1,参数2)-> 参数1+参数2
    
//例如：计算 x+y的值并返回结果
(x,y)-> x+y
//或者 打印出 x+y 的结果
(x,y)->System.out.println(x+y)

```

第三种：无参数数

```java
()-> 值
//如返回5
()->5
()->{ return 5 } 
//打印 5
()-> System.out.println(5)
()->{ System.out.println(5) }
```

第四种：单个参数

```java
(参数)-> 操作参数  =  参数-> 操作参数;
或者
(参数) -> {操作参数} = 参数 -> {操作参数}

//x 的倍数

x -> 2*x;
x -> System.out.println(2*x);
```

现在我们已经知道如何定义 lambda 的语法了，那 lambda 可以用在那些地方呢？就是函数式接口！

### 函数式接口

认识了 lambda 表达式之后，还有一个重要的概念与 lambda 表达式是分不开的,"函数式接口"。他是 lambda 展示的舞台。

**函数式接口（  functional interface）**：<font color='orange'>对于只有一个抽象方法的接口</font>， 需要这种接口的对象时， 就可以提供一个 lambda 表达式。 使用@FunctionalInterface注解修饰的类，编译器会检测该类是否只有一个抽象方法或接口，否则，会报错。可以有多个默认方法，静态方法。

我们先看一下 `java.util.function.Predicate` 类:

```java
package java.util.function;

import java.util.Objects;

@FunctionalInterface
public interface Predicate<T> {
    
    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }

    @SuppressWarnings("unchecked")
    static <T> Predicate<T> not(Predicate<? super T> target) {
        Objects.requireNonNull(target);
        return (Predicate<T>)target.negate();
    }
}
```

该方法只有一个方法为实现`boolean test(T t);` 而且也有注解`@FunctionalInterface` (非必要，只是在编译的时候验证是否只有一个接口)，而其他方法不是静态方法 就是默认实现了（`defualt`关键字）。

我们创建一个接口来构建函数式接口

新建抽象接口 `Counter`

```java
/**
 * 计算器
 */
@FunctionalInterface
public interface  Counter {
    // 自定义算法
    public abstract int algorithm(int x,int y);
}
```

编写测试类

```java
public class CounterTest {
    public static void main(String[] args) {
        Counter myCounter = (x,y)-> x*y;
        System.out.println(myCounter.algorithm(5,2));
        myCounter = (x,y)-> x-y;
        System.out.println(myCounter.algorithm(5,2));
    }
}
```

结果

```java
10
3
```



### 变量的作用域

####   访问外围方法或类中的变量

先来看一个例子：

```java

import javax.swing.*;
import java.awt.event.ActionListener;

public class TimerTest {
    /**
     * 定时打印文字
     * @param text 需要打印的文字
     * @param delay 时间间隔 毫秒
     */
    public static void repeatMessage(String text, int delay) {
        ActionListener listener = event ->{
            System.out.println(text);
        };
        new Timer(delay,listener).start();
    }

    public static void main(String[] args) throws InterruptedException {
        TimerTest.repeatMessage("hello world",1000);
      
        while (true){
            Thread.sleep(1000);
        }
    }

}
```

这样写是没有问题的。假设 lambda 表达式的代码可能会在 `algorithm`调用返回很久以后才运行， 而那时这个参数变量已经不存在了, 。 如何保留 text变量呢?    

接下来我们加深一下对 lambda 表达式的理解。lambda 表达式有3个部分：

1.  一个代码块；
2. 参数；
3. 自由变量的值，  这是指非参数而且不在代码中定义的变量。

在我们的例子中， 这个 lambda 表达式有 1 个自由变量 text。 表示 lambda 表达式的数据结构必须存储自由变量的值， 在这里就是字符串 "hello world"。 我们说它被 lambda 表达式捕获。

> 其中代码块和自由变量值有个术语就是”闭包“ 

**lambda 表达式可以捕获外围作用域变量的值。在 Java 中，要确保捕获的值是明确定义的，这里有个重要的限制。lambda 表达式中，只能引用不会改变的变量。**

其实在我们提供的例子中，`String text` 因为被 lambda表达式引用，不能给`text` 赋值，将会编译不通过,报错。

**之所以有这个限制是有原因的。 如果在 lambda 表达式中改变变量， 并发执行多个动作时就会不安全。另外如果在 lambda 表达式中引用变量， 而这个变量可能在外部改变，这也是不合法的，可以认为被捕获的遍历是隐式加了 final 关键字。 **

再看一个例子

```java
public static void repeatMessage(String text, int delay) {
        System.out.println(text);
        String name = "tom";
        ActionListener listener = event ->{
            //不合法的，有相同的参数名
            String name = "tom";
            System.out.println(text);
        };
        new Timer(delay,listener).start();
    }
```

**另外如果在 lambda 表达式中引用变量， 而这个变量可能在外部改变，这也是不合法的。这里同样适用命名冲突和遮蔽的有关规则。在 lambda 表达式中声明与一个局部变量同名的参数或局部变量是不合法的**  

### 方法引用

有时， 可能已经有现成的方法可以完成你想要传递到其他代码的某个动作。  比如定义个函数传入 `Student` 实例，获取 `student` 的 `name`。我们知道我们定义 实体类`Student` 中有个 `getName`方法刚好满足。接下来我们用代码演示自己用 lambda 表达式实现获取用户名和用现有的方法`getName`传入lambda中：

```java
public class MethodReferenceTest {
    public static void main(String[] args) throws InterruptedException {

        Student gin  = new Student(2,"gin",177);

        //自己实现
        Function<Student,String> f1 = student -> student.getName();
        //引用现有的方法  (Class:instanceMethod)
        Function<Student,String> f2 =Student::getName;

        System.out.println(f1.apply(gin)); // gin
        System.out.println(f2.apply(gin)); // gin


        // 获取自我介绍 （Class::staticMethod）
        Function<String,String> f3 = Student::getAboutMyself;
        System.out.println(f3.apply("高一3班"));

        // (object::instanceMethod)
        Function<String,String> f4 = gin::said;
        System.out.println(f4.apply("路飞"));
        
    }
}
```

输入的结果是:

```shell
gin
gin
我是来着高一3班的学生
路飞, 我要成为海贼王
```

#### 方法引用语法

从上面的例子中我们总结出了三种方法引用的语法：

- object::instanceMethod
- Class::staticMethod
- Class::instanceMethod

在前 2 种情况中， 方法引用等价于提供方法参数的 lambda 表达式。前面已经提到，`Student::getName` 等价于 `student -> student.getName()`类似地，`Math::pow` 等价于`（x，y) -> Math.pow(x, y)  `

对于第3种情况，第一个参数会成为方法的目标。例如：`String::compareToIgnoreCase` 等同于 `(a,b)->a.compareToIgnoreCase(b)`

> 当然在方法引用使用 `this` 和 `super` 也是合法的，如：`this::equals` 等同于 `x -> this.equals(x)`



### 构造器引用

构造器的引用与方法的引用很类似，只不过方法名为`new` 。例如 : `Student::new` 是 `Student` 构造器的引用。

再来看一个例子：

为数组添加一个构造器引用。例如 `int[]::new` 它有一个参数，就是数组的长度。等同于 `x-> new intp[x]`





> 本文参考：
>
> 1.  Java 核心技术卷 Ⅰ（第10版)
> 2. [菜鸟教程]( https://www.runoob.com/java/java8-lambda-expressions.html )
> 3. [掘金文章]( https://juejin.im/post/5ce66801e51d455d850d3a4a#heading-2 )