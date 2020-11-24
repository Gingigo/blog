# zookeeper 基础

## 概念

zookeeper 是一个开源的<font color='orange'>分布式协调服务系统</font>。Zookeeper 的设计目标是将那些复杂且容易出错的分布式协同服务封装起来，抽象出一个高效的原语集，并以一系列简单的接口提供给用户使用。

## 应用场景

三个著名开源是如何使用 zookeeper ：

- Hadoop：使用 zookeeper 做 Namenode 的高可用；
- HBase：保证集群中只有一个 master ，保存集群中的 RegionServer 列表，保存 hbase:mate 表的位置；
- Kafka：集群成员管理，controller 节点选举。

zookeeper 应用场景

- 配置管理（ configuration management ）；
  - 微服务中的需要集中化的配置管理。
- 命名服务
- DNS 服务
- 组成员管理( group membership )
  - 有机器退出和加入、选举master
- 各种分布式锁

zookeeper 适合于存储和协同相关的关键数据，不适合用于大数据存储。

## zookeeper 数据模型

zookeeper 的数据模型是层次模型（Google Chubby 也是这么做的）。层次模型常见于文件系统。层次模型和 key-value 模型是两种主流的数据模式主要基于两点考虑：

1. 文件系统的树型结构便于表达数据之间的<font color='orange'>层次关系</font>。
2. 文件系统的树型空间便于为不同的应用<font color='orange'>分配独立的命名空间</font>（namespace）。

zookeeper 的层次模型称作 data tree 。Data tree 的每个节点叫作 znode。不同于文件系统，每个节点都<font color='orange'>可以保存数据</font>。每个节点都<font color='orange'>有一个版本</font>（version）。版本从0开始计数。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/zookeeper/geek/z_01_01.png)

<center> 图01  zookeeper 数据结构图</center>

### data tree 接口

zookeeper 对外提供一个用来访问 data tree 的简化文件系统 API：

- 使用 UNIX 风格的路径名来定位 znode；
  - 例如：/A/X 表示 znode A 的子节点 X。
- znode 的数据只支持全量写入和读取；
  - 没有像通用文件系统那样支持部分写入和读取。
- data tree 的所有 API 都是 wait-free 的；
  - wait-free：正在执行中的 API 调用不会影响其他API的完成。
- data tree 的 API 都是对文件系统的 wait-free 操作，不直接提供锁这样的分布式协同机制。
  - 但是 data tree 的 API 非常强大，可以用来实现多种分布式协同机制。

### znode 分类

一个 znode 可以是持久性的、也可以是临时性的；znode 也可以是顺序性的。每个顺序性的 znode 关联一个唯一的单调递增整数。这个单调递增的整数是 znode 名字的后缀。

1. 持久性的 znode （PERSISTENT）：这样的 znode 在创建之后，即使发生 zookeeper 集群宕机或者 client 宕机也不会丢失。（除非手动删除，否则一直存在）
2. 临时性的 znode（EPHEMERAL）：client 宕机或者 client 在指定的 timeout 时间内没有给 zookeeper 集群发送消息，这样的 znode 就会消失。（临时节点的生命周期与客户端会话绑定）
3. 持久顺序性的 znode（PERSISTENT_SEQUENTIAL）: znode 除了具备持久性 znode 的特点之外，znode 的名字还具备顺序性。
4. 临时顺序性的 znode（EPHEMERAL_SEQUENTIAL）: znode 除了具备临时性 znode 的特点之外，znode 的名字具备顺序性

## zookeeper 安装

参考环境配置目录中的《zookeeper单机环境安装》

## zookeeper 基本操作

- 启动服务

  ```shell
  zkServer.sh start
  ```

- 启动客户端

  ```shell
  zkCli.sh
  ```

  - `help` : 对命令简单的介绍

  - `ls -R /`: 递归查找 `/` 下的 znode  备注： `-R` 表示递归

    ```shell
    [zk: localhost:2181(CONNECTED) 1] ls -R /
    /
    /zookeeper
    /zookeeper/config
    /zookeeper/quota
    ```

  - `create /app` 创建 znode

    ```shell
    [zk: localhost:2181(CONNECTED) 2] create /app
    Created /app
    [zk: localhost:2181(CONNECTED) 3] create /app/app1
    Created /app/app1
    [zk: localhost:2181(CONNECTED) 4] create /app/app2
    Created /app/app2
    [zk: localhost:2181(CONNECTED) 5] create /app/app1/p_1
    Created /app/app1/p_1
    [zk: localhost:2181(CONNECTED) 6] create /app/app1/p_2
    Created /app/app1/p_2
    [zk: localhost:2181(CONNECTED) 7] ls -R /app
    /app
    /app/app1
    /app/app2
    /app/app1/p_1
    /app/app1/p_2
    ```

  - create -e /lock 创建临时 znode ，用来做分布式锁

    1. client_1 加锁

       ```shell
       [zk: localhost:2181(CONNECTED) 1] create -e /lock
       Created /lock
       ```

    2. client_2 加锁失败，因为 client_1 锁未释放 

       ```shell
       [zk: localhost:2181(CONNECTED) 8] create -e /lock
       Node already exists: /lock
       ```

    3. client_2 监控 /lock 锁释放

       ```shell
       [zk: localhost:2181(CONNECTED) 9] stat -w /lock
       cZxid = 0x8
       ctime = Thu Jul 23 23:10:48 CST 2020
       mZxid = 0x8
       mtime = Thu Jul 23 23:10:48 CST 2020
       pZxid = 0x8
       cversion = 0
       dataVersion = 0
       aclVersion = 0
       ephemeralOwner = 0x100005e7f6e0001
       dataLength = 0
       numChildren = 0
       ```

    4. client_1 释放锁( 断开 session 连接)

       ```shell
       [zk: localhost:2181(CONNECTED) 2] quit
       2020-07-23 23:16:35,606 [myid:localhost:2181] - WARN  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1266] - An exception was thrown while closing send thread for session 0x100005e7f6e0001.
       EndOfStreamException: Unable to read additional data from server sessionid 0x100005e7f6e0001, likely server has closed socket
       	at org.apache.zookeeper.ClientCnxnSocketNIO.doIO(ClientCnxnSocketNIO.java:75)
       	at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:348)
       	at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1262)
       
       WATCHER::
       
       WatchedEvent state:Closed type:None path:null
       2020-07-23 23:16:35,712 [myid:] - INFO  [main:ZooKeeper@1618] - Session: 0x100005e7f6e0001 closed
       2020-07-23 23:16:35,717 [myid:] - ERROR [main:ServiceUtils@42] - Exiting JVM with code 0
       2020-07-23 23:16:35,722 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@566] - EventThread shut down for session: 0x100005e7f6e0001
       
       ```

    5. client_2 收到一个 WATCHER 通知 

       ```shell
       [zk: localhost:2181(CONNECTED) 10] 
       WATCHER::
       
       WatchedEvent state:SyncConnected type:NodeDeleted path:/lock
       ```

    6. client_2  尝试获得锁成功

       ```shell
       [zk: localhost:2181(CONNECTED) 13] create -e /lock
       Created /lock
       ```



## zookeeper 架构

zookeeper 客户端负责和 zookeeper 集群的交互。集群可以有两种模式

- standalone 模式
  - 处于 standalone 模式的 zookeeper 集群只有一个独立运行的 zookeeper 节点。有单点故障的问题。
- quorum 模式
  - 处于 quorum 模式的 zookeeper 集群包含多个 zookeeper 节点。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/zookeeper/geek/z_01_02.png)

<center>图02  zookeeper 架构图</center>

### Session

zookeeper  客户端和 zookeeper 集群中的某个节点创建一个 session。

- 客户端可以主动关闭 session；
- 如果 zookeeper 节点没有在 session 管理的 timeout 时间内收到客户端的消息，也会关闭 session
- 如果 zookeeper 客户端如果发现连接出错，会自动和其他 zookeeper 节点建立连接。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/zookeeper/geek/z_01_03.png)

<center>图03  zookeeper 重连接图</center>

### quorum 模式

- 集群中有一个 leader 节点，其他的是 follower 节点；
- leader 节点可以处理读写请求，follower 节点可以处理读请求；
- 如果 follower 节点在接收到写请求，会转发给 leader 来处理。

### 数据一致性

- 全局可线性（Linearizable）写入：先到达 leader 的写请求会被先处理，leader 决定写请求的执行顺序。
- 客户端 FIFO 顺序：来自给定客户端的请求按照发送顺序执行（客户端来说，先发送的请求先处理）









