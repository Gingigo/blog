# Web MVC 核心

## Spring MVC 架构演进

### 基础架构：Servlet

![](https://raw.githubusercontent.com/dddygin/image-storage/main/image/blog/image/framework/springboot/image-20201222212708057.png)

- 特点
  - 请求/响应式（Request/Response）
  - 屏蔽网络通讯的细节
- API 特性
  - 面向 HTTP 协议
  - 完整的生命周期



### 核心架构：[前端控制器](http://www.corej2eepatterns.com/FrontController.htm)

![](https://raw.githubusercontent.com/dddygin/image-storage/main/image/blog/image/framework/springboot/image-20201222213449298.png)

- [资源](http://www.corej2eepatterns.com/FrontController.htm)
- 实现：Spring Web MVC [DispatcherServlet](https://docs.spring.io/spring-framework/docs/1.0.0/javadoc-api/org/springframework/web/servlet/DispatcherServlet.html)

Spring Web MVC 架构

![](https://raw.githubusercontent.com/dddygin/image-storage/main/image/blog/image/framework/springboot/image-20201222213912939.png)

## 认识 Spring Web MVC

### Spring Web MVC  交互交互流程

![](https://raw.githubusercontent.com/dddygin/image-storage/main/image/blog/image/framework/springboot/image-20201222214244388.png)

### Spring MVC 组件

| 组件 Bean 类型                                               | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [HandlerMapping](https://docs.spring.io/spring/docs/5.0.6.RELEASE/spring-framework-reference/web.html#mvc-handlermapping) | 映射请求（Request）到处理器（Handler）加上其关联的拦截器（HandlerInterceptor）列表，其映射关系基于不同的 HandlerMapping 实现的一些标准细节。其中两种主要 HandlerMapping 实现， RequestMappingHandlerMapping支持标注 @RequestMapping 的方法， SimpleUrlHandlerMapping 维护精确的URI路径与处理器的映射 |
| HandlerAdapter                                               | 帮助 DispatcherServlet 调用请求处理器（Handler），无需关注其中实际的调用细节。比如，调用注解实现的 Controller 需要解析其关联的注解. HandlerAdapter的主要目的是为了屏蔽与 DispatcherServlet 之间的诸多细节。 |
| [HandlerExceptionResolver](https://docs.spring.io/spring/docs/5.0.6.RELEASE/spring-framework-reference/web.html#mvc-exceptionhandlers) | 解析异常，可能策略是将异常处理映射到其他处理器（Handlers） 、或到某个 HTML错误页面，或者其他 |
| [ViewResolver](https://docs.spring.io/spring/docs/5.0.6.RELEASE/spring-framework-reference/web.html#mvc-viewresolver) | 从处理器（Handler）返回字符类型的逻辑视图名称解析出实际的 View 对象，该对象将渲染后的内容输出到HTTP 响应中。 |
| [LocaleResolver](https://docs.spring.io/spring/docs/5.0.6.RELEASE/spring-framework-reference/web.html#mvc-localeresolver),<br/>[LocaleContextResolver](https://docs.spring.io/spring/docs/5.0.6.RELEASE/spring-framework-reference/web.html#mvc-timezone) | 从客户端解析出 `Locale` ，为其实现国际化视图。               |
| [MultipartResolver](https://docs.spring.io/spring/docs/5.0.6.RELEASE/spring-framework-reference/web.html#mvc-multipart) | 解析多部分请求（如 Web 浏览器文件上传）的抽象实现            |

### 实现

#### xml

实现 `Controller`

```java
@Controller
public class HelloWorldController {
    @RequestMapping("")
    public String index() {
    	return "index";
    }
}
```

spring mvc 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
    http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">
    <context:component-scan base-package="com.imooc.web"/>
        <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"/>
        <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter"/>
        <bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

部署 DispatcherServlet

```xml
<web-app>
    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app-context.xml</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

#### 注解

- 版本依赖
  - Spring Framework 3.1+

注解配置： `@Configuration` 

组件激活： `@EnableWebMvc`

自定义组件 ： `WebMvcConfigurer`

```java
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {

//     <!--<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">-->
//        <!--<property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>-->
//        <!--<property name="prefix" value="/WEB-INF/jsp/"/>-->
//        <!--<property name="suffix" value=".jsp"/>-->
//    <!--</bean>-->
    @Bean
    public ViewResolver viewResolver(){
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setViewClass(JstlView.class);
        viewResolver.setPrefix("/WEB-INF/jsp/");
        viewResolver.setSuffix(".jsp");
        return viewResolver;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new HandlerInterceptor() {
            @Override
            public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
                System.out.println("拦截中...");
                return true;
            }
        });
    }
}
```

web.xml 转变成

```java
public class DefaultAnnotationConfigDispatcherServletInitializer extends
        AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getRootConfigClasses() { // web.xml
        return new Class[0];
    }

    @Override
    protected Class<?>[] getServletConfigClasses() { // DispatcherServlet
        return new Class[]{DispatcherServletConfiguration.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```

扫描路径下的 Bean

```java
@ComponentScan(basePackages = "com.imooc.web")
public class DispatcherServletConfiguration {
}
```

**常用注解**

- 注册模型属性： `@ModelAttribute`
- 读取请求头： `@RequestHeader`
- 读取 Cookie： `@CookieValue`
- 校验参数： `@Valid` 、 `@Validated`
- 注解处理： `@ExceptionHandler`
- 切面通知： `@ControllerAdvice`

#### 自动装配

- 版本依赖
  - Spring Framework 3.1 +
  - Servlet 3.0 +

Servlet SPI

- Servlet SPI ServletContainerInitializer ，参考 Servlet 3.0 规范

Spring 适配 Servlet SPI

- SpringServletContainerInitializer

Spring SPI

- 基础接口： `WebApplicationInitializer`
- 编程驱动： `AbstractDispatcherServletInitializer`
- 注解驱动： `AbstractAnnotationConfigDispatcherServletInitializer`

## SpringBoot  简化 Spring MVC

完全自动装配 

- `DispatcherServlet` 的装配  ->   `DispatcherServletAutoConfiguration`
- 替换 `@EnableWebMvc` -> `WebMvcAutoConfiguaration`
- Servlet 容器的装配: `ServletWebServerFactoryAutoConfiguration`

