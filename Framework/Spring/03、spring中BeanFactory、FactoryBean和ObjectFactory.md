# 03、BeanFactory、FactoryBean和ObjectFactory

## BeanFactory

BeanFactory 是 IOC 底层容器。

- Spring IOC 所遵守的最底层和最基本的编程规范；
-  BeanFactory 只是个接口。实现类：DefaultListableBeanFactory、 ApplicationContext 等，都是附加了某种功能的实现；
- 定位：<font color='orange'>管理Bean的工厂</font>

BeanFactory 的方法，如图 01

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/framework/spring/spring_03_01.jpg)

<center>图 01 BeanFactory 的方法</center>

## FactoryBean

FactoryBean 是创建 Bean 的一种方式，帮助实现复杂的初始化逻辑。

- 当实例化 Bean 过程比较复杂，如果按照传统的方式需要配置大量信息，可实现 FactoryBean 接口重新 `getObject()` 返回初始化 Bean，实现复杂的初始逻辑；
- 定位：<font color='orange'>生成Bean的工厂</font>。

## ObjectFactory

ObjectFactory 主要用于 spring 中创建 Bean 的时候延迟加载，ObjectFactory#getObject 可能返回原始Bean 也有可能返回代理 Bean 对象。（可以理解就是利用 Lambda 的回调)。

> Bean 的初始化中 三级缓存和二级缓存存储的 factory 就是 ObjectFactory 类型。

- 可以用于依赖查找中的延迟功能
- 定位：<font color='orange'>延迟创建Bean的工厂</font>。

## 对比

- 定位不同：
  - BeanFactory 是管理Bean（查）、FactoryBean 是创建 Bean（创建）、ObjectFactory延迟创建Bean（创建）；

> 参考文章：
>
> 1、[BeanFactory与FactoryBean](https://www.iteye.com/blog/chenzehe-1481476)
>
> 2、小马哥讲spring 的《spring 依赖查找》
>
> 

