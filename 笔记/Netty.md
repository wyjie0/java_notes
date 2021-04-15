## JDK原生NIO程序的问题

1. NIO类库和和API繁杂，使用麻烦（比如对BUFFER的读取很麻烦）。
2. JDK NIO的bug，epoll bug会导致selector空轮询

## Netty的特点

1. 使用方便，对NIO进行封装
2. 高性能、吞吐量高
3. 解决了NIO的bug

## Netty的工作流程

- Server端工作原理

    ![image-20200730152904174](F:\java书和视频\笔记\images\面经\26.png)

Server端启动时绑定某个本地端口，将自己的NioServerSocketChannel注册到某个boss NioEventLoop的Selector上

server端包含一个boss NioEventLoopGroup和1个worker NioEventLoopGroup

NioEventLoopGroup相当于一个事件循环组，这个组里包含多个事件循环NioEventLoop，每个NioEventLoop包含一个Selector和一个事件循环线程

每个boos NioEventLoop循环执行的任务包含3步：

1. 轮询accept事件
2. 处理io任务，即accept事件，与client连接，生成一个SocketChannel，并封装称为NioSocketChannel，然后将NioSocketChannel注册到某个worker NioEventLoop的Selector上
3. 处理任务队列中的任务，包括用户调用eventloop.execute或schedule执行的非io任务。

每个worker NioEventLoop循环执行的任务包含3步：

1. 轮询read、write事件
2. 处理io任务，即read、write事件，在NioSocketChannel可读、可写事件发生时进行处理
3. 处理任务队列的任务

- Client端工作原理

    ![image-20200730153703588](F:\java书和视频\笔记\images\面经\27.png)

client端启动时connect到server，建立NioSocketChannel，并注册到某个NioEventLoop的Selector上

client端只包含1个NioEventLoopGroup，每个NioEventLoop循环执行的任务包含3步:

1. 轮询connect、read、write事件
2. 处理io任务，在NioSocketChannel连接建立、可读、可写事件发生时进行处理
3. 处理非io任务

## Netty如何解决TCP粘包/拆包问题

1. 消息定长发送：`FixedLengthFrameDecoder`类
2. 包尾增加特殊字符分割：
    1. 行分隔符：`LineBasedFrameDecoder`
    2. 自定义分隔符类：`DelimiterBasedFrameDecoder`

## Netty如何解决空轮询

- 空轮询

    NIO使用Selector来进行IO多路复用，Selector会轮询每个Channel，如果Channel中没有事件它就会阻塞。但是，Selector可能会在没有事件发生的时候返回一个空的事件组，就造成了空轮询。

- Netty解决

    netty通过线程不断循环检测select是否返回0，如果是就认为是一个空轮询，然后对其计数，如果空轮询次数达到一定的阈值就会重建Selector

    因为重新构造了selector，需要重新注册channnel到其上，并注册感兴趣事件，重新注册的过程中有机会检测channel的可用性