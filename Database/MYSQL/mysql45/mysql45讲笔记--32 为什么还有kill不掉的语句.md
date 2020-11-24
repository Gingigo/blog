# 32、为什么还有kill不掉的语句？

**MySQL 中两条 kill 命令**

- **kill query 线程id** ： 表示终止这个线程中正在执行的语句；
- **kill (connection) 线程id**：connection 可以省略，表示断开这个线程的连接。(当然如果这个线程有语句正在执行，也要先停止正在执行的语句的)。

**有一种“奇怪”现象**

在使用 MySQL 的时候，有没有遇到过这样的现象：使用了 kill 命令，却没能断开这个连接。再执行 show processlist 命令，看到这条语句的 Command 列显示的是 Killed。

### kill 的原理

kill query 并不是马上停止的意思，而是告诉线程说，这条语句已经不需要继续执行了，可以开始“执行停止的逻辑了”

**kill query  的流程**

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-32-01.png)

<center>图 1 kill query 成功的例子</center>
实现上，当用户执行 kill query thread_id_B 时，MySQL 里处理 kill 命令的线程做了两件事： 

1. 把 session B 的运行状态改成 THD::KILL_QUERY(将变量 killed 赋值为 THD::KILL_QUERY)；（说白了就是所在线程的状态改成 killed）
2.  给 session B 的执行线程发一个信号。  （发一个信号说，执行停止逻辑）

其实，session B 处于锁等待状态，如果只是把 session B 的线程状态设置 THD::KILL_QUERY，线程 B 并不知道这个状态变化，还是会继续等待。发一个信号的目的，就是让 session B 退出等待，来处理这个 THD::KILL_QUERY 状态。

 上面的分析中，隐含了这么三层意思： 

1. 一个语句执行过程中有多处“埋点”，在这些“埋点”的地方判断线程状态，如果发现线程状态是 THD::KILL_QUERY，才开始进入语句终止逻辑；

2. 如果处于等待状态，必须是一个可以被唤醒的等待，否则根本不会执行到“埋点”处；

3. 语句从开始进入终止逻辑，到终止逻辑完全完成，是有一个过程的。

说白了就是，**不是说停就停的**。必须触碰到埋点，才能知道有停止的状态，才能开始执行停止。

 ###  一个 kill 不掉的例子 

首先，执行 set global innodb_thread_concurrency=2，将 InnoDB 的并发线程上限数设置为 2；然后，执行下面的序列：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-32-02.png)

<center> 图 2 kill query 无效的例子 </center>
 可以看到： 

1. sesssion C 执行的时候被堵住了 ; (因为设置了并发上线为2 )
2. 但是 session D 执行的 kill query C 命令却没什么效果 (session C 等待的地方没有“埋点”)
3.  直到 session E 执行了 kill connection 命令，才断开了 session C 的连接，提示“Lost connection to MySQL server during query”;（只是断开了 session C 连接）
4.  但是这时候，如果在 session E 中执行 show processlist，你就能看到下面这个图。 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-32-03.png)

<center> 图 3 kill connection 之后的效果 </center>
这时候，id=12 这个线程的 Commnad 列显示的是 Killed。也就是说，客户端虽然断开了连接，但实际上服务端上这条语句还在执行过程中。

**session D 执行 kill 的流程**

1. 12 号线程的等待逻辑是这样的：每 10 毫秒判断一下是否可以进入 InnoDB 执行，如果不行，就调用 nanosleep 函数进入 sleep 状态。 
2. 虽然 12 号线程的状态已经被设置成了 KILL_QUERY，但是在这个等待进入 InnoDB 的循环过程中，并没有去判断线程的状态，因此根本不会进入终止逻辑阶段

**session E 执行 kill 的流程**

1. 把 12 号线程状态设置为 KILL_CONNECTION；
2. 关掉 12 号线程的网络连接。因为有这个操作，所以你会看到，这时候 session C 收到了断开连接的提示。

那为什么执行 show processlist 的时候，会看到 Command 列显示为 killed 呢？ 其实 ，这就是因为在执行 show processlist 的时候，有一个特别的逻辑：**如果一个线程的状态是KILL_CONNECTION，就把Command列显示成Killed。 **

所以其实，即使是客户端退出了，这个线程的状态仍然是在等待中。那这个线程什么时候会退出呢？

答案是，只有等到满足进入 InnoDB 的条件后，session C 的查询语句继续执行，然后才有可能判断到线程状态已经变成了 KILL_QUERY 或者 KILL_CONNECTION，再进入终止逻辑阶段。

### 无法kill小结

- **kill 无效的第一类情况，即：线程没有执行到判断线程状态的逻辑 **

  - 发生的原因： IO 压力过大，读写 IO 的函数一直无法返回，导致不能及时判断线程的状态。 
  
- **另一类情况是，终止逻辑耗时较长**

  -  超大事务执行期间被 kill。这时候，回滚操作需要对事务执行期间生成的所有新数据版本做回收操作，耗时很长。 
  - 大查询回滚。如果查询过程中生成了比较大的临时文件，加上此时文件系统压力大，删除临时文件可能需要等待 IO 资源，导致耗时较长。
  -  DDL 命令执行到最后阶段，如果被 kill，需要删除中间过程的临时文件，也可能受 IO 资源影响耗时较久。 

###  另外两个关于客户端的误解 

#### 第一个误解是：如果库里面的表特别多，连接就会很慢。

实际上，，当使用默认参数连接的时候，MySQL 客户端会提供一个本地库名和表名补全的功能。为了实现这个功能，客户端在连接成功后，需要多做一些操作：

1.  执行 show databases； 
2.  切到 db1 库，执行 show tables； 
3.  把这两个命令的结果用于构建一个本地的哈希表。 

在这些操作中，**最花时间的就是第三步在本地构建哈希表的操作**。所以，当一个库中的表个数非常多的时候，这一步就会花比较长的时间。

我们感知到的连接过程慢，其实并不是连接慢，也不是服务端慢，而是客户端慢。 

**措施**

- 如果在连接命令中加上 -A，就可以关掉这个自动补全的功能，然后客户端就可以快速返回了。
- 加–quick(或者简写为 -q) 参数，也可以跳过这个阶段

但是，这个**–quick 是一个更容易引起误会的参数，也是关于客户端常见的一个误解**。

-quick 可能加快了客户端，但是有可能会降低服务端的性能。

MySQL 客户端发送请求后，接收服务端返回结果的方式有两种： 

1.  一种是本地缓存，也就是在本地开一片内存，先把结果存起来。如果你用 API 开发，对应的就是 mysql_store_result 方法。 （默认）
2. 另一种是不缓存，读一个处理一个。如果你用 API 开发，对应的就是 mysql_use_result 方法。

如果加上 -quick 参数就会使用第二种不缓存的方式。

采用不缓存的方式时，如果本地处理得慢，就会导致服务端发送结果被阻塞，因此会让服务端变慢 。

**为什么叫 quick**

- 第一点，就是前面提到的，跳过表名自动补全功能
- 第二点，mysql_store_result 需要申请本地内存来缓存查询结果，如果查询结果太大，会耗费较多的本地内存，可能会影响客户端本地机器的性能；
- 第三点，是不会把执行命令记录到本地的命令历史文件。
