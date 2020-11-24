# rocketmq 环境搭建

## 单机环境

### Linux（Centos7）

需要的准备的环境**：

- centos7
- JDK 1.8+
- rocketmq

#### centos 安装 JDK1.8

- 下载 rpm 包

  - 地址： [点击下载适合的JDK版本](https://www.oracle.com/java/technologies/javase/javase8u211-later-archive-downloads.html) 
- 安装：
    1. 执行 `yum install jdk-8u231-linux-x64.rpm ` 
    2. 在 /urs/java 成 java 安装目录 
    3. 执行 `java -version` 验证是否成功

#### 安装 rocketMQ

- 下载二进制代码
  - [点击下载二进制文件](https://mirror.bit.edu.cn/apache/rocketmq/4.7.0/rocketmq-all-4.7.0-bin-release.zip)
- 安装：
  1. 解压：unzip -o rocketmq-all-4.7.0-bin-release.zip -d /urs/rocketmq-4.7.0
  2. 验证：进入 `rocketmq-4.7.0/bin` 执行`./mqnamesrv`,显示启动成功

#### 可能出现的问题

- “MQClientException: No route info of this topic”问题

  - **最好不要用 网络地址转换（NAT）端口转发模式**，RocketMQ在程序访问的时候只需要填写一个NameSrv地址，但是会从NameSrv拿Broker的地址，而且在这种模式下虚拟机需要访问物理机。
    - 解决:换成桥接模式下，或者程序和recketmq同一台物理机
  - **版本和导入的jar包对应不上**，官网最新版是 4.7.0 但是 jar 是 4.2.0
    - 解决： 修改jar版本 或者修改 recketmq版本
  - **防火墙没有开发9876端口**
    - 解决：添加端口到规则中，或者关闭防火墙

- broker启动失败

  - **没启动 NameServer**

    - 解决：先启动NameServer服务

  - **内存不够**

    - 解决： runbroker.sh 

      ```yaml
      JAVA_OPT="${JAVA_OPT} -server -Xms128m -Xmx128m -Xmn128m"
      ```

### window（环境配置恶心）

- 需要注意的点
  1. 必须有 Java 环境，而且<font color='red'>最重要的是 JAVA_HOME 的路径不允许有空字符</font>
  2. [下载二进制文件](http://rocketmq.apache.org/release_notes/release-notes-4.7.0/)
  3. 安装的路径也不要有<font color='red'>**空字符**</font>
- 如何启动 .cmd 文件
  1. 打开cmd控制台
  2. 用 `start xxx` 如 `start mqnamesrv`

