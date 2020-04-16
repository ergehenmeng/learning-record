#### 基本说明

* `ThreadPoolExecutor#prestartAllCoreThreads` 提前触发核心线程的创建
* `ctl属性` 标示线程池状态及线程池数量
  * `11100000000000000000000000000000` RUNNING 接受新任务,持续处理队列中的任务 
  * `00000000000000000000000000000000` SHUTDOWN 调用shutdown()方法,不再接受新任务,但要处理队列中的任务
  * `00100000000000000000000000000000` STOP 调用shutdownNow()方法,不再接受新任务,不再处理队列中的任务,中断正在运行的任务
  * `01000000000000000000000000000000` TYDYING 线程池正在停止运行,终止所有任务,销毁所以线程
  * `01100000000000000000000000000000` TERMINATED 线程池已经停止,任务已经执行完毕或清空,线程已经销毁
  * `11100000000000000000000000000000` ctl初始值
  * `11100000000000000000000000000000` ~CAPACITY
  * `00011111111111111111111111111111` CAPACITY

* `ThreadPoolExecutor#getTaskCount` 需要执行任务的数量
* `ThreadPoolExecutor#getCompletedTaskCount` 已经执行完的任务数
* `ThreadPoolExecutor#getLargestPoolSize` 曾经创建的最大线程数
* `ThreadPoolExecutor#getPoolSize` 当前线程池中的线程数
* `ThreadPoolExecutor#getActiveCount` 正在运行任务的线程数