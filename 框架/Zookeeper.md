> zk是一个高性能的分布式协调服务,使用该文件系统目录树作为数据模型

#### 应用场景

* 发布订阅 基于watcher实现
* 负载均衡

#### 权限功能

* `schema:id:permission` 
  * `schema` 鉴权策略
  * `id` 授权id
  * `permission` 权限

* 策略分类

| 策略类型 | 描述                                     |
| -------- | ---------------------------------------- |
| world    | 只能有一个用户: anyone (代表所有人,默认) |
| ip       | 使用ip地址认证                           |
| auth     | 使用已添加的用户认证                     |
| digest   | 使用"用户名:密码"认证                    |

* 权限分类

| 权限   | acl简写 | 描述                         |
| ------ | ------- | ---------------------------- |
| create | c       | 创建子节点                   |
| delete | d       | 删除子节点(仅下一级节点)     |
| read   | r       | 读取节点数据和显示子节点列表 |
| write  | w       | 设置节点数据                 |
| admin  | a       | 可以设置节点访问控制列表权限 |

* `getAcl` 读取acl权限
* `setAcl` 设置acl权限
* `addauth` 添加认证用户

* 权限使用示例
  * `setAcl /node world:anyone:cdrwa` 表示任何人都拥有所有权限
  * `setAcl /node ip:192.168.1.1:cdrwa` 表示该ip拥有所有权限
  * `addauth digest eghm:123456` 添加认证用户
  * `setAcl /node auth:eghm:cdrwa` eghm用户拥有所有权限
  * `setAcl /node digest:eghm:NcRWbBdsl5x9pEis3es7fVwmSJs=:cdrwa` 与上面一个类似,只是这个额外添加密码加密验证
  * `echo -n eghm:123456 | openssl dgst -binary -sha1 | openssl base64` 用户和密码加密后的命令

#### 基本命令

* `create /node test`  创建持久化节点
* `create -e /node test` 创建临时节点
* `create -s /node test` 创建顺序节点

#### 节点属性描述

* `dataVersion` 数据版本号,每次对节点进行set操作,`dateVersion`都会+1(即使设置相同值)
* `cversion` 子节点版本号 每当子节点有变化时,`cversion` 都会+1
* `aclVersion` acl的版本号
* `cZxid`  znode创建时的事务id,
* `mZxid` znode被修改时的事务id,即每次修改都会对znode的`mZxid`进行修改
* `ctime` 创建znode的时间戳
* `mtime` 修改znode的时间戳

#### watcher机制

* `wather` 只能监听当前节点和其子节点,不能监听孙子节点
* 

### zookeeper是如何保证事务的顺序一致性的

> 待定

### fast paxos 选主算法

* 某Server首先向所有Server提议自己要成为Leader,当其他Server收到提议后,解决`epoch`和`zxid`冲突,并接受对方的提议
* 然后向对方发送接收提议完成的消息,重复这个流程一定能选出Leader

### basic paxos 选主算法

* `选举线程` 由当前Server发起选举的线程担任,主要是对选举结果进行统计,并选出推荐的Server
* 选举线程向所有的Server发起一次询问(包括自己)
* 选举线程收到回复后,验证是否是自己发起的询问(验证`zxid`是否一致),然后后去对方的id(`myid`),并存储到当前询问对象列表中,最后获取对方提议的leader相关信息(`id`,`zxid`),并将这些信息存储到档次讯据的投票记录表中
* 收到所有Server回复后,计算出`zxid`最大的那个Server,并将这个Server相关信息设置为下一次要投票的Server
* 线程将当前`zxid` 最大的Server设置为当前要Server要推荐的Leader,如果此时获胜的Server获取到n/2+1个Server的投票,设置推荐Server为真正的Leader,并根据获胜的Server相关信息设置自己的状态,否则继续这个过程,直到Leader被选举出来
* 通过分析我们可以得出:要使Leader获取多数Server的支持,Server总数必须为奇数 2n+1,且存活的Server的数目不小于n+1

### 同步流程

* leader等待Server连接
* Follower连接leader,将最大的`zxid`发送给leader
* Leader根据Follower的`zxid`确定同步点
* 完成同步后通知follower已经成为uptodate状态,
* Follower收到uptodate消息后,可以重新接收client的请求进行服务