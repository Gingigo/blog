## 01、spring核心之IOC容器

**IOC(Inversion of Control)控制反转**：所谓控制反转，就是把原先我们代码里面需要实现的对象创
建、依赖的代码，反转给容器来帮忙实现。那么必然的我们需要创建一个容器，同时需要一种描述来让
容器知道需要创建的对象与对象的关系。

**DI(Dependency Injection)依赖注入**：就是指对象是被动接受依赖类而不是自己主动去找，换句话说就
是指对象不是从容器中查找它依赖的类，而是在容器实例化对象的时候主动将它依赖的类注入给它。

我们要定义个 IOC 容器从设计的角度来说有几个问题：

1. 如何维护对象与对象之间的关系？

   我们可以通过 xml，propertites 文件等语意化配置文件表示。

2. 描述对象的文件放在那里？

   可能是 classpath、数据库、URL网络资源等...

3. 不同的配置文件对对象的描述规则不一样，如何如何统一？

   在内部需要有一个统一的关于对象的定义，所有外部的描述都必须转化成统一的描述定义。

4. 如何对不同的文件进行解析，获得统一的管理？

   需要对不同的配置文件语法，采用不同的解析器。

了解了 **IOC** 设计角度的几个问题之后，我们来看看，spring 是如何解决这些问题的。接下来我们来看一个非常重要的接口 <font color='red'>**BeanFactory**</font>，为  IOC 容器提供了很多便利和基础服务。



## Spring 核心容器类图

### BeanFactory

Spring Bean 的创建是典型的**工厂模式**，这一系列的 Bean 工厂，也即 IOC 容器为开发者管理对象间的依赖关系提供了很多便利和基础服务，在 Spring 中有许多的 IOC 容器的实现供用户选择和使用，其相互关系如下：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/source/code/spring/spring_beanfactory_01.png)

<center>图 01 BeanFactory 类图</center>

这张类图**非常重要**，有几点是需要我们理解的：

- BeanFactory 是作为最顶层的一个类接口，它定义了 IOC 容器的基本功能规范。
- BeanFactory 有三个重要的子类，LIstableBeanFactory、HierarchicalBeanFactory、AutowireCappableBeanFactory。
- LIstableBeanFactory 是对 BeanFactory 进行扩展，使 BeanFactory 可以返回多个的Bean 。(BeanFactory 返回单个的 Bean )。
- HierarchicalBeanFactory 是将 BeanFactory 拥有分层次。(也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为父子关系)。
  - HierarchicalBeanFactory  的接口只定义了查看的接口，传入父级 BeanFactory 需要在 ConfigurableBeanFactory 中的 setParentBeanFactory 传入。
- AutowireCapableBeanFactory，对 BeanFactory 进行扩展，使 BeanFactory 拥有自动装配功能。
- DefaultListableBeanFactory 最终实现了所有的接口。

需要打开 idea 看一下 以上提到的几个接口和类，对他们有个基本的了解。

在 BeanFactory 里只对 IOC 容器的基本行为作了定义，根本不关心你的 Bean 是如何定义怎样加载的。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。

而要知道工厂是如何产生对象的，我们需要看具体的 IOC 容器实现，Spring 提供了许多 IOC 容器的 实 现 。 比 如
GenericApplicationContext ， ClasspathXmlApplicationContext等

ApplicationContext 是 Spring 提供的一个高级的 IOC 容器，它除了能够提供 IOC 容器的基本功能外，还为用户提供了以下的附加服务。从 ApplicationContext 接口的实现，我们看出其特点：

1. 支持信息源，可以实现国际化。（实现 MessageSource 接口）
2. 访问资源。(实现 ResourcePatternResolver 接口)
3. 支持应用事件。(实现 ApplicationEventPublisher 接口)



### BeanDefinition

Spring IOC 容器管理了我们定义的各种 Bean 对象及其相互关系，Bean 对象在 Spring 实现中是以 BeanDefinition 来描述的（如描述bean 的属性值、构造函数参数以及具体的实现和该bean 是不是单例等），其类图如下：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/source/code/spring/spring_beandefinition_02.png)

<center>图 02 BeanDefinition 类图</center>

- AttributeAccessor 是一个顶层的接口，主要的功能是抽象的描述“访问对象元数据的抽象接口”可以查看源代码理解
- AttributeAccessorSupport 类是实现了AttributeAccessor 接口的基本功能，提供了基本的属性访问功能,是一个工具类。
- **BeanDefinition**接口就比较重要点了，它是描述 bean 的属性，如bean是否单例，是否懒加载，bean的属性等。可以查看源代码加深理解。
- AbstractBeanDefinition抽象类是提供一个基础的bean定义，方便子类扩展。



### BeanDinfinitionReader

Bean 的解析过程非常复杂，功能被分的很细，应为需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。Bean 的解析主要就是对 Spring 配置文件的解析。这个解析过程通过 BeanDefintionReader 来完成，最后看看 Spring 中的 BeanDefinitionReader 的类图结构：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/source/code/spring/spring_beandefinitionreader_03.png)

<center>图03 BeanDefintionReader 类图 </center>

- BeanDefinitionReader，就是加载 Bean。如Bean一般来说都在配置文件中定义，从配置文件中获取bean的定义。
- XmlBeanDefinitionReader，就是专门读取XML文件里的bean定义的。

















> 参考文档:
>
> 1. 沽泡Tom老师笔记
> 2. https://javadoop.com/post/spring-ioc