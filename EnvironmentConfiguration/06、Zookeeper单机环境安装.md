# zookeeper单机环境安装

## 准备

- Centos 7 系统
- JDK 7+ 环境
- 下载 [zookeeper](https://zookeeper.apache.org/releases.html#download) 安装包,目前最新是 [3.6.1](https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.6.1/apache-zookeeper-3.6.1-bin.tar.gz)

### Centos 7 系统

可以参考 [VirtualBox 安装CentOS7](VirtualBox安装CentOS7 ) 

### JDK 7+ 环境

```shell
yum install java-1.8.0-openjdk.x86_64
```

```shell
[root@localhost local]# java -version
openjdk version "1.8.0_262"
OpenJDK Runtime Environment (build 1.8.0_262-b10)
OpenJDK 64-Bit Server VM (build 25.262-b10, mixed mode)
```

### 下载安装 zookeeper

下载

```shell
wget https://mirrors.bfsu.edu.cn/apache/zookeeper/zookeeper-3.6.1/apache-zookeeper-3.6.1-bin.tar.gz
```

解压

```shell
tar zxfv apache-zookeeper-3.6.1-bin.tar.gz -C /usr/local/
```

软连接

```shell
ln -s apache-zookeeper-3.6.1-bin/ zookeeper
```

添加到PATH 

```shell
vim /etc/profile
```

- ```shell
  # zookeeper
  export ZOOKEEPER_HOME=/usr/local/zookeeper/bin/
  export PATH=$PATH:$ZOOKEEPER_HOME
  
  export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE HISTCONTROL
  ```

```shell
source /etc/profile
```

## 启动

#### 配置文件

新建数据存储路径

```shell
mkdir /usr/local/zookeeper/data
```

复制模板

```shell
cd /usr/local/zookeeper/conf
```

```shell
cp zoo_sample.cfg zoo.cfg
```

修改配置

```shell
vim zoo.cfg
```

- 两个重要的配置项

  ```shell
  # the directory where the snapshot is stored.
  # do not use /tmp for storage, /tmp here is just 
  # example sakes.
  dataDir=/usr/local/zookeeper/data
  # the port at which the clients will connect
  clientPort=2181
  ```

#### 启动服务

启动命令

```shell
zkServer.sh start
```

检查日志文件是否出错

```shell
cd /usr/local/zookeeper/logs
```

```shell
grep -E -i "((exception)|(error))" *
```

> 显示文件中有"exception"或者“error”，如果为空，表示没有错误

检查数据文件

```shell
cd /usr/local/zookeeper/
```

```shell
[root@localhost zookeeper]# tree data
data
├── version-2
│   └── snapshot.0
└── zookeeper_server.pid
```

检查端口监听正常

```shell
netstat -an |grep 2181
```

- 结果

  ```shell
  tcp6       0      0 :::2181                 :::*                    LISTEN
  ```

  

