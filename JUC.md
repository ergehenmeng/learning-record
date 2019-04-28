####AtomicIntegerArray
> base=`unsafe.arrayBaseOffset` 数组头部保存数组长度的偏移量
> scale=`unsafe.arrayIndexScale` 获取数组中每个元素的长度比例
> 数组中每个元素的偏移量=base + scale* index 与如下结果一致:

```java
   int scale = unsafe.arrayIndexScale(int[].class);
   scale = 31 - Integer.numberOfLeadingZeros(scale);
   return ((long) index << scale) + base;
```
####AtomicLongFieldUpdater(类的成员变量操作原子性)
* 提供两种实现方式
   *  `CASUpdater`  如果虚拟机支``持long类型的cas
   *  `LockedUpdater` 采用synchronized锁的方式

####AtomicStampedReference与AtomicMarkableReference(作用基本相同)
* `AtomicStampedReference` 基于int方式(时间戳),改变过几次
* `AtomicMarkableReference` 基于boolean方式.是否被改变过


####AQS Node节点等待状态

* `CANCELLED` 值为1 当线程等待超时或者被中断，则取消等待，设等待状态为-1，进入取消状态则不再变化。
* `SIGNAL` 值为-1 后继节点处于等待状态，当前节点（为-1）被取消或者中断时会通知后继节点，使后继节点的线程得以运行
* `CONDITION` 值为-2 当前节点处于等待队列,节点线程等待在Condition上,当其他线程对condition执行signall方法时,等待队列转移到同步队列,加入到对同步状态的获取。
* `PROPAGATE` 值为-3 与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。
* `INITIAL` 值为0 代表初始化状态
