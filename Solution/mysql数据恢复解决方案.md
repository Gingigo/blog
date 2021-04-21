# MySQL数据恢复解决方案

如果已经发生了数据库的表和记录被删除了，请先执行下列步骤

1、对MySQL进行操作

```msyql
flush logs
```

2、如果有必要的话，关闭服务 防止数据写入

## 模拟数据被被删除并且恢复

数据恢复是基于 binlog 进行恢复，所以必须开启 binlog 日志，这里的 binlog 日志格式是 row 

开启 binlog， 如果`log_bin 为 ON,表示已经开启

```log
mysql> show variables like "%log_bin%";
+---------------------------------+-----------------------------+
| Variable_name                   | Value                       |
+---------------------------------+-----------------------------+
| log_bin                         | ON                          |
| log_bin_basename                | /var/lib/mysql/binlog       |
| log_bin_index                   | /var/lib/mysql/binlog.index |
| log_bin_trust_function_creators | OFF                         |
| log_bin_use_v1_row_events       | OFF                         |
| sql_log_bin                     | ON                          |
+---------------------------------+-----------------------------+
mysql> show variables like "%binlog_format%";
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+

mysql> show binary logs;
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000005 |       179 | No        |
| binlog.000006 |      1969 | No        |
+---------------+-----------+-----------+

```

未开启，需要打开 

```shell
vim /etc/my.cnf
```

添加配置

```conf
log_bin=mysql-bin
binlog_format=ROW
```

### 生成模拟数据

- 新建数据库

  ```mysql
  create database test;
  ```

- 新建数据表

  ```mysql
  CREATE TABLE `t1` (
    `id` int NOT NULL AUTO_INCREMENT,
    `name` varchar(32) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  
  CREATE TABLE `t2` (
    `id` int NOT NULL AUTO_INCREMENT,
    `name` varchar(32) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

- 插入数据

  ```mysql
  insert into t1(name) values('t1-01'),('t1-02'),('t1-03'),('t1-04');
  insert into t2(name) values('t2-01'),('t2-02'),('t2-03'),('t2-04');
  ```

### 直接恢复

#### 备份全量恢复

- 工具： `mysqldump`
- 条件：必须备份

##### 模拟场景

- 备份

```SHELL
[root@localhost ~]# mysqldump -uroot -p test > /home/backup/mysql/test.db
Enter password: 
[root@localhost ~]# vim /home/backup/mysql/test.db
```

- 误删数据

```mysql
mysql> drop table t1;
Query OK, 0 rows affected (0.05 sec)
```

- 还原数据

```mysql
mysql> source /home/backup/mysql/test.db
mysql> select * from t1;
+----+-------+
| id | name  |
+----+-------+
|  1 | t1-01 |
|  2 | t1-02 |
|  3 | t1-03 |
|  4 | t1-04 |
+----+-------+
4 rows in set (0.00 sec)

mysql> select * from t2;
+----+-------+
| id | name  |
+----+-------+
|  1 | t2-01 |
|  2 | t2-02 |
|  3 | t2-03 |
|  4 | t2-04 |
+----+-------+
4 rows in set (0.00 sec)
```

#### binlog 恢复数据

- 工具:`mysqldump`
- 条件：必须开启 binlog 日志

##### 模拟场景

- 误删数据

  ```mysql
  mysql> drop table t2;
  Query OK, 0 rows affected (0.07 sec)
  
  mysql> show tables;
  +----------------+
  | Tables_in_test |
  +----------------+
  | t1             |
  +----------------+
  1 row in set (0.01 sec)
  ```

- 还原数据

  ```mysql
  mysql> show binary logs;
  +------------------+-----------+-----------+
  | Log_name         | File_size | Encrypted |
  +------------------+-----------+-----------+
  | mysql-bin.000001 |      5655 | No        |
  +------------------+-----------+-----------+
  1 row in set (0.00 sec)
  ```

  ```shell
  # 根据 pos结束点恢复
  mysqlbinlog  --stop-position=5246 --database=test /var/lib/mysql/mysql-bin.000001  | mysql -uroot -p密码 -v test
  ```

  ```mysql
  mysql> show tables;
  +----------------+
  | Tables_in_test |
  +----------------+
  | t1             |
  | t2             |
  +----------------+
  2 rows in set (0.00 sec)
  ```

  

> [MySQL 数据恢复详解](https://www.jianshu.com/p/e692d8ae4b1f)
>
> [MySQL之mysqldump的使用](https://www.cnblogs.com/markLogZhu/p/11398028.html)
>
> [binlog恢复数据](https://www.cnblogs.com/YCcc/p/10825870.html)







