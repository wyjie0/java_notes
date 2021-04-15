



## 一、redis-cli命令

远程连接redis：
    `redis-cli -h hostname -p port -a  password`


## 二、Redis命令
### 1、字符串
#### 自增命令和自减命令
**incr**：
    incr key-name：将键存储的值加上1
**decr**：
    decr key-name：将键存储的值减去1
**incrby**：
    incrby key-name amount：将键存储的值加上整数amount
**decrby**：
    decrby key-name amount：将键存储的值减去整数amount
**incrbyfloat**
    incrbyfloat key-name amount：将键存储的值加上浮点数amount

#### 供redis处理子串和二进制位的命令
![](F:\Java书和视频\笔记\images\redis.png)

### 2、列表
redis的列表底层是用**链表**来实现的（称之为快速链表），此链表有特殊之处：首先**在列表元素较少的情况下会使用一块连续的内存存储，这个结构是ziplist，也就是压缩列表**。**当数据量增多的时候才会改成quicklist，将多个ziplist使用双向指针串起来使用。**
这样既满足快速的添加和删除性能， 又不会出现太大的空间冗余

#### 一些常用的列表命令
![](F:\Java书和视频\笔记\images\redis1.png)
#### 阻塞式的列表弹出命令以及在列表之间移动元素的命令
![](F:\Java书和视频\笔记\images\redis2.png)

### 3、集合
#### 一些常用的集合命令
![](F:\Java书和视频\笔记\images\redis3.png)
#### 组合和处理多个结合的命令
![](F:\Java书和视频\笔记\images\redis4.png)
![](F:\Java书和视频\笔记\images\redis5.png)

### 4、散列
与java中的hashmap不一样的是，Redis的散列的值只能是字符串（包括字符串、整数和浮点数）。而且，java的HashMap在字典很大时，rehash是个耗时的操作，需要一次性全部rehash，而Redis为了高性能，不能阻塞服务，所以采用了渐进式rehash
![](F:\Java书和视频\笔记\images\redis6.png)

渐进式rehash会在rehash的同时，保留新旧两个hash结构，查询时会同时查询两个hash结构，然后在后续的定时任务中以及hash的子指令中，循序渐进地将旧hash一点点迁移到新的hash结构中
#### 用于添加和删除键值对的散列操作
![](F:\Java书和视频\笔记\images\redis7.png)
#### Redis散列的高级特性命令
![](F:\Java书和视频\笔记\images\redis8.png)

### 5、有序集合

类似于java的SortedSet和HashMap的结合体，一方面它是一个set，保证了内部成员的**唯一性**，另一方面它可以给每个成员赋予一个分值，代表成员的排序权重

zset内部的排序功能是通过“**跳跃列表**”数据结构来实现的。跳跃类似于层级制，也类似于B+树。

![](F:\Java书和视频\笔记\images\redis9.png)

最下面一层所有的元素都会串起来，然后每隔几个元素挑选出一个代表，再将这几个代表使用另外一级指针串起来。然后在这些代表里再挑出二级代表，再串起来。跳跃列表之所以“跳跃”，是因为内部的元素可能处于好几个层次。
定位插入点时，先在顶层进行定位，然后下潜到下一级定位，一直下潜到最底层找到合适的位置，将新元素插进去

#### 常用的有序集合命令
![](F:\Java书和视频\笔记\images\redis10.png)
#### 有序集合的范围性数据获取命令和范围性数据删除命令，以及并集命令和交集命令
![](F:\Java书和视频\笔记\images\redis11.png)

### 6、==Stream==（支持多播可持久化消息队列）

![](F:\Java书和视频\笔记\images\redis45.png)



Redis Stream的结构如上图所示，它有一个消息链表，将所有加入的消息都串起来，**每个消息都有一个唯一的ID和对应的内容**。消息是持久化的，Redis重启过后，消息还在。

**每个Stream都有唯一的名称**，它就是Redis的key，在我们首次使用xadd指令追加消息时指定创建。

每个Stream都可以挂多个消费组，每个消费组都会有个游标**last_delivered_id**在Stream链表之上往前移动，表示当前消费组已经消费到哪条消息了。每个消费组都有一个 Stream 内唯一的名称，消费组不会自动创建，它需要单独的指令**xgroup create**进行创建，需要指定从 Stream 的某个消息 ID 开始消费，这个 ID 用来初始化last_delivered_id变量。

每个消费组的状态都是独立的，相互不受影响。也就是说同一份Stream内部的消息会被每个消费组都消费到。

同一个消费组可以挂接多个消费者，这些消费者之间是**竞争**关系，任意一个消费者读取了消息都会使游标**last_delivered**往前移动，每个消费者有一个组内唯一名称。

消费者 (Consumer) 内部会有个状态变量pending_ids，它记录了当前已经被客户端读取的消息，但是还没有 ack。如果客户端没有 ack，这个变量里面的消息 ID 会越来越多，一旦某个消息被 ack，它就开始减少。这个 pending_ids 变量在 Redis 官方被称之为PEL，也就是Pending Entries List，这是一个很核心的数据结构，它用来确保客户端至少消费了消息一次，而不会在网络传输的中途丢失了没处理。

#### 6.1 消息ID

消息 ID 的形式是**timestampInMillis-sequence**，例如1527846880572-5，它表示当前的消息在毫秒时间戳1527846880572时产生，并且是该毫秒内产生的第 5 条消息。消息 ID 可以由服务器自动生成，也可以由客户端自己指定，但是形式必须是整数 - 整数，而且必须是后面加入的消息的 ID 要大于前面的消息 ID。

#### 6.2 消息内容

消息内容是键值对

#### 6.3 增删改查

1. xadd：追加消息

2. xdel：删除消息，仅仅设置标志位，不影响消息总长度

3. xrange：获取消息列表，自动过滤已经删除的消息

4. xlen：消息长度

5. del：删除Stream

   ```shell
   # * 号表示服务器自动生成消息ID，后面顺序跟着一堆key/value
   # stream的名字就是codehole
   192.168.0.105:6379> xadd codehole * name wyjie age 24
   "1584001270409-0"
   192.168.0.105:6379> xadd codehole * name laowang age 24
   "1584001388822-0"
   192.168.0.105:6379> xadd codehole * name www age 22
   "1584001545607-0"
   192.168.0.105:6379> xlen codehole
   (integer) 3
   # - 表示最小值，+ 表示最大值
   192.168.0.105:6379> xrange codehole - +
   1) 1) "1584001270409-0"
      2) 1) "name"
         2) "wyjie"
         3) "age"
         4) "24"
   2) 1) "1584001388822-0"
      2) 1) "name"
         2) "laowang"
         3) "age"
         4) "24"
   3) 1) "1584001545607-0"
      2) 1) "name"
         2) "www"
         3) "age"
         4) "22"
   # 指定最大消息 ID 的列表
   192.168.0.105:6379> xrange codehole - 1584001388822-0
   1) 1) "1584001270409-0"
      2) 1) "name"
         2) "wyjie"
         3) "age"
         4) "24"
   2) 1) "1584001388822-0"
      2) 1) "name"
         2) "laowang"
         3) "age"
         4) "24"
   192.168.0.105:6379> xdel codehole 1584001388822-0
   (integer) 1
   192.168.0.105:6379> xlen codehole
   (integer) 2
   192.168.0.105:6379> xrange codehole - +
   1) 1) "1584001270409-0"
      2) 1) "name"
         2) "wyjie"
         3) "age"
         4) "24"
   2) 1) "1584001545607-0"
      2) 1) "name"
         2) "www"
         3) "age"
         4) "22"#### 6.4 独立消费
   ```

#### 6.4 独立消费

我们可以在不定义消费组的情况下进行Stream消息的独立消费，当Stream没有消息时，甚至可以阻塞等待。Redis设计了一个单独的消费指令**xread**，可以将Stream当成普通的消息队列来使用。使用xread时，我们可以完全忽略消费组的存在，就好比Stream就是一个普通的列表。

```shell
# 从Stream头部读取两条消息
192.168.0.105:6379> xread count 2 streams codehole 0-0
1) 1) "codehole"
   2) 1) 1) "1584002465385-0"
         2) 1) "name"
            2) "wyjie"
            3) "age"
            4) "24"
      2) 1) "1584002470578-0"
         2) 1) "name"
            2) "laowang"
            3) "age"
            4) "24"
# 从 Stream 尾部读取一条消息，毫无疑问，这里不会返回任何消息
127.0.0.1:6379> xread count 1 streams codehole $
(nil)
# 从尾部阻塞等待消息到来，下面的指令会堵住，直到新消息到来
192.168.0.105:6379> xread block 0 count 1 streams codehole $
# 我们从新打开一个窗口，在这个窗口往 Stream 里塞消息
192.168.0.105:6379> xadd codehole * name youming age 33
"1584002891951-0"
# 再切换到前面的窗口，我们可以看到阻塞解除了，返回了新的消息内容
# 而且还显示了一个等待时间，这里我们等待了103s
192.168.0.105:6379> xread block 0 count 1 streams codehole $
1) 1) "codehole"
   2) 1) 1) "1584002891951-0"
         2) 1) "name"
            2) "youming"
            3) "age"
            4) "33"
(103.78s)
```

客户端如果想要使用xread进行顺序消费，一定要记住当前消费到哪里了，也就是返回的消息ID。消磁继续调用xread时，将上次返回的最后一个消息ID作为参数传递进去就可以继续消费后续的消息

#### 6.5 创建消费组

![](F:\Java书和视频\笔记\images\redis46.png)

Stream通过xgroup create指令创建消费组，需要传递起始消息ID参数用来初始化last_delivered_id变量

```shell
# 表示从头开始消费
# codehole是要创建消费组的流的名字
# cg1是创建的消费组的名字
# 0-0表示消息ID，用来初始化last_delivered_id
192.168.0.105:6379> xgroup create codehole cg1 0-0
OK
# $ 表示从尾部开始消费，值接受新消息，当前Stream消息会全部忽略
192.168.0.105:6379> xgroup create codehole cg2 $
OK
# 获取stream消息
192.168.0.105:6379> xinfo stream codehole
 1) "length"
 2) (integer) 4	#	共4个消息
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "groups"
 8) (integer) 2 #	2个消费组
 9) "last-generated-id"
10) "1584002891951-0"
11) "first-entry" # 第一个消息
12) 1) "1584002465385-0"
    2) 1) "name"
       2) "wyjie"
       3) "age"
       4) "24"
13) "last-entry" # 最后一个消息
14) 1) "1584002891951-0"
    2) 1) "name"
       2) "youming"
       3) "age"
       4) "33"
# 获取 Stream 的消息组信息
192.168.0.105:6379> xinfo groups codehole
1) 1) "name"
   2) "cg1"
   3) "consumers"
   4) (integer) 0 # 该消费组还没有消费者
   5) "pending"
   6) (integer) 0 # 该消费组没有正在处理的消息
   7) "last-delivered-id"
   8) "0-0"
2) 1) "name"
   2) "cg2"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "1584002891951-0"
```

#### 6.6 消费

Stream 提供了 xreadgroup 指令可以进行消费组的组内消费，需要提供消费组名称、消费者名称和起始消息 ID。它同 xread 一样，也可以阻塞等待新消息。读到新消息后，对应的消息 ID 就会进入消费者的 PEL(正在处理的消息) 结构里，客户端处理完毕后使用 xack 指令通知服务器，本条消息已经处理完毕，该消息 ID 就会从 PEL 中移除。

```shell
# > 号表示从当前消费组的last_delivered_id后面开始读
# 每当消费者读取一条消息，last_delivered_id变量就会前进
# c1 是cg1消费组中的消费者的名称
192.168.0.105:6379> xreadgroup GROUP cg1 c1 count 1 streams codehole >
1) 1) "codehole"
   2) 1) 1) "1584002465385-0"
         2) 1) "name"
            2) "wyjie"
            3) "age"
            4) "24"
192.168.0.105:6379> xreadgroup GROUP cg1 c1 count 1 streams codehole >
1) 1) "codehole"
   2) 1) 1) "1584002470578-0"
         2) 1) "name"
            2) "laowang"
            3) "age"
            4) "24"
192.168.0.105:6379> xreadgroup GROUP cg1 c1 count 3 streams codehole >
1) 1) "codehole"
   2) 1) 1) "1584002475293-0"
         2) 1) "name"
            2) "www"
            3) "age"
            4) "22"
      2) 1) "1584002891951-0"
         2) 1) "name"
            2) "youming"
            3) "age"
            4) "33"
# 阻塞等待
192.168.0.105:6379> xreadgroup GROUP cg1 c1 block 0 count 1 streams codehole >
# 开启另一个窗口，往里塞消息
192.168.0.105:6379> xadd codehole * name lanying age 61
"1584005959421-0"
# 回到前一个窗口，发现阻塞解除了，收到新消息了
192.168.0.105:6379> xreadgroup GROUP cg1 c1 block 0 count 1 streams codehole >
1) 1) "codehole"
   2) 1) 1) "1584005959421-0"
         2) 1) "name"
            2) "lanying"
            3) "age"
            4) "61"
(50.37s)
# 观察消费组的信息
192.168.0.105:6379> xinfo groups codehole
1) 1) "name"
   2) "cg1"
   3) "consumers"
   4) (integer) 1 # 一个消费者
   5) "pending"
   6) (integer) 5 # 共 5 条正在处理的信息还没有 ack
   7) "last-delivered-id"
   8) "1584005959421-0"
2) 1) "name"
   2) "cg2"
   3) "consumers"
   4) (integer) 0 # 消费组 2 没有任何变化
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "1584002891951-0"
# 接下来 ack 一条消息
192.168.0.105:6379> xack codehole cg1 1584005959421-0
(integer) 1
192.168.0.105:6379> xinfo consumers codehole cg1
1) 1) "name"
   2) "c1"
   3) "pending"
   4) (integer) 4 # 变成了 4 条
   5) "idle"
   6) (integer) 345977 # 空闲了多长时间（ms）没有读取消息了
```

#### 6.7 Stream消息太多怎么办？

Redis 提供了一个定长的 Stream 功能。在 xadd 的指令提供一个定长长度 maxlen ，就可以将旧消息干掉，确保最多不超过指定长度

```shell
192.168.0.105:6379> xlen codehole
(integer) 5
192.168.0.105:6379> xadd codehole maxlen 3 * name xiaorui age 1
"1584006727675-0"
192.168.0.105:6379> xlen codehole
(integer) 3
192.168.0.105:6379> xrange codehole - +
1) 1) "1584002891951-0"
   2) 1) "name"
      2) "youming"
      3) "age"
      4) "33"
2) 1) "1584005959421-0"
   2) 1) "name"
      2) "lanying"
      3) "age"
      4) "61"
3) 1) "1584006727675-0"
   2) 1) "name"
      2) "xiaorui"
      3) "age"
      4) "1"
```

#### 6.8 消息如果忘记 ack 会怎样

Stream 在每个消费者结构中保存了正在处理中的消息 ID 列表 PEL，如果消费者收到了消息处理完了但是没有回复 ack，就会导致 PEL 列表不断增长，如果有很多消费组的话，那么这个 PEL 占用的内存就会变大。



### 7、发布与订阅

发布与订阅的特点是订阅者负责订阅频道，发送者负责向频道发送二进制字符串消息。每当有消息被发送至给定频道时，频道的所有订阅者都会收到消息。
#### 相关命令
![](F:\Java书和视频\笔记\images\redis12.png)
![](F:\Java书和视频\笔记\images\redis13.png)

### 8、排序
![](F:\Java书和视频\笔记\images\redis14.png)

### 9、基本的redis事务

* Redis的基本事务需要用到multi和exec命令，这种事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令。和关系数据库中的回滚不同，在Redis里面，被multi和exec命令包围的所有命令会一个接一个地执行，直到所有命令都执行完毕。当一个事务执行完毕之后，Redis才会处理其他客户端的命令。
  
* 要在Redis里面实现事务，首先需要执行multi命令，然后输入那些我们想要在事务里面执行的命令，最后再执行exec命令。

### 10、键的过期时间
* 在使用Redis存储数据的时候，有些数据可能在某个时间点之后就不再有用了，用户可以用del命令显示地删除这些无用数据，也可以通过Redis的过期时间（expiration）特性来让键在给定的时限之后自动被删除。
![](F:\Java书和视频\笔记\images\redis15.png)

### 11、配置主从服务器
#### 11.1 CAP原理

CAP原理是分布式存储的理论基石

* C : Consistent，一致性

* A : Availability，可用性

* P : Partition tolerance，分区容忍性

  分布式系统节点往往都是分布在不同的机器上进行网络隔离开的，这意味着必然会有网络断开的风险，这个网络断开的场景就叫做**网络分区**

  在网络分区发生时，两个分布式结点之间无法进行通信，我们对一个节点进行的修改操作将无法同步到另外一个节点，所以数据的“一致性”将无法满足，因为两个分布式节点的数据不再保持一致。除非我们牺牲“可用性”，也就是暂停分布式节点的服务，在网络分区发生时，不再提供修改数据的功能，直到网络完全恢复再继续对外提供服务

  **网络分区发生时，一致性和可用性两难全**

#### 11.2 主从同步

使用slaveof命令配置主从服务器时，只需在从服务器中配置
`slaveof host port`就行。但是如果主服务器设置了密码，那么从服务器也要设置与主服务器一样的密码。使用命令`config set requirepass "password"`。设置了密码过后，还需配置从服务器中的masterauth属性：`config set masterauth "masterPassword"`

##### **从服务器连接主服务器时的步骤**
![](F:\Java书和视频\笔记\images\redis16.png)

**从服务器在进行同步时，会清空自己所有的数据，并被替换成主服务器发来的数据**

## 三、Redis应用

### 1、分布式锁
使用指令 `set key_name true ex 5 nx`就可以实现分布式锁，这个指令就是setnx和expire组合在一起的原子指令
#### 超时问题
Redis的分布式锁不能解决超时问题，如果在加锁和释放锁之间的逻辑执行的太长，以至于超出了锁的超时限制，就会出现问题。这时锁已经过期，第二个线程重新持有了这把锁，但是紧接着第一个线程执行完了业务逻辑，就把锁释放了，第三个线程就会在第二个线程逻辑执行完之前拿到锁。
为了避免这个问题，redis分布式锁不要用于较长时间的任务。一个更加安全的方案为：为set指令的value参数设置为一个随机数，释放锁时先匹配随机数是否一致，再删除key

### 2、延时队列
使用列表作为队列使用时，会遇见队列为空的情况，此时客户端会陷入pop的死循环，不停地pop。使用blpop/brpop可以解决这个问题。如果线程一致阻塞，Redis的客户端连接就成了闲置连接，闲置过久，服务器一般会主动端看连接，减少闲置资源占用。这个时候blpop/brpop会**抛出异常**

#### 锁冲突处理
一般有3中策略来处理加锁失败：
1. 直接抛出异常，通知用户稍后重试
2. sleep一会再重试
3. 将请求转移至延时队列，过一会再试

其中延时队列可以通过redis的zset来实现。我们将消息序列化成一个字符串作为zset的成员，这个消息的到期处理时间作为分值，然后用多个线程轮训zset获取到期的任务进行处理

### 3、位图
使用setbit/getbit指令可以对redis中的位进行读写
<img src="F:\Java书和视频\笔记\images\redis17.png" style="zoom: 67%;" />
<img src="F:\Java书和视频\笔记\images\redis18.png" style="zoom: 67%;" />

<img src="F:\Java书和视频\笔记\images\redis19.png" style="zoom: 67%;" />

redis还提供了位图统计指令bitcount和位图查找指令bitpos。
bitcount用来统计指定位置范围内1的个数，bitpos用来查找指定范围内出现的第一个0或1。指定范围的参数start和end是字节索引，表示的是字符
<img src="F:\Java书和视频\笔记\images\redis20.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis21.png" style="zoom:67%;" />

Redis 3.2新增了bitfield指令，可以对指定**位片段**进行对鞋，但是最多只能处理64个连续的位
<img src="F:\Java书和视频\笔记\images\redis22.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis23.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis24.png" style="zoom:67%;" />

### 4、HyperLogLog
在解决统一问题的时候，如果数据量比较小，可以直接使用redis的set数据结构。如果数据量大，则需要使用更加节约存储空间的数据结构。**HyperLogLog就是Redis的高级数据结构。**

HyperLogLog提供了两个指令：pfadd用于增加计数和pfcount用于获取计数。但是它需要占据一定12k的存储空间。所以它不适合统计单个用户相关的数据

**场景**：大型网站中记录每天的UV数据
### 5、布隆过滤器
布隆过滤器可以理解为一个不怎么精确的set结构，当使用它的contains方法判断某个对象是否存在的时候，它可能会误判。但是布隆过滤器也不是特别不精确，只要参数设置得合理，它的精度可以控制得相对足够精确。

当布隆过滤器说某个值不存在时，这个值肯定不存在。但是当判断某个值存在时，这个值可能不存在

Redis官方提供的布隆过滤器是在Redis 4.0以插件功能提供的，布隆过滤器作为一个插件加载到Redis Server中，给Redis提供了强大的布隆去重功能

**场景**：邮箱系统的垃圾邮件过滤功能、爬虫系统中URL的去重、降低数据库IO请求，过滤不存在的row请求去磁盘查询、推荐系统

#### 布隆过滤器基本使用
`bf.add`：添加元素                       `bf.madd`：批量添加
`bf.exists`：查询元素是否存在          `bf.mexists`：批量判断

### 6、分布式限流
限流算法在分布式领域是一个经常被提起的话题，当系统的处理能力有限时，如何阻止计划外的请求继续对系统施压，这是一个需要重视的问题。

除了控制流量，限流还有一个应用目的是用于控制用户行为，避免垃圾请求。比如在贴吧社区中，用户的发帖、回复、点赞等行为都要严格受控，**一般要严格限定某行为在规定时间内允许的次数**，超过了次数就是非法行为。对非法行为，也许必须规定适当的惩处策略。（**网页提示操作太快是靠redis实现的**）

#### redis实现限流
首先我们来看一个常见 的简单的限流策略。系统要限定用户的某个行为在指定的时间里只能允许发生 N次，如何使用 Redis 的数据结构来实现这个限流的功能？ 

解决方案：
    这个限流需求中存在一个滑动时间窗口，可以使用zset来实现，使用zset中的分值来圈出这个时间窗口，zset中的成员使用毫秒时间戳。
    也就是用一个zset结构记录用户的行为历史，每一个行为都会作为zset中的一个key保存下来。同一个用户同一种行为用一个zset保存。

```java
public boolean isActionAllowed(String userId, String actionKey, int period, int maxCount) {    
    String key = String.format("hist:%s:%s", userId, actionKey);//每个用户的每个动作都是一张表
    long nowTs = System.currentTimeMillis();
    jedis.auth("123456");
    Pipeline pipeline = jedis.pipelined();
    pipeline.multi();
    pipeline.zadd(key, nowTs, ""+nowTs);//使用当前时间戳作为成员，时间作为分值    
    System.out.println(nowTs);    System.out.println(nowTs - period * 1000);   
    pipeline.zremrangeByScore(key, 0, nowTs-period * 1000);//删除当前时间60s之前的行为    
    Response<Long> response = pipeline.zcard(key);    pipeline.expire(key, period + 1);  
    pipeline.exec();
    pipeline.close();
    return response.get() <= maxCount;
}
```

#### “漏斗”限流算法
漏斗的剩余空间代表着当前行为可以持续进行的数量，漏斗的流水速率代表着系统允许该行为的最大频率。
用java实现漏斗算法如下：
```java
    public class FunnelRateLimiter {       
        static class Funnel {        
            int capacity;
            float leakingRate;
            int leftQuota;
            long leakingTs;
            public Funnel (int capacity, float leakingRate) {
            this.capacity = capacity;//5
            this.leakingRate = leakingRate;//1
            this.leftQuota = capacity;//5
            this.leakingTs = System.currentTimeMillis();
            }
            //漏水算法，每次灌水前都会被调用以触发漏水，给漏斗腾出空间来
            //能腾出多少空间取决于过去了多久以及流水的速率
            void makeSpace() {
                long nowTs = System.currentTimeMillis();
                long deltaTs = nowTs - leakingTs;
                int deltaQuota = (int) (deltaTs * leakingRate); 
                if (deltaQuota < 0) {//间隔时间太长，整数数字过大溢出
                    this.leftQuota = capacity;
                    this.leakingTs = nowTs;
                    return;
                }
                if (deltaQuota < 1) {//腾出空间太小，最小单位为1
                    return;
                }
                this.leftQuota += deltaQuota;
                this.leakingTs = nowTs;
                if (this.leftQuota > this.capacity) {
                    this.leftQuota = this.capacity;
                }
            }
            boolean watering(int quota) {
                makeSpace();
                if (this.leftQuota >= quota) {
                    this.leftQuota -= quota;
                    return true;
                }
                return false;
            }
        }
        private Map<String, Funnel> funnels = new HashMap<>();
        public boolean isActionAllowed(String userId, String actionKey,
                                                    int capacity, float leakingRate) {
            String key = String.format("%s:%s", userId, actionKey);
            Funnel funnel = funnels.get(key);
            if (funnel == null) {
                funnel = new Funnel(capacity, leakingRate);
                funnels.put(key, funnel);
            }
            return funnel.watering(1);
        }
    }


```

而在redis 4.0中，提供了一个限流redis模块，名字叫redis-cell。该模块也使用了漏斗算法，并提供了原子的限流命令。该模块只有一条指令：cl.throttle
<img src="F:\Java书和视频\笔记\images\redis25.png" style="zoom:67%;" />
上述指令的意思是：允许“用户老钱的回复行为“的频率为没60s最多30次，漏斗的初试容量为15，也就是说一开始可连续回复15个帖子，然后才开始受漏水速率的影响

### 7、GeoHash
Redis 3.2版本以后增加了地理位置GEO模块，意味着可以使用redis实现摩拜单车“**附近的Mobike**”、美团和饿了么“**附近的餐馆**”这样的功能
GeoHash 算法**将二维的经纬度数据映射到一维的整数**，这样所有的元素都将在挂载到一条线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。当我们想要计算「附近的人时」，首先将目标位置映射到这条线上，然后在这个一维的线上获取附近的点就行了。 

那这个映射算法具体是怎样的呢？它将整个地球看成一个二维平面，然后划分成了一系列正方形的方格，就好比围棋棋盘。所有的地图元素坐标都将放置于唯一的方格中。方格越小，坐标越精确。然后对这些方格进行整数编码，越是靠近的方格编码越是接近。那如何编码呢？一个最简单的方案就是切蛋糕法。设想一个正方形的蛋糕摆在你面前，二刀下去均分分成四块小正方形，这四个小正方形可以分别标记为 00,01,10,11 四个二进制整数。然后对每一个小正方形继续用二刀法切割一下，这时每个小小正方形就可以使用 4bit 的二进制整数予以表示。然后继续切下去，正方形就会越来越小，二进制整数也会越来越长，精确度就会越来越高。  

上面的例子中使用的是二刀法，真实算法中还会有很多其它刀法，最终编码出来的整数数字也都不一样。 

编码之后，每个地图元素的坐标都将变成一个整数，通过这个整数可以还原出元素的坐标，整数越长，还原出来的坐标值的损失程度就越小。对于「附近的人」这个功能而言，损失的一点精确度可以忽略不计。 
GeoHash 算法会继续对这个整数做一次 base32 编码 (0-9,a-z 去掉 a,i,l,o 四个字母) 变成一个字符串。在 Redis 里面，经纬度使用 52 位的整数进行编码，放进了 zset 里面，zset 的 value 是元素的 key，score 是 GeoHash 的 52 位整数值。zset 的 score 虽然是浮点数，但是对于 52 位的整数值，它可以无损存储。 

在使用 Redis 进行 Geo 查询时，我们要时刻想到它的**内部结构实际上只是一个 zset(skiplist)**。

#### Geo指令
Redis提供的geo指令有6个
**增加**：`geoadd key-name longtitude latitude name`
<img src="F:\Java书和视频\笔记\images\redis26.png" style="zoom:67%;" />
**距离**：geodist指令用来计算两个元素之间的距离，携带集合名称、两个名称和距离单位
<img src="F:\Java书和视频\笔记\images\redis27.png" style="zoom:67%;" />
其中距离单位可以是m、km、ml、ft，分别表示米、千米、英里和尺
**获取元素位置**：geopos指令可以获取集合中任意元素的经纬度坐标，可以一次获取多个
<img src="F:\Java书和视频\笔记\images\redis28.png" style="zoom:67%;" />
**获取元素的hash值**：geohash可以获取元素的经纬度编码字符串，它是base32编码。
<img src="F:\Java书和视频\笔记\images\redis29.png" style="zoom:67%;" />
**附近的公司**：georadiusbymember指令可以用来查询指定元素附近的其他元素
<img src="F:\Java书和视频\笔记\images\redis30.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis31.png" style="zoom:67%;" />
**根据坐标值查询附近的元素**：它可以根据用户的定位来计算“附近的车”，“附近的餐馆等”。
<img src="F:\Java书和视频\笔记\images\redis32.png" style="zoom:67%;" />

### 8、Scan
在平时线上 Redis 维护工作中，有时候需要从 Redis 实例成千上万的 key 中找出特定前缀的 key 列表来手动处理数据，可能是修改它的值，也可能是删除 key。这里就有一个问题，如何从海量的 key 中找出满足特定前缀的 key 列表来？ 

Redis 提供了一个简单暴力的指令**keys** 用来列出所有满足特定正则字符串规则的 key。 

<img src="F:\Java书和视频\笔记\images\redis33.png" style="zoom:67%;" />
这个指令使用简单，提供一个简单的正则字符串即可，但是有很明显的两个缺点：

1.  没有offset、limit参数，一次性突出所有满足条件的key，如果实例中的key的量特别大，查看起来会很麻烦
2. keys算法是**遍历算法**，复杂度是O(n)，如果实例中有千万级的key，这个指令就会导致Redis服务卡顿，所有读写Redis的其他的指令都会被延后甚至会超时报错，因为Redis是单线程程序，顺序执行所有指令，其他指令必须等到当前的keys指令执行完了才可以继续

为了解决上述问题，Redis在2.8版本中加入了指令**Scan**，特点是：
1. 复杂度虽然也是O（n），但是它是通过游标分步进行的，**不会阻塞线程**
2. **提供limit参数**，可以控制每次返回结果的最大条数
3. 同keys一样，它也提供模式匹配功能
4. 服务器不需要为游标保存状态，游标的唯一状态就是scan返回给客户端的游标整数
5. **返回的结果可能会有重复，需要客户端去重**
6. 遍历的过程中如果有数据修改，改动后的数据能不能遍历到是不确定的
7. 单次返回的结果是空的并不意味着遍历结束，而要看返回的游标值是否为零; 
#### Scan的使用
scan 参数提供了三个参数，第一个是 **cursor** 整数值，第二个是**key 的正则模式**，第三个是遍历的 **limit hint。**第一次遍历时，cursor 值为 0，然后将返回结果中第一个整数值作为下一次遍历的 cursor。一直遍历到返回的 cursor 值为 0 时结束。 

在使用前，往Redis插入10000条数据进行测试：
<img src="F:\Java书和视频\笔记\images\redis34.png" style="zoom:67%;" />

<img src="F:\Java书和视频\笔记\images\redis35.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis36.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis37.png" style="zoom:67%;" />
<img src="F:\Java书和视频\笔记\images\redis38.png" style="zoom:67%;" />
从上面的过程可以看到虽然提供的 limit 是 1000，但是返回的结果只有 10 个左右。因为这个 limit 不是限定返回结果的数量，而是限定服务器单次遍历的字典槽位数量(约等于)。如果将 limit 设置为 10，你会发现返回结果是空的，但是游标值不为零，意味着遍历还没结束。 

#### Redis的字典存储结构
Redis 中所有的 key 都存储在一个很大的字典中，这个字典的结构和 Java 中的 HashMap 一样，是一维数组 + 二维链表结构，第一维数组的大小总是 2^n(n>=0)，扩容一次数组大小空间加倍，也就是 n++。 

<img src="F:\Java书和视频\笔记\images\redis39.png" style="zoom:67%;" />

 **scan指令返回的游标就是第一维数组的位置索引，我们将这个位置索引称为槽 (slot)** 。如果不考虑字典的扩容缩容，直接按数组下标挨个遍历就行了。limit 参数就表示需要遍历的槽位数，之所以返回的结果可能多可能少，是因为不是所有的槽位上都会挂接链表，有些槽位可能是空的，还有些槽位上挂接的链表上的元素可能会有多个。**每一次遍历都会将 limit 数量的槽位上挂接的所有链表元素进行模式匹配过滤后，一次性返回给客户端。**
#### Scan的遍历顺序
scan的遍历顺序非常特别。它不是从第一维数组的第0位一直遍历到末尾，而是采用了**高位进位加法**来遍历。避免在字典扩容或缩容后槽位遍历重复和遗漏

高位进位加法从左边开始相加，进位向右边移动

## 四、Redis原理

### 1、线程IO模型

* Redis是==**单线程程序**==

* Redis单线程为什么还能这么快？

​		因为它所有的数据都在**内存**中，所有的运算都是内存级别的运算。在使用Redis指令时，对于那些时间复杂度为O（n）级别的指令，一定要谨慎使用，一不小心就可能导致Redis卡顿

* Redis单线程如何处理那么多的并发客户端连接？

  是因为非阻塞IO和多路复用

  ####  非阻塞IO

  当我们调用套接字的读写方法，默认它们是阻塞的，比如read 方法要传递进去一个参数n，表示读取这么多字节后再返回，如果没有读够线程就会卡在那里，直到新的数据到来或者连接关闭了，read 方法才可以返回，线程才能继续处理。而write 方法一般来说不会阻塞，除非内核为套接字分配的写缓冲区已经满了，write 方法就会阻塞，直到缓存区中有空闲空间挪出来了。

  非阻塞 IO 在套接字对象上提供了一个选项Non_Blocking，当这个选项打开时，读写方法不会阻塞，而是能读多少读多少，能写多少写多少。能读多少取决于内核为套接字分配的读缓冲区内部的数据字节数，能写多少取决于内核为套接字分配的写缓冲区的空闲空间字节数。读方法和写方法都会通过返回值来告知程序实际读写了多少字节。有了非阻塞 IO 意味着线程在读写 IO 时可以不必再阻塞了，读写可以瞬间完成然后线程可以继续干别的事了。
  
  #### 事件轮询（多路复用）

  非阻塞IO有个问题，那就是线程要读数据，结果读了一部分就返回了，线程如何知道何时才应该继续读，也就是在数据到来时，线程如何得到通知？同样，如果缓冲区满了， 写不完数据，剩下的数据何时才应该继续写？

  事件轮询API就是用来解决这个问题的。利用**select()系统调用**可以同时处理多个通道描述符的读写事件，这类系统调用也可称为**多路复用API**。现代操作系统的多路复用API使用epoll和kqueue。在Java中的事件轮询API就是NIO技术。

  #### 指令队列

  Redis会将每个客户端套接字都关联一个指令队列， 客户端的指令通过队列来排队进行顺序处理，先到先服务。

### 2、Redis的通信协议

#### 2.1 Redis序列化协议

Redis序列化协议将传输的结构数据分为5种最小单元类型，单元结束时统一加上回车换行符\r\n

1. 单行字符串	以+符号开头 `hello world	--->	+hello world\r\n`
2. 多行字符串    以$符号靠头，后跟你字符串长度 `hello world   --->    $11\r\nhello world\r\n`
3. 整数值     以：符号开头，后跟整数的字符串形式 `1024    --->    :1024\r\n`
4. 错误消息    以 - 符号开头 `-WRONGTYPE Operation against a key holding the wrong kind of value`
5. 数组    以*符号开头，后跟数组的长度 `[1,2,3]  -->  *3\r\n:1\r\n:2\r\n:3\r\n`
6. Null 用多行字符串表示，不过长度要写成-1 `$-1\r\n`
7. 空串 用多行字符串表示，长度填0 `$0\r\n\r\n`

### 3、持久化

Redis的持久化机制有两种，第一种是快照，第二种是AOF日志。**快照是一次全量备份**，是内存数据的二进制序列化形式；**AOF日志是连续的增量备份**，记录的是内存数据修改的指令文本，AOF日志在长期的运行过程中会变得无比庞大，数据库重启时需要加载AOF日志进行指令重放，这个时间就会很漫长。所以需要定期进行AOF重写，给AOF日志进行瘦身

#### 3.1 快照机制

Redis使用操作系统的多进程COW（Copy On Write)机制来实现快照持久化。在持久化时，会**产生一个子进程来专门处理持久化的工作**，它不会修改现有的内存数据结构，只是**对数据结构进行遍历提取**，然后序列化写到磁盘中。**父进程持续处理客户端请求，然后对数据结构进行不间断的修改**。

这个时候会使用操作系统的**COW**机制来进行数据段页面的分离。数据段是由很多操作系统的页面组合而成，当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面没有变化。

#### 3.2 AOF原理

AOF日志存储的是Redis服务器的顺序**指令序列**，只存储对内存进行修改的指令记录。

Redis提供了bgrewriteaof指令用于对AOF日志进行瘦身。其原理就是开辟一个**新的子进程**对内存进行遍历**转换成一系列Redis的操作指令**，序列化到一个**新的AOF日志**文件中

**文件同步**	在向硬盘写入文件时，至少会发生3件事。当调用file.write()方法（或其他编程语言里面的类似操作）对文件进行写入时，写入的内容首先会被存储到缓冲区，然后操作系统会在将来的某个时候将缓冲区存储的内容写入硬盘，而数据只有在被写入硬盘之后，才算是真正地保存到了硬盘里面。用户可以通过调用file.flush()方法来请求操作系统尽快地将缓冲区存储的数据写入硬盘里，但具体何时执行写入操作仍然由操作系统决定。除此之外，用户还可以命令操作系统将文件**同步**到硬盘，同步操作会一直阻塞直到指定的文件被写入硬盘为止。当同步操作执行完毕之后，即时系统出现故障也不会对同步的文件造成任何影响。

Redis的`appendfsync`配置选项对AOF文件的同步频率有影响：

```shell
always:每个Redis写命令都要同步写入硬盘。这样做会严重降低Redis的速度
everysec:每秒执行一次同步，显示地将多个写命令同步到硬盘
no:让操作系统来决定应该何时进行同步（不建议使用）
```

### 4、管道

当我们使用客户端对Redis进行一次操作时，客户端将请求传送给服务器，服务器处理完毕后，再将响应回复给客户端，这要花费一个网络数据包来回的时间。如果连续执行多条指令， 就会耗费好多数据传输的时间。

在需要执行大量命令的情况下，即使命令实际上并不需要放在事务里面执行，但是为了通过**一次发送所有命令来减少通信次数并降低延迟值**，**用户也可能会将命令包裹在`multi`和`exec`里面执行。但是，`multi`和`exec`并不是免费的——它们会消耗资源，并且可能会导致其他重要的命令被延迟执行**。

取而代之的是，我们可以使用`pipeline`来完成上述功能。通过将标准的Redis连接替换成pipeline连接，程序可以减少通信往返次数至原来的1/2到1/5.

### 5、事务

#### 1、Redis没有实现典型的加锁功能

因为加锁会造成长时间的等待，所以Redis为了尽可能地减少客户端的等待时间，并不会在执行watch命令时对数据进行加锁。相反地，**Redis只会在数据被其他客户端抢先修改了的情况下，通知执行了watch命令的客户端，这种做法被称为==乐观锁==**·

与事务相关的命令有：`watch 、multi/exec、unwatch、discard（很少使用）`

在用户使用watch命令对键进行监视之后，直到用户执行exec命令的这段时间里，如果有其他客户端抢先对被监视的键进行了替换、更新或删除操作，那么当用户尝试执行**exec**命令的时候，事务将失败并返回一个错误（之后用户可以选择重试或者放弃事务）。

**Redis禁止在multi和exec之间执行watch指令，而必须在multi之前做好盯住关键变量，否则会出错**

* **为什么Redis没有实现典型的加锁功能？**

  在访问以写入为目的数据的时候（SQL中的select for update），关系数据库会对被访问的数据进行加锁，直到事务被提交或者被回滚为止。如果有其他客户端试图对被加锁的数据行进行写入，那么该客户端将被阻塞，直到第一个事务执行完毕位置。加锁在实际使用中非常有效，基本上所有关系数据库都实现了这种加锁功能，它的缺点在于，持有锁的客户端运行越慢，等待解锁的客户端被阻塞的时间越长。

  因为加锁有可能会造成长时间等待，所以Redis为了尽可能地减少客户端的等待时间，并不会在执行watch命令时对数据行进行加锁，相反地，Redis只会在数据已经被其他客户端抢先修改了的情况下，通知执行了watch命令的客户端，这种做法被称为乐观锁，而关系数据库实际执行的加锁操作则被称为悲观锁。

* **为什么Redis不支持回滚？**

  在事务运行期间，虽然Redis命令可能会执行失败，但是Redis仍然会执行事务中余下的其他命令，而不会执行回滚操作。

  只有当被调用的Redis命令有语法错误时，这条命令才会执行失败（在将这个命令放入事务队列期间，Redis能够发现此类问题）。鉴于没有任何机制能避免程序员自己造成的错误，并且这类错误通常不会在生产环境中出现，所以Redis选择了**更简单、更快速**的无回滚方式来处理

#### 2. Redis事务满足的特性

而且在Redis中的事务不能算**原子性**，而仅仅满足了事务的**隔离性**。如下图所示：

<img src="F:\Java书和视频\笔记\images\redis40.png" style="zoom:50%;" />

### 6、内存

#### 6.1 内存回收机制

Redis并不总是可以将空闲内存立即归还给操作系统。

如果当前Redis内存有10G，当删除了1GB的key后，再去观察内存，内存变化不会太大。原因是操作系统回收内存是以页为单位，如果这个页上只要有一个key还在使用，那么它就不能被回收。Redis虽然删除了1GB的key，但是这些key分散到了很多页面中，每个页面都还有其他key存在，这就导致了内存不会立即被回收。

但是执行`flushdb`，会将整个数据库的key都删除

#### 6.2 内存分配算法

Redis为了保持自身结构的简单性，将内存分配的细节丢给第三方内存分配库去实现。目前Redis可以使用jemalloc（facebook）库来管理内存，也可以切换到tcmalloc（google）。Redis默认使用jemalloc，因为其性能更好

## 五、集群

### 1、Sentinel

在主从节点运行中，如果主节点突然宕机，会给业务带来极大的麻烦。Redis Sentinel给我们提供了一个高可用方案来抵抗结点故障，**当故障发生时可以自动进行主从切换，程序可以不用重启**。

<img src="F:\Java书和视频\笔记\images\redis41.png" style="zoom:67%;" />

Sentinel负责持续监控主从节点的健康，当主节点挂掉时，自动选择一个最优的从节点切换为主节点。客户端来连接集群时，会首先连接sentinel，通过sentinel来查询主节点的地址，然后再去连接主节点进行数据交互。当主节点发生故障时，客户端会重新向sentinel要地址，sentinel会将最新的主节点地址告诉客户端。如此应用程序将无需重启即可自动完成节点切换。

### 2、Codis

在大数据高并发的需求下，Redis集群方案可以将众多小内存的Redis实例综合起来，将分布在多台机器上的CPU核心的计算能力聚集在一起，完成海量数据存储和高并发读写操作。

Codis便是Redis集群方案之一，它是一个代理中间件，和Redis一样使用Redis协议对外提供服务，当客户端向Codis发送指令时，Codis负责将指令转发到后面的Redis实例来执行，并将返回结果再转回给客户端。

Codis上挂接的所有Redis实例构成了一个Redis集群，当集群空间不足时，可以通过动态增加Redis实例在实现扩容需求



![](F:\Java书和视频\笔记\images\redis42.png)

#### 2.1 Codis分片原理

Codis要负责将特定的key转发的特定的Redis实例，这种对应关系如何关是如何管理的？

Codis将所有的key默认划分为1024个槽位，它首先对客户端传过来的key进行CRC32运算计算哈希值并对1024求模，这个值就是对应key的槽位。每个槽位都会唯一映射到后面的多个Redis实例之一，Codis在**内存维护槽位和Redis实例的映射关系**

#### 2.2 不同的Codis实例之间槽位关系如何同步

如果 Codis 的槽位映射关系只存储在内存里，那么不同的 Codis 实例之间的槽位关系就无法得到同步。所以 Codis 还需要一个分布式配置存储数据库专门用来持久化槽位关系。Codis 开始使用 ZooKeeper，后来连 etcd 也一块支持了。

Codis 将槽位关系存储在 zk 中，并且提供了一个 Dashboard 可以用来观察和修改槽位关系，当槽位关系变化时，Codis Proxy 会监听到变化并重新同步槽位关系，从而实现多个 Codis Proxy 之间共享相同的槽位关系配置。

![](F:\Java书和视频\笔记\images\redis43.png)

#### 2.3 扩容

刚开始 Codis 后端只有一个 Redis 实例，1024 个槽位全部指向同一个 Redis。然后一个 Redis 实例内存不够了，所以又加了一个 Redis 实例。这时候需要对槽位关系进行调整，将一半的槽位划分到新的节点。这意味着需要对这一半的槽位对应的所有 key 进行迁移，迁移到新的 Redis 实例。

那 Codis 如何找到槽位对应的所有 key 呢？

Codis 对 Redis 进行了改造，增加了 SLOTSSCAN 指令，可以遍历指定 slot 下所有的 key。Codis 通过 SLOTSSCAN 扫描出待迁移槽位的所有的 key，然后挨个迁移每个 key 到新的 Redis 节点。

在迁移过程中，Codis 还是会接收到新的请求打在当前正在迁移的槽位上，因为当前槽位的数据同时存在于新旧两个槽位中，Codis 如何判断该将请求转发到后面的哪个具体实例呢？

Codis 无法判定迁移过程中的 key 究竟在哪个实例中，所以它采用了另一种完全不同的思路。当 Codis 接收到位于正在迁移槽位中的 key 后，会立即强制对当前的单个 key 进行迁移，迁移完成后，再将请求转发到新的 Redis 实例。

### 3、Redis Cluster

相对于 Codis 的不同，它是去中心化的，如图所示，该集群有三个 Redis 节点组成，每个节点负责整个集群的一部分数据，每个节点负责的数据多少可能不一样。这三个节点相互连接组成一个对等的集群，它们之间通过一种特殊的二进制协议相互交互集群信息。

![](F:\Java书和视频\笔记\images\redis44.png)

Redis Cluster 将所有数据划分为 16384 的 slots，它比 Codis 的 1024 个槽划分的更为精细，每个节点负责其中一部分槽位。槽位的信息存储于每个节点中，它不像 Codis，它不需要另外的分布式存储来存储节点槽位信息。

当 Redis Cluster 的客户端来连接集群时，它也会得到一份集群的槽位配置信息。这样当客户端要查找某个 key 时，可以直接定位到目标节点。

## 六、Spring整合Redis（直接使用Redis，没有用作缓存）

### 1、过时版本的整合

1. 首先添加依赖项：

   ```xml
   <dependency>
       <groupId>org.springframework</groupId>
       <artifactId>spring-context</artifactId>
       <version>5.2.3.RELEASE</version>
   </dependency>
           <!-- https://mvnrepository.com/artifact/org.springframework.data/spring-data-redis -->
   <dependency>
    	<groupId>org.springframework.data</groupId>
       <artifactId>spring-data-redis</artifactId>
       <version>2.2.4.RELEASE</version>
   </dependency>
           <!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
   <dependency>
       <groupId>redis.clients</groupId>
       <artifactId>jedis</artifactId>
       <version>3.1.0</version>
   </dependency>
   <dependency>
       <groupId>org.springframework</groupId>
       <artifactId>spring-test</artifactId>
       <version>5.2.3.RELEASE</version>
   </dependency>
   <dependency>
       <groupId>junit</groupId>
       <artifactId>junit</artifactId>
       <version>4.12</version>
       <scope>test</scope>
   </dependency>
   ```

2. 在spring配置文件中配置bean

   ```xml
   <!--扫描路径-->
   <context:component-scan base-package="service"></context:component-scan>
   <!--设置redis配置文件的路径-->
   <bean id="annotationPropertyConfigurerRedis"  class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
       <property name="order" value="1"/>
       <property name="ignoreUnresolvablePlaceholders" value="true"/>
       <property name="locations">
           <list>
               <value>classpath:redis.properties</value>
           </list>
       </property>
   </bean>
   <!--已经不再使用这种方式了-->
   <bean id="jedisConnFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
       <property name="hostName" value="${redis.host}"/>
       <property name="port" value="${redis.port}"/>
       <property name="password" value="${redis.password}"/>
       <property name="timeout" value="${redis.timeout}"/>
   </bean>
   
   <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
       <property name="maxIdle" value="${redis.maxIdle}"/>
       <property name="maxWaitMillis" value="${redis.maxWaitMillis}"/>
       <property name="maxTotal" value="${redis.maxTotal}"/>
       <property name="testOnBorrow" value="${redis.testOnBorrow}"/>
   </bean>
   
   <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
       <property name="connectionFactory" ref="jedisConnFactory"/>
       <property name="keySerializer">
           <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
       </property>
       <property name="valueSerializer">
           <bean class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer"/>
       </property>
       <property name="hashKeySerializer">
           <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
       </property>
       <property name="hashValueSerializer">
           <bean class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer"/>
       </property>
       <property name="enableTransactionSupport" value="true"/>
   </bean>
   ```

3. redis配置文件：

   ```properties
   redis.maxTotal=10
   redis.maxIdle=5
   redis.maxWaitMillis=2000
   redis.testOnBorrow=true
   redis.host=127.0.0.1
   redis.port=6379
   redis.timeout=0
   redis.password=123456
   redis.testWhileIdle=true 
   redis.timeBetweenEvictionRunsMillis=30000  
   redis.numTestsPerEvictionRun=50 
   ```

4. 编写service

[service代码]: F:\Java书和视频\笔记\代码\service代码.md

5. 编写测试代码

   ```java
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration(locations = "classpath:application.xml")
   public class RedisServiceTest {
       @Autowired
       private RedisService service;
       
       @Test
       public void testRedis() {
           boolean set = service.set("key", "val");
           System.out.println(set);
           String key = (String) service.get("key");
           System.out.println(key);
           service.hset("myhashmap", "name", "wwww");
           
           System.out.println(service.hget("myhashmap", "name"));
       }
   }
   ```

### 2、JedisConnectionFactory的设置连接方法过时(Deprecated)的解决方案

在配置`JedisConnectionFactory`时，从`Spring Data Redis 2.0`开始就已经不推荐直接设置连接的信息了，一方面为了使配置与建立连接工厂解耦，另一方面抽象出`Standalone`，`Sentinel`和`RedisCluster`**三种模式的环境配置类**和**一个统一的 Jedis 客户端连接配置类**

#### 2.1 JavaConfig配置

##### 不使用连接池

这里仅仅以`standalone`配置为例，其他情况类似

1. 首先创建`RedisStandaloneConfiguration`
2. 然后根据该配置来初始化Jedis连接工厂

```java
@Configuration
@ComponentScan(basePackages = {"service"})
@PropertySource("classpath:redis.properties")
public class Appconfig {    
	@Value("${redis.host}")
    private String host;
    @Value("${redis.password}")
    private String password;
    @Value("${redis.port}")
    private String port;
    @Value("${redis.database}")
    private String database;
    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration =
                new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(host);
        redisStandaloneConfiguration.setPassword(password);
        redisStandaloneConfiguration.setDatabase(Integer.valueOf(database));
        redisStandaloneConfiguration.setPort(Integer.valueOf(port));
        logger.warning(database);
        return new JedisConnectionFactory(redisStandaloneConfiguration);
    }
	//具体使用的是这个类
    @Bean("template")
    public RedisTemplate redisTemplate(JedisConnectionFactory jedisConnectionFactory) {
        return new StringRedisTemplate(jedisConnectionFactory);
    }
}
使用方式和第一种配置方式一样。
```

##### ==连接池配置==

以上配置使用的是直接连接redis的方式，即每次连接都创建新的连接。当并发量剧增时，这会带来性能上的开销，同时由于没有对连接数进行限制，则可能使服务器崩溃导致无法响应。所以我们一般都会建立连接池，事先初始化一组连接， 供需要redis服务的线程使用，首先我们定义连接池配置信息：

```java
@Configuration
@ComponentScan(basePackages = {"service"})
@PropertySource("classpath:redis.properties")
public class Appconfig {
	@Value("${redis.maxTotal}")
    private String maxTotal;
    @Value(("${redis.maxIdle}"))
    private String maxIdle;
    @Value("${redis.maxWaitMillis}")
    private String maxWaitMillis;
    //连接池配置
    @Bean
    public JedisPoolConfig jedisPoolConfig() {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        //最大空闲连接数
        jedisPoolConfig.setMaxIdle(Integer.valueOf(maxIdle));
        //最大连接数
        jedisPoolConfig.setMaxTotal(Integer.valueOf(maxTotal));
        //当池内没有可用连接时，最大等待时间
        jedisPoolConfig.setMaxWaitMillis(Integer.valueOf(maxWaitMillis));
        //...还可以配置其他属性
        return jedisPoolConfig;
    }
```

接下来配置连接工厂类。

连接工厂类`JedisConnectionFactory`对于Standalone模式没有提供参数为`(RedisStandaloneConfiguration, JedisPoolConfig)`的构造函数。虽然`JedisConnectionFactory`有一个内部`JedisClientConfiguration`的实现类，但是访问权限仅限包内，我们无法使用，它的作用也进京作为默认的客户端连接配置。

若想配置Standalone带有连接池配置的连接工厂类就比较麻烦，因为我们通常要自定义配置，但是`JedisClientConfiguration`并没有停工带有参数为`JedisPoolConfig`的方法，它内部主张使用构造器来构建。而且构造器是采用仅包内访问的内部类的形式，也就是意味着无法在外部访问这个内部类，所有智能联通过该接口的`builder()`方法实例化出一个构造器，同时这个构造器会自带一个默认的连接池的配置，所以我们要替换成自己的配置类。但是，返回的对象的静态类型为`JedisClientConfigurationBuilder`，它没有修改配置的接口方法，所以此处还得转型为`JedisPoolingClientConfigurationBuilder`，然后调用其poolConfig即可替换为我们需要的配置。

```java
@Bean
public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory) {
    return new StringRedisTemplate(redisConnectionFactory);
}
//Jedis连接工厂
@Bean
public RedisConnectionFactory redisConnectionFactory(JedisPoolConfig jedisPoolConfig) {
    //单机版jedis
    RedisStandaloneConfiguration redisStandaloneConfiguration = 
        new RedisStandaloneConfiguration();
    //设置redis服务器的host或者ip地址
    redisStandaloneConfiguration.setHostName(host);
    //设置密码
    redisStandaloneConfiguration.setPassword(password);
    //设置默认使用的数据库
    redisStandaloneConfiguration.setDatabase(Integer.valueOf(database));
    //设置redis的服务端口号
    redisStandaloneConfiguration.setPort(Integer.valueOf(port));
    //获得默认的连接池构造器
    JedisClientConfiguration.JedisPoolingClientConfigurationBuilder jpcb = 
        (JedisClientConfiguration.JedisPoolingClientConfigurationBuilder) JedisClientConfiguration.builder();
    //指定jedisPoolConfig来修改默认的连接池构造器
    jpcb.poolConfig(jedisPoolConfig);
    //通过构造器来构造jedis客户端配置
    JedisClientConfiguration jedisClientConfiguration = jpcb.build();
    //JedisConnectionFactory实现了RedisConnectionFactory
    return new JedisConnectionFactory(redisStandaloneConfiguration, jedisClientConfiguration);
}

```

很显然这里利用==策略模式==来使得连接工厂可以根据不同的方案来创建相应的连接，创建连接过程与配置连接信息完全解耦，方便以后拓展jedis更多的配置信息。	

### 3. 将Redis作为缓存

除了以上配置将Redis引入外，要将Redis作为缓存，还需要手动配置cacheManager。

## 七、SpringBoot整合Redis

### 1、搭建基本环境

1. 使用Idea中的Spring Initializr来创建Spring Boot工程，选中Web、MySQL和MyBatis

2. 导入数据库文件，创建出department和employee表

   [文件位置]: F:\Java书和视频\SpringBoot视频\springboot核心篇+整合篇-尚硅谷\02尚硅谷SpringBoot整合篇\课件\文档\cache实验

   ![](F:\Java书和视频\笔记\images\SpringBoot_Redis\pic1.png)

3. 创建JavaBean封装数据

4. 整合MyBatis操作数据库

   1. 配置数据源信息（在application.properties中加上如下配置）

      ```propertoes
      spring.datasource.url=jdbc:mysql://localhost:3306/spring_cache
      spring.datasource.username=root
      spring.datasource.password=19960421wyj
      spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
      ```

   2. 使用注解版的MyBatis

      1. @MapperScan指定需要扫描的mapper接口所在的包

         ```java
         @SpringBootApplication
         @MapperScan("com.wyjie.springboot.mapper")
         public class SpringBoot01CacheApplication {
         
             public static void main(String[] args) {
                 SpringApplication.run(SpringBoot01CacheApplication.class, args);
             }
         
         }
         
         ```

      2. 创建Mapper接口，在接口上标注`@Mapper`

         ```java
         @Mapper
         public interface EmployeeMapper {
             
             @Select("select * from employee where id=#{id}")
             public Employee getEmpById(Integer id); 
             
             @Update("update employee set lastName=#{lastName}," +
                     "email=#{email}, gender=#{gender}, d_id=#{dId}" +
                     "where id=#{id}")
             public void updateEmp(Employee employee);
             
             @Delete("delete from employee where id=#{id}")
             public void deleteEmp(Integer id);
             
             @Insert("insert into employee(lastName, email, gender, d_id)" +
                     "values(#{lastName},#{email},#{gender},#{dId})")
             public void insertEmp(Employee employee);
         }
         ```

      3. 创建Service类和Controller类，测试是否能从数据库中获取数据

### 2、 快速体验缓存（未使用Redis）

步骤：

1. 开启基于注解的缓存：`@EnableCaching`

   ```java
   @SpringBootApplication
   @MapperScan("com.wyjie.springboot.mapper")
   @EnableCaching
   public class SpringBoot01CacheApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(SpringBoot01CacheApplication.class, args);
       }
   
   }
   ```

2. 标注缓存注解：

   `@cacheable` ：将方法的运行结果进行缓存，以后再要相同的数据，则从缓存中获取，不用调用方法

   **属性**：

   * CacheManager用于管理多个缓存组件，对缓存的真正CRUD操作，在缓存组件中，每一个缓存组件有自己唯一的一个名字

   * cacheNames/value：指定缓存的名字

   * key：缓存数据时使用的key；可以用它来指定，默认是使用方法参数的值，可写EL表达

     * #id：参数id的值，也可以写为#a0, #p0, #root.args[0]

   * keyGenerator：key的生成器；可以自己指定key的生成器的组件id

     ​			**key和keyGenerator只写其中一个**

   * cacheManager：指定缓存管理器

   * condition：指定符合条件的情况下才缓存

   * unless：当unless指定的条件为true，方法的返回值不缓存。与condition相反
   * sync：缓存是否使用异步模式

   ![image-20200314161306839](C:\Users\wyj\AppData\Roaming\Typora\typora-user-images\image-20200314161306839.png)

   

如果不使用缓存，那么每一次请求都会去查询数据库

![image-20200314155616327](C:\Users\wyj\AppData\Roaming\Typora\typora-user-images\image-20200314155616327.png)

使用缓存的代码如下：

```java
@Service
public class EmployeeService {
    
    @Autowired
    EmployeeMapper employeeMapper;
    
    @Cacheable(cacheNames = "emp", key="#id", condition="#id>0")
    public Employee getEmp(Integer id) {
        System.out.println("查询"+id+"号员工");
        Employee empById = employeeMapper.getEmpById(id);
        return empById;
    }
}
```

`@cacheEvict`

缓存清除。key用来指定要清除的数据

`@cachePut`

既调用方法，又更新缓存数据。修改了数据库中的某个数据，同时更新缓存 

**运行时机**

1. 先调用目标方法
2. 将目标方法的结果缓存起来

#### ==缓存的工作原理==

1. 自动配置类：`CacheAutoConfiguration`

2. 缓存的配置类`@Import({CacheAutoConfiguration.CacheConfigurationImportSelector.class, CacheAutoConfiguration.CacheManagerEntityManagerFactoryDependsOnPostProcessor.class})`

   ```java
   org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration
   org.springframework.boot.autoconfigure.cache.JCacheCacheConfiguration
   org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration
   org.springframework.boot.autoconfigure.cache.HazelcastCacheConfiguration
   org.springframework.boot.autoconfigure.cache.InfinispanCacheConfiguration
   org.springframework.boot.autoconfigure.cache.CouchbaseCacheConfiguration
   org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration
   org.springframework.boot.autoconfigure.cache.CaffeineCacheConfiguration
   org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration
   org.springframework.boot.autoconfigure.cache.NoOpCacheConfiguration
   ```

3. 哪个配置类自动生效

   ```java
   @ConditionalOnBean({Cache.class})
   @ConditionalOnMissingBean({CacheManager.class})
   @Conditional({CacheCondition.class})
   ```

   默认生效`SimpleCacheConfiguration`

   ```java
   @ConditionalOnMissingBean({CacheManager.class})
   @Conditional({CacheCondition.class})
   class SimpleCacheConfiguration {
       SimpleCacheConfiguration() {
       }
   
       @Bean
       ConcurrentMapCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers) {
           ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
           List<String> cacheNames = cacheProperties.getCacheNames();
           if (!cacheNames.isEmpty()) {
               cacheManager.setCacheNames(cacheNames);
           }
   
           return (ConcurrentMapCacheManager)cacheManagerCustomizers.customize(cacheManager);
       }
   }
   ```

   给容器中注册了一个ConcurrentMapCacheManager，它的作用是将数据保存在ConcurrentMap中。而在实际开发中，经常使用的是缓存中间件：redis、memcache、ehcache。

   **运行流程**

   ```java
   @Cacheable:
   1、方法运行之前，先去查询Cache（缓存组件），按照cacheNames指定的名字获取
   	（CacheManager先获取相应的缓存），第一次获取缓存如果没有Cache组件会自动创建
   2、去Cache中查找缓存的内容，使用一个key，默认是方法的参数值
       key是按某种策略生成的，默认是使用keyGenerator生成，默认使用SimpleKeyGenerator生成key
       生成策略为：
       	如果没有参数：key=new SimpleKey()
       	如果有一个参数：key=参数的值
       	如果有多个参数：key = new SimpleKey(params)
   3、没有查到缓存就调用目标方法
   4、将目标方法返回的结果放入缓存
   ```

   **==核心==**

   1. 使用CacheManager按照名字来得到Cache组件
   2. key是使用keyGenerator生成的，默认是SimpleKeyGenerator()

### 3. 整合Redis作为缓存

1. 引入Redis的starter

   当引入了redis相关的场景过后，RedisAutoConfiguration就起作用了，自动配置类配置了连接工厂RedisConnectionFactory和RedisTemplate。而使用Spring时，这两个类是需要我们自己配置的。

2. 配置redis（==只用在application.properties配置文件中添加相关配置，与Spring整合Redis比较学习==）

   ```properties
   spring.redis.host=192.168.0.105
   spring.redis.port=6379
   spring.redis.password=123456
   ```

3. 测试Redis（现在仅仅是使用Redis作为数据库）

   RedisAutoConfiguration类给我们提供了RedisTemplate（存储对象）和StringRedisTemplate（存储字符串）。

   在保存对象的时候需要对象是**可序列化的**，默认使用的序列化器是JDK默认的序列化器，在存储对象的时候会存一些我们看不懂的字符。

   ![image-20200315095620553](C:\Users\wyj\AppData\Roaming\Typora\typora-user-images\image-20200315095620553.png)

   我们可以自己写一个序列化器加入到IOC容器中，将对象转为json再存储（==在Spring中配置RedisTemplate的时候自己设置序列化器==）

   ```java
   @Bean
   public RedisTemplate<Object, Employee> empRedisTemplate(
       RedisConnectionFactory redisConnectionFactory
   ) {
       RedisTemplate<Object, Employee> template = new RedisTemplate<>();
       template.setConnectionFactory(redisConnectionFactory);
       template.setDefaultSerializer(new Jackson2JsonRedisSerializer<Employee>(Employee.class));
       return template;
   }
   ```

   ![image-20200315095818181](C:\Users\wyj\AppData\Roaming\Typora\typora-user-images\image-20200315095818181.png)

### 4. 测试缓存（将Redis作为缓存）

1. 引入redis的starter以后，容器中保存的是RedisCacheManager

2. RedisCacheManager帮我们创建RedisCache作为缓存组件；RedisCache通过操作Redis来缓存数据。==所以，只要添加了redis的starter，那么就默认使用的是Redis作为缓存了==。而且默认保存数据时，如果k-v为对象，是利用序列化保存的

   ![image-20200315101039389](C:\Users\wyj\AppData\Roaming\Typora\typora-user-images\image-20200315101039389.png)
   1. 引入的redis的starter，cacheManager变为了RedisCacheManager

   2. 默认创建RedisCacheManager操作redis的时候使用的是RedisTemplate<Object,Object>

   3. RedisTemplate<Object,Object>是默认使用jdk的序列化机制，所以保存的数据我们看不懂

   4. ==我们需要自定义CacheManager==

      ```java
      @Bean
      public RedisCacheManager employeeCacheManager(RedisConnectionFactory redisConnectionFactory) {
          RedisSerializer<String> redisSerializer = new StringRedisSerializer();
          Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = 
              new Jackson2JsonRedisSerializer(Object.class);
      
          RedisCacheConfiguration configuration = RedisCacheConfiguration.defaultCacheConfig()
              .entryTtl(Duration.ZERO)
              .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer))
              .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
              .disableCachingNullValues();
          RedisCacheManager redisCacheManager = RedisCacheManager.builder(redisConnectionFactory)
              .cacheDefaults(configuration)
              .build();
          return redisCacheManager;
      }
      ```

      

## Redis可以做什么（Redis实战）

### 使用Redis构建web应用
1、登录和cookie缓存
2、使用redis实现购物车
3、网页缓存
4、数据行缓存
5、网页分析

### 使用Redis构建支持程序
1、记录日志
2、计数器和统计数据
3、查找IP所述城市以及国家
4、服务的发现与配置

### 使用Redis构建应用程序组件
1、自动补全
2、分布式锁
3、计数信号量
4、任务队列
5、消息拉取
6、文件分发

### 基于搜索的应用程序
1、使用redis进行搜索（有序搜搜）
2、广告定向
3、职位走索