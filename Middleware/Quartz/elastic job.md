# Elastic Job

## Quartz 的不足：

- 作业只能通过 DB 抢占随机负载，无法协调
- 任务不能分片
- 作业日志可视化监控统计



## Elastic Job 

- 分布式协调度协调：用 ZK 实现注册中心
- 错过执行作业重触发（Misfire）
- 支持并行调度（任务分片）
- 作业分片一致性，保证同一分片在分布式环境中仅一个执行实例
- 弹性扩容缩容
- 失效转移 failover
- 支持作业生命周期操作
- 丰富的作业类型
- 运维平台

### 三种任务类型

1. SimpleJob：简单实现，未经过任何封装的类型。需要实现 SimpleJob 接口。
2. DataFlowJob：用于处理数据流，必须实现 fetchData() 和 processData() 的方法，一个用来获取数据，一个用来处理获取到的数据。
3. ScriptJob：脚本作业类型，支持 shell，python、perl 等所有类型的脚本

### 配置

- 需要启动 ZK 注册中心
  - config 节点：存储任务的配置信息，包含执行类，cron 表达式，分片算法类，分片数量，分片参
    数等等
  - instances 节点：同一个 Job 下的 elastic-job 的部署实例。一台机器上可以启动多个 Job 实例，也就
    是 Jar 包。instances 的命名是 IP+@-@+PID。只有在运行的时候能看到。
  - leader 节点：任务实例的主节点信息，通过 zookeeper 的主节点选举，选出来的主节点信息。在
    elastic job 中，任务的执行可以分布在不同的实例（节点）中，但任务分片等核心控制，
    需要由主节点完成。
    - election：主节点选举
    - sharding：分片
    - failover：失效转移
  - servers 节点：任务实例的信息，主要是 IP 地址，任务实例的 IP 地址
  - sharding 节点：任务的分片信息，子节点是分片项序号，从 0 开始。分片个数是在任务配置中设置
    的
    - instance：执行该分片项的作业运行实例主键
    - running：分片项正在运行的状态
    - failover：如果该分片项被失效转移分配给其他作业服务器，则此节点值记录执行此分片的作业服务器 IP
    - misfire：是否开启错过任务重新执行
    - disabled：是否禁用此分片项
- 三部曲（ Core - Type - Lite）
  - **JobCoreConfiguration** :用于提供作业核心配置信息，如：作业名称、CRON 表达式、分片总数等。
  - **JobTypeConfiguration**:有 3 个子类分别对应 SIMPLE, DATAFLOW 和 SCRIPT 类型作业，提供 3 种作
    业需要的不同配置，如：DATAFLOW 类型是否流式处理或 SCRIPT 类型的命令行等。Simple 和 DataFlow 需要指定任务类的路径。
  - **JobRootConfiguration**:有 2 个子类分别对应 Lite 和 Cloud 部署类型，提供不同部署类型所需的配
    置，如：Lite 类型的是否需要覆盖本地配置或 Cloud 占用 CPU 或 Memory
    数量等

### 分片策略

任务分片，是为了实现把一个任务拆分成多个子任务，在不同的 ejob 示例上执行。

- AverageAllocationJobShardingStrategy，基于平均分配算法的分片策略，也是默认的分片策略
- OdevitySortByNameJobShardingStrategy，根据作业名的哈希值奇偶数决定 IP 升降序算法的分片策略
- RotateServerByNameJobShardingStrategy，根据作业名的哈希值对服务器列 表进行 轮转 的分片 策略
- 自定义分片策略，实现 JobShardingStrategy 接口并实现 sharding 方法。

























