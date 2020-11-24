# 43 | 要不要使用分区表？

###  分区表是什么？ 

创建一个表：

```mysql
CREATE TABLE `t` (
  `ftime` datetime NOT NULL,
  `c` int(11) DEFAULT NULL,
  KEY (`ftime`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY RANGE (YEAR(ftime))
(PARTITION p_2017 VALUES LESS THAN (2017) ENGINE = InnoDB,
 PARTITION p_2018 VALUES LESS THAN (2018) ENGINE = InnoDB,
 PARTITION p_2019 VALUES LESS THAN (2019) ENGINE = InnoDB,
PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE = InnoDB);
insert into t values('2017-4-1',1),('2018-4-1',1);
```

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-43-01.png)

<center> 图 1 表 t 的磁盘文件 </center>
> 可以通过 `find / -name "*.ibd"` 找到 msyql 存储文件的位置。

在表 t 中初始化插入了两行记录，按照定义的分区规则，这两行记录分别落在 p_2018 和 p_2019 这两个分区上。

可以看到，这个表包含了一个.frm 文件和 4 个.ibd 文件，每个分区对应一个.ibd 文件。也就是说：

- 对于引擎层来说，这是 4 个表；
- 对于 Server 层来说，这是 1 个表。

**证明对于InnoDB 来说是 4 个表（InnoDB）**

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-43-02.png)

<center> 图 2 分区表间隙锁示例 </center>
从图2可知到，如果对于 InnoDB 来说是一个表，那么 session A 会在 2017-4-1 到 2018-4-1 加一个间隙锁（利用优化2）。但 session B 中 2018-2-1 能插入，而 2017-12-1 却被锁住。

很明显，如果我们把分区看成四个表就容易解释了。session A 执行后会对 p_2018 这个区（表）加一个2017-4-1 到 supermun 的间隙锁。而且三个区是不同的表格，不会受到间隙锁的影响，所以 session B 的第一条正确执行了，而第二条受到了 session A 的间隙锁影响，阻塞了。

**证明对于InnoDB 来说是 4 个表（MyISAM）**

首先用 alter table t engine=myisam，把表 t 改成 MyISAM 表；然后，我再用下面这个例子说明，对于 MyISAM 引擎来说，这是 4 个表。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-43-03.png)

<center> 图 3 用 MyISAM 表锁验证 </center>
在 session A 里面，用 sleep(100) 将这条语句的执行时间设置为 100 秒。由于 MyISAM 引擎只支持表锁，所以这条 update 语句会锁住整个表 t 上的读。

但我们看到的结果是，session B 的第一条查询语句是可以正常执行的，第二条语句才进入锁等待状态。

这正是因为 MyISAM 的表锁是在引擎层实现的，session A 加的表锁，其实是锁在分区 p_2018 上。因此，只会堵住在这个分区上执行的查询，落到其他分区的查询是不受影响的。

#### 手动分表和分区有什么区别

- 划分依据：按照年份来划分，我们就分别创建普通表 t_2017、t_2018、t_2019 等等。手工分表的逻辑，也是找到需要更新的所有分表，然后依次执行更新。在性能上，这和分区表并没有实质的差别。
- 划分方式：分区表和手工分表，一个是由 server 层来决定使用哪个分区，一个是由应用层代码来决定使用哪个分表。因此，从引擎层看，这两种方式也是没有差别的。

### 分区的诟病

其实这两个方案的区别，主要是在 server 层上。从 server 层看，我们就不得不提到分区表一个被广为诟病的问题：**打开表的行为**。 

#### 分区策略

**每当<font color='orange'>第一次</font> 访问一个分表的时候，MySQL 需要把所有的分区都访问一遍。**

**一个典型的报错情况是这样的**：如果一个分区表的分区很多，比如超过了 1000 个，而 MySQL 启动的时候，<font color='orange'>open_files_limit</font> 参数使用的是默认值 1024，那么就会在访问这个表的时候，由于需要打开所有的文件，导致打开表文件的个数超过了上限而报错。

下图就是创建的一个包含了很多分区的表 t_myisam，执行一条插入语句后报错的情况。 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-43-04.png)

<center> 图 4 insert 语句报错 </center>
可以看到，这条 insert 语句，明显只需要访问一个分区，但语句却无法执行。 

这时，你一定从表名猜到了，这个表我用的是 MyISAM 引擎。是的，因为使用 InnoDB 引擎的话，并不会出现这个问题。 (但是InnoDB 和  MyISAM 第一次都会访问全部分区一遍)。

**MySQL分区策略**：

- MyISAM 分区表使用的分区策略，我们称为通用分区策略（generic partitioning），每次访问分区都由 server 层控制。通用分区策略，是 MySQL 一开始支持分区表的时候就存在的代码，在文件管理、表管理的实现上很粗糙，因此有比较严重的性能问题。 （MySQL 从 5.7.17 开始，将 MyISAM 分区表标记为即将弃用 (deprecated)， 从 MySQL 8.0 版本开始，就不允许创建 MyISAM 分区表了，只允许创建已经实现了本地分区策略的引擎  ）
- 从 MySQL 5.7.9 开始，InnoDB 引擎引入了本地分区策略（native partitioning）。这个策略是在 InnoDB 内部自己管理打开分区的行为。 
- 目前来看，只有 InnoDB 和 NDB 这两个引擎支持了本地分区策略。 

### 分区表的 server 层行为

如果从 server 层看的话，一个分区表就只是一个表。 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-43-05.png)

<center>图 5 分区表的 MDL 锁</center>
![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-43-06.png)

<center> 图 6 show processlist 结果 </center>

可以看到，虽然 session B 只需要操作 p_2107 这个分区，但是由于 session A 持有整个表 t 的 MDL 锁，就导致了 session B 的 alter 语句被堵住。

> 这也是 DBA 经常说的，分区表，在做 DDL 的时候，影响会更大。如果你使用的是普通分表，那么当你在 truncate 一个分表的时候，肯定不会跟另外一个分表上的查询语句，出现 MDL 锁冲突。 

### 小结分区

 到这里我们小结一下： 

- MySQL 在第一次打开分区表的时候，需要访问所有的分区； 
-  在 server 层，认为这是同一张表，因此所有分区共用同一个 MDL 锁； 
- 在引擎层，认为这是不同的表，因此 MDL 锁之后的执行过程，会根据分区表规则，只访问**必要的分区**。

而关于“**必要的分区”**的判断，就是根据 SQL 语句中的 where 条件，结合分区规则来实现的。比如我们上面的例子中，where ftime=‘2018-4-1’，根据分区规则 year 函数算出来的值是 2018，那么就会落在 p_2019 这个分区。

###  分区表的应用场景 

分区表的一个显而易见的优势是对业务透明，相对于用户分表来说，使用分区表的业务代码更简洁。还有，分区表可以很方便的清理历史数据。

如果一项业务跑的时间足够长，往往就会有根据时间删除历史数据的需求。这时候，按照时间分区的分区表，就可以直接通过 alter table t drop partition …这个语法删掉分区，从而删掉过期的历史数据。

这个 alter table t drop partition …操作是直接删除分区文件，效果跟 drop 普通表类似。与使用 delete 语句删除数据相比，优势是速度快、对系统影响小。

### 小结

需要注意的是，我是以范围分区（range）为例和你介绍的。实际上，MySQL 还支持 hash 分区、list 分区等分区方法。你可以在需要用到的时候，再翻翻手册。 

实际使用时，分区表跟用户分表比起来，有两个绕不开的问题：一个是第一次访问的时候需要访问所有分区，另一个是共用 MDL 锁。

因此，如果要使用分区表，就不要创建太多的分区。我见过一个用户做了按天分区策略，然后预先创建了 10 年的分区。这种情况下，访问分区表的性能自然是不好的。这里有两个问题需要注意： 

1.  分区并不是越细越好。实际上，单表或者单分区的数据一千万行，只要没有特别大的索引，对于现在的硬件能力来说都已经是小表了。 
2.  分区也不要提前预留太多，在使用之前预先创建即可。比如，如果是按月分区，每年年底时再把下一年度的 12 个新分区创建上即可。对于没有数据的历史分区，要及时的 drop 掉。 

至于分区表的其他问题，比如查询需要跨多个分区取数据，查询性能就会比较慢，基本上就不是分区表本身的问题，而是数据量的问题或者说是使用方式的问题了。
