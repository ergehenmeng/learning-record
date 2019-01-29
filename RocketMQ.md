##命令
#### 命令均在bin目录下执行
* 启动nameSrv服务 
  * `nohup mqnamesrv &`
* 启动broker服务
  * `nohup mqbroker -n localhost:9876 &`
* 默认日志在/root/logs/rocketmqlogs/下

* 手动创建topic `./mqadmin updateTopic`
  * `-b` 如果`-c`为空则该项必填,表示topic创建在该broker上
  * `-c` 如果`-b`为空则该项必填,表示topic创建在该集群上(可通过`clusterList`查询)
  * `-h` 打印帮助
  * `-n` 必填 namesrv服务列表地址 ip:port;ip:port
  * `-p` 指定新topic权限(W|R|WR)
  * `-r` 可读队列 默认8
  * `-w` 可写队列 默认8
  * `-t` topic名称 必填
* ps `./mqadmin updateTopic -c DefaultCluster -n localhost:9876 -t TopicTest`
* 删除topic `./mqadmin deleteTopic`
* 创建订阅组 `./mqadmin updateSubGroup`
* 删除订阅组 `./mqadmin deleteSubGroup`
* 更新Broker配置 `./mqadmin updateBrokerConfig`
* 查看topic列表 `./mqadmin topicList`
* 查看topic路由 `./mqadmin topicRoute`
* 查看topic统计信息 `./mqadmin topicStatus`
* 查看Broker统计信息 `./mqadmin brokerStatus`
* 根据消息id查询消息 `./mqadmin queryMsgById`
* 根据消息key查询消息 `./mqadmin queryMsgByKey`
* 根据Offset查询消息 `./mqadmin queryMsgByOffset`
* 查询Producer的网络连接 `./mqadmin producerConnection`
* 查询Constomer的网络连接 `./mqadmin consumerConnection`
* 查询订阅组消费状态 `./mqadmin consumerProgress`
* 查询集群消息 `./mqadmin clusterList`
* 添加(更新)K/V配置信息 `./mqadmin updateKvConfig`
* 删除K/V配置信息 `./mqadmin deleteKvConfig`
* 添加(更新)Project Group配置信息 `./mqadmin updateProjectGroup`
* 删除Project Group 配置信息 `./mqadmin deleteProjectGroup`
* 获取Project Group 配置信息 `./mqadmin getProjectGroup`
* 设置消费进度 `./mqadmin resetOffsetByTime`
* 清除特定Broker权限 `./mqadmin wipeWritePerm`
* 获取Constomer消费进度 `./mqadmin getConsumerStatus`

> 针对服务器部署docker时,多网卡导致rocketmq找不到响应的地址
> /conf/broker.conf中增加如下

* namesrvAddr = 127.0.0.1
* brokerIP1 = 72.127.2.8
* 启动broker执行 `nohup sh mqbroker -n localhost:9876 -c ../conf/broker.conf autoCreateTopicEnable=true & ` 