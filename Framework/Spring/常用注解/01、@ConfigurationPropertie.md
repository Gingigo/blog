# 02、@ConfigurationPropertie

`@ConfigurationPropertie` 有什么用？

将外部化环境参数绑定到属性中，可以通过 `@EnableConfigurationProperties` 激活 `@ConfigurationPropertie`  并且注册到 Spring 容器中。

## 用法

**Pig.java**

```java
@ConfigurationProperties(prefix="model.pig")
@Setter
@ToString
public class Pig {
    private String name;
    private Integer age;
    private String address;
}
```

**application.properties**

```properties
model.pig.name=Paige
model.pig.age=5
```

**idea 中 Environment variables**

```properties
model.pig.address=英国
```

**ConfigurationPropertiesFeature.java**

```java
@SpringBootApplication
@EnableConfigurationProperties(value = {Pig.class})
public class ConfigurationPropertiesFeature {
    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = 
            SpringApplication.run(ConfigurationPropertiesFeature.class, args);
        Pig pig = applicationContext.getBean(Pig.class);
        System.out.println(pig);
    }
}
```

输出结果

```log
Pig(name=Paige, age=5, address=英国)
```

## 结论

- `@ConfigurationPropertie`  绑定数据的来源不仅仅来自与配置文件 application.properties,而是来源与整个环境参数。
- `@EnableConfigurationProperties` 可以激活 `@ConfigurationPropertie` ，并且将绑定的类的 Bean 注册到 Spring 容器中。

## **用途**

1、用于配置数据库多数据源

**application.properties**

```properties
#数据源
spring.datasource.druid.write.url=jdbc:mysql://localhost:3306/jpa
spring.datasource.druid.write.username=root
spring.datasource.druid.write.password=1
spring.datasource.druid.write.driver-class-name=com.mysql.jdbc.Driver

spring.datasource.druid.read.url=jdbc:mysql://localhost:3306/jpa
spring.datasource.druid.read.username=root
spring.datasource.druid.read.password=1
spring.datasource.druid.read.driver-class-name=com.mysql.jdbc.Driver
```

**DruidDataSourceConfig**

```java
@Configuration
public class DruidDataSourceConfig {
    /**
     * DataSource 配置
     * @return
     */
    @ConfigurationProperties(prefix = "spring.datasource.druid.read")
    @Bean(name = "readDruidDataSource")
    public DataSource readDruidDataSource() {
        return new DruidDataSource();
    }


    /**
     * DataSource 配置
     * @return
     */
    @ConfigurationProperties(prefix = "spring.datasource.druid.write")
    @Bean(name = "writeDruidDataSource")
    @Primary
    public DataSource writeDruidDataSource() {
        return new DruidDataSource();
    }
}
```



## 参考

> 1、[注解@ConfigurationProperties使用方法](https://www.cnblogs.com/tian874540961/p/12146467.html)





