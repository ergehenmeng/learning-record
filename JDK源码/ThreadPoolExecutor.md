#### 基本说明

* `ThreadPoolExecutor#prestartAllCoreThreads` 提前触发核心线程的创建
* `ctl属性` 标示线程池状态及线程池数量
  * `11100000000000000000000000000000` running 接受新任务,持续处理队列中的任务 
  * `00000000000000000000000000000000` shutdown 调用shutdown()方法,不再接受新任务,但要处理队列中的任务
  * `00100000000000000000000000000000` stop 调用shutdownNow()方法,不再接受新任务,不再处理队列中的任务,尝试中断正在运行的任务
  * `01000000000000000000000000000000` tidying 线程池正在停止运行,终止所有任务,线程已中断,仅仅做一些扫尾工作
  * `01100000000000000000000000000000` terminated 线程池已经停止,任务已经执行完毕或清空,线程已经销毁
  * `11100000000000000000000000000000` ctl初始值 
  * `11100000000000000000000000000000` ~capacity
  * `00011111111111111111111111111111` capacity 最大线程数


```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
//线程最大数 29位 最大可创建的线程数
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
//以下表示表示线程池的一些状态,通过int值的高三位表示
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
//用于计算计算线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//当前线程池中的创建的线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }
//状态或线程数换算使用
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

#### 线程池的一些常用方法

* `ThreadPoolExecutor#getTaskCount` 需要执行任务的数量
* `ThreadPoolExecutor#getCompletedTaskCount` 已经执行完的任务数
* `ThreadPoolExecutor#getLargestPoolSize` 曾经创建的最大线程数
* `ThreadPoolExecutor#getPoolSize` 当前线程池中的线程数
* `ThreadPoolExecutor#getActiveCount` 正在运行任务的线程数

#### 源码分析

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
// 如果核心线程都没有达到上限,则直接创建核心线程
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true)){
              return;
            }
          //核心线程创建失败
            c = ctl.get();
        }
  //此时表示要么创建核心线程失败,要么核心线程已经够了
  //如果线程池是运行中,则尝试将任务添加到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
          //双重判断线程池状态,如果线程池不在运行状态,则拒绝并移除队列中的该任务
            if (! isRunning(recheck) && remove(command)){
               reject(command);
            }else if (workerCountOf(recheck) == 0)
              //防止shutdown状态下,没有任务线程,但队列中还有没有执行完的任务
                addWorker(null, false);
        }else if (!addWorker(command, false)) //队列满了直接拒绝
            reject(command);
    }
```

```java
  //添加工作线程
  private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
      int c = ctl.get();
      int rs = runStateOf(c);
      //判断线程池状态,如果是非运行状态并且不是addWorker(null,false)则直接返回失败,此时线程池没有待执行的任务,也已经不是运行状态了
      if (rs >= SHUTDOWN &&
          ! (rs == SHUTDOWN &&
             firstTask == null &&
             ! workQueue.isEmpty()))
        return false;

      for (;;) {
        int wc = workerCountOf(c);
        //线程上限了
        if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize)) {
          return false;
        }
        //线程数增加成功(仅仅将已有的线程数+1,并没有创建线程),跳出所有循环
        if (compareAndIncrementWorkerCount(c)) {
          break retry;
        }
        //增加线程数失败
        c = ctl.get(); 
        //其他操作已经将线程池状态变更,重新走流程
        if (runStateOf(c) != rs)
          continue retry;
      }
    }
    // 可以创建线程了
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
      //将该任务包装成Worker,内部关联一个新的线程
      w = new Worker(firstTask);
      final Thread t = w.thread;
      if (t != null) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
          
          int rs = runStateOf(ctl.get());
          //线程池状态为运行状态,或者关闭状态但还有任务需要执行
          if (rs < SHUTDOWN ||
              (rs == SHUTDOWN && firstTask == null)) {
            if (t.isAlive()) 
              throw new IllegalThreadStateException();
            //将任务线程添加到工作组中
            workers.add(w);
            int s = workers.size();
            if (s > largestPoolSize)
              //此处仅仅为了记录线程池曾经创建过的最大线程数
              largestPoolSize = s;
            workerAdded = true;
          }
        } finally {
          mainLock.unlock();
        }
        //启动线程
        if (workerAdded) {
          t.start();
          workerStarted = true;
        }
      }
    } finally {
      //线程并没有启动成功,可能是任务执行过程中抛异常了,进行失败处理
      if (! workerStarted)
        addWorkerFailed(w);
    }
    return workerStarted;
  }
```

```java
//任务线程对象,该类实现了AQS和Runable类,即可以作为锁使用也是任务
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
        
        final Thread thread;
        
        Runnable firstTask;
        
        volatile long completedTasks;
        
        Worker(Runnable firstTask) {
            setState(-1); 
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
  //前面调用的t.start()实际上就是thread.start()最终执行的就该方法
        public void run() {
          //最最最核心的方法,执行任务操作
            runWorker(this);
        }
}


final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
  //先释放锁,因为在构造方法中setState(-1),在w.lock时将会因为获取不到锁而阻塞
    w.unlock();
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
          //线程池已经停止,或者当前线程已中断(后面是进行再次判断线程池状态,说是双检测,感觉有点脱裤子放屁)
          //该处 !wt.isInterrupted()目的是,防止两次中断操作抛异常的问题(麻蛋,jdk代码为了简洁,可读性完全不顾了)
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) 
                && !wt.isInterrupted()) {
              //该处也只是打了个中断标示符,后面没有响应中断的代码,表示即便是线程中断了依旧可以将该代码运行完毕,在下一轮中中断判断
               wt.interrupt();
            }
            try {
              //可以自定义一些前置处理
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                  //后置处理
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
      //在超时时间过了之后依旧没获取到任务
        completedAbruptly = false;
    } finally {
      //线程任务运行异常或者响应了中断抛出的异常
        processWorkerExit(w, completedAbruptly);
    }
}

//从队列中获取任务,该方法是可能导致阻塞(即队列中木有任务了)
  private Runnable getTask() {
    boolean timedOut = false; 

    for (;;) {
      int c = ctl.get();
      int rs = runStateOf(c);
      //线程池已经停止了,该任务线程将会被中断
      if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
        decrementWorkerCount();
        return null;
      }

      int wc = workerCountOf(c);

      // 超过最大线程数,或者当前线程数比核心线程数多且设置了超时回收,或者允许核心线程在在超时后回收
      boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

      if ((wc > maximumPoolSize || (timed && timedOut))
          && (wc > 1 || workQueue.isEmpty())) {
        if (compareAndDecrementWorkerCount(c))
          return null;
        continue;
      }

      try {
        //阻塞获取队列中任务
        Runnable r = timed ?
          workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
        if (r != null)
          return r;
        //超时依旧没获取到
        timedOut = true;
      } catch (InterruptedException retry) {
        timedOut = false;
      }
    }
  }


//没有任务或者任务异常了,需要退出工作线程
  private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //表示因为任务执行异常才退出的
    if (completedAbruptly) 
      decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      completedTaskCount += w.completedTasks;
      workers.remove(w);
    } finally {
      mainLock.unlock();
    }
		//如果是因为线程池停止导致的工作线程的退出则尝试执行中止操作,如果不是则跳过
    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
      if (!completedAbruptly) {
        //如果是因为设置了允许核心线程超时则保留最多保留一个线程
        //其他情况的超时保留核心线程
        int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
        if (min == 0 && ! workQueue.isEmpty())
          min = 1;
        if (workerCountOf(c) >= min)
          //相当于整个Thread.run()方法执行完毕,系统会自动退出该线程
          return; 
      }
      //表示是因为任务执行异常导致的退出,重新添加个空的任务
      addWorker(null, false);
    }
  }


```

```java
//submit提交任务

  //Runable没有返回值,返回的Future对象的get()一直为null,该方法的作用是为了提交任务后能知道任务的运行状态
//底层也是将Runnable包装成Callable对象进行操作
  public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
  }
//确实有返回值存在
  public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    //将任务包装成FutureTask对象,该对象实现了Runnable接口,新添加的任务默认state=NEW状态
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
  }

//运行任务
  public void run() {
    //新增任务,或者能将当前线程与任务绑定的任务才能执行
    if (state != NEW || !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread())) {
      return;
    }
    try {
      Callable<V> c = callable;
      //只有新增的任务才会执行
      if (c != null && state == NEW) {
        V result;
        boolean ran;
        try {
          result = c.call();
          ran = true;
        } catch (Throwable ex) {
          result = null;
          ran = false;
          //运行异常 NEW -> COMPLETING -> EXCEPTIONAL
          setException(ex);
        }
        if (ran)
          //运行成功 NEW -> COMPLETING -> NORMAL
          set(result);
      }
    } finally {
      runner = null;
      int s = state;
      if (s >= INTERRUPTING)
        handlePossibleCancellationInterrupt(s);
    }
  }
		//设置返回值
    protected void set(V v) {
      //先将任务改为完成中,再设置执行结果的返回值,再设置为已完成
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
          //通知所有等待的该任务结果的线程
            finishCompletion();
        }
    }

//通知等待的线程(如果有的话)
  private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
      if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
        for (;;) {
          //递归唤醒所有等待的线程 WaitNode是一个单项链表
          Thread t = q.thread;
          if (t != null) {
            q.thread = null;
            LockSupport.unpark(t);
          }
          WaitNode next = q.next;
          if (next == null)
            break;
          q.next = null;
          q = next;
        }
        break;
      }
    }
    //空方法
    done();
    callable = null;    
  }

// 获取执行的结果	
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
      //表示可能未开始执行或在执行中
        if (s <= COMPLETING)
          //阻塞等待
            s = awaitDone(false, 0L);
      //对结果状态进行处理
        return report(s);
    }

//带有超时时间的获取结果
  public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
      throw new NullPointerException();
    int s = state;
    if (s <= COMPLETING &&
        //如果在指定的超时时间过后,任务依旧没有执行完成则抛异常
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
      throw new TimeoutException();
    return report(s);
  }



  private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
      //当前线程中断则移除等待的节点
      if (Thread.interrupted()) {
        removeWaiter(q);
        throw new InterruptedException();
      }

      int s = state;
      //任务已经执行完了,可以直接获取结果
      if (s > COMPLETING) {
        if (q != null)
          q.thread = null;
        return s;
      } else if (s == COMPLETING) {
        //已经在处理中了 马上会处理完,只放弃cpu执行权
        Thread.yield();
      } else if (q == null) {
        q = new WaitNode();
      } else if (!queued)
        //将q作为waiters节点,原来的waiters节点作为q的next节点
        queued = UNSAFE.compareAndSwapObject(this, waitersOffset, q.next = waiters, q);
    	}else if (timed) {//表示超时等待
        nanos = deadline - System.nanoTime();
        if (nanos <= 0L) {
          //等待时间已过,移除之前的等待的节点
          removeWaiter(q);
          return state;
        }
        //阻塞nanos纳秒
        LockSupport.parkNanos(this, nanos);
      } else {
        //在没有设置超时等待的线程,经过几个循环都会进入该方法进行阻塞
        LockSupport.park(this);
      }     
    }
  }

//移除节点
    private void removeWaiter(WaitNode node) {
        if (node != null) {
          //当前节点线程清空,同时将该节点在整个waiters中去除
            node.thread = null;
            retry:
            for (;;) {
              //从首节点开始递归调用
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                  
                    if (q.thread != null) {
                      pred = q;
                    }else if (pred != null) {
                      //因为我们前面设置node.thread=null,该节点可能就是我们要移除的节点
                      //将q节点从链表中去除(thread=null了)
                        pred.next = s;
                      //此处为双检机制,在并发情况下,可能pred.thread被其他线程已经设置为null因此需要重新遍历
                        if (pred.thread == null) {
                           continue retry;
                        } 
                      //q.thread=null && pred=null 表示q节点是头节点
                    }else if (!UNSAFE.compareAndSwapObject(this, waitersOffset, q, s)) {
                      //此处continue retry相当于整个双循环从头再来
                       continue retry;
                    }
                }
                break;
            }
        }
    }


//取消可能已经在执行的任务
    public boolean cancel(boolean mayInterruptIfRunning) {
      //新增任务才能取消
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    
            if (mayInterruptIfRunning) {
                try {
                  //按理说,新增的任务应该没有绑定线程, 个人感觉 此时判断是在并发情况下,run()会给(不是新增的 && 没有绑定线程的任务)绑定线程
                    Thread t = runner;
                    if (t != null)
                      //由线程池响应中断
                        t.interrupt();
                } finally {
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
          //唤醒可能等待该结果的线程
            finishCompletion();
        }
        return true;
    }

```

