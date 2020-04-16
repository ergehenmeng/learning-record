> ReentrantLock是一个独占锁,默认非公平锁,

#### 公平锁

```java
  protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      //判断队列是否有比当前线程更靠前的节点存在,如果没有则尝试获取锁
      if (!hasQueuedPredecessors() &&
          compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    //当前线程已经持有该锁(重入)
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
```



#### 非公平锁(默认)

```java
  final void lock() {
    //不管能不能获取到锁,先尝试一波再说
    if (compareAndSetState(0, 1))
      setExclusiveOwnerThread(Thread.currentThread());
    else
      //尝试获取锁
      acquire(1);
  }

  protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
  }
  final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      //此处不判断队列是否有其他节点,先抢一波试试看
      if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
```

#### Condition

```java
//条件等待  
public final void await() throws InterruptedException {
    if (Thread.interrupted())
      throw new InterruptedException();
    //包装为一个条件节点并添加到条件等待队列中(条件等待队列为单向链表)
    //同时会清除队列中非条件节点(节点已经在同步队列中或者已取消,此时没必要在条件等待队列中占坑)
    Node node = addConditionWaiter();
    //释放当前线程持有的锁资源
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
      //阻塞当前线程
      LockSupport.park(this);
      if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
        break;
    }
    //将线程添加到同步队列中以便于获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
      interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
      unlinkCancelledWaiters();
    if (interruptMode != 0)
      reportInterruptAfterWait(interruptMode);
  }
	
  //条件唤醒(唤醒一个),唤醒首节点
  private void doSignal(Node first) {
    do {
      if ( (firstWaiter = first.nextWaiter) == null)
        lastWaiter = null;
      first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
  }

  final boolean transferForSignal(Node node) {
    //首节点状态变更失败,尝试变更首节点的下一个节点
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
      return false;
    //状态变更成功,将该节点假如到同步队列中,p为node节点的前置节点
    Node p = enq(node);
    int ws = p.waitStatus;
    //p节点已取消,或者 将p节点改为signal失败,则直接唤醒该node节点(相当于不用在同步队列中排队.没看出来此处的用意)
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
      LockSupport.unpark(node.thread);
    return true;
  }

```

