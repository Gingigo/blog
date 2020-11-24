## Redis 特性

**一、 速度快**

- 读写速度： 10w OPS
- 数据存在哪里： 内存
- 什么语言写： C语言（50000 lines）
- 线程模型： 单线程

**二、持久化**

Redis 所有数据保持在内存中，对数据的更新将异步地保持在磁盘上。

**三、多种数据结构**

- String
  - BitMaps
  - HyperLogLog
- Hash Table
- Linked List
- Sets
- Sorted Sets
  - GEO

**四、支持多种语言**

- Java
- PHP
- Python
- Ruby
- Lua
- Node

**五、功能丰富**

- 发布订阅
- Lua 脚本
- 事务
- pipeline

**六、简单 **

- 不依赖于外部地库
- 单线程模型

**七、主从复制**

**八、高可用、分布式**

- 高可用 -> Redis-Sentinel(v2.8) 支持高可用
- 分布式 -> Reidis-Cluster(v3.0) 支持分布式

## 使用场景

1. 缓存系统 ： 

2. 计数器

3. 消息队列

4. 排行榜

5. 社交网络

6. 实时系统

## Redis 常用配置

- deamonize（是否是守护进程）

  默认配置是 no

  建议设置成 yes （因为打印的日志，就能输到指定的位置）

- port

  6379

- logfile

  日志名字

- dir

  日志文件存储的路径

