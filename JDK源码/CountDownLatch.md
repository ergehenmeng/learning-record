> 在完成一组正在其他线程中执行的操作前,允许一个或多个线程一直等待,与`CyclicBarrier`类似

#### CyclicBarrier与CountDownLatch区别

* `CyclicBarrier`基于`ReentrantLock`实现的,在最后一个线程执行完任务之前,其他线程必须等待,支持重置操作
* `CountDownLatch` 基于`AQS`实现的,线程在调用`countDown()`后如果state归零则唤醒所有等待的线程,不支持重置

```java
//CountDownLatch基于AQS的共享锁实现的 await()是获取共享锁的方法,会调用以下方法
//注意:在state不等于0时表示获取锁失败,等于0是获取成功
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
// countDown()方法则是释放锁,会调用以下方法
// 注意:当state归零时,会唤醒所有的线程继续执行
protected boolean tryReleaseShared(int releases) {
  for (;;) {
    int c = getState();
    if (c == 0)
      return false;
    int nextc = c-1;
    if (compareAndSetState(c, nextc))
      return nextc == 0;
  }
}
```

