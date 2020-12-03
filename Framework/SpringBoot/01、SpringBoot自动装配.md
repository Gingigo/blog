# SpringBoot 自动装配

##  Spring 模式注解

[模式注解](https://github.com/spring-projects/spring-framework/wiki/Spring-Annotation-Programming-Model#stereotype-annotations)是一种用于声明在应用中扮演“组件”角色的注解。

### 常用的模式注解

| Spring Framework | 场景说明           | 起始版本 |
| ---------------- | ------------------ | -------- |
| `@Repository`    | 数据仓储模式注解   | 2.0      |
| `@Component`     | 通用组件模式注解   | 2.5      |
| `@Service`       | 服务模式注解       | 2.5      |
| `@Controller`    | Web 控制器模式注解 | 2.5      |
| `@Configuration` | 配置类模式注解     | 3.0      |



### 装配方式

#### XML `<context:component-scan>`

```xml
<context:component-scan base-package="com.gin.spring.boot" />
```

#### Annotation `@ComponentScan`

```java
@ComponentScan(basePackages = "com.gin.spring.boot")
```



### 自定义模式注解

#### `@Component`派生性

#### `@Component`层次性



## Spring @Enable 模块装配

Spring Framework 3.1 开始支持”@Enable 模块驱动“。所谓“模块”是指具备相同领域的功能组件集合， 组合所形成一个独立的单元。比如 Web MVC 模块、AspectJ代理模块、Caching（缓存）模块、JMX（Java 管 理扩展）模块、Async（异步处理）模块等。

### 实现方式

#### 注解驱动方式

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

```java
@Configuration
public class DelegatingWebMvcConfiguration extends
WebMvcConfigurationSupport {
...
}
```

#### 接口编程方式

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
...
}
```

```java
public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {
    ...
}
```

### 自定义 @Enable 模块

#### 注解驱动方式

- 新建注解 `@EnableHelloWorld`

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @Documented
  @Import(HelloWorldConfiguration.class)
  public @interface EnableHelloWorld {
  }
  ```

- 新建类 `HelloWorldConfiguration` 用来生成 Bean

  ```java
  public class HelloWorldConfiguration {
      @Bean
      public String helloWorld(){
          return "Hello, World";
      }
  }
  ```

- 新建 BootStrap 类 用上 `@EnableHelloWorld` 

  ```java
  @EnableHelloWorld
  public class EnableHelloWorldBootstrap {
      public static void main(String[] args) {
          ConfigurableApplicationContext context 
              = new SpringApplicationBuilder(EnableHelloWorldBootstrap.class)
                  .web(WebApplicationType.NONE)
                  .run(args);
          String helloWorld  = context.getBean("helloWorld",String.class);
          System.out.println("helloWorld Bean : " + helloWorld);
          context.close();
      }
  }
  ```

- 输出

  ```log
  helloWorld Bean : Hello, World
  ```



  #### 接口编程方式

- 新建注解 `@EnableHelloWorldImportSelector`

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @Documented
  @Import(HelloWorldImportSelector.class)
  public @interface EnableHelloWorldImportSelector {
  }
  ```

- 新建 `HelloWorldImportSelector` 类，并实现 `ImportSelector`

  ```java
  public class HelloWorldImportSelector implements ImportSelector {
      @Override
      public String[] selectImports(AnnotationMetadata importingClassMetadata) {
          return new String[]{HelloWorldConfiguration.class.getName()};
      }
  }
  ```

- 新建 BootStrap 类 用上 `@EnableHelloWorldImportSelector`

  ```java
  @EnableHelloWorldImportSelector
  public class EnableHelloWorldImportSelectorBootstrap {
      public static void main(String[] args) {
          ConfigurableApplicationContext context
                  = new SpringApplicationBuilder(EnableHelloWorldImportSelectorBootstrap.class)
                  .web(WebApplicationType.NONE)
                  .run(args);
          String helloWorld = context.getBean("helloWorld", String.class);
          System.out.println("helloWorld Bean : " + helloWorld);
          context.close();
      }
  }
  ```

- 输出

  ```log
  helloWorld Bean : Hello, World
  ```

#### 对比

- 接口编程方式相对于注解驱动方式更加灵活，中间多了一层选择。



## Spring 条件装配

从 Spring Framework 3.1 开始，允许在 Bean 装配时增加前置条件判断

### 条件注解举例

| Spring 注解    | 场景说明       | 起始版本 |
| -------------- | -------------- | -------- |
| `@Profile`     | 配置化条件装配 | 3.1      |
| `@Conditional` | 编程条件装配   | 4.0      |



### 实现方式

配置方式-`@Profile`

#### 例子:累加

- 新建接口 `CalculateService` 

  ```java
  public interface CalculateService {
      /**
       * 从多个整数 sum 求和
       * @param values 多个整数
       * @return sum 累加值
       */
      Integer sum(Integer... values);
  }
  ```

- JDK7 实现方式 `Java7CalculateService`

  ```java
  @Profile("Java7")
  @Service
  public class Java7CalculateService implements CalculateService {
      @Override
      public Integer sum(Integer... values) {
          System.out.println("Java 7 for 循环实现 ");
          int sum = 0;
          for (int i = 0; i < values.length; i++) {
              sum += values[i];
          }
          return sum;
      }
      public static void main(String[] args) {
          CalculateService calculateService = new Java7CalculateService();
          System.out.println(calculateService.sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
      }
  }
  ```

- JDK7 实现方式 `Java8CalculateService`

  ```java
  @Profile("Java8")
  @Service
  public class Java8CalculateService implements CalculateService {
      @Override
      public Integer sum(Integer... values) {
          System.out.println("Java 8 Lambda 实现");
          int sum = Stream.of(values).reduce(0, Integer::sum);
          return sum;
      }
      public static void main(String[] args) {
          CalculateService calculateService = new Java8CalculateService();
          System.out.println(calculateService.sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
      }
  }
  ```

- 新建 Bootstrap 类 `ProfileCalculateServiceBootstrap` 选用 Profile 

  ```java
  @SpringBootApplication(scanBasePackages = "com.gin.auto.configure.service")
  public class ProfileCalculateServiceBootstrap {
  
      public static void main(String[] args) {
          ConfigurableApplicationContext context = new SpringApplicationBuilder(ProfileCalculateServiceBootstrap.class)
                  .web(WebApplicationType.NONE)
                  .profiles("Java8")
                  .run(args);
          // CalculateService Bean 是否存在
          CalculateService calculateService = context.getBean(CalculateService.class);
          System.out.println("calculateService.sum(1...10) : " +
                  calculateService.sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
          // 关闭上下文
          context.close();
      }
  }
  ```

- 结果，输出 JDK8 的实现方式 

  ```log
  Java 8 Lambda 实现
  calculateService.sum(1...10) : 55
  ```

> 如果 profiles("Java8")  填写成 profiles("Java7") 结果会使用 JDK7 的实现方式

编程方式-`@Conditional`

#### 例子：自定注解实现条件装配

- 新建自定义注解 `ConditionalOnSystemProperty`

  ```java
  /**
   * Java 系统属性 条件判断
   */
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ ElementType.TYPE, ElementType.METHOD })
  @Documented
  @Conditional(OnSystemPropertyCondition.class)
  public @interface ConditionalOnSystemProperty {
      /**
       * Java 系统属性名称
       * @return
       */
      String name();
  
      /**
       * Java 系统属性值
       * @return
       */
      String value();
  }
  ```

- 注解生效类 `OnSystemPropertyCondition`

  ```java
  public class OnSystemPropertyCondition implements Condition {
      @Override
      public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
          MultiValueMap<String, Object> attributes =
                  metadata.getAllAnnotationAttributes(ConditionalOnSystemProperty.class.getName());
          String propertyName = String.valueOf(attributes.get("name").get(0));
          String propertyValue = String.valueOf(attributes.get("value").get(0));
          String javaPropertyValue = System.getProperty(propertyName);
          return propertyValue.equals(javaPropertyValue);
      }
  }
  ```

- 新建 Bootstrap 类 `ConditionalOnSystemPropertyBootstrap`

  ```java
  /**
   * 系统属性条件引导类
   */
  public class ConditionalOnSystemPropertyBootstrap {
      @Bean
      @ConditionalOnSystemProperty(name = "user.name", value = "Administrator")
      public String helloWorld() {
          return "hello world Administrator";
      }
      public static void main(String[] args) {
          ConfigurableApplicationContext context = new SpringApplicationBuilder(ConditionalOnSystemPropertyBootstrap.class)
                  .web(WebApplicationType.NONE)
                  .run(args);
          // 通过名称和类型获取 helloWorld Bean
          String helloWorld = context.getBean("helloWorld", String.class);
          System.out.println("helloWorld Bean : " + helloWorld);
          // 关闭上下文
          context.close();
      }
  }
  ```

- 结果

  ```log
  helloWorld Bean : hello world Administrator
  ```

> 如果 user.name 属性配不上，context.getBean("helloWorld", String.class) 找不到对应的 Bean

## Spring Boot 自动装配

在 Spring Boot 场景下，基于约定大于配置的原则，实现 Spring 组件自动装配的目的。其中包含一下技术

> 什么是“约定的大于配置”，一句话搞定“自带默认配置”

### 底层装配技术

- Spring 模式注解装配
- Spring `@Enable` 模块装配
- Spring 条件装配装配
- Spring 工厂加载机制
  - 实现类： `SpringFactoriesLoader`
  - 配置资源： `META-INF/spring.factories`

### 自动装配举例

- 新建 `HelloWorldAutoConfiguration` 自动装配类

  ```java
  /**
   * HelloWorld 自动装配
   */
  @Configuration // Spring 模式注解装配
  @EnableHelloWorld // Spring @Enable 模块装配
  @ConditionalOnSystemProperty(name = "user.name", value = "Administrator") // 条件装配
  public class HelloWorldAutoConfiguration {
  }
  ```

- 新建 `META-INF/spring.factories`

  ```factories
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.gin.auto.configure.spring.factories.HelloWorldAutoConfiguration
  ```

- Bootstrap 类

  ```java
  /**
   * {@link EnableAutoConfiguration} 引导类
   */
  @EnableAutoConfiguration
  public class EnableAutoConfigurationBootstrap {
      public static void main(String[] args) {
          ConfigurableApplicationContext context = new SpringApplicationBuilder(EnableAutoConfigurationBootstrap.class)
                  .web(WebApplicationType.NONE)
                  .run(args);
          // helloWorld Bean 是否存在
          String helloWorld = context.getBean("helloWorld", String.class);
          System.out.println("helloWorld Bean : " + helloWorld);
          // 关闭上下文
          context.close();
      }
  }
  ```

- 结果

  ```log
  helloWorld Bean : Hello, World
  ```

## 总结

Spring Boot 自动装配不是什么新技术，是基于 Spring Framework 实现的，但能带来简化的配置，使用者更加容易上手。

