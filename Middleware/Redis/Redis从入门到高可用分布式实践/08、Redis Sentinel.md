# Redis Sentinel

### 主从复制高可用

两个作用：

1. 为主节节点提供备份。当主节点宕机时，从节点会有数据的完整副本。
2. 为主节点提供读的分流。实现读写分离，可以减轻主节点的读压力。

存在问题：

1. 手动故障转移。就是一旦主节点出现故障，那么故障转移基本上是需要手动完成的。
2. 写能力和存储能力受限制。写只能写在一个节点上，而且存储也是一个节点上存储，从节点只是主节的备份，数据存储大小还是受限制主节点。

如果主从复制的架构中，主节点发生故障，整个过程需要进行手工的切换，这样非常麻烦，而且容易出现操作失误。所以 Redis 为我们提供了 **Redis Sentinel**

### Redis Sentinel (哨兵模式)

哨兵模式是一种特殊的模式，首先 Redis 提供了哨兵的命令，哨兵是一个独立的进程，作为进程,它会独立运行。其原理是**哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例。** 

如下图是 Redis Sentinel 基本架构：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-01.png)

<center>图 1 Redis Sentinel 架构</center>
如果一个 sentinel 对 Redis 服务器进行监控，可能会出现问题，为此，我们可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式。 （高可用）

其中 sentinel 有两个作用：

- 通过发送命令，让Redis服务器返回监控其运行状态，包括主服务器和从服务器。
- 当 sentinel 监测到master宕机，会自动将slave切换成master，然后通过**发布订阅模式**通知其他的从服务器，修改配置文件，让它们切换主机。

**sentinel 模式下的故障切换流程：**

1. 多个sentinel发现并确认master有问题 （高可用，只有多个才能 sentinel发现问题，才能发起投票，确定 master 是否有问题）
2. 选举出一个sentinel作为领导（因为要领导节点要主导后续的故障切换）
3. 选出一个slave作为master
4. 通知其余slave成为新的master的slave
5. 通知客户端主从变化 （客户端从 sentinel 获取即可）
6. 等待老的master复活成为新master的slave

> 1. 每隔1秒，每个Sentinel节点会向主节点、从节点、其余Sentinel节点发送一条ping命令做一次心跳检测，来确认这些节点当前是否可达。当这些节点超过down-after-milliseconds没有进行有效回复，Sentinel节点就会判定该节点为主观下线。
>
> 2. 如果被判定为主观下线的节点是主节点，该Sentinel节点会通过sentinel is master-down-by-addr命令向其他Sentinel节点询问对主节点的判断，当超过<quorum>个数，Sentinel节点会判定该节点为客观下线。如果从节点、Sentinel节点被判定为主观下线，并不会进行后续的故障切换操作。
>
> 3. 对Sentinel进行领导者选举，由其来进行后续的故障切换（failover）工作。选举算法基于Raft。
>
> 4. Sentinel领导者节点开始进行故障切换。
>
> 5. 选择合适的从节点作为新主节点。
>
> 6. Sentinel领导者节点对上一步选出来的从节点执行slaveof no one命令让其成为主节点。
>
> 7. 向剩余的从节点发送命令，让它们成为新主节点的从节点，复制规则和parallel-syncs参数有关。
>
> 8. 将原来的主节点更新为从节点，并将其纳入到Sentinel的管理，让其恢复后去复制新的主节点。
>
> 参考于： https://www.cnblogs.com/ivictor/p/9755065.html 

### 安装配置

**步骤**：

1. 配置开启主从节点
2. 配置开启 sentinel 监控主节点。（sentinel 是特殊的 redis）
3. 实际应该是多台机器（高可用，分配在不同的机器）
4. 详细配置节点

**最终目标架构：**

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-02.png)

<center>图2 配置目标架构</center>
**各个组件的配置情况：**

- 主节点

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-03.png)

  <center>图 3 主节点配置图</center>

- 从节点

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-04.png)

  <center>图 4 从节点配置图</center>

- sentinel 主要配置

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-05.png)

  <center>图 5 sentinel 主要配置</center>
- 注意：
    1. 这里没有给出具体的端口，是应为，要配置三个，26379、26380、26381，自己带进去即可
  - sentinel 配置说明：
    1. 第 4 个配置。意思是sentinel 监控 名字为 mymster的节点，该节点的 ip 为 127.0.0.1，端口7000，其中的 2 是代表至少有2个sentinel 认为 mymaster 有问题，才能执行故障转移。
    2. 第 5 个配置。意思是 sentinel监控 mymaster 发现 30000毫秒都没有转成正常，就认为该节点故障。
    3. 第 6 个配置。当故障转移后，会选择新的master 节点，这时从节点需要重新同步主节点，这里的 1 就是代表同步复制的从节点个数（一个个复制，这样能减少master 的压力）。
    4. 第 7 个配置。代表的是故障转移时间。
  

### 客户端

**客户端实现的基本原理：**

1. 客户端发送 masterName 返回Sentinel节点集合，遍历 Sentinel 节点集合，获取一个可用的 Sentinel 节点；
2. 在可用的 Sentinel 节点上执行 `sentinel get-master-addr-by-name masterName`,返回master 节点的 ip 和 port；
3. 在返回的master节点上执行 role 或者 role replication 验证是否是master节点；
4. 如果 master 节点发生了变化，Sentinel 是能感知地到的，然后通知到客户端（其实是发布订阅模式,客户端订阅了 Sentinel 监控信息频道，如果 master发生了变化，就通知客户端改变到新的master节点 ）。

**客户端接入流程参数：**

1. Sentinel地址集合
2. masterName

注意：sentinel 采用的不是代理模式（master向sentinel拿数据，sentinel 再去master拿数据），而且代理模式在这里的话性能差，redis 是快速著称的，不适合。

**Jedis**

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-06.png)

<center>图 6 sentinel 在 Jedis 的基本调用</center>
### 故障转移

#### 三个定时

1. 每10秒每个sentinel对master和slave执行info

   - 发现 slave 节点（虽然sentinel 没有直接对 sslave节点之间监控，但是通过在 master 节点上执行 info replication 获取 slave 节点的信息）

   - 确认主从关系 （确认所有节点的角色，会对每一个节点执行 info）

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-07.png)

     <center>图 7 每10秒执行 info</center>

2. 每2秒每个sentinel 通过master节点的channel交换信息（pub/sub）

   - 通过 \_\_sentinel\_\_:hello 频道交互

   - 交互对节点的“看法”和自身信息

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-08.png)

     <center>图 8 每2秒进行信息交换</center>

3. 每1秒每个 sentinel 对其他 sentinel 和 redis 执行ping

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/08-09.png)

   <center>图 9 每1秒进行ping</center>

#### 主观下线和客观下线

**两个重要的参数**：

sentinel monitor \<masterName\> \<ip\> \<port\> \<quorum\> # sentinel 监控，名为msterName 的节点。quorum 表示启动故障转移的法定人数。

sentinel down-after-milliseconds \<masterName\> \<timeout\>  # sentine 监控名为masterName的主节点及其从节点，在timeout的时间内，未回应，就主观认为该节有故障。

- 主观下线：每个 Sentinel 节点对 Redis 节点失败的“偏见”。 这个Sentinel会向同时监控这个主服务器的所有其他Sentinel发送查询命令，判断它们是否也任务主服务器下线(包括主观下线和客观下线) 
- 客观下线：所有 Sentinel 节点对 Redis 节点失败“达成共识”（超过quorum个统一）

> slave 节点主观下线就可以了，而master节点需要做故障转移，所以需要主观下线。

#### 领导者选举（发生在认为客观下线之后）

- 原因：只需要一个 Sentinel 节点来完成故障转移（所以多个sentinel就要选择一个出来完成故障转移负责人）
- 选举：通过执行 Sentinel is-master-down-by-addr 命令，希望成为领导者
  - 作用一：和其他 Sentinel 交换master节点是否出现故障，确认下线判定。
  - 作用二：咨询其他的Sentinel节点，希望 sentinel 节点中的领导者，完成master节点的故障转移
- 选举流程：
  1. 每个主观下线的 Sentinel 节点向其他 Sentinel 节点发送命令，要求将它设置为其他领导者。
  2. 收到命令的 Sentinel 节点 如果没有同意其他 Sentinel 节点发送的命令，那么将同意该请求，否则拒绝。
  3. 如果 Sentinel 节点发现自己的票数超过Sentinel集合半数且超过quorum，那么它将成为领导者（所以建议将 Sentinel 的节点设置为 >= 3,的奇数）。
  4. 如果此过程有多个 Sentinel 节点成为了领导者，那么将等待一段时间重新进行选举。

#### 故障转移（sentinel 领导者节点完成）

流程：

1. 从 slave 节点中选出一个 “适合的” 节点作为新的 master 节点

2. 对上面的 slave 节点执行 slaveof no one 命令成为 master 节点。

3. 向剩余的 slave 节点发送命令，让它们成为新 master 节点的 slave 节点，辅助规则和 parallel-syncs

   > parallel-syncs：表示同时给多少个 slave 节点同步数据。其实已经是做了优化了，导出一份RDB文件，但是，网络传输就是同时传输。这样会占用一定的网络资源，可能会影响 master 性能

4. 更新对原来 master 节点配置为 salve，并保持着对其“关注”，当其恢复后命令复制新的 master 节点。

选择“合适的” slave 节点：

1. 选择 slave-priority（slave 节点优先级）最高的 slave节点，如果存在则返回，不存在则继续（一般不设置）

   > 应用：假设有两个 slave 节点，slave A节点的物理机器好一点，slave B节点的物理机器差一点。这时我们可以设置 slave A 比slave B的 slave-priority 更高，这样优先选择 slave A

2. 选择复制偏移量最大的 slave 节点（复制的最完整），如果存在则返回，不存在则继续

3. 选择 runid 最新的 slave 节点 (启动最早的节点)

### 运维问题

#### 节点运维

##### 节点下线

**下线原因**

- 机器下线：例如过保等情况
- 机器性能不足：例如 CPU、内存、硬盘、网络等
- 节点自身故障：例如服务不稳定等

**主节点下线**

做手工下线转移，在任意一个sentinel 节点执行sentinel failover \<masterName\>。因为这是手工转移执行下线，所以没有主观下线、客观下线和领导者选举，自己进行 master 节点转移

**从节点下线**（savle 和 sentinel 节点）

临时下线还是永久下线，例如是否做一些清理工作。但是要考虑读写分离的情况。

Sentinel 节点同上。

##### 节点上线

- 主节点：sentinel failover 进行替换。
- 从节点：slaveof 即可。
- sentinel 节点：参考其他 sentinel 节点配置启动即可。



### 小结

- Redis Sentinel 是 Redis 的高可用实现方案：故障发现、故障自动转移、配置中心、客户端通知。
- Redsi Sentinel 是 Redis 2.8 开始正式可以投入生产环境中的。
- 尽可能在不同的物理上部署 Redis Sentinel 所有节点
- Redis Sentinel 节点 >=3 ,最好为奇数。
- Redis Sentinel 的数据节点和普通节点没有什么区别。
- 客户端初始化时连接的是 Sentinel 节点集合，不再是具体的Redis 节点，但，Sentinel 只是配置中心，不是代理。
- Redis Sentinel 通过三个定时任务实现了 Sentinel 节点对于主节点、从节点、其余 Sentinel节点的监控
- Reids Sentinel 在对主节点失败判定时，分为主观下线和客观下线。
- 看懂 Redis Sentinel 故障转移日志对于理解 Redis Sentinel 以及排除问题非常有帮助。
- Redis Sentinel 实现读写分离高可用可以依赖 Sentinel 节点的消息通知，获取 Redis 节点的状态变化。