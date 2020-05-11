#### 命令参数

* `-Xmn` 新生代堆的大小
* `-Xss` 线程栈的大小 
* `-Xmx` 最大堆
* `-Xms` 最小堆
* `-XX:SurvivorRatio` 年轻代Eden与Survivor比值
* `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=.` 在OOM时,自动生成堆转储文件

#### 垃圾收集器

##### 串行收集器(Serial)

> 古老的单线程收集器,`stop the world` 时间可能比较长 适合单核的桌面cpu

* 新生代和老年代都会使用Serial收集器
* 新生代采用复制算法
* 老年代采用标记-整理算法

##### 并行收集器(Parallel)

> 采用多线程并行处理GC,内存和核心数够多时,效率更好,它也被成为**吞吐量优先的垃圾收集器**

##### 并行收集器(Old Parallel)

> 与Parallel唯一区别就是老年代的gc算法不同 它执行的步骤:标记-汇总-压缩. 汇总步骤与清除的不同之处在于,他将依然幸存的对象分发到gc预先处理好的不同区域,算法相对清理来说比较复杂(?????)

##### 并发收集器(CMS)

> 是一种以获取最短回收停顿时间为目标的收集器,适合B/S系统的服务器

##### G1收集器

> 为了代替cms收集器而创建的,cms在长时间持续运行时会产生很多问题(????)

| Young             | Tenured      | Jvm options                                   |
| ----------------- | ------------ | --------------------------------------------- |
| Serial            | Serial       | -XX:+UseSerialGC                              |
| Parallel Scavenge | Parallel Old | -XX:+UsePrarallelGC  -XX:+UseParallelOldGC    |
| Parallel New      | CMS          | -XX:+UseParallelNewGC -XX:+UseConcMarkSweepGC |
| G1                | G1           | -XX:+UseG1GC                                  |

