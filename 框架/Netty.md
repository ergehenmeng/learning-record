> 发送消息的两种方式

* 直接写入通道, 消息从`ChannelPipeline`的尾节点开始
* 写入`ChannelHandlerContext`对象, 从`ChannelPipeline`的下个handler开始

> `ChannelHandler` 常用方法

* `handlerAdded`  handler添加到实际上下文中的准备事件
* `handlerRemoved` 将handler从实际上下文删除时的事件
* `exceptionCaught` 处理

> `ChannelInboundHandler` 常用方法

* `channelRegistered`  `ChannelHandlerContext`的Channel被注册到`EventLoop`时;
* `channelUnregistered` `ChannelHandlerContext`的Channel从`EventLoop`中注销时;
* `channelActive`  `ChannelHandlerContext`的Channel已激活;
* `channelInactive` `ChannelHanderContxt`的Channel结束生命周期;
* `channelReadComplete`  消息读取完成后执行;
* `userEventTriggered` 一个用户事件被处罚;
* 

