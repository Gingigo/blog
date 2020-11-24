# RocketMQ 问题汇总

### 连接不上问题

1. <font color='red'>Exception in thread "main" org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to 172.17.0.1:10911 failed</font>

   [解决]( https://blog.csdn.net/q258523454/article/details/82716027  )：

   1. 修改 broker.conf  文件。`vim conf/broker.conf`

   2. 添加 rocketmq 所在机子的 ip

      ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/questions-and-answers/01/rocketmq_01.png)

   3. 关闭 broker。`sh bin/mqshutdown broker`

   4. 重新启动，` nohup sh bin/mqbroker -n localhost:9876 -c conf/broker.conf & `

2. 23

3. 23

4. 23