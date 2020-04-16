> 信号量是一个共享资源控制的计数器, 和`CountDownLatch`类似,`Semaphore`支持公平模式和非公平模式
>
> 类似于停车场一样,车位数量是固定的,车辆满了就无法继续停车,只能干等着,直到有车出来才能继续进入



```java
//非公平模式下获取锁
final int nonfairTryAcquireShared(int acquires) {
  for (;;) {
    int available = getState();
    int remaining = available - acquires;
    if (remaining < 0 ||
        //尝试获取资源
        compareAndSetState(available, remaining))
      return remaining;
  }
}

//公平模式
protected int tryAcquireShared(int acquires) {
  for (;;) {
    //有比当前线程更靠前的节点
    if (hasQueuedPredecessors())
      return -1;
    int available = getState();
    int remaining = available - acquires;
    if (remaining < 0 ||
        compareAndSetState(available, remaining))
      return remaining;
  }
}

//释放锁
protected final boolean tryReleaseShared(int releases) {
  for (;;) {
    int current = getState();
    //将可用锁再加回去
    int next = current + releases;
    if (next < current)
      throw new Error("Maximum permit count exceeded");
    if (compareAndSetState(current, next))
      return true;
  }
}

//减少锁,不能增加
final void reducePermits(int reductions) {
  for (;;) {
    int current = getState();
    int next = current - reductions;
    if (next > current) 
      throw new Error("Permit count underflow");
    if (compareAndSetState(current, next))
      return;
  }
}

//将可用锁置为零	
final int drainPermits() {
  for (;;) {
    int current = getState();
    if (current == 0 || compareAndSetState(current, 0))
      return current;
  }
}


```

