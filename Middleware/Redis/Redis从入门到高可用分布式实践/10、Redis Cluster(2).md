# Redis Cluster(2)

## 集群伸缩

### 伸缩原理

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-01.png" style="zoom:75%;" />

<center>图1 集群伸缩图</center>
- 缩的情况：对6381和6384从节点中下线，将所管理的槽分配给集群中的节点。可能是因为节点异常或者其他情况。将

- 伸的情况：对6385节点加入集群中，集群中的节点抽出一部分slots 分配给新加入的节点。可能是扩容、或者替换某些异常节点的情况。

**集群伸缩的原理就是槽和数据在节点之间的移动 。**

### 扩容集群

- 准备节点

  - 集群模式
  - 配置和其他节点统一
  - 启动后是孤儿节点

- 加入集群

  - 加入集群的命令
    - `cluster meet ip port`
  - 验证节点是否加入命令
    - `cluster nodes`
  - 作用
    - 为其他迁移槽和数据实现扩容
    - 作为从节点负责故障转移
  - 加入集群 - redis-trib.rb
    - 命令 `redis-trib.rb add node new_host:new_port existing_host existing_port --slave master-id <arg>`
    - 在生产环境中建议用 redis-trib.rb能避免新节点已经加入了其他集群，造成故障。

- 添加从节点

- 迁移槽和数据

  - 槽迁移计划

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-02.png)

  - 迁移数据

    - 原理

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-03.png)

    - 流程图

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-04.png)

### 缩容集群

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-05.png" style="zoom:50%;" />

- 下线迁移槽

  - 节点下的槽转移到其他节点

    <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-06.png" style="zoom:50%;" />

- 忘记节点

  - 先下从节点再下主节点

  - 命令

    `cluster forget downNodeId`

  - 对所有节点都执行该命令

    <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-07.png" style="zoom:80%;" />

- 关闭节点



## 客户端路由

### moved 重定向

**为什么会有moved 重定向？**

因为在集群模式下，数据的存放是按照槽存放的，而每一个节点管理其中的部分槽，当客户端连接其中一个节点时，如果操作的key 落在该节点上则直接操作即可，否则，就会返回key落在的某个节点的信息，再重定向去操作。

- moved 重定向流程图**

  <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-08.png" style="zoom: 67%;" />

- **槽命中**

  <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-09.png" style="zoom:67%;" />

  - 可以通过命令 `cluster keysslot key`获取key对应的槽

- **槽不命中（moved 异常）**

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-11.png)

- **redis-cli -c 集群方式连接** 

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-12.png)

### ask 重定向

**为什么需ask重定向？**

在集群模式中，当节点A 正在迁移槽到节点 B，这时候如果有数据存入节点A，就会回复ask重定向到节点B。保证了数据不会丢失和效率高。

- ask重定向流程图

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-13.png)

- move 和 ask 对比

  - 相同
    - 两者都是客户端的重定向
    - 
  - 不同
    - moved ： 槽已经迁移；ask ：槽迁移中

### smart 客户端

**为什么需要smart 客户端？**

假设集群有1000个节点。我们不可能一个个去试那个节点可以操作 key，这样性能太差了。

- smart 客户端实现原理

  - 目标：追求性能

  - 原理

    1. 从集群中选一个可运行节点，使用 cluster slots 初始化槽和节点映射

    2. 将 cluster slots 的结果映射到本地，为每一个节点创建JedisPool

    3. 执行命令

       1. 通过 crc(16) mod 16384计算出槽，从JedisPool 中选合适直连的Jedis
       2. 如果连接成功就返回结果，否则随机选取任意一个节点执行命令，返回move 重定向信息，之后更新cluster slots 初始化槽和节点映射缓存。
       3. 循环执行第 2 步骤，如果超过5次未成功，就返回 `Too many cluster redirection`

       ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-14.png)

    4. 其中要注意处理好moved 和 ask 重定向异常。

- smart 客户端使用 JedisCluster

  - JedisCluster 基本使用

    - 使用技巧

      1. 单例：内置了所有节点的连接池。
      2. 无需手动借还连接池。
      3. 合理设置 commons-pool。

    - 例子：

      ```java
    package dddy.gin.practice;
      
      ```
  
    import lombok.extern.slf4j.Slf4j;
      import redis.clients.jedis.HostAndPort;
    import redis.clients.jedis.Jedis;
      import redis.clients.jedis.JedisCluster;
      import redis.clients.jedis.JedisPoolConfig;
  
      import java.util.ArrayList;
      import java.util.HashSet;
      import java.util.List;
      import java.util.Set;
  
      /**
       * redis 集群工厂类
          */
        @Slf4j
        public class JedisClusterFactory {
          private JedisCluster jedisCluster;
          List<String> nodes = new ArrayList();
      
          public JedisClusterFactory() {
              nodes.add("127.0.0.1:9000");
              nodes.add("127.0.0.1:9001");
              nodes.add("127.0.0.1:9002");
              nodes.add("127.0.0.1:9003");
              nodes.add("127.0.0.1:9004");
              nodes.add("127.0.0.1:9005");
          }
      
          public void init(){
              JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
              Set<HostAndPort> nodeSet = new HashSet();
              nodes.stream().forEach(node ->{
                  String[] arr = node.split(":");
                  if (arr.length>1){
                      nodeSet.add(new HostAndPort(arr[0],Integer.parseInt(arr[1])));
                  }
              });
              jedisCluster = new JedisCluster(nodeSet,PracticeConfig.TIMEOUT,jedisPoolConfig);
          }
          public void destroy(JedisCluster jedisCluster){
              if (jedisCluster!=null){
                  jedisCluster.close();
              }
          }
      
          public static void main(String[] args) {
              JedisClusterFactory factory =  new JedisClusterFactory();
              factory.init();
              JedisCluster jedisCluster = factory.jedisCluster;
              jedisCluster.set("hello","world");
              log.info(jedisCluster.get("hello"));
              factory.destroy(jedisCluster);
          }
  
      }
  
      ```
    
      ```
  
  - 多节点命令实现
  
    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-15.png)
  
  - 批量命令实现
  
    - mget、mset 必须在同个槽中
  
    - 四种批量优化方法
  
      1. 串行mget（简单的for循环key 一个一个的查找value）
  
         ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-16.png)
  
      2. 串行IO（循环遍历key，通过 crc(16)/16384 进行分组，不同的组再去不同的节点取value）
  
         ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-17.png)
  
      3. 并行IO(采用多线程 对串行IO进行优化)
  
         ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-18.png)
  
      4. hash_tag （当一个key包含 {} 的时候，就不对整个key做hash，而仅对 {} 包括的字符串做hash）
  
         ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-19.png)
  
    - 四种方法对比
  
      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-20.png)
  



## 故障转移

### 故障发现

原理：

- 通过 ping/pong 消息实现故障发现：不需要sentinel

  - ping/pong消息是节点之间信息交互，比如包含节点槽的消息和节点故障的消息等。

- 主观下线和客观下线

  - 主观下线：某个节点认为另一个节点不可用，“单个节点的认知”

  - 主观下线流程：

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-21.png)

  - 客观下线：当半数以上持有槽的主节点都标记某个节点主观下线

  - 客观下线流程：

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-22.png)

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-23.png)

### 故障恢复

- 资格检查

  - 每个从节点检查与故障主节点的断线时间。如果超过了 `cluster-node-timeout * cluster-slave-validity-factor`的值就取消资格，其中 `cluster-node-timeout` 的默认值是15000毫秒，`cluster-slave-validity-factor` 默认值是10。如果都是用默认配置超过150秒，就没有资格成为主节点的可能。

- 准备选举时间

  - 不同的从节点数据延迟是不一样的，当主节点出现了故障，为了保证数据的完整性，准备的选举时间也是不同的，与主节点同步的偏移量高的延迟就低，更快进入选举状态，与主节点同步偏移量低的延迟就高，更慢的进入选举状态。这样做是为了同步偏移量高的更有可能成为主节点。

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-24.png)

- 选举投票

  - 超过半数的主节点投票就可以成为主节点。偏移量高的，因选举时间长，更有可能获得更多的票数，更有可能成为主节点

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-25.png)

- 替换主节点

  - 流程：
    1. 当前从节点取消复制变为主节点（slaveof no one）
    2. 执行`clusterDelSlot`撤销主节点负责的槽，并执行`clusterAddSlot`包这些槽分配给自己
    3. 向集群广播自己的`pong`消息，表明已经替换了故障节点，成为主节了。

### 故障转移演练

## 开发和运维常见的问题

### 集群完整性

- 参数：`cluster-require-full-coverage` 默认是 `yes` 。意思是是否集群的所有节点都在线且16384个槽都分配的状态，才对外提供服务。
  - 集群中16384个槽全部可用：保证了数据完整性
  - 节点故障或者正在转移故障：<font color='red'>(error) CLUSTERDOWN the cluster is dwon</font>
- 到多少的业务无法容忍，<font color='red' >建议设置成 no</font>

### 带宽开削

集群的消息模式是点对点的（P2P），节点之间会进行Ping/Pong 消息交换,这样随着集群的扩大，就会产生较大的带宽消耗。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-26.png)

- 官方建议：集群的大小为最大为1000个节点
- Ping/Pong 消息
- 不容忽视的带宽消耗

- 影响带宽的三个方面：
  - 消息发送的频率：节点发现与其它节点最后通信时间超过 `cluster-node-timeout/2`就会之间发送ping 消息。
  - 消息数据量：slots槽数组(2kb空间) 和整个集群 1/10 的状态 数据（10个节点状态约为1kb）
  - 节点部署的机器规模：集群分布的机器越多且每台机器划分的节点越均匀，则集群内整体的可用带宽越高。(如90 个节点分布在3台集群，带宽高)。
- 优化：
  - 避免"大"集群，避免多个业务使用同一个集群，大业务可用多集群
  - cluster-node-timeout：带宽和故障转移速度的均衡
  - 尽量均匀分配到多台机器上：保证高可用和宽带

### Pub/Sub 广播

在集群模式下，对一个节点发布消息，集群内的节点都会订阅到发布的消息，这样会有一个问题就是节点的带宽开销会很大

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/10-27.png)

- 问题：publish 在集群每个节点广播，加重带宽 
- 解决：单独走一套 Redis Sentinel 

### 集群倾斜

- 数据倾斜：内存不均

  - 可能的原因：

    - 节点和槽分配不均匀

      - `redis-trib.rb info ip:port` 查看节点、槽、键值分布
      - `redis-trib.rb rebalance ip:port` 进行均衡（谨慎使用）

    - 不同槽对应键值数量差异较大

      - crc16正常情况下比较均匀
      - 可能存在hash_tag
      - `cluster countkeysinslot {slot}`获取槽对应键值个数

    - 包含bigkey

      - bigkey：例如大字符串、几百万的元素的hash、set等
        - 如何找bigkey：在从节点中执行`redis-cli --bigkeys`
      - 优化bigkey：优化数据结构，大集合拆分成小集合

    - 内存相关配置不一致（如数据压缩）

      - `hash-max-ziplist-value`、`set-max-intset-entries`等配置不同

      - 优化：定期”检查“配置一致性

        

- 请求倾斜：热点

  - 热点key：重要的key 或者 bigkey
  - 优化：
    - 避免bigkey
    - 不用用hash_tag 
    - 当一致性不高时，可以用本地缓存+MQ

### 读写分离

- 只读连接：集群模式的节点不接受任何读写请求。
  - 重定向到负责槽的主节点
  - readonly 命令可以读：连接级别命令
- 读写分离：更加复杂
  - 同样的问题：复杂延迟、读取过期数据、从节点故障
  - 修改客户端：cluster slaves {nodeId}

### 数据迁移

官方迁移工具：redis-trib.rb import

- 只能从单节点迁移到集群
- 不支持在线迁移：source需要停写
- -不支持断点续传
- 单线程迁移：影响速度

在线迁移：

- 唯品会 ：redis-migrate-tool
- 豌豆荚： redis-port

### 集群 vs 单机

- 集群的限制
  - key批量操作支持有限：例如mget、mset必须在同一个slot
  - key事务和Lua支持有限：操作的key必须在同个节点
  - key是数据分区的最小粒度：不支持bigkey分区
  - 不支持多个数据库：集群模式下只有一个数据库db 0
  - 复杂只支持一层：不支持树型复制结构（主从复制）
- 分布式Redis不一定好
  - Redis Cluster：满足容量和性能的扩展性，很多业务”不需要“（QPS达不到怎么高）
    - 大多数客户端性能会”降低“
    - 命令无法使用跨节点使用：mget、mset、keys、scan、flush、sinter等
    - Lua和事务无法跨节点使用
    - 客户端维护更加复杂：SDK和应用本身消耗（例如更多的连接池）
- 很多场景Redis Sentinel 已经最高好了