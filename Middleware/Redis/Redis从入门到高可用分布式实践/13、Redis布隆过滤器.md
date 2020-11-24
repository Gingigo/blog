# Redis 布隆过滤器

### 大数据验证是否存在的问题

- 问题：现在又50亿个电话号码，现在有10万个电话号码，要快速准确判断这些号码是否已经存在？
  - 通过数据库查询：实现快速有点难
  - 数据预放在集合中：50亿*8字节 = 40GB （内存浪费 或者不够）
  - hyperloglog：准确有点难（存在误差率）

- 类似的问题

  - 垃圾邮件过滤
  - 稳住处理软件（例如word）错误单次检测
  - 网络爬虫重复url检测
  - Hbase 行过滤

### 布隆过滤器原理

- 原理图

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/13-01.png)

- 构建
  - 参数：
    - m个二进制向量
    - n个预备数据
    - k 个hash函数
  - 构建布隆过滤器
    - n个预备数据 执行 k 个hash函数落在m个二进制向量中，值就标记为1.
  - 判断元素是否存在
    - 重新走一遍上面流程，如果都是1，则表明存在，否则，不存在

### 误差率

- 肯定存在误差率：恰好都命中了（布隆过滤认为存在，不一定存在，但是认为不存在，就一定不存在）

- 直观因素：m/n 的比率，hash 函数的个数

- 实际误差率公式

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/13-02.png)



### 本地布隆过滤器

- 现有库：guava

  - 问题：

    - 容量受限制（tomcat 或者 jvm的限制）

    - 多个应用存在多个布隆过滤器，构建复杂（数据共享有问题和共享session的问题一样）

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/13-03.png)



### Redis 布隆过滤器

- 原理：基于位图实现

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/13-04.png)

- 实现方法：
  - 定义布隆过滤器构造参数：m、n、k、误差概率
  - 定义布隆过滤器的操作函数：add 和 contain
  - 封装Redis位图操作
  - 开发测试样例
- 问题：
  - 速度慢：比本地慢，输在网络。
    - 解决：单机部署，单机部署，与应用同机房甚至机架
  - 容量受限制：Redis最大字符串为 512MB 和 Redis 单机容量。
    - 解决：基于Reids Cluster 实现

### Redis 分布式布隆过滤器

- 需要基于Redis 布隆过滤器进行二次过滤
- 基于pipeline提高效率

