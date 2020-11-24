# 01、@import

`@Import` 有什么用？

引用一句官方注释的话：“提供与springxml中的`<import/>`元素等效的功能”。其实本质的功能就是将 Bean 注册到 容器中。

## 用法

### 1、直接使用 @import 

**Cat.java**

```java
@Data
public class Cat {
    private String name;

    public Cat() {
        this.name = "lili";
    }
}
```

**Dog.java**

```java
@Data
public class Dog {
    private String name;

    public Dog() {
        this.name = "cici";
    }
}
```

**ImportFeature.java**

```java
@Import(value = {Cat.class, Dog.class})
public class ImportFeature {

    public static void main(String[] args) {
        // 注册并启动 Spring 应用上下文
        AnnotationConfigApplicationContext context =
            new AnnotationConfigApplicationContext(ImportFeature.class);

        Cat cat = context.getBean(Cat.class);
        Dog dog = context.getBean(Dog.class);

        System.out.println(cat);
        System.out.println(dog);

        // 显示地关闭 Spring 应用上下文
        context.close();
    }
}
```

输出结果

```log
Cat(name=lili)
Dog(name=cici)
```

### 2、使用 @import 和 <font color='orange'>ImportSelector</font> 

**ImportBean.java**

```java
public class ImportBean implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.gin.model.Cat","com.gin.model.Dog"};
    }
}
```

**ImportSelectorFeature**

```java
@Import(value = {ImportBean.class})
public class ImportSelectorFeature {

    public static void main(String[] args) {
        // 注册并启动 Spring 应用上下文
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ImportFeature.class);

        Cat cat = context.getBean(Cat.class);
        Dog dog = context.getBean(Dog.class);

        System.out.println(cat);
        System.out.println(dog);


        // 显示地关闭 Spring 应用上下文
        context.close();
    }
}
```

输出结果

```log
Cat(name=lili)
Dog(name=cici)
```

### 3、使用 @import 和 <font color='orange'>ImportBeanDefinitionRegistrar</font> 

**ImportBeanByDefinitionRegistrar.java**

```java
public class ImportBeanByDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                        BeanDefinitionRegistry registry) {
        RootBeanDefinition catDefinition = new RootBeanDefinition(Cat.class);
        RootBeanDefinition dogDefinition = new RootBeanDefinition(Dog.class);

        registry.registerBeanDefinition("cat",catDefinition);
        registry.registerBeanDefinition("dog",dogDefinition);
    }
}
```

**ImportBeanDefinitionRegistrarFeature.java**

```java
@Import({ImportBeanByDefinitionRegistrar.class})
public class ImportBeanDefinitionRegistrarFeature {
    public static void main(String[] args) {
        // 注册并启动 Spring 应用上下文
        AnnotationConfigApplicationContext context = 
            new AnnotationConfigApplicationContext(ImportFeature.class);

        Cat cat = context.getBean(Cat.class);
        Dog dog = context.getBean(Dog.class);

        System.out.println(cat);
        System.out.println(dog);


        // 显示地关闭 Spring 应用上下文
        context.close();
    }
}
  
```
输出结果

```log
Cat(name=lili)
Dog(name=cici)
```

## 结论

- 生成 Bean 并注册到容器中。

## 用途

- 用于导入第三方包到spring容器中。