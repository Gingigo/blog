# MySQL是怎么保证主备一致的？

在 MySQL 中主备结构是保证数据运行高可用的基础，那怎么来保证主备的数据一致呢？

答案是 **binlog** ，接下来我们就来看一下 binlog 是怎么实现主备一致的。

### MySQL 主备的基本原理

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-24-01.png)

<center>图 1 MySQL 主备切换流程</center>
在状态1中，客户端写数据到节点A，生成A节点的binlog，A节点将数据推送给B节点（A节点的备库），这样就起到了A、B 节点数据一致了。<font color='orange'>虽然写的操作是在A节点(主库)，但是我们建议把B节点（备库）设置成Readonly，因为防止B节点误操作修改数据，和有时可通过readonly判断主备库。</font>

> 你或许会问，节点B 设置成 readonly会不会不能备份数据？
>
> 其实不会，因为 readonly 设置对超级用户是无效的，而用于同步更新的线程，就是拥有超级权限

**实例**

节点 A 到 B 这条线的内部流程是什么样的，图 2 中画出的就是一个 update 语句在节点 A 执行，然后同步到节点 B 的完整流程图。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-24-02.png)

<center>图 2 主备流程图</center>
备库 B 跟主库 A 之间维持了一个长连接。主库 A 内部有一个线程，专门用于服务备库 B 的这个长连接。一个事务日志同步的完整过程是这样的： (重要！)

1. 在**备库** B 上通过 change master 命令，设置主库 A 的**IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog**，这个位置包含文件名和日志偏移量。
2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 **io_thread 和 sql_thread**。其中 io_thread 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给 B。(<font color='orange'>这里需要注意的是 数据是主库推送给备库的！</font>)
4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）
5. sql_thread 读取中转日志，**解析**出日志里的命令，并执行

### binlog 的三种格式

- **statement**
  - 格式：binlog“忠实”地记录了 SQL 命令，甚至连注释也一并记录了（一模一样）
  - 缺点：在执行一些语句的时候，可能会导致binlog在备库上执行得不到和主库一样的逻辑
  - 优点：占用空间少
- **row**
  - 格式：SQL 语句进行了转换，保证了**执行的逻辑**和主库一模一样
  - 缺点：很占空间 （生成的 binlog 数据多 ）
  - 优点：确保了 binlog 从主库同步过来，执行的逻辑一定相同
- **mixed**
  - 格式：MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式
  - 缺点：日志复杂（两种混合在一起）
  - 优点：整合了补足了以上两种的缺点

**为什么建议 binlog 格式设置成 row？**

因为，方便对数据进行恢复。如果你我操作进行增删改，binlog 入设置成 row，它就会记录详细的日志，让你能恢复数据（这也是他为什么日志多的原因，记得太细）

###  循环复制问题 

我们可以认为正常情况下主备的数据是一致的。也就是说，图 1 中 A、B 两个节点的内容是一致的。其实，图 1 中我画的是 M-S 结构，但实际生产上使用比较多的是双 M 结构，也就是图 3 所示的主备切换流程

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-24-03.png)

<center>图 3 MySQL 主备切换流程 -- 双 M 结构</center>
对比图 9 和图 1，你可以发现，双 M 结构和 M-S 结构，其实区别只是多了一条线，即：节点 A 和 B 之间总是互为主备关系。这样在切换的时候就不用再修改主备关系。

但是，双 M 结构还有一个问题需要解决。

业务逻辑在节点 A 上更新了一条语句，然后再把生成的 binlog 发给节点 B，节点 B 执行完这条更新语句后也会生成 binlog。（建议你把参数 log_slave_updates 设置为 on，表示备库执行 relay log 后生成 binlog）。

那么，如果节点 A 同时是节点 B 的备库，相当于又把节点 B 新生成的 binlog 拿过来执行了一次，然后节点 A 和 B 间，会不断地循环执行这个更新语句，也就是循环复制了。这个要怎么解决呢？

**server id**:表示这个事务是在那个库上执行的（当我们进行M-S 或者M-M的时候会设置不同的**server id**）。

而且在row格式下， binlog 日志中有记录日志对应的 server id 即改变时有那台数据库产生的。 

因此，我们可以用下面的逻辑，来解决两个节点间的循环复制的问题：

1. 从节点 A 更新的事务，binlog 里面记的都是 A 的 server id；
2.  传到节点 B 执行一次以后，节点 B 生成的 binlog 的 server id 也是 A 的 server id； 
3. 再传回给节点 A，A 判断到这个 server id 与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了。



### 疑问

1. 主库 A 从本地读取 binlog，发给从库 B ，binlog 日志时从 page cache还是disk呢 ？

   答：1.如果这个文件现在还在 page cache中，那就最好了，直接读走； 

   ​		2.  如果不在page cache里（已经刷盘了），就只好去磁盘读 。

