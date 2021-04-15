Java NIO主要由下面三个核心部分组成：

* Channel
* Buffer
* Selector

## 1、Channel和Buffer

### 1.1 Channel

基本上，所有的IO操作在NIO中都从一个Channel开始。Channel有点像流，数据可以从Channel流向Buffer，也可从Buffer流向Channel

![image-20200325090554777](F:\Java书和视频\笔记\images\NIO\NIO1.png)

Channel和流的区别在于：

1. Channel既可以向Buffer写数据，Buffer也可以向Channel写数据，但是流的数据流向通常是单向的
2. Channel可以**异步**地读写
3. **通道中的数据总是先读到一个Buffer，或者总是要从一个Buffer写入**

主要的Channel实现有：

* `FileChannel`：从文件中读写数据
* `DatagramChannel`：通过UDP从网络中读写数据
* `SocketChannel`：通过TCP读写网络中的数据
* `ServerSocketChannel`：监听新进来的TCP连接，像Web服务器那样，对每一个新进来的连接都会创建一个SocketChannel

这些具体实现包括UDP和TCP网络IO，以及文件IO

FileChannel的使用示例：

```java
public static void main(String[] args) throws IOException {
    String url = ChannelTest.class.getClassLoader().getResource("test").getFile();
    RandomAccessFile file = new RandomAccessFile(url, "rw");
    //通过RandomAccessFile的实例来获取fileChannel
    FileChannel fileChannel = file.getChannel();

    ByteBuffer buffer = ByteBuffer.allocate(48);
    //先把数据读取到Buffer中
    int bytesRead = fileChannel.read(buffer);
    while (bytesRead != -1) {
        System.out.println("Read: " + bytesRead);
        buffer.flip();//从写模式切换到读模式
        while (buffer.hasRemaining()) {
            System.out.print((char) buffer.get());
        }
        buffer.clear();
        bytesRead = fileChannel.read(buffer);
    }

    file.close();
}
```

### 1.2 Buffer

#### 1.2.1 Buffer的基本用法

使用Buffer读写数据一般遵循以下4个步骤：

1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用clear()方法或者Compact()方法

当向Buffer写入数据时，Buffer会记录下下了多少数据。一旦要读取数据，需要通过flip()方法将Buffer从写模式切换到读模式。

一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。clear()方法会清空整个缓冲区，compact()方法只会清除已经读过的数据。任何未读的数据都被移到**缓冲区的起始处**，新写入的数据将放到缓冲区未读取数据的后面

#### 1.2.2 Buffer的capacity，position和limit

缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存，这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存

Buffer有三个属性：

1. capacity
2. position
3. limit

图示如下：

![image-20200325095233353](F:\Java书和视频\笔记\images\NIO\NIO3.png)

在两种模式下，capacity的值都是相同的，表示Buffer能够容纳数据的量。而在**写模式下**，position表示当前写入数据的位置，初始值为0，最大为capacity-1；limit表示最多能写多少数据。而在**读模式下**，position表示当前读取数据的位置，当将Buffer从写模式切换到读模式，position会被重置为0；而limit则是在写模式下position的位置

#### 1.2.3 Buffer的类型

主要的Buffer实现有：

* `ByteBuffer`
* `CharBuffer`
* `DoubleBuffer`
* `FloatBuffer`
* `IntBuffer`
* `LongBuffer`
* `ShortBuffer`
* `MappedByteBuffer`：用于表示内存映射文件

#### 1.2.4 给Buffer分配空间

每一个Buffer类都有一个静态方法allocate，调用这个方法，传入给定的空间大小即可获得一个Buffer对象

```java
ByteBuffer buffer = ByteBuffer.allocate(48);
```

#### 1.2.5 向Buffer中写数据

写数据到Buffer有两种方式：

* 从Channel写到Buffer
* 通过Buffer的put()方法写到Buffer里

**从Channel写到Buffer**

```java
int bytesRead = fileChannel.read(buffer);
```

**通过put方法写Buffer**

```java
buffer.put(127);
```

put()还有许多重载方法，使用的时候可以了解一下

#### 1.2.6 从Buffer中读取数据

从Buffer中读取数据也有两种方式：

* 从Buffer读取数据到Channel
* 使用get()方法从Buffer中读取数据

**从Buffer中读取数据到Channel**

```java
//从buffer中读取数据的Channel中，也就是buffer写数据到Channel
int bytesWrite = fileChannel.write(buffer);
```

**使用get方法从Buffer中读取数据**

```java
byte aByte = buffer.get();
```

get()还有许多重载方法，使用的时候可以了解一下

#### 1.2.7 rewind方法

`buffer.rewind()`方法将position重置为0，所以可以重读Buffer中的数据，limit保持不变。

#### 1.2.8 clear和compact方法

一旦读完Buffer中的数据，需要让Buffer准备好再次被写入，可以通过clear或compact方法来实现。

如果调用的是clear方法，position将被重置为0，limit被设置成capacity-1。这就表示Buffer被清空了，但是**==其中的数据没有被删除==**，只是这些标记告诉我们可以从哪里开始往Buffer里写数据。如果Buffer中有一些未读的数据，调用clear方法，数据将“被遗忘”。

而compact()方法将position设到最后一个未读元素的正后面，limit属性依然想clear方法一样， 这样就不会覆盖未读的数据

#### 1.2.9 mark和reset方法

通过调用buffer.mark()方法可以标记Buffer中的一个特定position，之后可以通过调用buffer.reset()方法恢复到这个position。

```java
buffer.mark();
//dosomething...(比如：读取数据)
buffer.reset();
```

#### 1.2.10 Buffer之间的比较

当满足下列条件时，表示两个Buffer相等：

1. 有相同的类型（byte，char，int等）
2. Buffer中position到limit之间的元素的个数相等
3. Buffer中position到limit之间的元素相等（读取过的数据不比较）

compareTo方法比较两个Buffer的剩余元素，如果满足下列条件，则认为一个Buffer“小于”另一个Buffer：

1. 第一个不相等的元素小于另一个Buffer中对应的元素（优先级更高）
2. 所有元素都相等，但第一个Buffer比另一个先耗尽（第一个buffer的position大于第二个）



## 2、 Scatter和Gather

Scatter（分散）从Channel中读取是指在读操作时将读取的数据写入多个buffer中。

Gather（聚集）写入Channel是指在写操作时将多个buffer的数据写入同一个Channel

### 2.1 Scattering Reads

![image-20200325111754949](F:\Java书和视频\笔记\images\NIO\NIO4.png)

```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body = ByteBuffer.allocate(1024);
ByteBuffer[] buffers = {header, body};
fileChannel.read(buffers);
```

Buffer首先被插入到数组，然后再将数组作为channel.read()的输入参数。read方法按照buffer在数组中的顺序将从channel中读取的数据写入到buffer，当一个buffer被写满后，channel紧接着向另一个buffer中写

### 2.2 Gathering Writes

![image-20200325112252629](F:\Java书和视频\笔记\images\NIO\NIO5.png)

```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body = ByteBuffer.allocate(1024);
ByteBuffer[] buffers = {header, body};
fileChannel.write(buffers);
```

buffers数组是write()方法的入参，write()方法会按照buffer在数组中的顺序，将数据写入到channel，注意只有position和limit之间的数据才会被写入。因此，如果一个buffer的容量为128byte，但是仅仅包含58byte的数据，那么这58byte的数据将被写入到channel中

## 3、通道之间的数据传输

FileChannel的transferFrom()方法可以将字节数据从给定的**可读取字节通道**（可以是其他通道，比如SocketChannel）传输到调用transferFrom()方法的通道的文件中。

```java
RandomAccessFile from  = new RandomAccessFile("test", "rw");
FileChannel fromChannel = from.getChannel();
RandomAccessFile to = new RandomAccessFile("toTest", "rw");
FileChannel toChannel = to.getChannel();

long position = 0;
long count = fromChannel.size();
toChannel.transferFrom(fromChannel, position, count);
```

方法的参数position表示从position处开始向目标文件写入数据，count表示最多传输的字节数，但是实际传入的数据根据源通道实际的数据个数而定。

transferTo()方法将数据从FileChannel传输到其他的channel中。

## 4、Selector

Selector允许单线程处理多个Channel，并能够知晓Channel是否为诸如读写事件做好准备的组件。如果应用打开了多个连接（通道），但每个连接的流量都很低，则可以选择使用Selector

![image-20200325091036847](F:\Java书和视频\笔记\images\NIO\NIO2.png)

**异步IO中的核心对象就是Selector**。Selector是注册对各种I/O事件感兴趣的地方，而且当哪些事件发生时，就是Selector的对象告诉我们所发生的事件

首先需要创建一个Selector对象

`Selector selector = Selector.open();//创建Selector`

然后将对不同的通道对象调用register方法，以便注册我们对这些对象中发生的I/O事件的兴趣。

为了接收连接，我们需要先创建一个ServerSocketChannel。我们要监听的每一个端口都需要一个ServerSocketChannel。

```java
ServerSocketChannel ssc = ServerSocketChannel.open();
//与Selector一起使用时，Channel必须处于非阻塞模式下，这意味着不能将FileChannel与Selector一起使用，因为FileChannel不能切换到非阻塞模式，而套接字通道可以
ssc.configureBlocking(false);
//将ServerSocketChannel绑定的给定的端口
ServerSocket ss = ssc.socket();
InetSocketAddress address = new InetSocketAddress(8088);
ss.bind(address);
```

下一步是将新打开的ServerSocketChannel注册到Selector上：

`SelectionKey selectionKey = ssc.register(selector, SelectionKey.OP_ACCEPT);`

register方法的第一个参数总是Selector，第二个参数是OP_ACCEPT，这里它指定我们想要监听accept事件，也就是在新的连接建立时所发生的事件。**这是适用于ServerSocketChannel的唯一事件类型**

返回值SelectionKey代表这个通道在此Selector上的这个注册。当某个Selector通知我们某个事件传入时，它是通过提供对应与该事件的SelectionKey来进行的

现在已经注册了我们对一些I/O事件的兴趣，下面进入循环来处理事件

```java
int readyChannels = selector.select();
Set<SelectionKey> selectedKeys = selector.selectedKeys();
Iterator iterator = selectedKeys.iterator();
while (iterator.hasNext()) {
	SelectionKey key = (SelectionKey) iterator.next();
	//处理I/O事件
}
```

首先调用Selector的select()方法，这个方法会阻塞，直到至少有一个已注册的事件发生。当一个或者更多的事件发生时，select()方法将返回所发生的事件的数量。

接下来，我们调用 `Selector` 的 `selectedKeys()` 方法，它返回发生了事件的 `SelectionKey` 对象的一个集合。

我们通过迭代 `SelectionKeys` 并依次处理每个 `SelectionKey` 来处理事件。对于每一个 `SelectionKey`，我们必须确定发生的是什么 I/O 事件，以及这个事件影响哪些 I/O 对象。

现在我们应该对SelectionKey调用readyOps()方法，检查发生了什么类型的事件

```java
if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {
	//接收新的连接
}
```

因为我们知道这个服务器套接字上有一个传入的连接在等待，所以可以安全地接受它，不用担心accept()操作会被阻塞：

```java
//接收新的连接
ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
SocketChannel sc = serverSocketChannel.accept();
```

下一步是将新连接的SocketChannel设置为非阻塞的，而且由于接受这个连接的目的是为了读取来自套接字的数据，所以我们还必须将SocketChannel注册到Seletor上：

```java
sc.configureBlocking(false);
SelectionKey selectionKey = sc.register(selector, SelectionKey.OP_READ);
```

当来自一个套接字的数据到达时，它会触发一个I/O事件，这会导致在主循环中调用Selector.select()方法，这一次SelectionKey被标记为OP_READ事件：

```java
if ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
	//读取数据
	SocketChannel socketChannel = (SocketChannel)key.channel();
	//...
}
```

**完整示例如下**

```java
public class SelectorEcho {

    private int[] ports;
    private ByteBuffer echoBuffer = ByteBuffer.allocate(1024);

    public SelectorEcho(int[] ports) throws IOException {
        this.ports = ports;
        go();
    }

    private void go() throws IOException {
        //创建一个selector
        Selector selector = Selector.open();
        //为每一个要监听的端口创建一个ServerSocketChannel
        for (int i = 0; i < ports.length; i++) {
            ServerSocketChannel ssc = ServerSocketChannel.open();
            ssc.configureBlocking(false);
            ServerSocket socket = ssc.socket();
            InetSocketAddress address = new InetSocketAddress(ports[i]);
            socket.bind(address);

            SelectionKey key = ssc.register(selector, SelectionKey.OP_ACCEPT);

            System.out.println("准备监听端口：" + ports[i]);
        }

        while (true) {
            int readyChannels = selector.select();
            if (readyChannels == 0) continue;
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                if ((key.readyOps() & SelectionKey.OP_ACCEPT)
                    == SelectionKey.OP_ACCEPT) {
                    //接收新的连接
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);

                    //将新的channel注册到selector
                    SelectionKey key1 = sc.register(selector, SelectionKey.OP_READ);
                    keyIterator.remove();

                    System.out.println("收到新的连接：" + sc);
                }
                if ((key.readyOps() & SelectionKey.OP_READ)
                    == SelectionKey.OP_READ) {
                    //读取数据并返回数据
                    SocketChannel sc = (SocketChannel) key.channel();

                    int bytsEchoed = 0;
                    while (true) {
                        echoBuffer.clear();
                        int r = sc.read(echoBuffer);
                        if (r <= 0) break;
                        echoBuffer.flip();
                        sc.write(echoBuffer);//回复同样的数据
                        bytsEchoed += r;
                    }
                    keyIterator.remove();
                }
            }
        }
    }
}

```

**Selector、SelectionKey、ServerSocketChannel和SocketChannel关系图**：

![image-20200330153010962](F:\Java书和视频\笔记\images\NIO\nio9.png)

## 5、SocketChannel

SocketChannel是一个连接到TCP网络套接字的通道，可以通过以下两种方式打开SocketChannel：

* 直接调用SocketChannel的静态方法open来打开，并连接到互联网上的某台服务器
* 当一个新连接到达ServerSocketChannel时，会创建一个SocketChannel

### 5.1 打开SocketChannel

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("www.baidu.com", 8080));
```

SocketChannel的读取和写入方法和FileChannel一样。

### 5.2 非阻塞模式

可以设置SocketChannel为非阻塞模式，设置过后，就可以在异步模式下调用connect()、read()和write()方法了

## 6、DatagramChannel

DatagramChannel是一个能收发UDP包的通道，因为UDP是无连接的网络协议，所以不能像其他通道那样读取和写入，它发送和接收的是数据包。

### 6.1 打开DatagramChannel

```java
DatagramChannel datagramChannel = DatagramChannel.open();
//在UDP端口9999上接收数据包
datagramChannel.socket().bind(new InetSocketAddress(9999));
```

### 6.2 发送和接收数据

```java
//从datagramChannel接收数据
ByteBuffer byteBuffer = ByteBuffer.allocate(48);
byteBuffer.clear();
SocketAddress socketAddress = datagramChannel.receive(byteBuffer);
//通过datagramChannel发送数据
String data = "this is a test for datagramchannel ..." + System.currentTimeMillis();
ByteBuffer buffer = ByteBuffer.allocate(48);
buffer.clear();
buffer.put(data.getBytes());
buffer.flip();
//将消息发送到com.wyjie服务器的UDP端口80
datagramChannel.send(buffer, new InetSocketAddress("com.wyjie", 80));
```

### 6.3 连接到特定的地址

可以将DatagramChannel连接到网络中的特定地址。但是由于UDP是无连接的，连接到特定地址并不会像TCP通道那样创建一个真正的连接，而是锁住DatagramChannel，让其只能从特定地址收发数据

```java
datagramChannel.connect(new InetSocketAddress("wyjie", 8888));
```

当连接后，也可像传统通道一样使用read和write方法

## 7、 管道

Java NIO管道是2个线程之间的单向数据连接。Pipe有一个Source通道和一个Sink通道，数会被写入Sink通道，从Source通道读取。

![image-20200326092753975](F:\Java书和视频\笔记\images\NIO\NIO6.png)

管道的使用方式如下：

```java
//首先需要打开管道
Pipe pipe = Pipe.open();
//要发送消息必须访问Pipe的Sink通道
Pipe.SinkChannel sink = pipe.sink();
//通过调用sinkChannel的write方法将数据写入到SinkChannel中
String data = "this is a test for sink channel ..." + System.currentTimeMillis();
ByteBuffer byteBuffer = ByteBuffer.allocate(48);
byteBuffer.clear();
byteBuffer.put(data.getBytes());

byteBuffer.flip();
while (byteBuffer.hasRemaining()) {
sink.write(byteBuffer);
}
//从管道中读取数据需要访问source通道
Pipe.SourceChannel source = pipe.source();
ByteBuffer byteBuffer1 = ByteBuffer.allocate(48);
//返回值表示有多少字节被读进了缓冲区
int counts = source.read(byteBuffer1);
```

## 8、Java NIO和IO

### 8.1 Java NIO和IO的主要区别

|   IO   |   NIO    |
| :----: | :------: |
| 面向流 | 面向缓冲 |
| 阻塞IO | 非阻塞IO |
|   无   |  选择器  |

* 面向流与面向缓冲

  Java NIO和IO之间第一个最大的区别，IO是面向流的，NIO是面向缓冲区的。Java IO面向流意味着每次从流中读一个或多个字节，直至读取完所有字节。它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取数据，需要先将它缓存到一个缓冲区。Java NIO的缓冲导向方法略有不同，**数据读取到一个它稍后处理的缓冲区**，需要时可在缓冲区中前后移动。这就增加了处理过程中的灵活性。但是，还需要检查是否该缓冲区中包含所有我们需要处理的数据。而且，需确保当更多的数据读入缓冲区时，不会覆盖缓冲区里尚未处理的数据

  

* 阻塞与非阻塞

  Java IO的各种流操作时阻塞的， 即当一个线程调用read或write方法时，该线程会被阻塞，直到有数据被读取，或者数据完全写入，**在此期间线程不能再做其他任何事情**（调用的本地方法来进行读写）。Java NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么也不会获取，而不是让线程阻塞，直到数据变得可以读取之前，该线程可以继续做其他的事情。非阻塞写也是如此。线程通常将非阻塞IO的空闲时间用于在其他通道上执行IO操作，所以一个单独的线程可以管理多个输入和输出通道。

  ![image-20200326103950559](F:\Java书和视频\笔记\images\NIO\NiO7.png)

  

  ![image-20200326104052264](F:\Java书和视频\笔记\images\NIO\nio8.png)

## 9、==NIO底层原理==

### 1. IO读写的底层流程

用户程序进行IO的读写，基本上会用到系统调用read&write，read把数据从内核缓冲区读取到进程缓冲区，write把数据从进程缓冲区复制到内核缓冲区，它们不等价于数据在内核缓冲区和磁盘之间的交换

![image-20200729210502124](F:\java书和视频\笔记\images\面经\21.png)

典型Java 服务端处理网络请求的典型过程：

1. 客户端请求：Linux通过网卡，读取客户端的请求数据，将数据读取到内核缓冲区
2. 获取请求数据：服务器从内核缓冲区读取数据到Java进程缓冲区
3. 服务端业务处理：Java服务端在自己的用户空间中处理客户端的请求
4. 服务端返回数据：Java服务端将已构建好的响应从用户缓冲区写入到内核缓冲区
5. 发送给客户端：Linux内核通过网络I/O，将内核缓冲区中的数据写入网卡，网卡通过底层的通讯协议，会将数据发送到目标客户端

### 2. 四种主要的IO模型（操作系统IO模型）

1. 阻塞IO（Blocking IO）

    传统的IO模型都是同步阻塞的IO模型

    **阻塞与非阻塞**

    阻塞IO指的是需要内核IO操作彻底完成后，才返回到用户空间，执行用户的操作。阻塞指的是用户空间程序的执行状态，用户空间程序需等到IO操作彻底完成。在Java中，默认创建的socket是阻塞的

    ![image-20200729213004642](F:\java书和视频\笔记\images\面经\22.png)

    当用户进程调用了recvfrom这个系统调用（读取数据）过后，kernel就开始了IO的第一个阶段：准备数据。对于Network io来说，很多时候护具一开始还没有到达（比如，还没有收到一个完整的UDP包），这个时候kernel就要等待足够的数据到达。而在**用户进程这边会一直阻塞，当kernel等到数据准备好后，将数据复制到用户进程缓冲区过后，用户进程才会解除阻塞。**

    - 优点

        程序简单，在阻塞等待数据期间，用户线程挂起。用户线程基本不会占用CPU资源

    - 缺点

        一般情况下会为每个连接配套一条独立的线程，或者说一条线程维护一个连接成功的IO流的读写。在并发量小的情况下，这个没有什么问题。但是，当在高并发的场景下，需要大量的线程来维护大量的网络连接，内存、线程切换开销会非常巨大。因此，基本上，BIO模型在高并发场景下是不可用的。

2. 非阻塞IO（Non-blocking IO）

    ![image-20200729220011511](F:\java书和视频\笔记\images\面经\25.png)

    当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以**再次发送read操作**（需要不断的发起IO系统调用）。**一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回（这个时候用户进程处于阻塞状态）**。所以，用户进程其实是需要不断的主动询问kernel数据好了没有。

    - 优点

        每次发起的 IO 系统调用，在内核的等待数据过程中可以立即返回。用户线程不会阻塞，实时性较好。

    - 缺点

        需要不断的重复发起IO系统调用，这种不断的轮询，将会不断地询问内核，这将占用大量的 CPU 时间，系统资源利用率较低。

3. IO 多路复用

    ![image-20200729213716306](F:\java书和视频\笔记\images\面经\23.png)

    当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程（**这个时候也会阻塞**）。
    这个图和blocking IO的图其实并没有太大的不同，事实上，还更差一些。因为这里需要使用两个system call (select 和 recvfrom)，而blocking IO只调用了一个system call (recvfrom)。但是，用select的优势在于它可以同时处理多个connection。（多说一句。所以，如果处理的连接数不是很高的话，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。）
    在IO multiplexing Model中，实际中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process其实是一直被block的。只不过**process是被select这个函数block，而不是被socket IO给block。**

    **多路复用模型与非阻塞IO是有关系的，对于进程每一个可以查询到的socket，一般都设置为non-blocking模式**

    - 优点

        用select/epoll的优势在于，它可以同时处理成千上万个连接（connection）。与一条线程维护一个连接相比，I/O多路复用技术的最大优势是：系统不必创建线程，也不必维护这些线程，从而大大减小了系统的开销。

    - 缺点

        本质上，select/epoll系统调用，属于同步IO，也是阻塞IO。都需要在读写事件就绪后，自己负责进行读写，也就是说这个读写过程是阻塞的。

4. 异步IO

    ![image-20200729213853751](F:\java书和视频\笔记\images\面经\24.png)

    户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，**当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。**

### 3. 同步IO与异步IO的区别

 **A synchronous I/O operation causes the requesting process to be blocked until that ==I/O operation== completes;
  An asynchronous I/O operation does not cause the requesting process to be blocked;**
两者的区别就在于**synchronous IO做”IO operation”的时候会将process阻塞**。按照这个定义，**之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO**（==这也就是为什么有同步非阻塞IO这个叫法的原因了吧==）。有人可能会说，non-blocking IO并没有被block啊。这里有个非常“狡猾”的地方，定义中所指的”IO operation”是指真实的IO操作，就是例子中的recvfrom这个system call。non-blocking IO在执行recvfrom这个system call的时候，如果kernel的数据没有准备好，这时候不会block进程。但是，当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内，进程是被block的。而asynchronous IO则不一样，当进程发起IO 操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block。

## 10、==直接内存==

### 1. Java传统socket发送数据

传统的Java Socket编程每次发送数据的时候，都会申请一块直接内存（堆外），然后从堆内复制到堆外，最后再调用send方法。**为什么要把数据从堆内复制到堆外呢？堆内的对象地址会随着gc改变，在send的时候会崩**

### 2.DirectBuffer

Java复用堆外内存，将JVM堆内的对象关联到堆外，在回收堆内对象的时候触发一个回收堆外对象的操作。

## 11、零拷贝

### 1. 操作系统零拷贝

- mmap

    经过四次用户态、内核态切换，数据首先从磁盘经过DMA拷贝到kernel，然后从kernel中经过cpu拷贝到socket缓冲，最后从socket缓冲经过DMA拷贝到协议栈

- sendFile

    经过三次用户态、核心态切换，数据首先从磁盘经过DMA拷贝到kernel，然后从kernel直接DMA拷贝到协议栈

### 2. NIO零拷贝

NIO的零拷贝使用`transferTo`方法