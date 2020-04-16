> 可以延迟获取元素的阻塞队列,**队列头的元素时最先到期的元素**
>
> 内部实现 ReentrantLock+PriorityQueue(与PriorityBlockingQueue类似),且内部元素必须实现Delayed接口

```java
	//添加元素
  public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
      //将原始添加带有优先级的队列中
      q.offer(e);
      //如果当前添加的元素时优先级最高的,则唤醒等待的线程
      if (q.peek() == e) {
        leader = null;
        available.signal();
      }
      return true;
    } finally {
      lock.unlock();
    }
  }

  public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
      for (;;) {
        E first = q.peek();
        //队列为空,等待元素
        if (first == null)
          available.await();
        else {
          long delay = first.getDelay(NANOSECONDS);
          //头元素已经到期,直接移除头元素并返回
          if (delay <= 0) {
            return q.poll();
          }
          //头元素依旧没有到期
          first = null; 
          //表示有其他线程已经在,则当前线程阻塞
          if (leader != null) {
            available.await();
          } else {
            //设置当前线程为主要获取元素的线程
            Thread thisThread = Thread.currentThread();
            leader = thisThread;
            try {
              available.awaitNanos(delay);
            } finally {
              if (leader == thisThread)
                leader = null;
            }
          }
        }
      }
    } finally {
      //有元素时才唤醒阻塞的线程(减少线程切换代价)
      if (leader == null && q.peek() != null)
        available.signal();
      lock.unlock();
    }
  }
//与take方法类似,该方法支持超时时间,如果超时依旧没有获取到,则返回null
//线程池最大线程数减少到核心线程就是通过poll方法判断的
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null) {
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    if (nanos <= 0)
                        return null;
                    first = null; // don't retain ref while waiting
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```



