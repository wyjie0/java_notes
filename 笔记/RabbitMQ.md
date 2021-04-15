## 一、相关概念

RabbitMQ整体上是一个生产者与消费者模型，主要负责接收、存储和转发消息。模型结构如下图所示：

![image-20200319174859962](F:\Java书和视频\笔记\images\idea\rabbitMQ\mq1.png)

### 1、生产者和消费者

生产者负责创建消息，然后发布到RabbitMQ中。消息一般可以分为两部分：**消息体**和**标签**。消息的标签用来表述这条消息，比如一个交换器的名称和路由键（可以表示消息传输的方向）。**RabbitMQ根据标签把消息发送给感兴趣的消费者。**

消费者连接到RabbitMQ服务器，并订阅到队列上。当消费者消费一条消息时，只是消费消息的消息体，因为在消息路由的过程中，消息的标签会被丢弃，存入队列的消息只有消息体。

**Broker**：消息中间件的服务节点

对于RabbitMQ来说，一个RabbitMQ Broker可以简单地看作一个RabbitMQ服务节点。下图展示了生产者将消息传入RabbitMQ Broker，以及消费者从中消费消息的整体流程：

![image-20200319175455882](F:\Java书和视频\笔记\images\idea\rabbitMQ\mq2.png)

### 2、队列

队列是 RabbitMQ 的内部对象，用于存储消息。

多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。

### 3、交换器、路由键、绑定

交换器（Exchange）是生产者与队列之间的中间交互部分。生产者交消息发送到Exchange，由Exchange将消息路由到一个或者多个队列中。

路由键：生产者将消息发给交换器的时候会指定一个路由键，用来指定这个消息的路由规则，这个路由键需要与交换器类型和绑定键结合在一起才能有作用。

绑定：RabbitMQ通过绑定将交换器和队列关联起来。在绑定的时候一般会指定一个绑定键，这样RabbitMQ就知道如何正确地将消息路由到队列了

![image-20200319180205228](F:\Java书和视频\笔记\images\idea\rabbitMQ\mq3.png)

其中，Producer到Exchange之间有一个路由键，在路由消息的时候，Exchange会检查路由键和哪一个绑定键相等或大致相同，来路由消息。

### 4、交换器类型

常用的交换器类型有4中：fanout、direct、topic、headers

**fanout**

它会把所有发送到该交换器上的消息路由到所有与该交换器绑定的队列中

**direct**

direct类型的交换器会把消息路由到那些路由键和绑定键完全匹配的队列中

![image-20200319180635028](F:\Java书和视频\笔记\images\idea\rabbitMQ\mq4.png)

如图所示，如果设置路由键为warning，则会将消息路由到队列1和队列2，如果是info或debug，则只会路由到队列2

**topic**

与direct的区别在于，这个类型的交换器对路由键和绑定键进行模糊匹配。规则为：

* 路由键为一个“.”分隔的字符串（被“.”分隔的每一段独立的字符串称为一个单词），如：“com.rabbitmq.client”
* 绑定键与路由键的命名方式相同
* 绑定键中用“*”来匹配一个单词，“#”用于匹配多个单词

![image-20200319181702451](F:\Java书和视频\笔记\images\idea\rabbitMQ\mq5.png)

**headers**

headers类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据消息中的内容中的headers属性来进行匹配。

## 二、客户端开发

### 1、连接RabbitMQ

下面的代码用于在给定的参数下连接RabbitMQ：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(USERNAME);
factory.setPassword(PASSWORD);
//有RabbitMQ创建的虚拟消息服务器，如果没有可不设置
factory.setVirtualHost(virtualHost);
factory.setHost(IP ADDRESS);
factory.setPort(PORT);
Connection conn = factory.newConnection();
```

也可以选择使用URI的方式来实现：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri( "amqp:lluserName:password@ipAddress:portNumber/virtualHost");
Connection conn = factory.newConnection();
```

Connection 接口被用来创建一个Channel:

`Channel channel = conn.createChannel()`

*Connection可以用来创建多个Channel实例，但是Channel实例不能在线程间共享，应用程序应该为每一个线程开辟一个Channel。某些情况下，Channel的操作可以并发执行，但是在其他情况下会导致在网络上出现错误的通信帧交错，同时也会影响发送方确认机制的运行，所以**多线程间共享Channel实例是非线程安全的***

### 2、使用交换器和队列

在使用交换器和队列之前应该声明它们：

```java
channel.exchangeDeclare(exchangeName, "direct", true) ;
String queueName = channel. queueDeclare().getQueue( );
channel.queueBind (queueName, exchangeName, routingKey);
```

#### 2.1 exchangeDeclare方法详解

exchangeDeclare有多个重载方法，这些重载方法由下面这个方法中缺省的某些参数构成的：

```java
DeclareOk exchangeDeclare(String exchange, String type, 		boolean durable ,boolean autoDelete, 
        boolean internal, Map<String, Object> arguments) throws IOException;
//DeclareOk用来标识成功声明了一个交换器
```

* exchange：交换器的名称
* type：交换器的类型
* durable：设置是否持久化。持久化可以将交换器存盘，在服务器重启的时候不会丢失相关信息
* autoDelete：设置是否自动删除。自动删除的前提是至少有一个队列或者交换器与这个交换器绑定，之后所有与这个交换器绑定的队列或者交换器都与此解绑，则这个交换器自动删除。
* internal：设置是否是内置的。如果设置为true，则表示是内置的交换器，客户端程序无法直接发送消息到这个交换器，只能通过交换器路由到交换器的方式。
* arguments：其他一些结构化参数，比如alternate-exchange

删除交换器的方法如下：

```java
Exchange.DeleteOk exchangeDelete(String exchange) throws IOException ;

Exchange.DeleteOk exchangeDelete(String exchange, boo1ean ifUnused) throws IOException;
```

其中exchange表示交换器的名称，而ifUnused用来设置是否在交换器没有被使用的情况下删除。如果设置为true，则只有在交换器没有被使用的情况下才能被删除。如果设置为false，则无论如何这个交换器都要被删除。

#### 2.2 queueDeclare方法详解

queueDeclare的重载方法如下：

```java
Queue.DeclareOk queueDec1are() throws IOException;

Queue.DeclareOk queueDeclare (String queue , boolean durable , boolean exclusive, boolean autoDelete, Map<Str ing, Object> arguments) throws IOException;
```

无参方法默认创建一个由RabbitMQ命名的、非持久化的、排他的、自动删除的队列。

参数说明：

* queue：队列的名称
* durable：是否持久化。如果设置为true，则将队列持久化到磁盘，在服务器重启的时候可以保证不丢失信息
* exclusive：设置是否排他。如果一个队列被声明为排他队列，该队列仅**对首次声明它的连接可见**，并在连接断开时自动删除。排他队列是基于连接可见的，同一个连接的不同信道是可以同时访问同一连接的排他队列；“首次”是指如果一个连接声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同；即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除，这种队列适用于一个客户端同时发送和读取消息的应用场景。
* autoDelete: 设置是否自动删除。为true 则设置队列为自动删除。自动删除的前提是：至少有一个消费者连接到这个队列，之后所有与这个队列连接的消费者都断开时，才会自动删除。不能把这个参数错误地理解为: "当连接到此队列的所有客户端断开时，这个队列自动删除"，因为生产者客户端创建这个队列，或者没有消费者客户端与这个队列连接时，都不会自动删除这个队列。、

*生产者和消费者都能够使用queueDeclare来声明一个队列，但是如果消费者在同一个信道上订阅了一个队列，就无法再声明队列了。必须先取消订阅，然后将信道置位“传输”模式，之后才能声明队列*

删除队列：

`Queue.De1eteOk queueDe1ete(String queue) throws IOExcept ion;`
		`Queue.De1eteOk queueDe1ete(String queue, boo1ean ifUnused, boolean ifEmpty) throws IOException;`

清空队列：

`Queue.PurgeOk queuePurge(String queue ) throws IOException;`

#### 2.3 queueBind方法

`Queue.BindOk queueBind(String queue, String exchange, String routingKey)`

`Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String, Object> arguments)`

* rountingKey：用来绑定队列和交换器的路由键

### 3、发送消息

发送消息可以使用Channel类的basicPublish方法，其重载方法如下：

`void basicPub1ish (String exchange , String routingKey, BasicProperties props , byte[] body) throws IOException;`

`void basicPub1ish (String exchange , String routingKey, boolean mandatory, BasicProperties props , byte[] body) throws IOException;`

`void basicPub1ish (String exchange , String routingKey, boolean mandatory, boolean immediate, BasicProperties props , byte[] body) throws IOException;`

* exchange：交换器的名称，指明消息要发送到哪个交换器中，如果设置为空字符串，则消息会被发送到RabbitMQ默认的交换器中

* routingKey：路由键，交换器根据路由键将消息存储到相应队列

* props：消息的基本属性集，可以使用属性构建器来构建，

  ```java
  new AMQP.BasicProperties.Builder().build();
  ```

  其包括的属性有：

  ```xml
  contentType、contentEncoding、
  headers(Map<String,Object>)、deliveryMode、priority、correlationld、replyTo、expiration、messageld、timestamp、type、userld、appld、clusterId。
  ```
  
* mandatory：若为true，表示如果消息经过交换器但是没有路由成功，那么消息会返还给生产者。若为false，则消息直接丢弃

  生产者可以调用channel.addReturnListener来添加ReturnListener监听器实现接收交换器返还的消息

  ```java
  channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, true, MessageProperties.PERSISTENT_TEXT_PLAIN,
  message.getBytes());
  channel.addReturnListener(new ReturnListener() {
  @Override
  public void handleReturn(int i, String s, String s1, String s2, AMQP.BasicProperties basicProperties, byte[] bytes) throws IOException {
  String msg = new String(bytes);
  System.out.println("Basic.Return返回的结果是：" + msg);
  }
  });
  ```

* immediate：若为true，表示如果交换器在将消息路由到队列时发现队列上没有任何消费者，那么这条消息将不会存入队列中。当与路由键匹配的所有队列都没有消费者时，该消息会通过Basic.Return返回至生产者（**RabbitMQ 3.0开始不支持这个参数了**）

* 

### 4、消费消息

RabbitMQ的消费模式分为两种：推模式（Push）和拉模式（Pull）。推模式采用Basic.Consume进行消费，而拉模式调用Basic.Get进行消费

#### 4.1 推模式

推模式的消费代码一般如下：

```java
//设置客户端最多未被ack的消息的个数
channel.basicQos(64);
Consumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        System.out.println("recv message " + new String(body));
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        channel.basicAck(envelope.getDeliveryTag(), false);
    }
};
channel.basicConsume(QUEUE_NAME, consumer);
```

#### 4.2 拉模式

通过channel.basicGet方法可以单条地获取消息，其返回值是GetResponse。

`GetResponse basicGet(String queue , boolean autoAck) throws IOException;`

## 三、RabbitMQ实现延时队列功能

### 1、过期时间

过期时间分为队列的过期时间和消息的过期时间。

**消息的过期时间**

1. 可以在声明队列的时候设置消息的过期时间，只需在参数中提供`x-message-ttl`属性即可。如果不设置过期时间，则此消息不会过期；如果设置为0，则表示除非此时可以直接将消息传递到消费者，否则消息就会立刻丢弃。**这是从队列的角度设置队列中每个消息的过期时间**

2. 也可以给每个消息设置一个过期时间，需要在调用basicPublish方法的时候通过properties的方式来设定。

   第一种设置队列TTL属性的方法，一旦消息过期，就会从队列中马上抹去，而在第二个方法中，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的。

### 2、死信队列

当消息在一个队列中变为死信之后，它能被重新分配到一个死信交换器（DLX）中，这个交换器中，绑定这个交换器的队列就叫做死信队列。

消息变为死信的情况：

* 消息被拒绝，并且设置requeue为true；
* 消息过期
* 队列达到最大长度

### 3、延迟队列

延迟队列是指当消息被发送之后，并不马上提供给消费者，而是等待一段时间后消费者才能对这个消息进行消费。RabbitMQ不提供延迟队列的功能，但是可以组合过期时间和死信队列模拟出延迟队列的功能

消费者订阅死信队列，生产者将消息发布到普通队列并设定一定的过期时间，那么在等到消息过期过后，才会被传入死信队列， 这是消费者才能消费消息。这就模拟出了延时队列的功能

### 4、RPC实现

RPC是一种通过网络来请求远端服务器提供服务，而不用了解底层网络的细节。RPC的主要功用是：简化构建分布式计算，在提供强大的远程调用能力时不损失本地调用的语义简洁性。

在RabbitMQ中实现RPC很简单，客户端发送请求消息的时候，只需指定一个回调队列就行。

```
String callBackQueue = Channel.queueDeclare().getQueue();
BasicProperties props = new 
BasicProperties.Builder().replyTo(callBackQueue).build();
channel.basicPublish("","rpc_queue",props,msg.getBytes());
```

其中，replyTo就是用来指定一个回调队列的，可以为每个客户端指定一个单一的回调队列。还应该为每一个请求设置一个唯一的correlationId。之后在回调队列接收到回复的消息时，可以根据这个属性匹配到相应的请求。

![image-20200321153416366](F:\Java书和视频\笔记\images\idea\rabbitMQ\mq6.png)

RPC的处理过程如下：

1. 当客户端启动时， 创建一个匿名的回调队列
2. 客户端为RPC请求设置2个属性，replyTo用来告知RPC服务端回复请求时的目的队列，即回调队列；correlationId用来标记一个请求
3. 请求被发送到rpc_queue队列中
4. RPC服务端监听rpc_queue队列中的请求，当请求到来时，服务端会处理并且把带有结果的消息发送个客户端。接收到的队列就是replyTo设定的回调队列。
5. 客户端监听回调队列，当有消息时，检查correlationId属性，如果请求匹配，那就是结果

## 为什么需要消息队列？消息队列的好处

当系统中出现“生产“和“消费“的速度或稳定性等因素不一致的时候，就需要消息队列，作为抽象层，弥合双方的差异。“ **消息** ”是在两台计算机间传送的数据单位。消息可以非常简单，例如只包含文本字符串；也可以更复杂，可能包含嵌入对象。消息被发送到队列中，“ **消息队列** ”是在消息的传输过程中保存消息的**容器** 。

举几个例子

1）业务系统触发短信发送申请，但短信发送模块速度跟不上，需要将来不及处理的消息暂存一下，缓冲压力。就可以把短信发送申请丢到消息队列，直接返回用户成功，短信发送模块再可以慢慢去消息队列中取消息进行处理。

2）调远程系统下订单成本较高，且因为网络等因素，不稳定，攒一批一起发送。

3）任务处理类的系统，先把用户发起的任务请求接收过来存到消息队列中，然后后端开启多个应用程序从队列中取任务进行处理。



**消息队列的好处**

3.1、提高系统响应速度

使用了消息队列，生产者一方，把消息往队列里一扔，就可以立马返回，响应用户了。无需等待处理结果。

处理结果可以让用户稍后自己来取，如医院取化验单。也可以让生产者订阅（如：留下手机号码或让生产者实现listener接口、加入监听队列），有结果了通知。或者约定将结果放在某处，无需通知。

3.2、提高系统稳定性

考虑电商系统下订单，发送数据给生产系统的情况。电商系统和生产系统之间的网络有可能掉线，生产系统可能会因维护等原因暂停服务。如果不使用消息队列，电商系统数据发布出去，顾客无法下单，影响业务开展。两个系统间不应该如此紧密耦合。应该通过消息队列解耦。同时让系统更健壮、稳定。

**异步化**、**解耦**、**消除峰值**

## 四、Spring整合RabbitMQ

[Spring AMQP官方文档](https://docs.spring.io/spring-amqp/docs/2.2.5.RELEASE/reference/html/#_introduction)

## 五、SpringBoot整合RabbitMQ

### 1、自动配置

1. RabbitAutoConfiguration自动配置类
2. 自动配置了连接工厂ConnectionFactory
3. RabbitProperties 封装了RabbitMQ的配置，只需在`application.properties`文件中加上相应的配置即可
4. RabbitTemplate：给RabbitMQ发送和接收消息
5. AmqpAdmin：RabbitMQ系统管理功能组件

```java
public void contextLoads() {
    //声明一个交换器，可以直接new，也可以使用ExchangeBuilder来构建
    admin.declareExchange(new DirectExchange("exchange.direct"));
    //声明队列，也可以值解new Queue()
    admin.declareQueue(QueueBuilder.nonDurable("testQueue").autoDelete().build());
    //声明一个绑定，将交换器和队列绑定起来
    admin.declareBinding(new Binding("testQueue", Binding.DestinationType.QUEUE,
                                     "exchange.direct","wyjie.news", null));
    Map<String, Object> map = new HashMap<>();
    map.put("msg", "这是第一个消息");
    map.put("data", Arrays.asList("helloworld", 123, true));
    //对象被默认序列化以后发送出去
    template.convertAndSend("exchange.direct","wyjie.news",
                            map);
    Object o = template.receiveAndConvert("testQueue");

    System.out.println(o.getClass());
}
```

发送的消息为对象的时候，会使用默认的序列化工具，我们可以自定义序列化工具，自己创建一个配置类，然后加入MessageConverter：

```java
@Configuration
public class MyAMQPConfig {
    
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

我们要实现监听功能，只需在方法上添加@RabbitListener注解，并给出queue属性，就可以将队列上收到的消息传给方法参数，然后进行处理，但是要让@RabbitListenner起作用，必须开启@EnableRabbit（放在主配置类之上）：

```java
@EnableRabbit //开启基于注解额RabbitMQ
@SpringBootApplication
public class SpringBoot02AmqpApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBoot02AmqpApplication.class, args);
    }
}

@Service
public class BookService {
    
    @RabbitListener(queues = "testQueue")
    public void receive(Book book) {
        System.out.println("收到消息：" + book.toString());
    }
}
```

