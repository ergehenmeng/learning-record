Mysql

#### explain分析:
* **id** SELECT识别符。这是SELECT的查询序列号 
* **select_type** 查询类型如下:

 > * 1.SIMPLE: 简单查询 不使用union或子查询 
 > * 2.PRIMARY: 最外层的查询
 > * 3.UNION: UNION中的第二个或后面的SELECT语句
 > * 4.DEPENDENT UNION: UNION中的第二个或后面的SELECT语句,取决于外面的查询

执行过程:
* redo log日志,在mysql中是固定大小的,可配置为一组4个文件,每个文件1G,这个是InnDB引擎的日志

* mysql就是采用的WAL技术(write-ahead-loggin)先写日志,再写磁盘

* checkpoint要擦除的位置,write pos要写入的位置

* server层日志称之为binlog

* `innodb_flush_log_at_trx_commit=1` 每次事务的redo log都能写入磁盘,mysql异常重启时不会丢失数据

* `sync_binlog=1 `每次事务的binlog都持久化到磁盘

* `sync_binlog=0` 每次提交事务都只write,不fsync

* `sync_binlog=N` (N>1) 表示每次提交事务都write,但累计N个事务后才fsync

  

* show variables like 'transaction_isolation' 查看事务隔离级别

* > set autocommit=0 会把这个线程的自动提交关掉,并且不会自动提交,直到你主动执行commit或者rollback或者断开链接,通常建议 set autocommit=1

* > commit work and chain 提交之后再自动开启新事务,省去begin的开销

* 查询超过60秒的事务  `select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60`

* set autocommit=0 ，每次commit/rollback之后，再执行一个SQL语句自动开启事务，就是这么个设定

* `start transaction with consistent snapshot` 立即启动事务 begin/start transaction并不是一个事务的起点,只有真正执行到第一个操作语句的时候才启动
####索引说明
* 主键索引的叶子节点存的是整行数据,也称之为聚簇索引(clustered index)
* 非主键索引的叶子节点存放的是主键的值,也称之为二级索引(secondary index)
* 因此在使用二级索引时,先搜索二级索引树(B+),找到主键后再通过主键索引树搜索一次,这个过程称之为回表
* InnoDB的数据是按数据页为单位读写的,也就是说当读取一条的记录时,会以页为单位整体读到内存,InnoDB默认页大小为16KB
* InnoDB,如果数据页没有在内存中时,在不影响一致性前提下,会将更新操作写入到change buffer中,在下次访问数据页时,将数据页读到内存然后执行change buffer与这个也有关的操作(merge操作)
* `innodb_change_buffer_max_size` 用来设置 change buffer在 buffer pool中占比
* `change buffer`在写多读少的业务中比较合适,反之在更新后又访问这个数据页,会将这个数据页加载到内存,然后merge,起不到change buffer的作用,而且增加维护成本
* 重新进行索引统计 `analyze table ....`  可以纠正mysql在查询时使用慢索引,explain中预估扫描行数不准时,可以采用此方法
* `optimize table` = 'recreate + analyze' 
* `alter table t engine = InnoDB` 自动复制并生成新的表
* 强制使用某索引 force index 例如 select a from t force index(b) where b = 1;

####唯一索引和普通索引
* 3与5之间插入4分析:
> 第一类情况,要更新的目标页在内存中
   * 唯一索引:找到3与5之间的位置,判断是否有冲突值,插入这个值,执行结束;
   * 普通索引:找到3与5之间的位置,插入这个值,执行结束;
> 第二类情况,要更新的目标页不在内存中
   * 唯一索引:将数据页加载到内存,判断是否有冲突,插入这个值,执行结束;
   * 普通索引:将更新记录写入到change buffer,执行结束  


####全局锁
* 对整个数据库加锁,命令: Flush table with read lock (FTWRL),数据库会处于只读状态,其他线程执行时会被阻塞(增删改,数据定义,事务提交)
* 主要做全库的逻辑备份
* –single-transaction InnoDB在导出数据之前启动一个事务来保证一致性视图,可以不使用FTWRL,但MyISAM等引擎不支持事务需要使用FTWRL
* set global readonly=true也可以实现全库只读,但不建议用,原因如下:
   * 有些系统通过readonly来判断是否为主库还是从库
   * 异常处理机制不一样,FTWRL命令在客户端发生异常断开时会自动释放这个全局锁,而readonly客户端发生异常时,数据库一直保持readonly状态
####表级锁
* 语法:`lock tables ... read/write` 释放锁 `unlock tables ... read/write`
* 上述操作会限制其他线程的读或写,同时当前线程也会限制为只读或者只写
####表级锁MDL(元数据锁)meta data lock
* 数据结构变更,例如删除一列,修改一列等
* 不需要显式使用,在访问一个表时会动态添加
* 作用:保证读写的正确性
* 5.5版本有效
* 读锁之间不互斥
* 读写锁之间,写锁之间互斥,保证变更表结构操作的安全性
* ![](https://i.imgur.com/Jyd5bDL.png)

####行级索
* 在InnoDB中行级锁是在需要时才添加,但并不是不需要就立即释放,而是等到事务结束时才释放,即两阶段锁协议
* 在事务中如果需要锁多行,最可能出现锁冲突的,最可能影响并发度的锁尽量往后放
* 死锁
* `lock in share mode` 共享

![](https://i.imgur.com/2oGuK8L.png)

* 上述死锁处理方案:
   * 第一种:直接进入等待,直到超时,超时时间可以通过 innodb_lock_wait_timeout设置(该值默认时间是50s)
   * 第二种:发起死锁检查,主动回滚死锁链中的某一个事务,让其他事务得以执行,inndb_deadlock_detect设置为on
* show engine innodb status 死锁sql查询

* **innodb_file_per_table** 参数:
  * off 表的数据存放在系统共享表空间,也就是和数据字典放在一起
  * on 每个InnoDB表数据均以.ibd文件存储,在5.6.6之后默认,如果drop table的时候直接删除这个文件(推荐)
* alter table A engine=InnoDB 将A表数据复制到B表,表结构等不变

* **sort_buffer_size** mysql为排序开辟出来的内存空间,如果要排序的数据小,则直接在内存中排序,如果数据量太大,不得不利用磁盘文件辅助排序
* **max_length_for_sort_data** 控制排序的行数据的长度


* 四大特性:原子性,一致性, 隔离性,持久性
	![](https://i.imgur.com/jscNor0.png) 	
![](https://i.imgur.com/0bhQoUI.png)
![](https://i.imgur.com/6BpW1FX.png)

##复合索引最左索引
* 复合索引 index(a,b)
* select * from user  where a=? and b= ? 使用索引
* select * from user where a=? 使用索引
* select * from user where b = ? 不使用索引

##离线安装 
* `mysqld --defaults-file=C:\windows\my.ini mysql`

<img src="事务流程图.jpg" style="width:500px;"/>

##数据库优化

* `innodb_buffer_pool_size` Innodb的缓冲池 默认为128M,一般调整为物理内存的80%,过大会发生swap页交换
* `innodb_buffer_pool_instances` 缓冲池里面实例的大小,一般不超过`innodb_read_io_threads `+ `innodb_write_io_threads` 建议为实例和线程数量比例为1:1
* `innodb_read_io_threads` IO读线程
* `innodb_write_io_threads` IO写线程
* `innodb_log_buffer_size` InnoDB重做日志缓冲池的大小  默认8M,在高并发时会增加写入磁盘的IO操作
* 触发redo log写盘操作的方式
  * 在该缓存池占用一半时,后台线程会触发主动写盘,注意由于这个事务没有提交,这个写盘操作只是write,而没有调用fsync,也就是只留了文件系统的page cache
  * InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。
  * 

<img src="redo_bin_log.png" style="width:600px;"/>

* `innodb_flush_log_at_trx_commit` 这个参数可以控制重做日志从缓存写入文件刷新到磁盘中的策略，默认值为 1

  * 0 表示每次事务提交时都只是把redo log留在redo log buffer中
  * 1 表示每次事务提交都将redo log直接持久化到磁盘中
  * 2 表示每次事务提交时都是将redo log写到page cache中

* `innodb_max_dirty_pages_pct` 脏页的比例上限 默认75%

  <img src="D:\learning\cc44c1d080141aa50df6a91067475374.png" style="width:400px;"/>

  

* `innodb_io_capacity` 磁盘读写能力

* `innodb_flush_neighbors ` 1:在将脏页flush到文件的时候, 如果相邻的数据页也是脏页,也会"连坐",flush到磁盘, 0:则不会"连坐"(SSD建议设置为此值)

* `sort_buffer_size` 排序缓存大小 

* `slow_query_log` 慢sql日志开关

* `long_query_time` 慢sql时间,超过该值,会将sql记录到慢查询日志中(slow log)

* `binlog_cache_size ` 单个线程内binlog cache所占的内存大小

* `binlog_group_commit_sync_delay` 表示延迟多少微秒后才调用 fsync

* `binlog_group_commit_sync_no_delay_count ` 表示累积多少次以后才调用 fsync。

``` mysql

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

* `max_length_for_sort_data` 单行数据长度超过该值后,mysql会自动切换为rowid排序规则
* `tmp_table_size ` 内存临时表大小 用于排序
* `default_tmp_storage_engine` 内存临时表使用的引擎

```mysql
-- 随机取一条 
select count(*) into @C from t;
set @Y = floor(@C * rand());
set @sql = concat("select * from t limit ", @Y, ",1");
prepare stmt from @sql;
execute stmt;
DEALLOCATE prepare stmt;
```

```mysql
-- 伪随机算法,id连续时可以使用
select max(id),min(id) into @M,@N from t ;
set @X= floor((@M-@N+1)*rand() + @N);
select * from t where id >= @X limit 1;
```

* `performance_schema=on` 开启sys数据库,mysql默认只显示`mysql`和`infomation_schema`数据库
* `flush tables t with read lock` 关闭指定的表
* `flush tables with read lock;` 关闭所有打开的表 

加锁原则

> 原则 1：加锁的基本单位是 next-key lock。希望你还记得，next-key lock 是前开后闭区间。
>
> 原则 2：查找过程中访问到的对象才会加锁。
>
> 优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
>
> 优化 2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
>
> 一个 bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。

5.7新增的查询重写功能

```mysql

mysql> insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");

call query_rewrite.flush_rewrite_rules();
```















