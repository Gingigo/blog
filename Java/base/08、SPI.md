# SPI

## 什么是 SPI？

SPI(Service Provider Interface): [维基百科的解释](https://en.wikipedia.org/wiki/Service_provider_interface) ,看起来有点懵。
简单的理解就是，顶层已经定义好了抽象接口，实现的类会放在常用扩展的目录中，SPI 机制能找到接口对应的实现类，并且实例化。

> 一句话概括：找实现类的机制

## SPI 能做什么？
JDBC 为例，JDK 自带了 `java.sql.Driver` 对数据库驱动做了标准，但是每一个厂家的驱动实现是不一样的(如:MySQL、Oracle),JDK 不可能把每一种实现都
写到 JDK 中 这样代码臃肿不好维护，如升级了 MySQL 驱动还需要重新发个 JDK 版？？？ 本质就是违背了可拔插的原则。

现在 SPI 能解决上面提到的问题？

- JDK 只需要定义驱动的标准

- 厂家实现驱动定义的标准即可

- 将厂家定义好的驱动打成 jar 包，放到 JDK 常用扩展的目录中

- SPI 机制即可调用到厂家的实现的类

## 实现 SPI

原理:

- 定义好抽象的接口 interface A

- 在资源目录下的`META-INF/services`新建一个文件 `xxx.xxx.A` (A 的全类名)

- 在文件 `xxx.xxx.A` 中写下实现 A 接口的实现类的名称 `xxx.xxx.AImpl`(A实现类的全类名)

- 当需要用到 A 接口时，通过 `ServiceLoader.load`或者`Service.providers` 可以获取到实现类的实例

    - 通过拼接 `META-INF/services` +`/` + `xxx.xxx.A` 可以定位到文件的位置
    
    - 读取文件拿到每一行字符串（每一行就是一个 A 接口的实现类的全类名 `xxx.xxx.AImpl`）
    
    - 通过反射实例化实现类
    
- 这样实现类就被引入到项目中了

> 这里可能会涉及到线程上下文类加载器的知识，这里就不展开了。

## JDK 中的数据库驱动案例


以 MySQL 使用 SPI 实现驱动为例为例:

- `META-INF/services` 新建一个名称为要实现 SPI 功能的接口名(如：`java.sql.Driver` )

- 文件内容填写实现该接口的类(如：`com.mysql.cj.jdbc.Driver`)

- `com.mysql.cj.jdbc.Driver` 实现类内容如下：

  ```java
  public class Driver extends NonRegisteringDriver implements java.sql.Driver {
      public Driver() throws SQLException {
      }
  
      static {
          try {
              DriverManager.registerDriver(new Driver());
          } catch (SQLException var1) {
              throw new RuntimeException("Can't register driver!");
          }
      }
  }
  ```

- 初始化 JDBC 中的 MySQL 驱动

  ```java
  Class<?> clazz = Class.forName("com.mysql.jdbc.Driver"); // ①
  ```

-   ① 行代码是通过反射初始化自身并向 `DriverManager` 注册了自己，并将驱动的注册信息存储到 `registeredDrivers`

    ```java
    public class Driver extends NonRegisteringDriver implements java.sql.Driver {
        static {
                try {
                    java.sql.DriverManager.registerDriver(new Driver());
                } catch (SQLException E) {
                    throw new RuntimeException("Can't register driver!");
                }
            }
        ...
    }
    ```

     ```java
    public static synchronized void registerDriver(java.sql.Driver driver,
                DriverAction da)
            throws SQLException {
    
            /* Register the driver if it has not already been added to our list */
            if(driver != null) {
                registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
            } else {
                // This is for compatibility with the original DriverManager
                throw new NullPointerException();
            }
    
            println("registerDriver: " + driver);
    
        }
     ```

-   驱动向`DriverManager`注册的时候调用 `registerDriver` 方法，这是会触发 `DriverManager.class` 初始化，从而调用静态代码块

    ```java
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
    ```

    ```java
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                    Iterator<Driver> driversIterator = loadedDrivers.iterator();
                    try{
                        while(driversIterator.hasNext()) {
                            driversIterator.next();
                        }
                    } catch(Throwable t) {
                    // Do nothing
                    }
                    return null;
                }
            });
    ```

-   `ServiceLoader.load(Driver.class)` 是一个非常重要的类中一个重要方法，根据 `META-INF/services/java.sql.Driver` 找到 MySQL 驱动，返回一个 `LazyIterator` 对象，通过调用 `next()` 用反射的方式实例化 MySQL 驱动类

    ```java
    private S nextService() {
        ...
                try {
                    c = Class.forName(cn, false, loader);
                } catch (ClassNotFoundException x) {
                    fail(service,
                         "Provider " + cn + " not found");
                }
        ...
            }
    ```

## 实战

定义一个抽象接口 `SpiService.java`

```java
/**
 * SPI 抽象接口
 */
public interface SpiService {
    void execute();
}
```

定义抽象接口的实现类 `SpiServiceImplA.java` 和 `SpiServiceImplA.java`

```java
/**
 * SPI 服务实现类 A
 */
public class SpiServiceImplA implements SpiService {
    @Override
    public void execute() {
        System.out.println("SPI 实现类 A 被调用");
    }
}
```

```java
/**
 * SPI 服务实现类 B
 */
public class SpiServiceImplB implements SpiService {
    @Override
    public void execute() {
        System.out.println("SPI 实现类 B 被调用");
    }
}

```

在`META-INF/services` 目录中新建文件名为`dddy.gin.spi.SpiService`,内容如下：

```txt
dddy.gin.spi.SpiServiceImplA
dddy.gin.spi.SpiServiceImplB
```

通过 `ServiceLoader.load()` 方式加载资源

```java
/**
 * 通过 ServiceLoader 加载 SPI 机制实例
 */
public class SpiTestByServiceLoader {
    public static void main(String[] args) {
        ServiceLoader<SpiService> load = ServiceLoader.load(SpiService.class);
        Iterator<SpiService> iterator = load.iterator();
        while (iterator.hasNext()){
            SpiService ser = iterator.next();
            ser.execute();
        }
    }
}
```

通过`Service` 方式加载

```java
/**
 * 通过 Service 加载 SPI 机制实例
 */
public class SpiTestByService {
    public static void main(String[] args) {
        Iterator<SpiService> providers = Service.providers(SpiService.class);
        while (providers.hasNext()) {
            providers.next().execute();
        }
    }
}
```

输出

```log
SPI 实现类 A 被调用
SPI 实现类 B 被调用
```

## 参考

> [深入理解SPI机制](https://www.jianshu.com/p/3a3edbcd8f24)

