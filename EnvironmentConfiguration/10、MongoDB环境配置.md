# MongoDB 环境配置

## 下载

- [官网下载](https://www.mongodb.com/try/download/community)

  - Version: 4.4.3
  - Platform: Centos8.0
  - Package: tag

- [下载链接](https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel80-4.4.3.tgz)

  ```shell
  [root@localhost software]# pwd
  /opt/software
  [root@localhost software]# curl https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel80-4.4.3.tgz -o mongodb-linux-x86_64-rhel80-4.4.3.tgz --progress
  ```



## 安装

- 解压

  ```shell
  [root@localhost software]# tar -xzf mongodb-linux-x86_64-rhel80-4.4.3.tgz -C /usr/local/
  ```

- 软连接

  ```shell
  [root@localhost software]# cd /usr/local/
  [root@localhost local]# ln -s mongodb-linux-x86_64-rhel80-4.4.3/ mongodb
  ```

- 添加到Path路径

  ```shell
  [root@localhost ~]# export PATH=/usr/local/mongodb/bin:$PATH
  ```

## 启动

### 创建数据库目录

- 数据存储目录：`/data/mongo`
- 日志文件目录：`/data/mongodb`

```shell
[root@localhost data]# sudo mkdir -p /data/mongo
[root@localhost data]# sudo mkdir -p /data/mongodb
[root@localhost data]# sudo chown `whoami` /data/mongo
[root@localhost data]# sudo chown `whoami` /data/mongodb
```

### 启动

```shell
[root@localhost data]# mongod --dbpath /data/mongo --logpath /data/mongodb/mongod.log --fork
about to fork child process, waiting until server is ready for connections.
forked process: 1442
child process started successfully, parent exiting
[root@localhost data]# mongo
```



## 工具

- [官网地址](https://www.mongodb.com/try/download/database-tools)

  - [rpm包](https://fastdl.mongodb.org/tools/db/mongodb-database-tools-rhel80-x86_64-100.2.1.rpm)

  ```shell
  [root@localhost data]# curl -O -k https://fastdl.mongodb.org/tools/db/mongodb-database-tools-rhel80-x86_64-100.2.1.rpm
  [root@localhost data]# yum install mongodb-database-tools-rhel80-x86_64-100.2.1.rpm 
  ```

## 导入样板数据

- 下载样板数据

  ```shell
  curl -O -k https://raw.githubusercontent.com/tapdata/geektime-mongodb-course/master/aggregation/dump.tar.gz
  ```

- 解压

  ```shell
  tar -xvf dump.tar.gz
  ```

- 导入数据

  ```shell
  mongorestore /data/dump
  ```