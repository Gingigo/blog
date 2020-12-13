# 02、理解SpringApplication

`SpringApplication` 构造方法

```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
         // 推断 web 应用类型
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		this.bootstrappers = new ArrayList<>(getSpringFactoriesInstances(Bootstrapper.class));
         // 加载应用上下文初始器
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
         // 加载应用事件监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
         // 推断启动类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```



## 基本使用

### 普通方式

```java
public class DiveInSpringBootApplication {
    public static void main(String[] args) {
        SpringApplication springApplication = 
            new SpringApplication(DiveInSpringBootApplication.class);
        springApplication.setBannerMode(Banner.Mode.CONSOLE);
        springApplication.setWebApplicationType(WebApplicationType.NONE);
        springApplication.setAdditionalProfiles("prod");
        springApplication.setHeadless(true);
    }
}
```

### Builder 方式

```java
public class DiveInSpringBuilderBootApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext springApplication
                = new SpringApplicationBuilder(DiveInSpringBuilderBootApplication.class)
                .bannerMode(Banner.Mode.CONSOLE)
                .web(WebApplicationType.NONE)
                .profiles("prod")
                .headless(true)
                .run(args);
    }
}
```

## 准备阶段

### 配置 Spring Boot Bean 源

Java 配置 Class 或 XML 上下文配置文件集合，用于 Spring Boot BeanDefinitionLoader 读取 ，并且将配置源解析加载为
Spring Bean 定义

- 数量：一个或多个以上
- 配置方式：
  - `@Configuration` 或其他注解
  -  `XML` 文件

### 推断 Web 应用类型

根据当前应用 ClassPath 中<font color='orange'>是否存在相关实现类</font>来推断 Web 应用的类型,包括：

- Web Reactive： `WebApplicationType.REACTIVE`
- Web Servlet：`WebApplicationType.SERVLET`
- 非 Web：`WebApplicationType.NONE`

参考于：`org.springframework.boot.WebApplicationType#deduceFromClasspath`

```java
static WebApplicationType deduceFromClasspath() {
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;
	}
```

### 推断引导类

参考于：org.springframework.boot#deduceMainApplicationClass

```java
private Class<?> deduceMainApplicationClass() {
		try {
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {
				if ("main".equals(stackTraceElement.getMethodName())) {
					return Class.forName(stackTraceElement.getClassName());
				}
			}
		}
		catch (ClassNotFoundException ex) {
			// Swallow and continue
		}
		return null;
	}
```

通过代码可知，推导引用类是通过变量方法栈匹配方法名为 `main` 的类，进行推导的。

### 加载应用上下文初始器

利用 Spring 工厂加载机制，实例化 ApplicationContextInitializer 实现类，并排序对象集合。

参考于：`org.springframework.boot.getSpringFactoriesInstances`

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
    	// 获取 /MATE-INF/spring.factories 下实现类 ApplicationContextInitializer.class的类名
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    	// 利用反射实例化
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    	//排序
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```



### 加载应用事件监听器

利用 Spring 工厂加载机制，实例化 ApplicationListener 实现类，并排序对象集合

## 运行阶段

`SpringApplication#run`

```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		DefaultBootstrapContext bootstrapContext = createBootstrapContext();
		ConfigurableApplicationContext context = null;
		configureHeadlessProperty();
    	 // 获取监听器
		SpringApplicationRunListeners listeners = getRunListeners(args); 
    	// 发布应用正在启动
		listeners.starting(bootstrapContext, this.mainApplicationClass);
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			context.setApplicationStartup(this.applicationStartup);
			prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

### 运行监听器

`SpringApplicationRunListener` 监听多个运行状态方法

```java
public interface SpringApplicationRunListener {

	/**
	 * Spring 应用刚启动
	 */
	default void starting(ConfigurableBootstrapContext bootstrapContext) {
		starting();
	}

	/**
	 * ConfigurableEnvironment 准备妥当，允许将其调整
	 */
	default void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
			ConfigurableEnvironment environment) {
		environmentPrepared(environment);
	}

	/**
	 * ConfigurableApplicationContext 准备妥当，允许将其调整
	 */
	 */
	default void contextPrepared(ConfigurableApplicationContext context) {
	}

	/**
	 * ConfigurableApplicationContext 已装载，但仍未启动
	 */
	default void contextLoaded(ConfigurableApplicationContext context) {
	}

	/**
	 * ConfigurableApplicationContext 已启动，此时 Spring Bean 已初始化完成
	 */
	default void started(ConfigurableApplicationContext context) {
	}

	/**
	 * Spring 应用正在运行
	 */
	default void running(ConfigurableApplicationContext context) {
	}

	/**
	 * Spring 应用运行失败
	 */
	default void failed(ConfigurableApplicationContext context, Throwable exception) {
	}

}
```



### 创建 Environment

根据准备阶段的推断 Web 应用类型创建对应的 ConfigurableEnvironment 实例

- Web Reactive： `StandardReactiveWebEnvironment`
- Web Servlet： `StandardServletEnvironment`
- 非 Web： `StandardEnvironment`

参考于：`org.springframework.boot#getOrCreateEnvironment()`

```java
	private ConfigurableEnvironment getOrCreateEnvironment() {
		if (this.environment != null) {
			return this.environment;
		}
		switch (this.webApplicationType) {
		case SERVLET:
			return new StandardServletEnvironment();
		case REACTIVE:
			return new StandardReactiveWebEnvironment();
		default:
			return new StandardEnvironment();
		}
	}
```

### 创建 Spring 应用上下文

根据准备阶段的推断 Web 应用类型创建对应的 ConfigurableApplicationContext 实例

- Web Reactive： `AnnotationConfigReactiveWebServerApplicationContext`
- Web Servlet： `AnnotationConfigServletWebServerApplicationContext`
- 非 Web： `AnnotationConfigApplicationContext`

参考于：`org.springframework.boot.ApplicationContextFactory`

```java
ApplicationContextFactory DEFAULT = (webApplicationType) -> {
		try {
			switch (webApplicationType) {
			case SERVLET:
				return new AnnotationConfigServletWebServerApplicationContext();
			case REACTIVE:
				return new AnnotationConfigReactiveWebServerApplicationContext();
			default:
				return new AnnotationConfigApplicationContext();
			}
		}
		catch (Exception ex) {
			throw new IllegalStateException("Unable create a default ApplicationContext instance, "
					+ "you may need a custom ApplicationContextFactory", ex);
		}
	};
```

