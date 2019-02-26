####TCP粘包拆包
* **LineBasedFrameDecoder** 依次读取ByteBuf中的可读字节,判断是否有"\n"或者"\r\n",如果有,就以此位置为结束位置,从可读索引到结束位置组成一行,同时支持最大读取长度,如果读取最大行数后仍没有读取到换行符,则抛异常,忽略掉之前读取的数据
* **StringDecoder** 将接收到的对象转换为字符串,然后继续调用后面的Handler,上述两个组合就是按行切换的文本解码器
* **DelimiteBasedFrameDecoder** 以分隔符作为码流结束标示符的解码器
* **FixedLengthFrameDecoder** 定长解码器
####ByteBuffer 当前指针的位置position,缓冲池大小limit=capacity
* flip() 将总容量设置为当前位置(limit=position),将当前位置设置position=0,capacity不变
####ByteBuf 
* readerIndex与writerIndex开始都为0,随着写数据writerIndex会增加,读数据readerIndex会增加,但不会超过writerIndex,在读取之后,0~readerIndex的就会被视为discard的,调用discardReaderBytes方法会释放这部分空间,他的作用类似于ByteBuffer.compact方法.readerIndex与writerIndex之间的数据是可读取的 等价于ByteBuffer position与limit之间的数据,writerIndex与capacity之间的空间是可写的,等价于ByteBuffer limit与capacity之间的可用空间
* **discardReadBytes** 释放已经读取的空间(频繁操作会消耗性能)
* **read**操作只是指针位置的变换,并没有对原来数据造成影响;
* **clear** 不会清空缓冲区本身,而是readerIndex与writerIndex都归零;
* **markReaderIndex** 将当前的readerIndex备份到markedReaderIndex中;
* **resetReaderIndex** 将当前的readerIndex设置为markedReaderIndex;
* **markWriterIndex** 将当前的writerIndex备份到markedWriterIndex;
* **resetWriterIndex** 将当前的writerIndex设置为markedWriterIndex;
* **indexOf(int from,int to,byte value)** 在索引from到索引to中查找首次出现value的位置,如果没找到返回-1
* **bytesBefore(byte value)** 在readerIndex与writerIndex中查找首次出现value的位置,没找到返回-1;不改变readerIndex与writerIndex
* **bytesBefore(int length,byte value)** 在readerIndex与readerIndex+length期限查找首次出现value的位置,没找到返回-1;
* **byteBefore(int index,int length,byte value)** 在index与index+length查找首次出现value的位置,没找到返回-1;
* **forEachByte(ByteProcessor processor)** 遍历查找满足processor条件的byte的位置,没有找到返回-1;
* **forEachByteDesc(ByteProcessor processor)** 与上方法区别时 从readerIndex-1开始迭代到readerIndex
* ByteProcessor 定义了一些默认的查找方法:
  * FIND_NUL :第一个为空的下标
  * FIND_NON_NUL() :第一个不为空的下标
  * FIND_CR:\r查找:第一个\r的下标
  * FIND_LF:\n
  * FIND_CRLF:\r\n
  * FIND_NON_LINEAR_WHITESPACE : ' '或者'\t'
* **duplicate** :返回ByteBuf的复制对象,与原对象共享缓存区,但维护自己独立的读写索引,修改复制后的ByteBuf,原ByteBuf也会改变
* **copy**:完全复制,与原来的对象无关
* **slice**:返回当前ByteBuf的子缓存区,数据内容为原ByteBuf的起始位置readerIndex到writerIndex之间的数据,与原ByteBuf共享内容,但读写索引独立维护,原ByteBuf不会改变readerIndex与writerIndex
* **slice(int index,int length)** 返回ByteBuf的子缓存区 index到index+length之间的数据,读写索引独立维护
* **nioBuffer** 将当前ByteBuf缓存区转换为ByteBuffer,两者共享缓存区内容索引,对ByteBuffer的读写操作不会影响ByteBuf的读写索引,返回的ByteBuffer无法感知原ByteBuf的动态扩展操作
* **nioBuffer(int index,int length)**;将ByteBuf中index到index+length的缓冲区转换为ByteBuffer,两者共享缓冲区内容,其他与上面的方法一样
####ByteBuf API
![](https://i.imgur.com/mYeukxY.png)
![](https://i.imgur.com/euqAe5C.png)
readBytes(ByteBuf buf,int dstIndex,int length):disIndex表示写入到目标的开始位置,length是写入的长度,注意写入的长度如果大于buf之前的writerIndex,虽然会被写入进去,但是writerIndex没有改变,因此通过readableBytes计算可读长度不准确
![](https://i.imgur.com/aUnVGqD.png)
![](https://i.imgur.com/T6uyh3U.png)
![](https://i.imgur.com/FsvK8Lx.png)
![](https://i.imgur.com/kSePcxr.png)
![](https://i.imgur.com/D8wLzdP.png)
* **InBound事件** 通常由I/O触发(被动接受)如:TCP链路建立事件,链路关闭事件,读事件,异常通知事件等
* **OutBound事件** 通常由用户主动发起的网络I/O事件,如连接事件,绑定操作,消息发送等事件