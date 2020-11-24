# Redis 持久化

**什么是持久化？**

Redis 所有数据保持在内存中，对数据的更新将**异步地**保存到磁盘中

**持久化方式**

- 快照。存储某一时的数据 （时刻）

  - 例子： MySQL dump 和 Redis RDB

- 写日志。记录日志的变化 （时间段）

  - 例子：MySQL binlog 和 Redis AOF

### RDB

**什么是RDB？**

在 Redis 中，通过对内存数据做一个快照，并持久化到硬盘中的RDB文件。可以通过RDB文件载入到内存中，恢复到某一时刻的状态。如图 1

- RDB 文件是二进制
- RDB 文件存储在硬盘中
- RDB 是快照模式  
- 复制媒介

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-01.png" style="zoom:75%;" />

<center> 图1 RDB 创建与载入 </center>
#### RDB 触发机制三种方式

1. save（同步）

   - 结构如图2。整个过程同步，数据大容易发生阻塞

   - 文件策略：如果存在老的 RDB 文件，新替换老

   - 复杂度： O(n)

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-02.png)

     <center>图2 save 同步</center>

2. bgsave（异步）

   - 结构如图三。fork() 子进程进行创建 RDB，大多数情况下速度快，不会阻塞。
   - 文件策略和复杂度和 save 一样

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-03.png)

   <center>图3 bgsave 异步</center>

3. 自动

   - 结构图四。只要满足配置中的一项，就会触发自动备份

   - 相关配置

     - save 900 1                             # 900秒内发生一次改变就生成新的RDB文件
     - save 300 10
     - save 60 10000
     - dbfilename dump.rdb        # 生成rdb备份文件名,默认 dump.rdb
     - dir  ./                                      # 存储rdb备份文件的位置 ，默认当前路径
     - stop-writes-on-bgsave-error yes   # 如果 bgsave 发生了错是否停止写入，默认 yes
     - rdbcompression  yes          # rdb 文件是否采用压缩格式 ，默认 yes
     - rdbchecksum yes                 # 是否对rdb文件进行检验 ，默认 yes。
  - 
   
- 最佳配置（相对）
  
  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-05.png)
  
     <center>图 5 最佳配置</center>
   - 触发机制
  
     - 全量配置 。主从结构中自动生成rdb文件；
     - debug reload。如果进行不用清空内存的重启，会触发rdb文件生成
     - shutdown 。执行shutdown 会触发rdb文件生成
  
   - 2
  
     
  
     
  
     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-04.png)
  
     <center>图4 自动生成 RDB文件</center>



**save 和 bgsave**

| 命令   | save             | bgsave                     |
| ------ | ---------------- | -------------------------- |
| IO类型 | 同步             | 异步                       |
| 阻塞？ | 是               | 是（阻塞发生在fork子进程） |
| 复杂度 | O(n)             | O(n)                       |
| 优点   | 不会消耗额外内存 | 不阻塞客户端命令           |
| 缺点   | 阻塞客户端命令   | 需要fork，消耗内存         |

#### RDB 总结

1. RDB 是 Redis 内存到硬盘得快照，用于持久化。
2. save 通常会阻塞 Redis。
3. bgsave 不会阻塞 Redis，但是会 fork 新进程
4. save 自动配置满足任 一项会被执行。
5. 自动配置触发机制不容忽视

**RDB 有什么问题？**

- 耗时、耗性能

  - O(n)数据：耗时
  - fork():消耗内存，copy-on-write策略
  - Disk I/O 性能

- 不可控、丢失数据

  例如：

  | 时间戳 | save                  |
  | ------ | --------------------- |
  | T1     | 执行多个写命令        |
  | T2     | 满足RDB自动创建得条件 |
  | T3     | 再次执行多个写命令    |
  | T4     | 宕机                  |
  | T5     | 数据丢失              |

  

### AOF

**AOF 是什么？**

AOF 是 Redis 持久化方式之一，其原理是，每执行一条更改的操作，就会生成对应的日志，并且保存到 AOF 文件中。当需要恢复某一时刻的数据时，加载 AOF 文件到 Redis 中，逐条的运行命令，最后把数据准确的载入回内存。 （ 和数据binlog 类似 ）

#### AOF 的三种策略

> 很像 MySQL 中 sync_binlog 和 innodb_flush_log_at_trx_commit  的策略,可以查看mysql45第23章

- always （总是刷入AOF文件）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-06.png)

  <center>图5 always 结构图</center>

- everysec（每秒策略。时间可以设置，默认 1s）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-07.png)

  <center>图6 everysec 结构图 </center>

- no（由操作系统决定，操作系统觉得什么时候该刷，就刷）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-08.png)

<center>图7 no 结构图 </center>
**对比**

| 命令 | always     | everysec                    | no     |
| ---- | ---------- | --------------------------- | ------ |
| 优点 | 不丢失数据 | 1 / s  fsync，丢失 1 s 数据 | 不用管 |
| 缺点 | IO开销大   | 丢失1s数据                  | 不可控 |

#### AOF 重写

**为什么会有AOF重写？**

- AOF 持久化是通过保存被执行的写命令来记录数据库状态的，所以AOF文件的大小随着时间的流逝一定会越来越大；影响包括但不限于：对于Redis服务器，计算机的存储压力；AOF还原出数据库状态的时间增加；
- 为了解决AOF文件体积膨胀的问题，Redis提供了AOF重写功能：Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个文件所保存的数据库状态是相同的，但是新的AOF文件不会包含任何浪费空间的冗余命令，通常体积会较旧AOF文件小很多。

**重写规则是什么？**

例子：

<table>
    <tr>
        <th>原生AOF</th>
        <th>Savings</th>
  	</tr>
    <tr>
        <td>
            <table>
                <tr><td>set hello world</td></tr>
                <tr><td>set hello java</td></tr>
                <tr><td>set hello hehe</td></tr>
            </table>
        </td>
        <td>set hello hehe</td>
    </tr>
    <tr>
        <td>
            <table>
                <tr><td>incr counter</td></tr>
                <tr><td>incr counter</td></tr>
            </table>
        </td>
        <td>set counter 2</td>
    </tr>
    <tr>
        <td>
            <table>
                <tr><td>rpush mylist a</td></tr>
                <tr><td>rpush mylist b</td></tr>
                <tr><td>rpush mylist c</td></tr>
            </table>
        </td>
        <td>rpush mylist a b c</td>
    </tr>
    <tr>
        <td>
            <table>
                <tr><td>过期数据</td></tr>
            </table>
        </td>
        <td> </td>
    </tr>
</table>

**AOF 重写作用**

- 减少硬盘占用量
- 加快恢复速度

**AOF重写的两种方式**

- **bgrewriteaof**  (类似 RDB 中的bgsave，也会开启子进程)

  其中图9 中的 “AOF 重写” 实际上是将Redis中的内存进行一次回溯，回溯成AOF文件，并不是像上面说的将AOF文件进行抽象地重写成新的文件，而是在内存中进行重写。

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-09.png)

  <center>图9 bgrewriteaof 重写流程</center>

- AOF 重写配置 (**重写流程的本质还是用 “bgrewriteaof”** )
  - 什么时候重写

    - auto-aof-rewrite-min-size            # AOF文件多大了进行重写
    - auto-aof-rewrite-percentage       # AOF 文件增长率

  - 统计

    - aof_current_size                            # AOF当前大小（字节）
    - aof_base_size                                 # AOF 上次启动和重写的大小

  - 什么时候进行重写？（必须**同时满足**下面两个条件）

    1. aof_current_size  >auto-aof-rewrite-min-size
    2. aof_current_size - aof_base_size / aof_base_size > auto-aof-rewrite-percentage
#### AOF 重写流程

​    如图10

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/05-10.png)

<center>图10 AOF 重写流程</center>
#### AOF 配置

- appendonly yes                      # 启动 AOF 模式
- appendfilename “appendonly-${port}.aof”    #  aof文件名
- appendfsync everysec             # aof 的策略
- dir /bigdiskpath                         # aof文件持久化路径
- no-appendfsync-on-rewrite yes  # 不允许在AOF重写过程中进行追加操作，推荐 yes
- auto-aof-rewrite-min-size 64mb   # 超过64mb进行重写 （必须和auto-aof-rewrite-percentage同时满足）
- auto-aof-rewrite-percentage 100 # 文件增长率100%进行重写（必须和auto-aof-rewrite-min-size 的同时满足）

### RDB 和 AOF 抉择

#### RDB 和AOF 对比

| 命令                                 | RDB    | AOF      |
| ------------------------------------ | ------ | -------- |
| 启动优先级(宕机后加载数据选择优先级) | 低     | 高       |
| 体积                                 | 小     | 大       |
| 恢复速度                             | 快     | 慢       |
| 数据安全性                           | 丢数据 | 策略决定 |
| 轻重                                 | 重     | 轻       |

#### RDB “最佳策略 ”

- 建议无论主从数据库 “关闭” RDB
- 集中管理（如果按天备份数据 RDB很合，因为文件小，传输速度快，管理有优势）
- 主从，从开？（有时需要在从节点开启RDB，因为需要保存历史记录，控制力度）

#### AOF“最佳策略”

- 建议开启AOF（everysec）
- AOF重写集中管理（如果单机多部署，fork集中发生。分配给60%-70%给redis，给20%-30%给fork）
- 建议 everysec

#### 最佳策略

- 小分片（设置memory 为 4G，这样fork的量小，这样内存压力小）
- 缓存或者存储（小分片 会产生更多的线程，CUP会增加压力）
- 监控（硬盘、内存、负载、网络）
- 足够内存（不要全部分配给redis）

