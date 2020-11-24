![](https://img.hacpai.com/bing/20181016.jpg?imageView2/1/w/960/h/540/interlace/1/q/100)

# Java中创建对象的4种方式

## 第一种：new

通过关键字**new** 用选择对应的构造方法申请新的对象赋值给句柄

```java
@Data
public class Hello {
    public String greet;

    public static void main(String[] args) {
        Hello hello = new Hello();
        hello.setGreet("hello world");
        System.out.println(hello.getGreet());
    }
}
```

## 第二种：反射   newInstance

1. **反射的Class类** 

   java反射时通过在运行期间主动的让jvm去加载指定的.class文件生成Class对象，从而获取到该类的属性与方法，在调用`newInstance()` 方法构造对象 。

   ```java
   @Data
   public class Hello2 {
       public String greet="hello";
       public static void main(String[] args) throws Exception {
           Hello2 hello2 = new Hello2();
           hello2.setGreet("hello world");
           System.out.println(hello2.getGreet()); // hello world
   
           Class<Hello2> hello2Class = (Class<Hello2>) hello2.getClass();
           Hello2 hello21 = hello2Class.newInstance();
           System.out.println(hello21.getGreet()); // hello 
       }
   }
   ```
   
   > `newInstance()`  调用的是类中的默认构造方法，所以要保证默认的构造方法必须存在否则会抛出 `java.lang.InstantiationException` 异常

2. **反射的Constructor类**

   其实反射Class类用`newInstance()`创建对象就是反射的Constructor类创建对象的特殊应用，而反射Constructor更加灵活可以自主选择合适的构造方法。

   ```java
   @Data
   @AllArgsConstructor
   public class Hello3 {
       public String greet="hello";
       public static void main(String[] args) throws Exception {
           
           Class<Hello3> hello3Class = Hello3.class;
           Hello3 hello3 = hello3Class
                   .getConstructor(String.class)
                   .newInstance("hello world");
           System.out.println(hello3.getGreet()); // hello world
       }
   }
   ```
   
   > 反射方式的补充:
   >
   > `Hello2` 和 `Hello3 `采用了两种不同的方式生成Class对象，分别采用了`(Class<Hello2>) hello2.getClass()`和 `Hello3.class `，其实我们还有一种更加常用的一种方式，就是在配置jdbc的驱动的时候  `Class.forName(类路径)`

## 第三种：Clone

克隆一个相同的对象，clone分为浅克隆和深克隆（全克隆），克隆的方式两个步骤，1、 实现`Cloneable`接口

2、重写`clone()` 。而深克隆需要对类的属性进行克隆，以此类推。

```java
@Data
public class Hello4 implements Cloneable {
    public String greet="hello";

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    public static void main(String[] args) throws Exception {
        Hello4 hello4 = new Hello4();
        hello4.setGreet("hello world");
        Hello4 hello41 = (Hello4) hello4.clone();
        System.out.println(hello4.getGreet()==hello41.getGreet()); //true
    }
}
```

## 第四种：序列化

序列化生成对象的本质是将二进制重新反序列化成对象，生成的对象值相同，是深度的拷贝

```java
@Data
@AllArgsConstructor
public class Hello5 implements Serializable{
    public String greet="hello";

    public static void main(String[] args) throws Exception {
        ByteArrayOutputStream buf = new ByteArrayOutputStream();
        ObjectOutputStream o = new ObjectOutputStream(buf);
        Hello5 hello5 = new Hello5("hello world");
        o.writeObject(hello5);
        //start clone
        ObjectInputStream in = new ObjectInputStream(
            new ByteArrayInputStream(buf.toByteArray())
        );
        Hello5 hello51 = (Hello5) in.readObject();
        System.out.println(hello5.getGreet()==hello51.getGreet()); //false 
    }
}
```

> 参考 ：
>
> 《Java编程思想》第四版 中的  11、12章节

 [github代码](https://github.com/dddygin/intentional-learning/tree/master/learningJava/java-base/src/main/java/dddy/gin/create_object)


