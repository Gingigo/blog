# cKafka单机环境安装

## 安装

### 下载

```shell
wget https://apachemirror.sg.wuchna.com/kafka/2.6.0/kafka_2.13-2.6.0.tgz
```

### 解压

```shell
tar -xzf kafka_2.13-2.6.0.tgz -C /usr/local/
```

### 软连接

```shell
ln -s apache-zookeeper-3.6.1-bin/ zookeeper
```

### 进入目录

```shell
cd kafka
```

### 修改配置

-  查看本机ip

  ```shell
  [root@localhost ~]# ifconfig
  enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
          inet6 fe80::9f14:6706:b00d:cbc8  prefixlen 64  scopeid 0x20<link>
          ether 08:00:27:b7:5e:92  txqueuelen 1000  (Ethernet)
          RX packets 87  bytes 9816 (9.5 KiB)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 80  bytes 9806 (9.5 KiB)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  
  enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          inet 192.168.2.199  netmask 255.255.255.0  broadcast 192.168.2.255
          inet6 fe80::81d5:9ab9:d7d4:4e6e  prefixlen 64  scopeid 0x20<link>
          ether 08:00:27:a4:95:50  txqueuelen 1000  (Ethernet)
          RX packets 8  bytes 1328 (1.2 KiB)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 16  bytes 2021 (1.9 KiB)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  ```

  得到 IP 为<font color='orange'>192.168.2.199</font>

- 修改配置文件`server.properties`

  ```shell
  vim config/server.properties
  ```

  ```properties
  listeners=PLAINTEXT://192.168.2.199:9092
  advertised.listeners=PLAINTEXT://192.168.2.199:9092
  log.dirs=/tmp/kafka-logs
  zookeeper.connect=localhost:2181
  ```

### 测试

- 启动 zookeeper (默认配置)

  ```shell
  bin/zookeeper-server-start.sh config/zookeeper.properties
  ```

- 启动 kafka (需要打开新窗口)

  ```shell
  bin/kafka-server-start.sh config/server.properties
  ```

- 创建一个 TOPIC 来存储事件（events）

  ```shell
  bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server 192.168.2.199:9092
  ```

- 写事件到 TOPIC 中

  ```shell
  bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server 192.168.2.199:9092
  >hello
  >world
  >gin
  >oooo
  ```

> Ctrl+c 退出

- 读取事件

  ```shell
  bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server 192.168.2.199:9092
  hello
  world
  gin
  oooo
  ```

## 参考

> - [官网配置](http://kafka.apache.org/quickstart)

