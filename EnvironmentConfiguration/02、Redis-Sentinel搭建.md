# Redis Sentinel 搭建

本教程参考于，《一站式学习Redis，从入门到高可用分布式实践》中的第8章《Redis Sentinel》

集群搭建不涉及原理简介，需要了解原理可以复习该章节。

### 运行环境

- centos7 (本次是单机搭建，生产环境需要多台物理机)
- Redis 5.0.8

### 环境安装

#### centos7安装

自行安装centos 7 系统 这里就不在说明了。

#### 安装Redis

1.  下载 Redis ，现在的最新版本为 5.0.8。当前是在`/usr`下

   - 下载： `wget http://download.redis.io/releases/redis-5.0.8.tar.gz`

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_01.png)

2. 解压安装包

   - 解压：`tar -xvf redis-5.0.8.tar.gz`

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_02.png)

3. 建立软连接

   - 原因：方便管理和升级

   - 建立软连接：`ln -s redis-5.0.8 redis`

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_03.png)

4. 编译

   - 进入 redis ，执行编译。命令：`cd redis`，`make`。

   - <font color='red'>注意：这里可能会出现错误，如果无法编译请查看本文最后的 “可能出现的问题” 并寻找解决方法</font>。

5. 安装

   - 安装：`make install`

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_04.png) 
   
    
   
6.  启动

    1.  启动方式1：`redis-server` ,默认端口为 6379

        ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_05.png)

    2.  启动方式2：`redis-server --port 6380`,指定端口启动

        ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_06.png)

    3.  启动方式3：通过配置文件启动，也是我们推荐的启动方式，方便管理。`redis-server redis-xxxx.conf`

        - 在安装的redis目录中新建 data 文件。`mkdir data` 方便我们存储数据
        
        - 在安装的redis目录中新建 config 文件。`mkdir config`
- 复制 redis 目录中 redis.conf 文件到 config/ 目录中。`cp redis.conf ./config`
  
    - 进入 config 目录。`cd config`
    
    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_07.png)
    
    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_08.png)

### 开始搭建 Redis Sentinel

#### 最终的目标

##### 架构图

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_10.png" style="zoom:75%;" />

<center>Redis Sentinel 最终搭建目标架构图</center>
##### 实现功能点

- Setinel 集群
- Redis 主从复制
- Sentinel 集群实现 Redis 高可用（故障自动转移）

#### redis.conf 配置

**我们主要是体验搭建 Redis Sentinel,所以很多细枝末节的配置就没用添加进去，这个是基本的配置。**

在 redis/config 中新建以下配置

##### Redis 主从配置

- **Redis 主节点**： redis-7000.conf

  1.  新建 redis-7000.conf ：`cp redis-conf redis-7000.conf`

  2. 修改配置 redis-7000.conf：

     ```properties
     # redis 启动端口
     port 7000
     # 以守护进程的方式启动
     daemonize yes
     # redis 的进程id
     pidfile /var/run/redis-7000.pid
     # redis 日志文件名
     logfile "7000.log"
     # redis 数据存储位置
     dir "/usr/redis/data"
     ```

- **Redis 从节点1**：redis-7001.conf

  1. 新建 redis-7001.conf ：`cp redis-7000.conf redis-7002.conf`

  2. 修改配置 redis-7001.conf：

     ```properties
     # redis 启动端口
     port 7001
     # 以守护进程的方式启动
     daemonize yes
     # redis 的进程id
     pidfile /var/run/redis-7001.pid
     # redis 日志文件名
     logfile "7001.log"
     # redis 数据存储位置
     dir "/usr/redis/data"
     # 主从复制
     slaveof 127.0.0.1 7000
     ```
- **Redis 从节点2**：redis-7002.conf

  1. 新建 redis-7002.conf ：`cp redis-7001.conf redis-7002.conf`

  2. 修改配置 redis-7002.conf：

     ```properties
     # redis 启动端口
     port 7002
     # 以守护进程的方式启动
     daemonize yes
     # redis 的进程id
     pidfile /var/run/redis-7002.pid
     # redis 日志文件名
     logfile "7002.log"
     # redis 数据存储位置
     dir "/usr/redis/data"
     # 主从复制
     slaveof 127.0.0.1 7000
     ```
  

##### 启动 redis 主从

启动命令：

```she
redis-server redis-7000.conf
redis-server redis-7001.conf
redis-server redis-7002.conf
```

验证并查看主从信息:`redis-cli -p 7000 info replication`

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_11.png)

##### redis sentinel 配置

- **redis sentinel 节点1**：redis-sentinel-26379.conf

  - 新建 redis-sentinel-26389.conf ，可从 /urs/redis/redis-sentinel.conf 复制到   /urs/redis/config/redis-sentinel.conf 。在config 目录下执行`cp ../redis-sentinel.conf ./redis-sentinel-26389.conf `

  - 修改配置 redis-sentinel-26389.conf：

    ```properties
    port 26379
    daemonize yes
    pidfile /var/run/redis-sentinel-26379.pid
    logfile "26379.log"
    dir /usr/redis/data
    # 监控节点，且超过2个sentinel 任务故障，方可执行故障转移
    sentinel monitor mymaster 127.0.0.1 7000 2
    # 如果节点在 30000毫秒内未回应，就认为故障
    sentinel down-after-milliseconds mymaster 30000
    # 如果故障转移后，同时进行主从复制数为 1
    sentinel parallel-syncs mymaster 1
    # 故障转移的超时时间
    sentinel failover-timeout mymaster 180000
    sentinel deny-scripts-reconfig yes
    ```

- **redis sentinel 节点2**：redis-sentinel-26380.conf

  - 复制 `sed "s/26379/26380/g" redis-sentinel-26379.conf > redis-sentinel-26380.conf`

  - 修改配置 redis-sentinel-26380.conf：

    ```properties
    port 26380
    daemonize yes
    pidfile /var/run/redis-sentinel-26380.pid
    logfile "26380.log"
    dir /usr/redis/data
    # 监控节点，且超过2个sentinel 任务故障，方可执行故障转移
    sentinel monitor mymaster 127.0.0.1 7000 2
    # 如果节点在 30000毫秒内未回应，就认为故障
    sentinel down-after-milliseconds mymaster 30000
    # 如果故障转移后，同时进行主从复制数为 1
    sentinel parallel-syncs mymaster 1
    # 故障转移的超时时间
    sentinel failover-timeout mymaster 180000
    sentinel deny-scripts-reconfig yes
    ```
- **redis sentinel 节点3**：redis-sentinel-26381.conf

  - 复制 `sed "s/26379/26381/g" redis-sentinel-26379.conf > redis-sentinel-26381.conf`

  - 修改配置 redis-sentinel-26381.conf：

    ```properties
    port 26381
    daemonize yes
    pidfile /var/run/redis-sentinel-26381.pid
    logfile "26381.log"
    dir /usr/redis/data
    # 监控节点，且超过2个sentinel 任务故障，方可执行故障转移
    sentinel monitor mymaster 127.0.0.1 7000 2
    # 如果节点在 30000毫秒内未回应，就认为故障
    sentinel down-after-milliseconds mymaster 30000
    # 如果故障转移后，同时进行主从复制数为 1
    sentinel parallel-syncs mymaster 1
    # 故障转移的超时时间
    sentinel failover-timeout mymaster 180000
    sentinel deny-scripts-reconfig yes
    ```
    

##### 启动 redis sentinel

- 启动命令：	

  ```shell
  redis-sentinel redis-sentinel-26379.conf
  redis-sentinel redis-sentinel-26380.conf
  redis-sentinel redis-sentinel-26381.conf
  ```

- 验证：`ps -ef | grep redis-sentinel`

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_12.png)

至此 Redis 主从和 Redis Sentinel 已经搭建完成了。接下来验证故障转移。

### 故障转移演示

1. 编写 Java 客户端

   ```java
   package dddy.gin.practice;
   
   import lombok.extern.slf4j.Slf4j;
   import redis.clients.jedis.Jedis;
   import redis.clients.jedis.JedisSentinelPool;
   
   import java.util.HashSet;
   import java.util.Random;
   import java.util.Set;
   import java.util.concurrent.TimeUnit;
   
   /**
    * 演示故障转移
    */
   @Slf4j
   public class RedisSentinelFailOver {
       private static final String MASTER_NAME = "mymaster";
   
       public static void main(String[] args) {
           Set<String> sentinels = new HashSet<>();
           sentinels.add("127.0.0.1:26379");
           sentinels.add("127.0.0.1:26380");
           sentinels.add("127.0.0.1:26381");
           JedisSentinelPool sentinelPool = new JedisSentinelPool(MASTER_NAME, sentinels);
   
           while (true) {
               Jedis jedis = null;
   
               try {
                   TimeUnit.MILLISECONDS.sleep(1000);
                   jedis = sentinelPool.getResource();
                   int index = new Random().nextInt(100000);
                   String key = "k-" + index;
                   String value = "v-" + index;
                   jedis.set(key, value);
                   log.info("{} value is {}", key, value);
               } catch (Exception e) {
                   log.warn(e.getMessage());
               } finally {
                   if (jedis != null)
                       jedis.close();
               }
           }
   
       }
   }
   ```

2. 查看当前 Redis 集群的主节点 `redis-cli -p 26379 info`

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_15.png)

3. 启动 Java 客户端（这里可能报错，如果有错误请查看本文最后“可能出现的问题”）

   <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_16.png" style="zoom: 53%;" />

4. 模拟主节点意外宕机（通过`redis-cli -p 7000 info server`查看进程id，在通过 `kill -9 进程id`）

   <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_17.png" style="zoom:75%;" />

5. 查看 Java 客户端日志

   <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_13.png" style="zoom:65%;" />

6. 等待 Redis Sentinel 故障转移完成

   <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_14.png" style="zoom:54%;" />
   
7. 查看当前 Redis 集群的主节点 `redis-cli -p 26379 info`,已经转移了主节点到7001（或者7002）

   <img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/redis/sentinel/redis_sentinel_18.png" style="zoom:90%;" />

至此，整个故障转移已经演示完毕。

### 可能出现的问题

- **未安装 gcc**

  - 现象：

    ```shell
    MAKE hiredis
    cd hiredis && make static
    make[3]: 进入目录“/usr/redis-5.0.8/deps/hiredis”
    gcc -std=c99 -pedantic -c -O3 -fPIC  -Wall -W -Wstrict-prototypes -Wwrite-strings -g -ggdb  net.c
    make[3]: gcc：命令未找到
    make[3]: *** [net.o] 错误 127
    make[3]: 离开目录“/usr/redis-5.0.8/deps/hiredis”
    make[2]: *** [hiredis] 错误 2
    make[2]: 离开目录“/usr/redis-5.0.8/deps”
    make[1]: [persist-settings] 错误 2 (忽略)
        CC adlist.o
    /bin/sh: cc: 未找到命令
    make[1]: *** [adlist.o] 错误 127
    make[1]: 离开目录“/usr/redis-5.0.8/src”
    make: *** [all] 错误 2
    ```

  - 原因：Redis 是 C 语言开发，编译，安装之前必先确认是否安装 gcc 环境（gcc -v） 

  - 解决方法：安装 gcc 。`yum install -y gcc `


- **分配器allocator， 中MALLOC  这个 环境变量选择的默认库导致的** 


    - 现象：
    
      ```shell
      cd src && make all
      make[1]: 进入目录“/usr/redis-5.0.8/src”
          CC Makefile.dep
      make[1]: 离开目录“/usr/redis-5.0.8/src”
      make[1]: 进入目录“/usr/redis-5.0.8/src”
          CC adlist.o
      In file included from adlist.c:34:0:
      zmalloc.h:50:31: 致命错误：jemalloc/jemalloc.h：没有那个文件或目录
       #include <jemalloc/jemalloc.h>
                                     ^
      编译中断。
      ```


​      


    -  原因：分配器，默认的是 jemalloc ，而你的环境jemalloc ，而只有 libc。
    -  解决：手动选择分配器。`make` 替换为 `make MALLOC=libc` 即可。

-  Jedis 中的保护模式


   -  现象

      ```shell
      [main] INFO redis.clients.jedis.JedisSentinelPool - Redis master running at 127.0.0.1:7000, starting Sentinel listeners...
      [main] INFO redis.clients.jedis.JedisSentinelPool - Created JedisPool to master at 127.0.0.1:7000
      [main] WARN dddy.gin.practice.RedisSentinelFailOver - DENIED Redis is running in protected mode because protected mode is enabled, no bind address was specified, no authentication password is requested to clients. In this mode connections are only accepted from the loopback interface. If you want to connect from external computers to Redis you may adopt one of the following solutions: 1) Just disable protected mode sending the command 'CONFIG SET protected-mode no' from the loopback interface by connecting to Redis from the same host the server is running, however MAKE SURE Redis is not publicly accessible from internet if you do so. Use CONFIG REWRITE to make this change permanent. 2) Alternatively you can just disable the protected mode by editing the Redis configuration file, and setting the protected mode option to 'no', and then restarting the server. 3) If you started the server manually just for testing, restart it with the '--protected-mode no' option. 4) Setup a bind address or an authentication password. NOTE: You only need to do one of the above things in order for the server to start accepting connections from the outside.
      [main] WARN dddy.gin.practice.RedisSentinelFailOver - java.net.SocketException: Software caused connection abort: recv failed
      ```

   -  原因：redis 中没有关闭保护模式

   -  解决：在 redis 中，添加多一个配置 `protected-mode no`，然后重新启动 redis 服务