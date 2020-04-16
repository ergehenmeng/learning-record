> 读写锁与ReentrantLock一样,内部采用Sync实现AQS接口
>
> state却分为读写,高16位读锁,低16位写锁

```java
  static final int SHARED_SHIFT   = 16;

  static final int SHARED_UNIT    = (1 << SHARED_SHIFT);

  static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;

  static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
  
  //读锁判断,右移,将低位的16位抹去,只保留高位16
  static int sharedCount(int c)    { 
    return c >>> SHARED_SHIFT; 
  }
	//与运算 EXCLUSIVE_MASK = 0000000000000000111111111111111 = 0x0000FFFF,抹去高位只保留低位
  static int exclusiveCount(int c) {
    return c & EXCLUSIVE_MASK; 
  }
```

> 写锁(排他锁)

```java
//获取写锁  
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
      //w==0 表示没有写锁只有读锁
      if (w == 0 || current != getExclusiveOwnerThread())
        return false;
      //写锁数量上限
      if (w + exclusiveCount(acquires) > MAX_COUNT)
        throw new Error("Maximum lock count exceeded");
      setState(c + acquires);
      return true;
    }
    //公平锁时要判断是否有比当前等待更长的线程存在,如果存在则获取失败
    //非公平锁则直接尝试获取
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
      return false;
    //将线程与当前所对象绑定
    setExclusiveOwnerThread(current);
    return true;
  }
	//释放写锁
  protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
      throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    //判断写锁是否全部释放(主要是重入锁判断)
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
      setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
  }
```

> 读锁(共享锁)

```java
  protected final int tryAcquireShared(int unused) {
    
    Thread current = Thread.currentThread();
    int c = getState();
    //有写锁存在,并且当前线程与绑定的线程对象(写锁时会绑定线程对象)不一样(锁降级)
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
      return -1;
    int r = sharedCount(c);
    //尝试快速获取读锁
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        //高位state+1(重入依旧会+1)
        compareAndSetState(c, c + SHARED_UNIT)) {
      if (r == 0) {
        firstReader = current;
        firstReaderHoldCount = 1;
      } else if (firstReader == current) {
        firstReaderHoldCount++;
      } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
          cachedHoldCounter = rh = readHolds.get();
        else if (rh.count == 0)
          readHolds.set(rh);
        rh.count++;
      }
      return 1;
    }
    //再次获取读锁
    return fullTryAcquireShared(current);
  }
	
 //公平模式
 final boolean readerShouldBlock() {
   return hasQueuedPredecessors();
 }
//非公平模式
  final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
  }
  //判断队列的头节点(头结点不保存信息)的next节点是否为排他节点
  final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
      (s = h.next)  != null &&
      !s.isShared()         &&
      s.thread != null;
  }


 //完全体的获取共享锁
  final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
      int c = getState();
      if (exclusiveCount(c) != 0) {
        if (getExclusiveOwnerThread() != current)
          return -1;
      } else if (readerShouldBlock()) {
        if (firstReader == current) {
        } else {
          if (rh == null) {
            rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current)) {
              rh = readHolds.get();
              if (rh.count == 0)
                readHolds.remove();
            }
          }
          if (rh.count == 0)
            return -1;
        }
      }
      if (sharedCount(c) == MAX_COUNT)
        throw new Error("Maximum lock count exceeded");
      //获取读锁成功
      if (compareAndSetState(c, c + SHARED_UNIT)) {
        //第一次获取读锁
        if (sharedCount(c) == 0) {
          firstReader = current;
          firstReaderHoldCount = 1;
        } else if (firstReader == current) {
          firstReaderHoldCount++;
        } else {
          if (rh == null)
            rh = cachedHoldCounter;
          if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
          else if (rh.count == 0)
            readHolds.set(rh);
          rh.count++;
          cachedHoldCounter = rh; 
        }
        return 1;
      }
    }
  }

	  //释放共享锁
  protected final boolean tryReleaseShared(int unused) {
    //这坨代码可以大致理解为用来记录重入锁的重入次数
    Thread current = Thread.currentThread();
    if (firstReader == current) {
      if (firstReaderHoldCount == 1)
        firstReader = null;
      else
        firstReaderHoldCount--;
    } else {
      HoldCounter rh = cachedHoldCounter;
      if (rh == null || rh.tid != getThreadId(current))
        rh = readHolds.get();
      int count = rh.count;
      if (count <= 1) {
        readHolds.remove();
        if (count <= 0)
          throw unmatchedUnlockException();
      }
      --rh.count;
    }
    //释放锁
    for (;;) {
      int c = getState();
      int nextc = c - SHARED_UNIT;
      if (compareAndSetState(c, nextc))
        return nextc == 0;
    }
  }

```

#### 锁降级

> 在获取写锁释放之前又获取了读锁 称之为锁降级,如果在写锁释放后又获取读锁不能成为锁降级
>
> ReentrantReadWriterLock 支持锁降级,不支持锁升级

#### 锁降级的作用

* 锁降级后,由于当前所持有读锁,因此会阻塞其他的写锁,可以将本次写的数据被其他读锁锁感知(如果不获取读锁而直接释放写锁,其他线程获取了写锁,则该线程无法感知到其他写锁线程更新的数据),保证了数据的可见性
* 