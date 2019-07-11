
## Mysql
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
* innodb_flush_log_at_trx_commit=1 每次事务的redo log都能写入磁盘,mysql异常重启时不会丢失数据
* sync_binlog=1 每次事务的binlog都持久化到磁盘
* read uncommited 读未提交是指，一个事务还没提交时，它做的变更就能被别的事务看到。
* read commited 读提交是指，一个事务提交之后，它做的变更才会被其他事务看到。
* repeatable read 可重复读是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。

* show variables like 'transaction_isolation' 查看事务隔离级别
* > 显式开启事务 begin或start transaction, commit提交事务 rollback回滚
* > set autocommit=0 会把这个线程的自动提交关掉,并且不会自动提交,直到你主动执行commit或者rollback或者断开链接,通常建议 set autocommit=1
* > commit work and chain 提交之后再自动开启新事务,省去begin的开销
* 查询超过60秒的事务 select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
* set autocommit=0 ，每次commit/rollback之后，再执行一个SQL语句自动开启事务，就是这么个设定
* > start transaction with consistent snapshot 立即启动事务
####索引说明
* 主键索引的叶子节点存的是整行数据,也称之为聚簇索引(clustered index)
* 非主键索引的叶子节点存放的是主键的值,也称之为二级索引(secondary index)
> 因此在使用二级索引时,先搜索二级索引树(B+),找到主键后再通过主键索引树搜索一次,这个过程称之为回表
> InnoDB的数据是按数据页为单位读写的,也就是说当读取一条的记录时,会以页为单位整体读到内存,InnoDB默认页大小为16KB
> InnoDB,如果数据页没有在内存中时,在不影响一致性前提下,会将更新操作写入到change buffer中,在下次访问数据页时,将数据页读到内存然后执行change buffer与这个也有关的操作(merge操作)
> innodb_change_buffer_max_size 用来设置 change buffer在 buffer pool中占比
> change buffer在写多读少的业务中比较合适,反之在更新后又访问这个数据页,会将这个数据页加载到内存,然后merge,起不到change buffer的作用,而且增加维护成本
> 重新进行索引统计 analyze table .... 可以纠正mysql在查询时使用慢索引
> 强制使用某索引 force index 例如 select a from t force index(b) where b = 1;
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
* 语法:lock tables ... read/write 释放锁 unlock tables ... read/write
* 上述操作会限制其他线程的读或写,同时当前线程也会限制为只读或者只写
####表级锁MDL(元数据锁)meta data lock
* 数据结构变更,例如删除一列,修改一列等
* 不需要显式使用,在访问一个表时会动态添加
* 作用:保证读写的正确性
* 5.5版本有效
* 读锁之间不互斥
* 读写锁之间,写锁之间互斥,保证变更表结构操作的安全性

![](https://i.imgur.com/Jyd5bDL.png)

####行级索
* 在InnoDB中行级锁是在需要时才添加,但并不是不需要就立即释放,而是等到事务结束时才释放,即两阶段锁协议
* 在事务中如果需要锁多行,最可能出现锁冲突的,最可能影响并发度的锁尽量往后放
* 死锁

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

```
打开 optimizer_trace，只对本线程有效
```

```
SET optimizer_trace='enabled=on'; 
```

```
 @a 保存 Innodb_rows_read 的初始值
```

```
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';
```

```
select city, name,age from t where city='杭州' order by name limit 1000; 
```

```
查看 OPTIMIZER_TRACE 输出
```

```
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```

```
@b 保存 Innodb_rows_read 的当前值
```

```
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';
```

```
计算 Innodb_rows_read 差值select @b-@a;
``` 

##复合索引最左索引
* 复合索引 index(a,b)
* select * from user  where a=? and b= ? 使用索引
* select * from user where a=? 使用索引
* select * from user where b = ? 不使用索引

##离线安装 
* mysqld --defaults-file=C:\windows\my.ini mysql