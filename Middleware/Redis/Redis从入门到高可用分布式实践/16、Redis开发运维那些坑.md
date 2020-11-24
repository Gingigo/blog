# Redis 开发运维那些坑

### Linux内核优化

- **overcommit建议设置为1**

  - 原因：防止fork子进程失败，导致主进程阻塞

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-01.png)

  - 操作:

    ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-02.png)

  - 最佳实践

    - Redis 设置合理的maxmemory，保证机器有20%~30%的闲置内存
    - 集中化管理AOF重写和RDB的bgsave
    - 设置vm.overcommit_memory=1,防止极端情况下fork失败

- swappiness建议设置为0

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-03.png)

  - 生效：
    - 立即生效：echo {bestvalue} > /proc/sys/vm/swappiness
    - 永久生效：echo vm.swappiness={bestvalue} >> /etc/sysctl.conf
  - 最佳实践
    - Linux > 3.5 ,vm.swapiness=1，否则 vm.swapiness=0，从而实现俩个目标：
      - 物理内存足够的时候，使 Redis 足够快
      - 物理内存不足时，避免Redis死掉（如果当前Redis为高可用，死掉比阻塞更好）。

- THP建议关闭

  - 作用：加速fork，未开启内存页大小4kb，开启后内存页4M，但是会造成延迟

  - 建议：禁用，可能产生更大的内存开销

  - 设置方法：echo never>/sys/kernel/mm/transparent_hugepage/enabled

  - 坑：源码中是绝对路径，注意不同的发行版本的区别

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-04.png)

- OMM killer

  - 作用：内存使用超出，操作系统按照规则kill掉某些进程
  - 配置方法：/proc/{progress_id}/oom_adj 越小，被杀掉的概率越小
  - 运维经验：不要过度依赖此特性，应该合理管理内存

- NTP服务

  - 作用：统一每个节点时间

- ulimit

  - 作用：分配更多的句柄
  - 命令：ulimit -n

- TCP backlog 

  - 默认值：Redis中tcp-backlog默认值为511
  - 操作系统的 tcp-backlog至少要511
  - 查看方法：`cat /proc/sys/net/core/somaxconn`
  - 修改方法： `echo 511 > /proc/sys/net/core/somaxconn` 

### 安全的Redis

- 设置密码
  - 服务端配置
    - requirepass：节点密码  
    - masterauth：同步主节点也需要输入密码
  - 客户端连接：
    - auth命令和 -a参数
  - 建议：
    - 密码要足够复杂，防止暴力破解
    - masterauth不要忘记
    - auth 还是通过明文传输（https）
- 伪装危险命令
  - 服务端配置
    - rename-command为空或者随机字符
  - 客户端连接
    - 不可用或者使用指定随机字符
  - 建议：
    - 不支持config set动态设置
    - RDB和AOF如果包含rename-command之前的命令，将无法使用
    - config命令本身是再Redis内核使用到，不建议设置
- bind
  - 服务端配置
    - bind限制的是网卡，并不是客户端ip
  - 相关建议
    - bind不支持config set
    - bind 127.0.01需要谨慎
    - 如果存在外网网卡尽量去设置屏蔽
- 防火墙：杀手锏
  - 注意不要把redis屏蔽掉了
- 定期备份
- 不要默认端口
- 非使用root用户启动

### 热点key

- 客户端（定时统计）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-05.png)

- 代理（代理统计）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-06.png)

- 服务端（monitor）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-07.png)

- 机器端（抓包）

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-08.png)

- 对比

  ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/redis/practice/16-09.png)

