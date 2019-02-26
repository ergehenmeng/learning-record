
##Kafka
**启动命令**
* ./kafka-server-start.sh -daemon config/server.properties `server.properties`中指定链接zookeeper的地址
**创建topic命令**
* `describe` 打印描述信息
* ./kafka-topics.sh --create --zookeeper 127.0.0.1:21811 --replication-factor 1  --partition 1 --topic TEST-TOPIC --describe
* ./kafka-topics.sh --list --zookeeper 127.0.0.1:21811 查看topic列表
**生产者启动命令** 
* 需要指定broker地址 默认是当前机器的9092端口 `advertised.listeners=PLAINTEXT://127.0.0.1:9092`
* ./kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic TEST-TOPIC
**消费者启动命令**
* `from-beginning` 从头开始接受消息
* ./kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic TEST-TOPIC --from-beginning
##Zookeeper
####启动命令 ./zkServer.sh {start|start-foreground|stop|restart|status|upgrade|print-cmd}
* ./zkService.sh start 
* ./zkCli.sh -server host:port
####zoo.cfg配置说明
* **tickTime** zookeeper服务器之间或客户端与服务器端之间维持心跳的间隔
* **dataDir** zookeeper保存数据的目录,默认情况下zookeeper会将日志文件也保存在该目录下
* **dataLogDir** zookeeper日志文件目录
* **clientPort** 客户端链接zookeeper服务器的端口
####节点创建模式
* **PERSISTENT** 持久化
* **PERSISTENT_SEQUENTIAL** 持久化并带序列号
* **EPHEMERAL** 临时
* **EPHEMERAL_SEQUENTIAL** 临时并带序列号
* **PathChildrenCache** 监控一个ZNode节点的子节点的变化(增删改),会改变它的状态,包含最新的子节点,子节点的数据及状态,而状态的变更通过PathChildrenCacheListener,注意:第三个参数如果设置false,则不会对节点数据进行缓存
  * StartMode启动方式:
  * NORMAL:正常初始化
  * BUILD_INITIAL_CACHE:在调用start之前调用rebuild();
  * POST_INITIALIZED_EVENT:当Cache初始化数据后发送一个PathChildrenCacheEvent.Type#INITIALIZED事件
* **NodeCache** 监听某个特定的节点 
* **TreeCache** 监听树的所有节点PathChildrenCache + NodeCache结合体
* **LeaderSelector** 所有存活的客户端不间断轮流做leader
* **LeaderLatch** 一旦选举出leader后,除非有客户端挂掉重新触发选举,否则不会交出leader权限
* **InterProcessMutex** 可重入锁
* **InterProcessSemaphoreMutex** 不可重入锁
* **InterProcessReadWriteLock** 可重入读写锁 读写锁的类型 InterProcessMutex
* **MultiSharedLock** 锁容器,调用acquire()时,所有锁都被acquire(),如果请求失败,所有锁会release()
* **DistributedAtomicLong** 分布式计数器 Long类型
* **SharedCount** 分布式计数器 int类型
* **DistributedQueue** 分布式队列
* **DistributedIdQueue** 带id的分布式队列(可以移除元素)
* **DistributedPriorityQueue** 可以指定优先级的队列(put方法设置) priority越小越靠前,越先被消费
* **DistributedDelayQueue** 分布式延迟队列 消费需要隔一段时间才能收到消息,put方法指定未来的某个时间,如果设置的时间大于当前时间则会立即被消费
* **DistributedBarrier** 类似于CountDownLatch 栅栏
  * setBarrier() 阻塞在它上面的线程
  * waitOnBarrier() 需要阻塞的线程调用方法等待放行的条件
  * removeBarrier() 移除栅栏,所有等待的线程继续运行
* **DistributedDoubleBarrier** 双栅栏
