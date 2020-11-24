# 01 、spring xml 配置

## \<Bean\>

Spring IoC 容器管理一个或者多个bean。这些 bean 是使用提供给容器的配置元数据传教的（如  XML \<bean\> ）。这些 bean 定义被表示为 BeanDefinition对象(即读取 bean 配置的元数据，封装成 BeanDefinition )。其中包含一下元数据

- Bean 的类名：通常是定义的 bean 的实际实现类；
- Bean 的行为：它们陈述 bean 在容器中的行为(范围、生命周期回调等）；
- Bean 的依赖：对 bean 完成工作所需的其他 bean 的引用，这些引用也称为协作者或依赖关系；
- Bean 的配置：在新创建的对象中设置的其他配置设置，ーー例如，池的大小限制或管理连接池的 bean 中使用的连接数。

### \<bean\> 的元数据

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_01_01.png)

<center> 图 01 bean 的元数据</center>

#### Bean 的命名

基于 xml 的配置元数据中，可以使用 id 属性、 name 属性或两者来指定 bean 标识符。

- 格式:

  - ```xml
    <!-- 内部命名 -->
    <bean id="user" name="u1,u2" class="dddy.gin.entity.User"></bean>
    ```

  - ```xml
    <!-- 外部命名 -->
    <alias name="fromName" alias="toName"/>
    ```

- 属性：
  - id
    - 唯一性
    - 允许为空，系统会默认生成一个唯一的id
  - name
    - 唯一性
    - 允许多个名字，中间用逗号(,)、分号(;)或空白分隔
- 规范：
  
  - 约定是在命名 bean 时对实例字段名使用标准的 Java 约定（驼峰命名）

#### Bean 的初始化

Bean 定义 本质上是创建一个或者多个对象的配方，当被请求时，容器查看已命名 bean 的配方，并使用该 bean 定义封装的配置元数据来创建(或获取)实际对象。

- 格式

  - ```xml
    <bean id="user" class="dddy.gin.entity.User"></bean>
    ```

- 属性 class

  - 必须填写（requested）
  - **表示实例化的对象的类型(或类)**

> 静态内部类，的 class 怎么填写？用 “$”。
>
> 如果 class A{ public static class B {}; }
>
> ```xml
> <bean id="b" class="A$B"></bean>
> ```

- 实例化方式

  - 构造方法实例化

    - 可能需要默认构造方法

    - ```xml
      <bean id="exampleBean" class="examples.ExampleBean"/>
      ```

  - 静态工厂方法实例化

    - ```xml
      <bean id="clientService" 
            class="dddy.gin.bean.official.ClientService"
            factory-method="createInstance"></bean>
      ```

      ```java
      public class ClientService {
          private static ClientService clientService = new ClientService();
          private ClientService() {
          }
          public static ClientService createInstance() {
              return clientService;
          }
      }
      ```

    - factory-method：指定工厂方法本身的名称

  - 实例工厂方法实例化

    - ```xml
      <bean id="defaultServiceLocator" class="dddy.gin.bean.official.DefaultServiceLocator"></bean>
          <bean id="clientServiceDome" class="dddy.gin.bean.official.ClientServiceDome"
                factory-bean="defaultServiceLocator"
                factory-method="createClientServiceInstance">
          </bean>
      ```

    - ```java
      public class DefaultServiceLocator {
          private static ClientServiceDome clientServiceDome = new ClientServiceDome();
          public ClientServiceDome createClientServiceInstance() {
              return clientServiceDome;
          }
      }
      ```

    - ```java
      public class ClientServiceDome {
      }
      ```

    - factory-bean : 指定 class 属性的实例

    - factory-method：指定实例对象工厂方法本身的名称

    > 一个工厂类可以容纳多个工厂方法（即不同的 factory-method，返回不同的对象）

    

  - Bean 运行时类型

    - 特定bean的运行时类型并不容易确定，bean 元数据定义中的指定类只是初始类引用，可能与声明的工厂方法结合，或者作为FactoryBean类，这可能导致bean的不同运行时类型

    - 获取运行时 Bean 的类型，推荐方法是使用BeanFactory.getType调用指定的 bean 名称，并返回 BeanFactory 对象的类型

    - ```java
      public void getRuntimeTypeTest(){
              ApplicationContext context = new ClassPathXmlApplicationContext("bean.xml");
              Class clazz = context.getType("clientServiceDome");
              System.out.println(clazz);
      }
      ```

#### 依赖注入

依赖注入是一个过程，对象仅通过<font color='orange'>构造函数参数、工厂方法的参数或在构造或从工厂方法返回后在对象实例上设置的属性来定义它们的依赖项</font>(即与它们一起工作的其他对象)，容器在创建bean时注入这些依赖项。这个过程基本上是bean本身的反向(因此得名，控制的反转)

- 依赖注入的优点
  - 使用 DI 原则的代码更加整洁；
  - 对象具有其依赖关系时，解耦更加有效；
  - 测试更加容易。

##### 依赖注入的方式

- 构造器注入

  - 定义：基于构造器的 DI 是通过容器调用许多参数的构造函数来实现的，每一个参数表示一个依赖。（静态工厂方法来构造 bean 几乎是等效的）

  - 实现

    - 构造函数参数对象注入：

      - ```java
        public class ThingThree {
        }
        ```

      - ```java
        public class ThingTwo {
        }
        ```

      - ```java
        public class ThingOne {
            private ThingTwo two;
            private ThingThree three;
        
            public ThingOne(ThingTwo two, ThingThree three) {
                this.two = two;
                this.three = three;
            }
        }
        ```

      - ```java
        <bean id="thingOne" class="dddy.gin.bean.di.ThingOne">
            <constructor-arg name="two" ref="thingTwo"></constructor-arg>
            <constructor-arg name="three" ref="thingThree"></constructor-arg>
        </bean>
        <bean id="thingTwo" class="dddy.gin.bean.di.ThingTwo"/>
        <bean id="thingThree" class="dddy.gin.bean.di.ThingThree"/>
        ```

    - 构造函数参数简单类型注入

      - ```xml
        <bean id="thingOneBySimpleValue"  class="dddy.gin.bean.di.ThingOne">
            <constructor-arg name="x" value="0"></constructor-arg>
            <constructor-arg name="y" value="1"></constructor-arg>
        </bean>
        ```

      - ```java
        public ThingOne(int x, String y) {
            this.x = x;
            this.y = y;
        }
        ```

    -  构造函数参数类型注入

      - ```xml
        <bean id="thingOneByType" class="dddy.gin.bean.di.ThingOne">
            <constructor-arg type="int" value="0"></constructor-arg>
            <constructor-arg type="java.lang.String" value="1"></constructor-arg>
        </bean>
        ```

    -  构造函数参数索引注入

      - ```xml
        <bean id="thingOneByIndex" class="dddy.gin.bean.di.ThingOne">
            <constructor-arg index="0" ref="thingTwo"></constructor-arg>
            <constructor-arg index="1" ref="thingThree"></constructor-arg>
        </bean>
        ```

- 静态工厂方法或者实例方法（可以看作是构造器注入）

  - 重点：

    1. 静态工厂方法的参数由 < constructor-arg/> 元素提供，与实际使用的构造函数完全相同；
    2. 实例(非静态)工厂方法可以以本质上相同的方式使用(除了使用 factory-bean 属性而不是 class 属性之外) 

  - 实现

    - 在 ThingFour 添加3个方法;

      - ```java
        public ThingFour() {
        }
        private ThingFour(ThingTwo two, ThingThree three, int x) {
            this.two = two;
            this.three = three;
            this.x = x;
        }
        ```

      - ```java
        public static ThingFour getInstance(ThingTwo two, ThingThree three, int x) {
           return new ThingFour(two,three,x);
        }
        ```

      - ```xml
            <!-- 构造器注入-静态方法    -->
        <bean id="thingFourBystatic" class="dddy.gin.bean.di.ThingFour" factory-method="getInstance">
            <constructor-arg ref="thingTwo"></constructor-arg>
            <constructor-arg ref="thingThree"></constructor-arg>
            <constructor-arg value="3"></constructor-arg>
        </bean>
        ```

  

- setter  注入

  - 作用：在调用无参数构造函数或无参数静态工厂方法实例化 bean 之后，bean 上的容器调用 setter 方法可以实现基于 setter 的 DI。

  - 实现

    - ```java
      public class ThingFour {
          private ThingTwo two;
          private ThingThree three;
      
          private int x;
          private String y;
      
          public void setTwo(ThingTwo two) {
              this.two = two;
          }
      
          public void setThree(ThingThree three) {
              this.three = three;
          }
      
          public void setX(int x) {
              this.x = x;
          }
      
          public void setY(String y) {
              this.y = y;
          }
      
          public ThingTwo getTwo() {
              return two;
          }
          
      }
      ```

    - ```xml
      <bean id="thingFour" class="dddy.gin.bean.di.ThingFour">
          <property name="two" >
              <ref bean="thingTwo"/>
          </property>
          <property name="three" ref="thingThree"/>
      
          <property name="x" value="0"></property>
      </bean>
      ```

- p 名称注入(简化setter 注入)

  引入命名空间

  ```xml
  xmlns:p="http://www.springframework.org/schema/p"
  ```

  ```xml
  <bean id="thingFour2" class="dddy.gin.bean.di.ThingFour" 
        p:three-ref="thingThree"
        p:two-ref="thingTwo"
        p:x="1"
        p:y="2"
  />
  ```

  

- 构造器注入和setter注入的对比

  - 怎么选择注入方式：
    - 对于强依赖采用构造器注入，对于可选的依赖采用 setter 注入
  - 构造器
    - 构造器注入的组件总是以完全初始化的状态返回给客户端(调用)代码，（对象不可变）
    - 构造函数参数是一种糟糕的代码，这意味着类可能有太多的责任。
    - 会产出循环依赖问题。
  - setter
    - setter方法使该类的对象容易进行重新配置或重新注入。
    - 依赖项需要设置默认值，否则必须在代码使用依赖项的任何地方执行非空检查。
    - setter 注入能解决循环依赖问题。
  

##### 内部 bean 和外部 bean

内部 bean：

元素中的 `<property/>` 或 `<constructor-arg/>` 元素定义了一个内部 bean，如下面的示例所示:

```xml
<bean id="outer" class="...">
    <!-- instead of using a reference to a target bean, simply define the target bean inline -->
    <property name="target">
        <bean class="com.example.Person"> <!-- this is the inner bean -->
            <property name="name" value="Fiona Apple"/>
            <property name="age" value="25"/>
        </bean>
    </property>
</bean>
```

元素中的 `ref` 属性引用了外部定义的 bean

```xml
<bean id="thingOne" class="dddy.gin.bean.di.ThingOne">
    <constructor-arg name="two" ref="thingTwo"></constructor-arg>
    <constructor-arg name="three" ref="thingThree"></constructor-arg>
</bean>
<bean id="thingTwo" class="dddy.gin.bean.di.ThingTwo"/>
<bean id="thingThree" class="dddy.gin.bean.di.ThingThree"/>
```

##### 集合

实体类 CollectionsModel

```java
public class CollectionsModel {
    List<String> list;
    Set<String> set;
    Map<String,String> map;
    Properties properties;
    ThingOne one;

    public List<String> getList() {
        return list;
    }

    public void setList(List<String> list) {
        this.list = list;
    }

    public Set<String> getSet() {
        return set;
    }

    public void setSet(Set<String> set) {
        this.set = set;
    }

    public Map<String, String> getMap() {
        return map;
    }

    public void setMap(Map<String, String> map) {
        this.map = map;
    }

    public Properties getProperties() {
        return properties;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    public ThingOne getOne() {
        return one;
    }

    public void setOne(ThingOne one) {
        this.one = one;
    }
}
```

xml 配置

```xml
<bean id="collectionsModel" class="dddy.gin.bean.di.CollectionsModel">
    <property name="list">
        <list>
            <value>list01</value>
            <value>list02</value>
            <value>list03</value>
        </list>
    </property>

    <property name="set">
        <set>
            <value>s01</value>
            <value>s02</value>
            <value>s03</value>
        </set>
    </property>
    
    <property name="map">
        <map>
            <entry key="k01" value="v01"/>
            <entry key="k02" value="v02"/>
            <entry key="k03" value="v03"/>
        </map>
    </property>
    
    <property name="properties">
        <props>
            <prop key="p01">v01</prop>
            <prop key="p02">v02</prop>
            <prop key="p03">v03</prop>
        </props>
    </property>

    <property name="one">
        <null/>
    </property>

</bean>
```

