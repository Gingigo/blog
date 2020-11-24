# Redis 开发规范

### key 设计

- key名设计

  - 三大建议

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-01.png)

  - embstr

    - ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-02.png)

    > redis 有三种表达字符串的编码，分别是 int、embstr、raw。
    >
    > 查看编码命令是`object encoding key`
    >
    > redis 3种是39个字节 ，redis 4中是44个字节
    >
    > int ： 如果数据是整数就用int类型
    >
    > embstr：如果数据小于等于39个字符就用 embstr（可以连续redis Object 和 value 连续内存，而且是一次分配，这样就到达了减少分配次数和节省空间）
    >
    > raw：如果数据大于39 就用 raw

### value 设计

- bigkey

  - 强制:

    - string 类型控制在   10 KB 以内
    - hash、list、set、zset 元素个数不要超过5000

  - 危害

    - 网络阻塞
    - Redis阻塞（慢查询：hgetall、lrange、zrange）
    - 集群节点数据不平衡
    - 频繁序列化：应用服务CPU消耗

  - 发现

    - 应用异常（调用超时）
    - redis-cli --bigkeys （官方）
    - scan + debug object
    - 主动报警：网络流量监控、客户端监控

  - 删除

    - 阻塞：注意隐形删除（过期、raname）（不推荐）

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-03.png)

    - Redis 4.0: lazy delete(unlink 命令) 推荐

    - Redis 4.0 之前的要怎么删除

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-04.png)

  - 预防

    - 优化数据结构：例如二级拆分（按天、hash处理）
    - 物理隔离或者万兆网卡：不是治标方案
    - 命令优化：例如hgetall->hmget、hscan
    - 报警和定期优化

  - 总结

    - 牢记Redis单线程特性
    - 选择合理的数据结构和命令
    - 清楚自身OPS
    - 了解bigkey危害

### 合适的数据结构

- 实体类型（数据结构内存优化：例如ziplist，注意内存和性能的平衡）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-05.png)

- 一个例子三个方案（序曲 picId=>userId(100w)）

  - 方案一：全部 string ： set picId userId

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-06.png)

  - 方案二：一个hahs： hset allPics picId userId

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-07.png)

  - 方案三：若干个小hash： hset picId/100 picId%100 userId

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-08.png)

  - 对比

    - 内存对比

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-09.png)

    - 内存分析( ziplist 性能和存储消耗上做均衡)

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-10.png)

    - 优缺点

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/14-11.png)

### 键值生命周期

- Redis 不是垃圾桶
  - 周期数据需要设置过期时间，通过 object idle time 可以找出垃圾 key-value
  - 过期时间不宜集中：缓存穿透和雪崩等问题



### 命令使用技巧

1. 【推荐】O(N)以上命令关注N的数量
   - 例如 hgetall、lrange、smembers、zrange、sinter、等并非不能使用，但是需要明确N的值。有遍历的需求可使用hscan、sscan、zscan代替。
2. 【推荐】禁用命令
   - 禁止线上使用keys、flushall、flushdb等，通过redis的rename机制禁掉命令，或者使用scan的方式渐进式处理。
3. 【推荐】合理的使用select
   - redis的多数据库较弱，使用数字进行区分
   - 很多客户端支持较弱
   - 同时多业务用多数据库实际还是单线程处理，会有干扰
4. 【推荐】Redis事务功能较弱，不建议过多使用
   - Redis的事务功能较弱（不支持回滚），可以借助其他如java中的spring事务
   - 而且集群版本要求一次事务操作的key必须在一个slot上（可以使用hashtag功能解决，但是会造成key不均匀）
5. 【推荐】Redis集群版本在使用Lua上有特殊要求
   - 所有key，必须在1个slot上，否则直接返回error
6. 【建议】必要情况下使用monitor命令时，要注意不要长时间使用

### Java客户端优化

1. 【推荐】避免多个个应用使用一个Redis实例
   - 正例：不相干的业务拆分，公共数据做服务化
2. 【推荐】使用连接池，标准使用方式

