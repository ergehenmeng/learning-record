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