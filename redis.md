## redis
####linux安装
> * yum install binutils
> * yum install glibc
> * yum install glibc-kernheaders
> * yum install glibc-common
> * yum install glibc-devel
> * yum install gcc
> * yum install make
> * yum install vim
> * make MALLOC=libc
> * cp redis-server,redis-benchmark,redis-cli,redis.conf到指定目录
> * cp utils/redis_init_script /etc/init.d/redis 增加 # chkconfig 2345 80 90
> * chkconfig redis on 开机启动
> * ./redis-server & 启动
> * ./redis-cli 客户端链接

### window-redis:
> * redis-server --service-uninstall
> * redis-server --service-install redis.windows.conf

####Redis数据持久化
* **RDB** 按照一定的时间间隔对数据集创建基于时间点的快照,是默认的持久化方式方式
* **AOF** 记录Server收到的写操作到日志文件,在Server重启时,通过回放这些写操作,重新建立数据集
*
1. #900 秒后且至少 1 个 key 发生变化时创建快照
2. save 900 1
3. #300 秒后且至少 10 个 key 发生变化时创建快照
4. save 300 10
5. #60 秒后且至少 10000 个 key 发生变化时创建快照
6. save 60 10000
7. #可通过注释所有 save 开头的行来禁用 RDB 持久化
8. #创建快照时对数据进行压缩
9. rdbcompression yes
10. #快照名称
11. dbfilename dump.rdb
12. #存放快照的目录（AOF 文件也会被存放在此目录）
13. dir /usr/local/redis/
* `slowlog` subcommand 
  * `get` slowlog get 10 获取10条慢日志
  * `len` slowlog len 总慢日志数量
  * `reset` 清空日志
  * `config set slowlog-log-slower-than 2000` 命令多少毫秒才会保存slowlog中 单位微妙
* `rename-command` 重命名一些高危命令(生成环境上使用)
* `bigkeys` 查看key的大小(不一定准确,通过len进行判断的)
  * string 通过strlen判断
  * list 通过llen判断
  * hash 通过hlen判断
  * set 通过scard判断
  * sorted set通过zcard判断
* **Bitmaps** 
  * 并不是一个真实的数据结构,而是在string类型上一组面向bit操作的集合
  * `getbit` key offset value 设置值,value只能是0或1
  * `getbit` key offset 获取指定下标的值
  * `bitop` and,or,xor,not 执行两个string的位操作
  * `bitcount` 统计位的值为1的数量
  * `bitpos` 寻找第一个为0或者1的bit的位置
* **HASH** 命令:
  * `hset` 设置某个hash指定键的值 hset key field value
  * `hmset` 设置多个hash指定键的值 hmset key field value field value ....
  * `hget` 获取某个hash指定键的值 hget key field
  * `hmget` 获取多个hash指定键的值 hmget key field field field ...
  * `hsetnx` 不存在时创建并返回`OK`
  * `hgetall` 获取hash的所有键及值
  * `hexists` 判断某个hash指定键是否存在,如果存在返回1 否则返回0 
  * `hlen` 返回hash中所有键的数量
  * `hdel` 删除某个hash指定的键 可连续删除多个
  * `hkey` 返回hash的所有键
  * `hvals` 返回hash的所有键的值
  * `hincrby` 对指定键的值进行计算 增加一个整数值 返回新值
  * `hincrbyfloat` 对指定键的值进行计算增加一个浮点值 返回新值
* **EXPIRE** 超时 del set getset等会清空或者删除超时信息
  * `ttl` 返回key超时剩余时间(秒) 2.8版本之后 -2:表示key不存在 -1:表示key存在但未设置超时时间,2.6版本及之前均返回-1
  * `expire ` expire key seconds 设置key的超时时间,小于等于0执行删除操作
  * `pttl` 返回key超时剩余时间(毫秒) 只在2.6版本以上 
  * `expireat` expire key millisecond 与expire作用一样 不过**millisecond**是1970到现在的毫秒值,例如1293840000,如果小于当前时间则删除
* **LIST** 集合的值不具有唯一性
* **SET** 具有唯一性
  * `sadd`  key value1 value2 向集合中添加元素,并返回添加的个数,如果有重复则跳过
  * `smembers` 获取某个集合中所有的值
  * `sismember` 查看集合中是否包含指定的元素,有返回1 无返回0
  * `scard` 获取集合中元素的总个数
  * `srem` 删除集合中某个元素
  * `srandmember`  key count 随机查询集合中的元素
  * `spop`  key count 随机删除集合中的值 
  * `smove`  key1 key2 value 将key1的value移到key2集合中
  * `sdiff`  key1 key2... 差集 key1中存在 key2不存在
  * `sinter`  key1 key2... 交集 key1与key2 中公有的元素
  * `sunion`  key1 key2... 并集  
* **有序集合Sorted Set***
  * 元素唯一 score可重复
  * `zadd`  key score member socre member 添加或更新元素
  * `zcard`  key 获取key的元素的个数
  * `zcount`  key min max 在指定分数区间的元素个数
  * `zincrby`  key increment member 在指定元素上将分数增加increment
  * `zinterstore`  destination numberkeys key1 key2 将key1 key2集合元素的交集放入到destination集合中,score默认会相加
  * `zunionstore ` 并集
  * 
  * `zlexcount`  key min max 在指定元素min与max之间元素的个数 支持"[":包含,"(":不包含
  * `zrange`  key start stop [WITHSCORES] 根据下标查找集合中的元素-升序
  * `zrevrange` key start stop 根据下标查找元素的集合-升序
  * `zrangebylex`  key min max [LIMIT offset count] 查找在元素min与max之间的元素,支持类似mysql的分页方式
  * `zrangebyscore` key min max 通过分数查找元素-升序
  * `zrevrangebyscore` key min max 通过分数查找元素-降序
  * `zrank` key member 返回指定元素在集合中的下标-
  * `zrevrank` key member 返回指定元素咋集合中的排名,降序模式,0:最大
  * `zrem` key member member... 删除指定元素
  * `zremrangebylex` key min max 在指定元素min与max之间删除元素
  * `zremrangebyrank` key start stop根据下标删除元素
  * `zremrangebyscore` key min max 根据分数区间删除元素
  * `zscore` key member 返回元素的分数
  * `zscan` 指针方式遍历
  
* **[keyspace notifications](https://redis.io/topics/notifications)** 键空间通知事件 类似于redis的订阅,发布.默认关闭
  * `K` keyspace事件,事件以__keyspace@<db>__为前缀发布的
  * `E` keyevent事件,事件以__keyevent@<db>__为前缀发布的
  * `g` 一般性,非特定类型的命令,例如 del expire rename等
  * `$` 字符串特定命令
  * `l` 列表特定命令
  * `s` 集合特定命令
  * `h` 哈希特定命令
  * `z` 有序集合特定命令
  * `x` 过期事件 当某个键过期会产生该事件
  * `e` 驱逐事件 当某个键因maxmemory策略而被删除时,产生该事件(内存不足导致的清空部分存量数据,即过期策略)
  * `A` g$lshzxe上述所有命令 因此AKE意味着所有事件
  * `config set notify-keyspace-events ` 开启命令 必须包含`K` `E` 其中之一,否则不会发布任何事件
  * `psubscribe __keyevent@0__*` keyevent类型所有命令(前提开启上述事件),事件以`:`开始 即 `__keyevent@0__:expired` `__keyevent@0__:rename_to`等
* **事件类型说明**
  * `del` del事件
  * `rename` 两个事件,源键产生一个rename_from事件,目标键产生一个rename_to事件
  * `expire` expired事件
  * `set` 包含类似`setex` `setnx` `getset`产生 set事件 另外setex命令还产生expired事件
  * `mset` set事件
  * `setrange` setrange事件
  * `incr,decr、incrby和decrby` incrby事件
  * `incrbyfloat` incrbyfloat事件
  * `append` append事件
  * `lpush,lpushx` 产生lpush事件
  * `rpush` `rpushx` rpush事件
  * `rpop` rpop事件,如果被弹出的是最后一个元素还产生一个del事件
  * `lpop` 与上述类似
  * `linsert ` linsert事件
  * `lset` lset事件
  * `lrem` lrem事件 如果执行该命令后列表被清空还会有del事件
  * `ltrim` 与上述类似
  * `rpoplpush,brpoplpush` rpop,lpush事件并且保证 rpop在lpush之前,如果列表被清空额外产生del事件
  * `hset,hsetnx,hmset` hset事件
  * `hincrby ` hincrby 事件
  * `hincrbyfloat ` hincrbyfloat 事件
  * `hdel` hdel 事件 如果hash被清空额外产生del事件
  * `sadd` sadd 事件
  * `srem` srem 事件,集合被清空额外产生srem 事件
  * `smove` 为源键产生srem事件,目标键产生sadd事件
  * `spop` spop事件,集合被清空后额外产生del事件

