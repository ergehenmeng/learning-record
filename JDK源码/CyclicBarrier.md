> 允许一组线程进行相互等待,在达到某公共屏障点之前相互等待
>
> 即:线程达到障碍点时阻塞,直到所有线程都达到障碍点才能继续运行

```java
  //await()等待方法
  private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
  TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
      final Generation g = generation;
			//表示CyclicBarrier已经重置或则中断(breakBarrier)
      if (g.broken)
        throw new BrokenBarrierException();
			
      if (Thread.interrupted()) {
        breakBarrier();
        throw new InterruptedException();
      }
			
      int index = --count;
      //表示当前是最后一个线程达到障碍点,开始执行barrierCommand
      if (index == 0) {
        boolean ranAction = false;
        try {
          final Runnable command = barrierCommand;
          if (command != null)
            command.run();
          ranAction = true;
          //重置栅栏数并唤醒所有等待的线程
          nextGeneration();
          return 0;
        } finally {
          if (!ranAction)
            breakBarrier();
        }
      }
      //不是最后一个线程,进入阻塞状态
      for (;;) {
        try {
          if (!timed)
            trip.await();
          else if (nanos > 0L)
            nanos = trip.awaitNanos(nanos);
        } catch (InterruptedException ie) {
          if (g == generation && ! g.broken) {
            breakBarrier();
            throw ie;
          } else {
            Thread.currentThread().interrupt();
          }
        }
				
        if (g.broken)
          throw new BrokenBarrierException();
				//最后一个线程会重置generation,因此一般情况下两个generation已经不是同一批次的东西了
        if (g != generation)
          return index;

        if (timed && nanos <= 0L) {
          breakBarrier();
          throw new TimeoutException();
        }
      }
    } finally {
      lock.unlock();
    }
  }
```

