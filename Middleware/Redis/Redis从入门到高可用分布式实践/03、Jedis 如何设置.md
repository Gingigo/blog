# Jedis 如何设置

**Jedis 直连**

需要用的时候 new Jedis(),每次都需要进行 TCP 三次握手四次挥手。

- 优点：
  - 简单方便
  - 适用于少量长期连接的场景
- 缺点：
  - 每次新建/关闭TCP开销
  - 资源无法控制，存在连接泄露的可能
  - Jedis 对象线程不安全

**Jedis 连接池**

在容器中先 new 出一定的 Jedis 对象，需要用的时候从容器中取出来，用完了，还回容器中。 

- 优点：
  - Jedis预先生成，降低开销使用；
  - 连接池的形式保护和控制资源的使用。
- 缺点：
  - 相对于直连，使用相对麻烦，尤其在资源的管理上需要很多参数来保证，一旦规划不合理也会出现问题。

**如何使用规范的使用 Jedis 连接池中的对象？**

```java
Jedis jedis = null;
try{
    //1.从连接池中获取jedis对象
    jedis = jedisPool.getResource();
    //2.执行操作
    jedis.set("hello","world");
}catch(Excption e){
    e.printStackTrace();
}finally{
    if(jedis!=null)
        //归还资源到连接池
        jedis.close();
}
```



**Jedis 连接池配置**

*资源数控制*

| 参数名    | 含义                     | 默认值 | 使用建议 |
| --------- | ------------------------ | ------ | -------- |
| maxTotal  | 资源池最大连接数         | 8      |          |
| maxIdle   | 资源池允许最大空闲连接数 | 8      |          |
| minIdle   | 资源池允许最少空闲连接数 | 0      |          |
| jmxEnable | 是否开启jmx监控          | true   | 建议开启 |

*借还参数*

| 参数名             | 含义                                                         | 默认值       | 使用建议 |
| ------------------ | ------------------------------------------------------------ | ------------ | -------- |
| blockWhenExhausted | 当资源池用尽后，调用者是否要等待，只有为true时，下面的maxWaitMillis才会生效 | true         | true     |
| maxWaitMillis      | 当资源池用尽后，调用者的最大等待时间（毫秒）                 | -1：永不超时 | 1000     |
| testOnBorrow       | 向资源池借用连接时是否做连接有效性检测（ping），无效连接会被移除 | false        | false    |
| testOnReturn       | 向资源池归还连接否做连接有效性检测（ping），无效连接会被移除 | false        | false    |

**maxTotal**

举个例子：

1. 命令平均执行时间0.1ms=0.001s；
2. 业务需要 50000 QPS；
3. maxTotal理论值 = 0.001*50000=50个。实际只要偏大一些。

**maxIdle**

建议 maxIdle=maxTotal。

- 减少创新新连接的开销。

**minIdle**

建议预热minIdle。

- 减少第一次启动后新连接开销。



**常见问题**

- Timeout wating for idle objec （连接超时）
- Pool exhausted （资源耗尽）

**解决思路**

1. 慢查询阻塞：池子连接都被hang 住。（解决：设置命令的超时时间）
2. 资源池参数不合理：例如QPS高、池子小 （解决：合理设置参数）
3. 连接泄露（没有 close（），没有归还资源）（解决提高编程质量）
4. DNS异常等