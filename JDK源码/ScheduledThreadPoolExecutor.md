> 可以延迟执行或周期执行任务的线程池,继承`ThreadPoolExecutor`

#### 源码分析

> 延迟执行任务

```java
//如果delay为0表示立即执行 与execute没区别
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
  if (command == null || unit == null)
    throw new NullPointerException();
  //任务包装成ScheduledFutureTask并执行
  RunnableScheduledFuture<?> t = decorateTask(command, new ScheduledFutureTask<Void>(command, null, triggerTime(delay, unit)));
  delayedExecute(t);
  return t;
}


private void delayedExecute(RunnableScheduledFuture<?> task) {
  if (isShutdown())
    reject(task);
  else {
    super.getQueue().add(task);
    if (isShutdown() &&
        //在线程池关闭阶段,根据任务是否是周期的判断是否可以还能执行任务
        !canRunInCurrentRunState(task.isPeriodic()) &&
        remove(task)) {
      task.cancel(false);
    } else {
      //因为只添加到了延迟队列中,此处要确保有线程存在才可执行上述任务
      ensurePrestart();
    }
  }
}
```

> 延迟队列的核心类 `DelayedWorkQueue` 类似于`DelayQueue`  可以延迟获取队列中的元素,也是基于二叉堆实现的

```java
//添加元素到队列中
public boolean offer(Runnable x) {
  if (x == null)
    throw new NullPointerException();
  RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
  final ReentrantLock lock = this.lock;
  lock.lock();
  try {
    int i = size;
    //任务队列需要扩容
    if (i >= queue.length)
      grow();
    size = i + 1;
    if (i == 0) {
      queue[0] = e;
      //将元素下标与自身绑定,加速查找或删除任务
      setIndex(e, 0);
    } else {
      //根据二叉堆的添加规则将元素添加到指定位置
      siftUp(i, e);
    }
    //插入之后发现新插入的元素在堆的顶部,则尝试唤醒等待的线程
    if (queue[0] == e) {
      leader = null;
      available.signal();
    }
  } finally {
    lock.unlock();
  }
  return true;
}


//延迟获取元素
public RunnableScheduledFuture<?> take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();
  try {
    for (;;) {
      RunnableScheduledFuture<?> first = queue[0];
      if (first == null)
   			//队列中还木有元素
        available.await();
      else {
        long delay = first.getDelay(NANOSECONDS);
        if (delay <= 0)
          //时间到了,可以拉出去消费了,并将此元素从队列中删除
          return finishPoll(first);
        first = null; 
        //这些代码与DelayQueue#take基本一致
        if (leader != null)
          available.await();
        else {
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
    if (leader == null && queue[0] != null)
      available.signal();
    lock.unlock();
  }
}

// 任务执行
public void run() {
  boolean periodic = isPeriodic();
  if (!canRunInCurrentRunState(periodic))
    cancel(false);
  else if (!periodic)
    ScheduledFutureTask.super.run();
  //表示是个周期性任务,只有运行成功的新增任务才会返回true
  else if (ScheduledFutureTask.super.runAndReset()) {
    //设置下次要执行的时间
    setNextRunTime();
    //因为任务已经执行完了,如果要周期执行,需要重新添加到队列中
    //outerTask与this实际上是同一个对象,因此setNextRunTime()就是设置自己的运行周期等字段
    //所以如果重写线程池的decorateTask()方法,在某种程度上可以让任务执行一次后,执行自己自定义的方法,甚至可以中断周期调用
    reExecutePeriodic(outerTask);
  }
}

```

