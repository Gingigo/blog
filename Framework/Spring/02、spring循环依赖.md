# Spring 循环依赖问题

## 什么是循环依赖

循环依赖其实就是循环引用，也就是两个或则两个以上的bean互相持有对方，最终形成闭环。比如A依赖于B，B依赖于A。如图 A、B 互为循环依赖。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_01.png)

<center>图01 A、B 循环依赖图</center>

### 代码实现

A.java

```java
public class A {
    private B b;

    public void setB(B b) {
        this.b = b;
    }

    public B getB() {
        return b;
    }
}
```

B.java

```java
public class B {
    private A a;

    public void setA(A a) {
        this.a = a;
    }

    public A getA() {
        return a;
    }
}
```

cyclic-dependence.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="a" class="dddy.gin.bean.di.cyclic.A">
        <property name="b" ref="b"/>
    </bean>
    <bean id="b" class="dddy.gin.bean.di.cyclic.B">
        <property name="a" ref="a"/>
    </bean>
</beans>
```

测试类

```java
public class CircularDependency {
    @Test
    public void test01(){
        ApplicationContext context = new ClassPathXmlApplicationContext("cyclic-dependence.xml");
        A a = context.getBean(A.class,"a");
        B b = context.getBean(B.class,"b");
        System.out.println(a.);
    }
}
```

结果

```java
true
```

## 怎么检测是否存在循环依赖

检查循环依赖相对比较容易，Bean 再创建的时候可以给 Bean 打标签，如果递归调用回来发现正在创建中的话，即说明了循环依赖了。

具体代码在 `DefaultSingletonBeanRegistry` 中的 `getSingleton(String beanName, boolean allowEarlyReference)` 方法中有一个判断 `isSingletonCurrentlyInCreation(beanName)` 返回当前对象是否正在创建。

DefaultSingletonBeanRegistry.java

```java
/**
* 循环依赖重点：
* 	singletonObjects： 一级缓存，存储的是实例化并且初始化的单例 bean
* 	earlySingletonObjects： 二级缓存，存储的是实例化，但未初始化的 bean
* 	singletonFactories：三级缓存，存储的是实例化 bean 的工厂方法
*/         
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
...
public boolean isSingletonCurrentlyInCreation(String beanName) {
    // 如果是正在创建返回true，否则返回 false 并且添加到正在创建的容器中
    return this.singletonsCurrentlyInCreation.contains(beanName);
}
```

## 怎么解决循环依赖

### 相关概念

#### 三大步骤

spring 处理循环依赖，核心部分有三步走，`createBeanInstance`、`populateBean`、`initializeBean`:

- `createBeanInstance`: 使用默认的无参构造方法实例化 Bean；
- `populateBean` : 填充属性值，通过 setter 注入属性的值；
- `initializeBean`： 调用配置的 `init-method`属性指定的方法。

#### 四大容器

`DefaultSingletonBeanRegistry` 类在解决循环依赖中几个重要的容器 ：

-  `singletonObjects`： 一级缓存，存储的是实例化并且初始化的单例 Bean（map）;
-  `earlySingletonObjects`: 二级缓存，存储的是实例化，但未初始化的 Bean（map）;
-  `singletonFactories`：三级缓存，存储的是实例化 Bean 的工厂方法（map）;
- `singletonsCurrentlyInCreation`：存储正在创建的 Bean 的名字 `beanName` （set）;

### 断点跟踪

我先来看一下循环依赖的时序图

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_02.jpg)

<center> 图 02 循环依赖时序图</center>

启动我们的测试类，根据时序图我们会进入`refresh()`在进入`finishBeanFactoryInitialization()`等等，我们的断点就设置在 `DefaultListableBeanFactory` 类中的 `getBean(String name)` 方法。我们假设是 A 类实例化，然后在 B 类在实例化

0. 设置  `beanName`  为 a；

1. `getBean` 方法会调用 `doGetBean`。spring 中带 `do` 前缀的方法都是真实做事的方法；

2. 进入**①标签**中的`doGetBean` 中调用了 `Object sharedInstance = getSingleton(beanName);`

3. 继续跟踪到**②标签**中`getSingleton`方法，发现实现代码如下：

   ```java
   protected Object getSingleton(String beanName, boolean allowEarlyReference) {
       Object singletonObject = this.singletonObjects.get(beanName);
       if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
           synchronized (this.singletonObjects) {
               singletonObject = this.earlySingletonObjects.get(beanName);
               if (singletonObject == null && allowEarlyReference) {
                   ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                   if (singletonFactory != null) {
                       singletonObject = singletonFactory.getObject();
                       this.earlySingletonObjects.put(beanName, singletonObject);
                       this.singletonFactories.remove(beanName);
                   }
               }
           }
       }
       return singletonObject;
   }
   ```

   这是个重要的方法，主要的功能如下：

   - 根据`beanName` 获取一级缓存中的 Bean，如果查找存在则直接返回；
   - 如果在一级缓存中查找为空，则进入判断：
     - `isSingletonCurrentlyInCreation(beanName)` 这句可以理解为，将 `beanName` 添加到 `singletonsCurrentlyInCreation`容器中，并且返回 `false`;

4. 返回到**①标签**中的`doGetBean` 方法再调用**③标签**中 `getSingleton`方法，调用的代码如下：

   ```java
   // Create bean instance.
   if (mbd.isSingleton()) {
       sharedInstance = getSingleton(beanName, () -> {
           try {
               return createBean(beanName, mbd, args);
           }
           catch (BeansException ex) {
               // Explicitly remove instance from singleton cache: It might have been put there
               // eagerly by the creation process, to allow for circular reference resolution.
               // Also remove any beans that received a temporary reference to the bean.
               destroySingleton(beanName);
               throw ex;
           }
       });
       bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
   }
   ```

   - 传入的参数是`beanName` 和一个`ObjectFactory<?>`匿名函数；
   - **③标签**中 `getSingleton`方法，因为`Object singletonObject = this.singletonObjects.get(beanName);` 返回的是空，所以会执行`singletonObject = singletonFactory.getObject();`
   - 其中` singletonFactory.getObject()`是一个回调方法，会执行，刚刚传入`ObjectFactory<?>`匿名函数中的方法体，也就是第4步骤的代码中的`return createBean(beanName, mbd, args);`

5. 通过回调，调用`AbstractAutowireCapableBeanFactory` 中的**④标签** `createBean(beanName, mbd, args)`

6. ④标签中的方法`createBean`方法调用了`Object beanInstance = doCreateBean(beanName, mbdToUse, args);`进入了`doCreateBean`方法开始创建 Bean

7. 在`doCreateBean`方法中调用了三大步骤中`createBeanInstance`返回`beanName`对应的实例化对象，对应流程图中的**标签⑤**，代码如下;

   ```java
   if (instanceWrapper == null) {
       instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   ```

   - 使用默认的无参构造方法去实例化 Bean 对象

8. 接下来就去调用`addSingletonFactory`方法，将`beanName`，和单例工厂对象存储到三级缓存中:

   调用代码如下：

   ```java
   addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   ```

9. **标签⑥**中的`addSingletonFactory` 是一个重要的方法，代码如下：

   ```java
   protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
       Assert.notNull(singletonFactory, "Singleton factory must not be null");
       synchronized (this.singletonObjects) {
           if (!this.singletonObjects.containsKey(beanName)) {
               this.singletonFactories.put(beanName, singletonFactory);
               this.earlySingletonObjects.remove(beanName);
               this.registeredSingletons.add(beanName);
           }
       }
   }
   ```

   - 主要的功能是将`beanName `和 创建`beanName`对应的单例对象工厂存储到三级缓存中；

这时对应的四大容器的存储情况如下：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_03.png)

<center>图04 四大容器中的情况图01</center>

10. 接下来 `doCreateBean`方法开始执行三大步骤中的 `populateBean`填充 Bean 的属性，对应流程图中的**标签⑦**

11. 在`populateBean`方法中调用`applyPropertyValues(beanName, mbd, bw, pvs);`进入`applyPropertyValues`方法，对应**标签⑧**

12. `applyPropertyValues`方法中的调用`resolveValueIfNecessary`方法，解析引用对象，调用代码如下：

    ```java
    Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
    ```

13. 如果类中有引用对象会进入`resolveReference` 方法进行应用对象的解析，代码如下(对应标签⑨)

    ```java
    if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        return resolveReference(argName, ref);
    }
    ```

14. `resolveReference`  方法会调用`this.beanFactory.getBean(resolvedName);` 获取属性对应的`bean`对象,对应标签⑩；

> 以上步骤，如果我们结合 A 类、和 B 类，就是先解析了 A 类，并且实例化了 A 类，对应的对象时 a，但是 a 并未初始化，而且 a 类正在填充属性值，并且正在解析 B 类型的属性 b。

15. 我们看到`this.beanFactory.getBean(resolvedName);` 这个和我们刚刚开始的步骤中一样，只是重新跑了一遍 第 1 步骤到第 14 步骤，

    只是四大容器中发生了变化，变化如下：（A、B 对象实例化完成，但未初始化）

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_04.png)

<center>图04 四大容器中的情况图02</center>

到这里我们小结一下：

- 第一次：1-14 步骤是对 A 进行解析，但是 A 中有 B 类型的属性，所以需要注入 B 类型的 bean，而 B 类型的 Bean 没有在一级缓存，所以就有了第二次；
- 第二次：1-14 步骤是对 B 进行解析，但是 B 中也是 A 类型的属性，所以需要对 A 进行解析；所以有了第三次调用 `doGetBean` 对应①标签的位置；
- 第三次我们也是从这个方法开始，但是四大容器中存储的内容已经变为如图 04；

16. 这时调用`getSingleton` 方法，这是对应的是标签标签②，这是我们再来看一次 `getSingleton`的代码;

    ```java
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
    ```

    - 第一、第二次调用这个方法的时候因为`isSingletonCurrentlyInCreation(beanName)` 是 false 直接就返回空了（第3步骤）
    - 但是第三次来的时候，`isSingletonCurrentlyInCreation(beanName)` 条件是成立的，（这时的 `beanName` 为 a）
    - `singletonObject = this.earlySingletonObjects.get(beanName);` 为空，则`singletonObject == null && allowEarlyReference` 返回 true；
    - 通过`singletonFactory.getObject();`从三级缓存中获取并且生成 beanName 对应的对象或者代理对象；
    - 这时候会执行将`beanName ` 对应的（beanName，A 实例）添加到二级缓存，并且从三级缓存中移除（A（beanName，单例对象工厂））移除，四大容器里面的存储情况如图 05

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_05.png)

<center>图05 四大容器中的情况图03</center>

17. 这时就会返回 A 的实例 a,而 `doGetBean` 方法 会直接返回 a 实例；

18. 这时会一直返回到第二次调用中的 b 实例解析 A 类型属性中的值，即① -》⑩ -》⑨ -》⑧ 的步骤，继续执行；

19. 执行标签⑾的位置注入属性的值，这时的 b 对象已经完成了实例化和初始化，之后会调用`initializeBean`方法对应标签⑿；

20. 完成了 B 类型的对象的实例化和初始化，会返回到时序图中 ④-》③ 的位置；

21. 随后执行`afterSingletonCreation` 和 `addSingleton` ,对应标签是⒀ 和 ⒁；

    - 我们看一下 `afterSingletonCreation`  代码;

      ```java
      protected void afterSingletonCreation(String beanName) {
          if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
              throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
          }
      }
      ```

      - 就是移除将 B的beanName 从 `singletonsCurrentlyInCreation` 移除；

    - `addSingleton`代码：

      ```java
      protected void addSingleton(String beanName, Object singletonObject) {
          synchronized (this.singletonObjects) {
              this.singletonObjects.put(beanName, singletonObject);
              this.singletonFactories.remove(beanName);
              this.earlySingletonObjects.remove(beanName);
              this.registeredSingletons.add(beanName);
          }
      }
      ```

      - 将 b 添加到一级缓存中；
      - 将 b 相关的对象从二级和三级缓存中删除
      - 这时四大容器的存储如图06

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_06.png)

<center>图06 四大容器中的情况图05</center>

22. 这时 b 已经成功实例化和初始化，就要返回回去第一次调用的（a 实例中有 B 类型属性的位置即 第一次的⑧标签）

23. 获取到 b 的实例后,重复19-21步骤的操作，只是 a、b 反过来（或者 A 、B 反过来），其中四大容器中的变化为，图07

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_07.png)

<center>图07 四大容器中的情况图06</center>

到此，A、B 依赖循环的问题就解决了

总的流程如下:

高清图片：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_02_08.jpg)



## 问题

1、一级缓存能不能解决循环依赖？

答：不能，在三个级别的缓存中，放置的对象是有区别的，（1、是完成实例化且初始化的对象，2、是实例化但未初始化的对象），如果只有一级缓存，那么在并发操作下，有可能取到实例化但未初始化的对象，属性值未空，这个是有问题的？

2、二级缓存能不能解决？为什么要三级缓存？

答：理论上是二级缓存是可解决循环依赖，但是为什么需要三级缓存呢？，本质在于为了创建代理对象。三级缓存中放置的是，生成具体对象的一个匿名内部类，这个匿名类可能生成代理类，也可能是普通的实例对象，而使用三级缓存就保证了不管是否需要代理对象，都保证使用的是一个对象，而不会出现，前面使用的是普通 bean，后面使用代理类。