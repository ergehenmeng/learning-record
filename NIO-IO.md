#### NIO核心组成部分

* `Channel`
* `Buffer`
* `Selector`

#### Channel

> 类似于流,又与流有些不同

* 即可用从通道中读取数据,又可以写入数据到通道中,但流的读写通常是单向的
* 通道可以异步进行读写
* 通道的数据总是先读取到一个`Buffer` 或者写入到一个`Buffer`
  * 从通道读取数据到缓存区
  * 从缓存区写入数据到通道

> Channel实现类

* `FileChannel` 从文件读写数据
* `DatagramChannel` 通过UDP读写网格数据
* `SocketChannel` 通过TCP读写网络中的数据
* `ServerSocketChannel` 可以监听TCP链接,对每个新进来的链接创建`SocketChannel`

#### Buffer

> 缓冲区就是一块可以写入数据,可以从中读取数据的内存区域,这块内存被包装成`Buffer`对象并提供api方便访问该内存

* 读写数据一般遵循以下步骤
  * 写入数据到Buffer
  * 调用`flip()`方法 (将Buffer从写模式切换到读模式)
  * 从`Buffer`中读取数据
  * 调用`clear()`方法或`compact()`方法

* 属性或方法说明
  * `capacity` 读写模式含义是一样的 表示内存块的大小
  * `position` 写模式下,`position`代表当前的位置 初始化=0,写入数据后向前移动,最大`position=capacity-1`
  * `position`读模式下,表示从`position`位置开始读取数据,读取后向前移动到下一个可读的位置
  * `limit` 写模式下,`limit`表示你最多能写多少数据到Buffer中,`limit=capacity`
  * `limit`读模式,表示最多能读取多少数据
  * `rewind()`将`position`设置为0,表示可以重新读取Buffer中的所有数据,`limit`保持不变
  * `clear()` 会将`position=0` `limit=capacity`,换句话说Buffer被清空了,Buffer中的数据并未清除
  * `compact()`会将所有未读的数据拷贝到Buffer的起始位置,将`position`设置到最后一个未读元素后面 `limit=capactiy`
  * `mark()`标记一个特定`position`
  * `reset()`恢复之前标记的`position`值

#### 文件描述符(File descriptor)

> 是一个索引值,指向内核为每一个进程所维护的该进程打开文件的记录表

#### 阻塞IO

> 当用户进程调用`recvfrom`之后,内核开始IO的第一个阶段:准备数据(可能数据一开始还没准备好),这个过程需要等待,也就是说数据被拷贝到操作系统内核的缓存区中是需要过程的,而用户进程这边,整个进程会被阻塞,当内核等待数据就绪后,他会将数据从内核空间拷贝到用户空间,然后内核返回结果,用户进程解除阻塞状态,重新运行.即:IO执行的两个阶段都被block了

#### 非阻塞IO

> 用户进程发起read操作后,如果内核中的数据还没准备好,他不会block用户进程,而是会返回error,从用户角度来看,他发起read操作后并不需要等待直接能获取一个结果.用户进程判断结果是error时,就知道数据还没准备好,他可以再次发送read操作.一旦内核中的数据准备好了,并且又再次收到用户进程的system call,那么他马上就会将数据拷贝到用户内核中,然后返回 即:用户进程需要不断的主动询问内核数据好了没有

#### IO多路复用

> 当用户进程调用select,整个进程就会block,而内核会监视所有selector负责的socket,任意一个socket中的数据准备好了,select就会返回,这个时候用户进程再次调用read操作,将数据从内核拷贝到用户内存,select的优势在于他可以同时处理多个connection,从用户进程角度来看,同样一直被block的,只不过是被select函数锁block,而不是被socket IO给block

#### 异步IO

> 用户进程发起read操作后,立即就可以做其他的事了,而另一方面,从内核角度来说,他收到一个asynchronous read之后,首先他会立即返回,不会对用户进程产生任何block,然后内核会等待数据准备完成,然后将数据拷贝到用户内存中,当这一切准备就绪后,内核会给用户进程发送一个signal,告诉他read操作已完成

