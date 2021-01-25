# JVM 内存空间



## 运行时数据区域

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/JVM/shengsiyuan/memory.png)

## 堆

### new 关键字对象的创建过程

1. 在堆内存中创建出对象的实例。
2. 为对象的实例成员变量赋初值。
3. 将对象的引用返回。

### 分布

1. 指针碰撞(前提是堆中的空间通过一个指针进行分割，一侧是已经被占用的空间，另一侧是未被占用的空间)
2. 空闲列表(前提是堆内存空间中已被使用与未被使用的空间是交织在一起的，这时，虚拟机就需要通过一个列表来记录哪些空间是可以使用的， 哪些空间是已被使用的，接下来找出可以容纳下新创建对象的且未被使用的空间，在此空间存放该对象，同时还要修改列表上的记录)

## 对象

### 结构

1. 对象头
2. 实例数据
3. 对齐填充

### 引用对象的方式

1. 使用句柄的方式
2. 使用直接指针的方式（HotSpot）

## 栈

测试栈溢出

```java
/**
 * 虚拟机栈溢出测试
 */
public class Test02 {
    private int length;

    public int getLength() {
        return length;
    }

    public void test() throws InterruptedException {
        length++;
        Thread.sleep(1);
        test();
    }

    public static void main(String[] args) {
        //测试调整虚拟机栈内存大小为：  -Xss160k，此处除了可以使用JVisuale监控程序运行状况外还可以使用jconsole
        Test02 test02 = new Test02();
        try {
            test02.test();
        } catch (Throwable e) {
            System.out.println(test02.getLength());//打印最终的最大栈深度为：2587
            e.printStackTrace();
        }
    }
}
```

## 元空间

元空间溢出

```java
/**
 * 元空间内存溢出测试
 * 设置元空间大小：-XX:MaxMetaspaceSize=100m
 * 关于元空间参考：https://www.infoq.cn/article/java-permgen-Removed
 */
public class Test03 {
    public static void main(String[] args) {
        for (;;){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Test03.class);
            enhancer.setUseCache(false);
            enhancer.setCallback((MethodInterceptor) (obj, method, ags, proxy) -> proxy.invokeSuper(obj, ags));
            System.out.println("Hello World");
            enhancer.create();// java.lang.OutOfMemoryError: Metaspace
        }

    }
}

```