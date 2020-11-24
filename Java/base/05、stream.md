# Stream

流（Stream）：提供了一种让我们可以比集合更高的概念级别上指定计算的数据视图。

- 特点：
  - 高级别数据视图
  - 遵循 “做什么而非怎么做”的原则
- 与集合对比
  - 流并不存储元素，可能存储在底层集合中，或者按需生成；
  - 流的操作不会修改其数据源；
  - 流的操作是尽可能惰性执行的。

例子1：过滤国家名字长度超过6的国家名

```java
/**
 * 认识一下 JDK 8 中的新特 stream
 */
public class HelloStream {
    public static void main(String[] args) {
        String[] countries = new String[]{"China", "Mongolia", "Korea", "South Korea", "Japan", 
                "Philippines", "Vietnam", "Cambodia" };
        List<String> names = Arrays.asList(countries);
        //获取名字大于6的国家名
        getLongNameCount(names,6);
    }

    private static void getLongNameCount(List<String> names, int length) {
        long count = names.stream().filter(e -> e.length() > length).count();
        System.out.println(count); // 5
    }

}
```

## 创建流

- 空流:

  ```java
  Stream<Object> empty = Stream.empty();
  ```

- 数组 -> 流

  ```java
  String[] countries = new String[]{"China", "Mongolia"};
  Stream<String> stream = Stream.of(countries);
  ```

- 集合 -> 流

  ```java
  List<String> names = Arrays.asList(countries);
  Stream<String> stream = names.stream();
  ```

- 种子流

  - Stream.iterate

    ```java
    //生成 0 1 2...
    Stream<BigInteger> generateInt = Stream.iterate(BigInteger.ZERO,n->n.add(BigInteger.ONE));
    ```

  - Stream.generate

    ```java
      //生成随机数 
    Stream<Double> random = Stream.generate(Math::random);
    ```
  
- 简单类型

  ```java
  //基本类型都有对应的Stream 这里演示了IntStream 几个简单的 API
  IntStream.of(1, 3, 2, 3, 4);
  IntStream.range(0,100);
  ```

  

## 使用流

- **Stream<T> filter(Predicate<? super T> predicate);**

  - 产生一个流，它包含当前流中所有满足断言条件的元素

  - 应用场景：多用用于筛选数据（不改变数据结构）

  - 使用例子：

    ```java
    //过滤 countries，长度大于 6 
    private static void getLongNameCountByArray(String[] countries, int length) {
            Stream<String> names = Stream.of(countries);
            long count = names.filter(e -> e.length() > length).count();
            System.out.println(count);
    }
    ```

- **\<R\> Stream\<R\> map(Function<? super T, ? extends R> mapper)**

  - 产生一个流，它包含将 mapper 应用于当前中所有元素所产生的结果

  - 应用场景：对数据进行处理（不改变容量，可以数据内容）

  - 使用例子

    ```java
    //转换小写
    private static void convertLowerCase(Stream<String> stream) {
            Stream<String> lowerStream = stream.map(String::toLowerCase);
            lowerStream.forEach(System.out::println);
    }
    ```

- **flatMap**

  - 产生一个流，它是通过将 mapper 应用于当前流中所有元素产生的结果连接到一起而获得的。

  - 应用场景：每个元素都返回 Stream\<T\>，并将其汇总到一个结果集中

  -  使用例子

    ```java
    public static void main(String[] args){
       //例子["i","love","you"] ->[["i"],["l","o","v","e"],["y","u","u"]]
    	Stream<String> names = Stream.of(new String[]{"i","love","you"}); 
        //通过 map 返回 Stream<Stream>
        convertStreamByMap(names);
        //通过 flatMap
        convertStreamByFlatMap(names);
    }
    
    private static void convertStreamByFlatMap(Stream<String> stream) {
        Stream<String> sss = stream.flatMap(UsedStream::letters);
    }
    
    private static void convertStreamByMap(Stream<String> stream) {
        Stream<Stream<String>> sss = stream.map(UsedStream::letters);
    }
    
    //字符串-> list -> stream
    // you -> ["y","o","u"]
    public static Stream<String> letters(String s){
        List<String> result = new ArrayList<>();
        for (int i = 0; i < s.length(); i++) {
            result.add(s.substring(i,i+1));
        }
        return result.stream();
    }
    
    ```

- **Stream\<T\> limit(long maxSize)**

  - 抽取指定数量

  - 应用场景：返回指定个数的元素

  - 例子：

    ```java
    private static void limitStream() {
        Stream<Double> random = Stream.generate(Math::random);
        random.limit(10).forEach(System.out::println);
    }
    ```

- **Stream\<T\> skip(long n)**

  - 跳过指定个数的数量

  - 例子

    ```java
    private static void skipStream() {
        Stream<Integer> integerStream = Stream.of(new Integer[]{3,2,1,5,6,5,0,7});
        integerStream.skip(2).forEach(System.out::println);
    }
    ```

- **Stream\<T\> concat(Stream<? extends T> a, Stream<? extends T> b)**

  - 拼接两个Stream

- **Stream\<T\> distinct()**

  - 返回去重元素

- **Stream\<T\> sorted()**

  - 按顺序返回元素

- **Stream\<T\> peek(Consumer<? super T> action)**

  - 获取数据调用方法

  - 例子：

    ```java
    private static void peekStream() {
        Stream<Double> powers = Stream.iterate(1.0, p->p*2)
            .peek(e -> System.out.println("Fetching: "+e)).limit(10);
    
        powers.forEach(e->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException interruptedException) {
                interruptedException.printStackTrace();
            }
        });
    }
    ```

## 约简

约简是一种总结操作，从流中获得答案。

- **long count()**

  - 返回流中元素个数

- **Optional\<T\> min(Comparator<? super T> comparator)**

  - 返回流中元素最小值

- **Optional\<T\> max(Comparator<? super T> comparator)**

  - 返回流中元素最大值

- **Optional<T> reduce(BinaryOperator<T> accumulator)**

  - 可用于求和、乘积、字符拼接、最大值和最小值等

  - 例子

    ```java
    private static void getStreamInfo() {
        Stream<Integer> integerStream1 = Stream.of(new Integer[]{3,2,1,5,6,5,0,7});
        Stream<Integer> integerStream2 = Stream.of(new Integer[]{3,2,1,5,6,5,0,7});
        Optional<Integer> max = integerStream1.max(Integer::compareTo);
        Optional<Integer> min = integerStream2.min(Integer::compareTo);
        System.out.println("max="+max.orElse(0));
        System.out.println("min="+min.orElse(0));
        
        
        //计算总数
        Stream<Integer> integerStream3 = Stream.of(new Integer[]{3,2,1,5,6,5,0,7});
        Optional<Integer> sum = integerStream3.reduce(Integer::sum);
        System.out.println(sum.get());
    }
    ```

- **Optional<T> findFirst()**

  - 返回第一个元素

- **Optional<T> findAny()**

  - 返回任意一个元素

- **boolean anyMatch(Predicate<? super T> predicate)**

  - 返回流中是否有满足断言

## Optional

Optional\<T\> 对象是一种包装器对象，Optional\<T\> 类型被当作一种更安全的方式，用来替代类型 T 的引用。主要的用法是：它在值不存在的情况下会产生一个可替代物，而只有在值存在的情况下才会使用这个值。

- 创建 Optional

  - empty() 产生一个空的Optional

    ```java
    private static final Optional<?> EMPTY = new Optional<>();
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }
    ```

  - of(value) 根据传入的值产生一个 Optional,如果 value 为空抛出空指针异常

    ```java
    public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
    }
    ```

  - ofNullable(value) 产生一个允许为空结果的 Optional

    ```java
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
    ```

- T orElse(T other)

  - 产生一个 Optional 的值，或者在该 Optional 为空时，产生一个 other。

- T orElseGet(Supplier<? extends T> other)

  - 产生一个 Optional 的值，或者在该 Optional 为空时，产生调用 other 的结果。

- \<X extends Throwable\> T orElseThrow(Supplier<? extends X> exceptionSupplier)

  - 产生这个 Optional 的值，或者在该值 Optional 为空时，抛出调用 exceptionSupplier 的结果。

- T get()

  - 产生这个 Optional 的值，或者在该 Optional 为空时，抛出一个 NoSuchElementException 对象

- boolean isPresent()

  - 如果该 Optional 不为空，则返回 true。

- Optional\<U> flatMap(Function<? super T, Optional\<U>> mapper)

  - flatMap 可以对流进行操作。

    ```java
    //计算 倒数平方根
    Optional<Double> optionalDouble = Optional.of(2.0)
        .flatMap(UsedOptional::inverse)
        .flatMap(UsedOptional::SquareRoot);
    
    ... 
        
    public static Optional<Double> SquareRoot(Double x) {
        return x < 0 ? Optional.empty() : Optional.of(Math.sqrt(x));
    }
    public static Optional<Double> inverse(Double x) {
        return x < 0 ? Optional.empty() : Optional.of(1/x);
    }
    ```

  Optional Api 调用代码：

  ```java
  public class UsedOptional {
      public static void main(String[] args) {
          Integer[] integers =new Integer[]{3, 2, 1, 5, 6, 5, 0, 7};
          Stream<Integer> integerStream1 = Stream.of(integers);
          Stream<Integer> integerStream2 = Stream.of(integers);
          Optional<Integer> integerOptional1 = integerStream1
                  .findAny()
                  .filter(i -> i == 100);
  
          Optional<Integer> integerOptional2 = integerStream2
                  .findAny()
                  .filter(i -> i == 3);
  
          //有值返回值，无值放回默认值 或者异常
          Integer result1 = integerOptional1.orElse(0);
          Integer result2 = integerOptional2.orElse(0);
          System.out.println(result1);
          System.out.println(result2);
  
          Integer result3 = integerOptional2.orElseGet(()->2);
          System.out.println(result3);
          // Integer result4 = integerOptional1.orElseThrow(IllegalStateException::new);
          // System.out.println(result4);
  
          //存在则处理，不存在则跳过
          Optional.empty().ifPresent(System.out::println);
          Optional.of(3).ifPresent(System.out::println);
  
  
          //计算 倒数平方根
          Optional<Double> optionalDouble = Optional.of(2.0)
                  .flatMap(UsedOptional::inverse)
                  .flatMap(UsedOptional::SquareRoot);
          System.out.println(optionalDouble.get());
      }
  
      public static Optional<Double> SquareRoot(Double x) {
          return x < 0 ? Optional.empty() : Optional.of(Math.sqrt(x));
      }
      public static Optional<Double> inverse(Double x) {
          return x < 0 ? Optional.empty() : Optional.of(1/x);
      }
  }
  
  ```

## 收集结果

- 转换
  - stream -> 数组
    - `stream.toArray(String[]::new)`
  - stream -> Collection
    - List 
    - stream.collect(Collectors.toList())
    - Set
      - stream.collect(Collectors.toSet())
  - 更多类型 参考`java.util.strea.Collectors`

- 遍历

  - stream.iterator()
  - stream.forEach()

- 聚合函数

  - stream.count()
  - stream.getMax()
  - 更多函数 参考`java.util.stream.Stream`

代码：

```java
/**
 * stream 收集结果集
 */
public class CollectResults {
    public static void main(String[] args) {
        String[] strings = new String[]{"a","b","","","c","d","e","f"};
        Stream<String> stringStream = Stream.of(strings);
        Stream<String> stringStream1 = Stream.of(strings);
        Stream<String> stringStream2= Stream.of(strings);
        Stream<String> stringStream3= Stream.of(strings);
        Stream<String> stringStream4= Stream.of(strings);
        Stream<String> stringStream5= Stream.of(strings);
        String[] results = stringStream.filter(s -> !s.equals("")).toArray(String[]::new);
        for (String result : results) {
            System.out.println(result);
        }

        // stream -> list 、set 、string 、foreach
        List<String> list = stringStream1.collect(Collectors.toList());
        Set<String> set = stringStream2.collect(Collectors.toSet());
        String join = stringStream3.collect(Collectors.joining(","));
        System.out.println(join);

        stringStream4.forEach(System.out::println);
        Iterator<String> iterator = stringStream5.iterator();

        // stream<Object> -> map<id,Object>
        stream2Map();

    }

    private static void stream2Map() {

        Person p1 = new Person(1,"gin",25);
        Person p2 = new Person(2,"yy",24);
        Person p3 = new Person(3,"mm",21);
        Person p4 = new Person(3,"sw",25);
        Person[] people = new Person[]{p1,p2,p3};
        Person[] people2 = new Person[]{p1,p2,p3,p4};

        Stream<Person> stream = Stream.of(people);
        Stream<Person> stream2 = Stream.of(people2);
        //id 不允许重复 java.lang.IllegalStateException: Duplicate key ...
        Map<Integer,Person> map= stream.collect(Collectors.toMap(Person::getId,p->p));
        System.out.println(map);
        // 可以重复，根据情况而定选取第一个还是第二个
        Map<Integer,Person> map2= stream2.collect(Collectors.toMap(Person::getId,p->p,(v1,v2)->v2));
        System.out.println(map2);
    }

    public static class Person{
        public int id;
        public String name;
        public int age;

        public Person(int id, String name, int age) {
            this.id = id;
            this.name = name;
            this.age = age;
        }

        @Override
        public String toString() {
            return "Person{" +
                    "id=" + id +
                    ", name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }

        public int getId() {
            return id;
        }

        public String getName() {
            return name;
        }

        public int getAge() {
            return age;
        }
    }
}
```

## 分组与分区

- 分组：Collectors.groupingBy
- 分区：Collectors.partitioningBy
- 下游收集器（和 mysql 中 聚合函数一致）
  - countingt() ，分组收集元素的个数
  - summingt() ，分组引元累计和
  - maxByt()，分组中最大的元素
  - minByt()，分组中最小的元素
  - toSet(),分组结果集为 Set 集合

代码：

```java
/**
 * 分组和分区
 */
public class GroupOperate {
    public static void main(String[] args) {
        Person p1 = new Person(1, "gin", 25);
        Person p2 = new Person(2, "yy", 24);
        Person p3 = new Person(3, "mm", 21);
        Person p4 = new Person(3, "sw", 25);
        Person[] people = new Person[]{p1, p2, p3, p4};
        //分组
        Map<Integer, List<Person>> map = Stream.of(people).collect(Collectors.groupingBy(Person::getId));
        System.out.println(map);

        //分区
        Map<Boolean, List<Person>> ageIs25 = Stream.of(people).collect(Collectors.partitioningBy(p -> p.age == 25));
        List<Person> listBy25 = ageIs25.get(true);
        List<Person> listByNot25 = ageIs25.get(false);
        System.out.println(listBy25);
        System.out.println(listByNot25);

        // 下游收集器 - 聚合函数
        //各个分组的总数
        Map<Integer, Long> count = Stream.of(people)
                .collect(
                        Collectors.groupingBy(Person::getId, Collectors.counting()));
        System.out.println(count);
        // 结果集是set
        Map<Integer, Set<Person>> set = Stream.of(people)
                .collect(
                        Collectors.groupingBy(Person::getId, Collectors.toSet()));
        System.out.println(set);


    }

    public static class Person {
        public int id;
        public String name;
        public int age;

        public Person(int id, String name, int age) {
            this.id = id;
            this.name = name;
            this.age = age;
        }

        @Override
        public String toString() {
            return "Person{" +
                    "id=" + id +
                    ", name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }

        public int getId() {
            return id;
        }

        public String getName() {
            return name;
        }

        public int getAge() {
            return age;
        }
    }
}

```



## 并行流

- 特点

  - 无序、效率高

- 启动方式

  - stream.parallel()

- 示例

  ```java
  public class ParallelStream {
      public static void main(String[] args) {
          IntStream stream = IntStream.of(1, 2, 3, 4, 5, 6, 7);
          stream.parallel().forEach(System.out::println);
      }
  }
  ```


## 其他问题

1. 异常：stream has already been operated upon or closed

    每次创建的stream只能使用一次，要使用的话 得重新创建

2. 























