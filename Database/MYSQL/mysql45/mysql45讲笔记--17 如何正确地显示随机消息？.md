# 17 | 如何正确地显示随机消息?

如何从一个单词表中随机选出三个单词？这个表的建表语句和初始数据的命令如下：

```mysql

mysql> CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

## 内存临时表

 首先，你会想到用 order by rand() 来实现这个逻辑。 

```mysql

mysql> select word from words order by rand() limit 3;
```

我们先用 explain 命令来看看这个语句的执行情况。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-17-01.png)

<center>图 1 使用 explain 命令查看语句的执行情况</center>
Extra 字段显示 Using temporary ，表示的是需要使用临时表；Using filesort，表示的是需要执行排序操作。

你觉得对于临时内存表的排序来说，它会选择哪一种算法呢？

这个就要回到上一章的的一个结论，**对于 InnoDB 表来说**,执行全字段排序会减少磁盘访问，因此会被优先选择。

我强调了“InnoDB 表”，你肯定想到了，**对于内存表，回表过程只是简单地根据数据行的位置，直接访问内存得到数据，根本不会导致多访问磁盘。**优化器没有了这一层顾虑，那么它会优先考虑的，就是用于排序的行越小越好了，所以，MySQL 这时就会选择 rowid 排序。

这条语句的执行流程是这样的：

1. 创建一个临时表。这个临时表使用的 memory 引擎，表里面有两个字段，第一个字段为double 类型，为了后面描述方便，记为字段R，第二个字段是varchar（64）类型，记为字段 W。并且，这个表没有建索引。
2. 从words 表中，按主键顺序取出所有的word 值，对于每一个word 值，调用rand（） 函数生成一个大于0小于1的随机小数和word 分别存入临时表中的 R 和W 字段，到此，扫描行数是10000。
3. 现在临时表有 10000 行数据了，接下来你要在这个没有索引的内存临时表上，按照字段 R 排序。
4. 初始化 sort_buffer。sort_buffer 中有两个字段，一个是double 类型， **另一个是整型**。 
5. 从内存临时表中一行一行地取出 R 值和位置信息（我后面会和你解释这里为什么是“位置信息”），分别存入 sort_buffer 中的两个字段里。这个过程要对内存临时表做全表扫描，此时扫描行数增加 10000，变成了 20000。
6. 在 sort_buffer 中根据 R 的值进行排序。注意，这个过程没有涉及到表操作，所以不会增加扫描行数。
7. 排序完成后，取出前三个结果的位置信息，依次到内存临时表中取出 word 值，返回给客户端。这个过程中，访问了表的三行数据，总扫描行数变成了 20003。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-17-02.png)

<center>图 2 随机排序完整流程图 1</center>
图中的 pos 就是位置信息，你可能会觉得奇怪，这里的“位置信息”是个什么概念？在上一篇文章中，我们对 InnoDB 表排序的时候，明明用的还是 ID 字段

这也就是排序模式里面，rowid 名字的来历。实际上它表示的是：每个引擎用来唯一标识数据行的信息。

- 对于有主键的 InnoDB 表来说，这个 rowid 就是主键 ID；
- 对于没有主键的 InnoDB 表来说，这个 rowid 就是由系统生成的；
- MEMORY 引擎不是索引组织表。在这个例子里面，你可以认为它就是一个数组。因此，这个 rowid 其实就是数组的下标。



到这里，我来稍微小结一下：**order by rand() 使用了内存临时表，内存临时表排序的时候使用了 rowid 排序方法。**

## 磁盘临时表

那么，是不是所有的临时表都是内存表呢？

其实不是的，temp_table_size 这个配置限制了内存表的大小，默认是 16M。如果临时表大小超过了 temp_table_size,那么内存临时表就会转成磁盘临时表。

磁盘临时表使用引擎默认是 InnoDB，是由参数 internal_tmp_disk_storage_engine 控制的。

当使用磁盘临时表的时候，对应的就是一个没有显式索引的 InnoDB 表的排序过程。

为了复现这个过程，我把 tmp_table_size 设置成 1024，把 sort_buffer_size 设置成 32768, 把 max_length_for_sort_data 设置成 16。

```mysql

set tmp_table_size=1024;
set sort_buffer_size=32768;
set max_length_for_sort_data=16;
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* 执行语句 */
select word from words order by rand() limit 3;

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-17-03.png)

<center>图 3 OPTIMIZER_TRACE 部分结果</center>
因为将 max_length_for_sort_data 设置成 16，小于 word 字段的长度定义，所以我们看到 sort_mode 里面显示的是 rowid 排序，这个是符合预期的，参与排序的是随机值 R 字段和 rowid 字段组成的行。

这时候你可能心算了一下，发现不对。R 字段存放的随机值就 8 个字节，rowid 是 6 个字节（至于为什么是 6 字节，就留给你课后思考吧），数据总行数是 10000，这样算出来就有 140000 字节，超过了 sort_buffer_size 定义的 32768 字节了。但是，number_of_tmp_files 的值居然是 0，难道不需要用临时文件吗？

**这个 SQL 语句的排序确实没有用到临时文件**，采用是 MySQL 5.6 版本引入的一个新的排序算法，即：**优先队列排序算法**。接下来，我们就看看为什么没有使用临时文件的算法，也就是归并排序算法，而是采用了优先队列排序算法。

其实，我们现在的 SQL 语句，只需要取 R 值最小的 3 个 rowid。但是，如果使用归并排序算法的话，虽然最终也能得到前 3 个值，但是这个算法结束后，已经将 10000 行数据都排好序了。

也就是说，后面 9997 行是有序的了，但，我们的查询并不需要这些数据是有序的。所以，想一下就明白了，这浪费了非常多的计算量。

**而优先队列算法，就可以精确地只得到三个最小值**，执行流程如下：

1. 对于这10000 个准备排序的（R,rowid）,先取前三行，构造成一个堆；
2. 取下一行（R‘，rouwid’），跟当前堆里卖弄最大的R比较，如果R‘小于R，把这个（R，rowid）从堆中去掉，换成（R’，rowid‘）；
3. 重复第二步骤，直到第10000个（R’，rowid‘）完成比较。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-17-04.png)

<center>图 4 优先队列排序算法示例</center>
图 4 是模拟 6 个 (R,rowid) 行，通过优先队列排序找到最小的三个 R 值的行的过程。整个排序过程中，为了最快地拿到当前堆的最大值，总是保持最大值在堆顶，因此这是一个最大堆。

图3 的 OPTIMIZER_TRACE结果中， filesort_priority_queue_optimization 这个部分的 chosen=true，就表示使用了优先队列排序算法，这个过程不需要临时文件，因此对应的 number_of_tmp_files 是 0。

这个流程结束后，我们构造的堆里面，就是这个 10000 行里面 R 值最小的三行。然后，依次把它们的 rowid 取出来，去临时表里面拿到 word 字段，这个过程就跟上一篇文章的 rowid 排序的过程一样了。

> 那上篇文章中为什么没有使用优先队列呢？
>
> ```msyql
> select city,name,age from t where city='杭州' order by name limit 1000  ;
> ```
>
> 原因是，这条 SQL 语句是 limit 1000，如果使用优先队列算法的话，需要维护的堆的大小就是 1000 行的 (name,rowid)，**超过了我设置的 sort_buffer_size 大小**，所以只能使用归并排序算法。

## 随机排序方法

我们先把问题简化一下，如果只随机选择一个 word 值，那怎么做呢？

### 随机算法 1

1. 取得这个表得主键id 得最大值M 和最小值N；
2. 用随机函数生成一个最大值到最小值直接得数 X = （M-N）* rand() +N;
3. 取不小于 X 得第一个ID行。

```mysql
mysql> select max(id),min(id) into @M,@N from t ;
set @X= floor((@M-@N+1)*rand() + @N);
select * from t where id >= @X limit 1;
```

这个算法效率很高，但是有局限性，id必须连续，不能有空洞，否则会出现概率不准确和找不到对应的X值问题。

### 随机算法 2

1. 取得整个表中的行数，记为C。
2. 取得 Y = floor（C.rand()）。floor 函数这里的作用就是取整数部分。
3. 再用 limt Y，1 取得一行。

```mysql

mysql> select count(*) into @C from t;
set @Y = floor(@C * rand());
set @sql = concat("select * from t limit ", @Y, ",1");
prepare stmt from @sql;
execute stmt;
DEALLOCATE prepare stmt;
```

随机算法2 解决了算法 1 里面明显的概率不均匀问题。

MySQL 处理 limit Y,1 的做法就是按顺序一个一个地读出来，丢掉前 Y 个，然后把下一个记录作为返回结果，因此这一步需要扫描 Y+1 行。再加上，第一步扫描的 C 行，总共需要扫描 C+Y+1 行，执行代价比随机算法 1 的代价要高。

 当然，随机算法 2 跟直接 order by rand() 比起来，执行代价还是小很多的。 

### 随机算法 3

现在，我们再看看，如果我们按照随机算法 2 的思路，要随机取 3 个 word 值呢？你可以这么做

1.  取得整个表的行数，记为 C； 
2.  根据相同的随机方法得到 Y1、Y2、Y3； 
3.  再执行三个 limit Y, 1 语句得到三行数据。 

```mysql

mysql> select count(*) into @C from t;
set @Y1 = floor(@C * rand());
set @Y2 = floor(@C * rand());
set @Y3 = floor(@C * rand());
select * from t limit @Y1，1； //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行
select * from t limit @Y2，1；
select * from t limit @Y3，1；
```

