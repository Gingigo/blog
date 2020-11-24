# zookeeper *quorum*模式

## 准备

1、三台安装了 zookeeper 的服务器

> 并且强烈建议您使用奇数个服务器,如果您只有两个服务器，那么您处于这样一种情况: 如果其中一个服务器出现故障，那么就没有足够的机器来形成多数选举。两台服务器本质上比单台服务器更不稳定，因为存在两个单点故障。

## 安装

安装 zookeeper 可以参考[《zookeeper单机环境安装》](./zookeeper单机环境安装)

其中配置文件有些差异（先举个例子）

```shell
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

- tickTime:  zookeeper 使用的以毫秒为单位的基本时间单位（zookeeper中的一个时间单元）。它用于执行心跳，**最小会话**超时为 tickTime 的两倍。
- dataDir: 存储内存数据库快照的位置，以及(除非另有指定)数据库更新的事务日志。
- clientPort: 侦听客户端连接的端口。
- initLimit：follower 在启动过程中，会从 leader 同步所有最新数据，然后确定自己能够对外服务的起始状态。leader 允许F在 initLimit 时间内完成这个工作。（随着集群越大，数据同步会越慢，有必要适当调大这个参数了）其中 5 代表 tickTime * 5 = 10000ms
- syncLimit：在运行过程中，leader 负责与 zookeeper 集群中所有机器进行通信，例如通过一些心跳检测机制，来检测机器的存活状态。如果 leader 发出心跳包在 syncLimit 之后，还没有从follower 那里收到响应，那么就认为这个 follower 已经不在线了
- server.X ： 
  - 其中 zoo1 表示的是 zookeeper 所在的服务器 ip/域名；
  - 第一个端口 2888 是用来表示 *quorum* 通信的端口（更具体地说 zookeeper 服务器使用这个端口将 follower 连接到 leader ）
  - 第二个端口 3888 是用于 leader 选举（当一个新的领导者出现时，跟随者使用这个端口打开一个到领导者的 TCP 连接。因为默认的领导者选举也使用 TCP，我们目前需要另一个端口进行领导者选举）

## 准备配置配置文件

我们是用单机下的集群,以下是需要准备的目录和文件

**准备存储路径**

```shell
[root@localhost zookeeper]# tree quorum-data/
quorum-data/
├── d2181
│   ├── myid
├── d2182
│   └── myid
└── d2183
    └── myid
```

这里有个特备重要的的点，<font color='orange' >在 quorum 模式下，需要在指定的存储路径中添加`myid`，其中 myid 的为唯一性的数值</font>。我这里分别是 1，2，3.

其中，一下三个文件是放在`/usr/local/zookeeper/conf` 目录下的

**zoo_2181.cfg**

```shell
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/quorum-data/d2181
clientPort=2181
server.1=localhost:3333:3334
server.2=localhost:4444:4445
server.3=localhost:5555:5556
```

**zoo_2182.cfg**

```shell
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/quorum-data/d2182
clientPort=2182
server.1=localhost:3333:3334
server.2=localhost:4444:4445
server.3=localhost:5555:5556
```

**zoo_2182.cfg**

```shell
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/quorum-data/d2183
clientPort=2183
server.1=localhost:3333:3334
server.2=localhost:4444:4445
server.3=localhost:5555:5556

```

## 启动集群

```shell
zkServer.sh start  zoo_2181.cfg
```

启动第一个节点会抛出异常，这个是正常的，因为我们在配置项中配置了三个节点，但是目前只有一个节点。认为他和其他两节点建立不了连接，所以报错。

```shell
zkServer.sh start  zoo_2182.cfg
```

```shell
zkServer.sh start  zoo_2183.cfg
```

## 验证

```shell
ps -ef |grep 2181
```

```shell
ps -ef |grep 2182
```

```shell
ps -ef |grep 2183
```

如果**都有**输出一大坨代码，哈哈，就启动成功了,一下是执行`ps -ef |grep 2182`输出的

```shell
root      3425     1  0 17:01 pts/0    00:00:01 java -Dzookeeper.log.dir=/usr/local/zookeeper/bin/../logs -Dzookeeper.log.file=zookeeper-root-server-localhost.localdomain.log -Dzookeeper.root.logger=INFO,CONSOLE -XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError=kill -9 %p -cp /usr/local/zookeeper/bin/../zookeeper-server/target/classes:/usr/local/zookeeper/bin/../build/classes:/usr/local/zookeeper/bin/../zookeeper-server/target/lib/*.jar:/usr/local/zookeeper/bin/../build/lib/*.jar:/usr/local/zookeeper/bin/../lib/zookeeper-prometheus-metrics-3.6.1.jar:/usr/local/zookeeper/bin/../lib/zookeeper-jute-3.6.1.jar:/usr/local/zookeeper/bin/../lib/zookeeper-3.6.1.jar:/usr/local/zookeeper/bin/../lib/snappy-java-1.1.7.jar:/usr/local/zookeeper/bin/../lib/slf4j-log4j12-1.7.25.jar:/usr/local/zookeeper/bin/../lib/slf4j-api-1.7.25.jar:/usr/local/zookeeper/bin/../lib/simpleclient_servlet-0.6.0.jar:/usr/local/zookeeper/bin/../lib/simpleclient_hotspot-0.6.0.jar:/usr/local/zookeeper/bin/../lib/simpleclient_common-0.6.0.jar:/usr/local/zookeeper/bin/../lib/simpleclient-0.6.0.jar:/usr/local/zookeeper/bin/../lib/netty-transport-native-unix-common-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-transport-native-epoll-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-transport-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-resolver-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-handler-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-common-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-codec-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/netty-buffer-4.1.48.Final.jar:/usr/local/zookeeper/bin/../lib/metrics-core-3.2.5.jar:/usr/local/zookeeper/bin/../lib/log4j-1.2.17.jar:/usr/local/zookeeper/bin/../lib/json-simple-1.1.1.jar:/usr/local/zookeeper/bin/../lib/jline-2.11.jar:/usr/local/zookeeper/bin/../lib/jetty-util-9.4.24.v20191120.jar:/usr/local/zookeeper/bin/../lib/jetty-servlet-9.4.24.v20191120.jar:/usr/local/zookeeper/bin/../lib/jetty-server-9.4.24.v20191120.jar:/usr/local/zookeeper/bin/../lib/jetty-security-9.4.24.v20191120.jar:/usr/local/zookeeper/bin/../lib/jetty-io-9.4.24.v20191120.jar:/usr/local/zookeeper/bin/../lib/jetty-http-9.4.24.v20191120.jar:/usr/local/zookeeper/bin/../lib/javax.servlet-api-3.1.0.jar:/usr/local/zookeeper/bin/../lib/jackson-databind-2.10.3.jar:/usr/local/zookeeper/bin/../lib/jackson-core-2.10.3.jar:/usr/local/zookeeper/bin/../lib/jackson-annotations-2.10.3.jar:/usr/local/zookeeper/bin/../lib/commons-lang-2.6.jar:/usr/local/zookeeper/bin/../lib/commons-cli-1.2.jar:/usr/local/zookeeper/bin/../lib/audience-annotations-0.5.0.jar:/usr/local/zookeeper/bin/../zookeeper-*.jar:/usr/local/zookeeper/bin/../zookeeper-server/src/main/resources/lib/*.jar:/usr/local/zookeeper/bin/../conf: -Xmx1000m -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /usr/local/zookeeper/bin/../conf/zoo_2182.cfg
root      3557  1298  0 17:06 pts/0    00:00:00 grep --color=auto 2182

```

## 连接

```shell
zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

-  server 指定需要连接的 ip + port，可以连接集群中的多个，其中每个服务的 ip + port 需要以 `,` 隔开

输出日志：

```shell
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:java.io.tmpdir=/tmp
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:java.compiler=<NA>
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:os.name=Linux
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:os.arch=amd64
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:os.version=4.18.0-147.el8.x86_64
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:user.name=root
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:user.home=/root
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:user.dir=/usr/local/apache-zookeeper-3.6.1-bin/conf
2020-07-25 17:12:40,289 [myid:] - INFO  [main:Environment@98] - Client environment:os.memory.free=23MB
2020-07-25 17:12:40,291 [myid:] - INFO  [main:Environment@98] - Client environment:os.memory.max=247MB
2020-07-25 17:12:40,291 [myid:] - INFO  [main:Environment@98] - Client environment:os.memory.total=29MB
2020-07-25 17:12:40,297 [myid:] - INFO  [main:ZooKeeper@1005] - Initiating client connection, connectString=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@379619aa
2020-07-25 17:12:40,300 [myid:] - INFO  [main:X509Util@77] - Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation
2020-07-25 17:12:40,304 [myid:] - INFO  [main:ClientCnxnSocket@239] - jute.maxbuffer value is 1048575 Bytes
2020-07-25 17:12:40,314 [myid:] - INFO  [main:ClientCnxn@1703] - zookeeper.request.timeout value is 0. feature enabled=false
2020-07-25 17:12:40,322 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1154] - Opening socket connection to server localhost/127.0.0.1:2181.
2020-07-25 17:12:40,322 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1156] - SASL config status: Will not attempt to authenticate using SASL (unknown error)
2020-07-25 17:12:40,325 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@986] - Socket connection established, initiating session, client: /127.0.0.1:58564, server: localhost/127.0.0.1:2181
Welcome to ZooKeeper!
JLine support is enabled
2020-07-25 17:12:40,393 [myid:127.0.0.1:2181] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1420] - Session establishment complete on server localhost/127.0.0.1:2181, session id = 0x10000e25bd70001, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
```

## 测试

```shell
ls -R /
```

输入内容

```shell
/
/zookeeper
/zookeeper/config
/zookeeper/quota
```

我们可以从启动日志中发现这样一条日志,证明是客户端连接的是 2181 这个服务。

```shell
Opening socket connection to server localhost/127.0.0.1:2181.
```

我们需要验证是否正常切换 leader 节点，所以我们先打开一个新的窗口先停掉 2181 这个服务

窗口2：

```shell
[root@localhost conf]# zkServer.sh stop zoo_2181.cfg
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo_2181.cfg
Stopping zookeeper ... STOPPED
```

窗口1，会收到一个通知

窗口1：

```shell
[zk: 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183(CONNECTED) 1] 2020-07-25 17:22:54,049 [myid:127.0.0.1:2181] - WARN  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1272] - Session 0x10000e25bd70001 for sever localhost/127.0.0.1:2181, Closing socket connection. Attempting reconnect except it is a SessionExpiredException.
EndOfStreamException: Unable to read additional data from server sessionid 0x10000e25bd70001, likely server has closed socket
	at org.apache.zookeeper.ClientCnxnSocketNIO.doIO(ClientCnxnSocketNIO.java:75)
	at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:348)
	at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1262)

WATCHER::

WatchedEvent state:Disconnected type:None path:null
2020-07-25 17:22:54,433 [myid:127.0.0.1:2182] - INFO  [main-SendThread(127.0.0.1:2182):ClientCnxn$SendThread@1154] - Opening socket connection to server localhost/127.0.0.1:2182.
2020-07-25 17:22:54,433 [myid:127.0.0.1:2182] - INFO  [main-SendThread(127.0.0.1:2182):ClientCnxn$SendThread@1156] - SASL config status: Will not attempt to authenticate using SASL (unknown error)
2020-07-25 17:22:54,434 [myid:127.0.0.1:2182] - INFO  [main-SendThread(127.0.0.1:2182):ClientCnxn$SendThread@986] - Socket connection established, initiating session, client: /127.0.0.1:37532, server: localhost/127.0.0.1:2182
2020-07-25 17:22:54,442 [myid:127.0.0.1:2182] - INFO  [main-SendThread(127.0.0.1:2182):ClientCnxn$SendThread@1420] - Session establishment complete on server localhost/127.0.0.1:2182, session id = 0x10000e25bd70001, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null

```

从日志中，可以看出，2181 服务停止，客户端连接了 2182 服务。

窗口1：

```shell
[zk: 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183(CONNECTED) 1] ls -R /
/
/zookeeper
/zookeeper/config
/zookeeper/quota
[zk: 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183(CONNECTED) 2] 
```



遇到的问题

- 配置文件需要放到 conf 目录，否则会报错

  ```shell
  [root@localhost quorum]# zkServer.sh start-foreground  zoo_2182.cfg 
  /usr/bin/java
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/bin/../conf/zoo_2182.cfg
  grep: /usr/local/zookeeper/bin/../conf/zoo_2182.cfg: No such file or directory
  grep: /usr/local/zookeeper/bin/../conf/zoo_2182.cfg: No such file or directory
  mkdir: cannot create directory ‘’: No such file or directory
  2020-07-25 16:59:51,781 [myid:] - INFO  [main:QuorumPeerConfig@173] - Reading configuration from: /usr/local/zookeeper/bin/../conf/zoo_2182.cfg
  2020-07-25 16:59:51,789 [myid:] - ERROR [main:QuorumPeerMain@98] - Invalid config, exiting abnormally
  org.apache.zookeeper.server.quorum.QuorumPeerConfig$ConfigException: Error processing /usr/local/zookeeper/bin/../conf/zoo_2182.cfg
  	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.parse(QuorumPeerConfig.java:197)
  	at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:124)
  	at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:90)
  Caused by: java.lang.IllegalArgumentException: /usr/local/zookeeper/bin/../conf/zoo_2182.cfg file is missing
  	at org.apache.zookeeper.server.util.VerifyingFileFactory.doFailForNonExistingPath(VerifyingFileFactory.java:54)
  	at org.apache.zookeeper.server.util.VerifyingFileFactory.validate(VerifyingFileFactory.java:47)
  	at org.apache.zookeeper.server.util.VerifyingFileFactory.create(VerifyingFileFactory.java:39)
  	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.parse(QuorumPeerConfig.java:179)
  	... 2 more
  Invalid config, exiting abnormally
  2020-07-25 16:59:51,790 [myid:] - INFO  [main:ZKAuditProvider@42] - ZooKeeper audit is disabled.
  2020-07-25 16:59:51,792 [myid:] - ERROR [main:ServiceUtils@42] - Exiting JVM with code 2
  
  ```

  - 解决方法: 将配置文件移动到  conf 目录下

- 在集群模式下存储路径缺少 myid 文件 

  ```shell
  myid file is missing 
  ```

  - 解决: 添加 myid 文件，并且填入唯一的数字

