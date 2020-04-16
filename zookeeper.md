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