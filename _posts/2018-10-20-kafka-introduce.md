---
layout:     post                    # 使用的布局（不需要改）
title:      消息队列            # 标题
subtitle:   Kafka          #副标题
date:       2018-10-20              # 时间
author:     tanjiquan               # 作者
header-img: img/in-post/kafka-bg.png    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 消息队列
---

## (一）消息队列作用
    1.1 解耦
    1.2 峰值处理、缓冲
    1.3 异步通信
    1.4 扩展性
    1.5 顺序保证
 
## (二）常用消息队列
RabbitMQ:分布式,支持多种MQ协议,重量级（）
<br> ***ActiveMQ***:与RabbitMQ类似（ActiveMQ 一般用作在web中，将request请求发到ActiveMQ中，再webserver再将request请求拿出来进行处理，这个请求并不高）
<br> ***ZeroMQ***:以库的形式提供,使用复杂,无持久化
<br> ***redis***:纯内存性好,持久化较差
<br> ***Kestrel***:单机,持久化
<br> ***kafka***:分布式,较长时间持久化,高性能,轻量灵活
<br>
<br>&emsp;&emsp;  ***RabbitMQ***也是常见的消息对列,它支持多种MQ的协议,jms啊,等多种协议等等, 它的缺点比较重,erlang，server段维护消费状态，比较复杂.另外一个ActiveMQ也和RabbitMQ类似,支持的协议比较多
<br>&emsp;&emsp;  ***ZeroMQ***是一个socket的通信库,它是以库的形式提供的,所以说你需要写程序来 实现消息系统,它只管内存和通信那一块,持久化也得自己写,还是那句话它是用来实现消息队列的一个库,
其实在storm里面呢,storm0.9之前,那些spout和bolt, bolt和bolt之间那些底层的通信就是由ZeroMQ来通信的,它并不是一个消息队列, 就是一个通信库,在0.9之后呢,因为license的原因,
ZeroMQ就由Netty取代了, Netty本身就是一个网络通信库嘛,所以说更合适是在通信库这一层,不应该是 MessageQueue这一层
<br>&emsp;&emsp;  ***Redis***,本身是一个内存的KV系统,但是它也有队列的一些数据结构,能够实现一 些消息队列的功能,当然它在单机纯内存的情况下,性能会比较好,持久化做的稍差,当持久化的时候性能下降的会比较厉害
<br>&emsp;&emsp;  ***Kestrel***,这个也是在storm里面可以配合来使用的消息队列,是twitter开源的消息队列,也是单机,它提供的持久化是以日志的情况来写的,如果用的时候,你需要 自己来搭建几台Kestrel的服务,自己处理负载均衡,自己去做容错
<br>&emsp;&emsp;  ***Kafka***,的亮点,天生是分布式的,不需要你在上层做分布式的工作,另外有较长时间持久化,前面基本消费就干掉了,另外在长时间持久化下性能还比较高,顺序读和顺序写,
另外还通过sendFile这样0拷贝的技术直接从文件拷贝到网络,减少内存的拷贝,还有批量。Linux中的sendfile()以及Java NIO中 的FileChannel.transferTo()方法都实现了零拷贝的功能,而在Netty中也通过在 
FileRegion中包装了NIO的FileChannel.transferTo()方法实现了零拷贝。

## (三）Kafka介绍
kafka 是一个高吞吐的分布式消息系统，是一种发布订阅的消息系统。
<br>版本说明，kafka_2.12-0.11.0.3.tgz   其中2.12 表示scala版本，0.11.0.3 表示kafka版本。
<br>***整体架构如下图***
![avatar](https://ws1.sinaimg.cn/large/006tNbRwly1fwm39mwgwtj31kw0u57am.jpg)

### 3.1 Kafka基本术语
**zookeeper**: 管理broker集群，管理元数据。
<br>**producer**: 消息生产者。
<br> **consumer**: 消息消费者，向broker发送消息。
<br> **consumerGroup**: 每个 Consumer 具有一个 group id 用于标记其消费者组。具有相同 group id 的 Consumer，共同消费某个（或某些）topic。
<br> **broker**: kafka集群的server， 负责处理消息读、写请求，存储消息，可以由一个或多个服务器组成，每个服务器叫做一个broker。
<br> **topic**: 消息队列主题/分类，一个topic分成多个partition（一个topic的partition分布在多台机器中），一个partition只能对应一个broker，一个broker管理多个partition。
<br> **partition**: 一个topic中的消息数据按照多个分区组织，分区是kafka是消息队列组织的最小单位，一个分区可以看做是一个队列。副本也是一partition为单位。副本之间以复制为关系关联。
<br>（1）为了实现负载均衡，Kafka尽量将所有的Partition均匀分配到整个集群上。一个典型的部署方式是一个Topic的Partition数量大于Broker的数量。
<br>（2）为了实现高可用，kafka可以配置配个分区的副本，副本存储在不同的broker上。但是副本数量必须小于等于broker的数量，并且副本数量不能为0。
<br> **offset**：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息.
某一个消费组，当前对于某一个topic的某一个分区的消费偏移量。
<br> **partition leader**: 每个partition有多个副本，其中有且仅有一个作为Leader，Leader是当前负责数据的读写的partition。
<br> **partition follower**: Follower跟随Leader，所有写请求都通过Leader路由，数据变更会广播给所有Follower，Follower与Leader保持数据同步。
如果Leader失效，则从Follower中选举出一个新的Leader。当Follower与Leader挂掉、卡住或者同步太慢，leader会把这个follower从“in sync replicas”（ISR）列表中删除，重新创建一个Follower。
![avatar](https://ws3.sinaimg.cn/large/006tNbRwly1fwg8kekp7nj31jc0sazsv.jpg)
  ***图解***
<br> **a.强有序**: 对于一个topic，右边的生产者partition写的顺序和左边消费者读的顺序相同。
对于partion内是强有序的，多个partition组成一个topic，每个partition可能在不同的机器上。所以一个topic不能保证强有序。partition在多机上，消费者不保证顺序读取。	
<br> **b.负载策略**: producer写入，写入时key-value策略，若key 为空就是负载均衡策略，若不为空则是hash策略
<br> **c.序列**: 每个消息都对应一个序列号，一个topic就是对一组消息的归纳。
<br> **d. kafka是怎么生产消息,消费消息,和怎么存储消息的**
（1）kafka里面的消息是有topic来组织的,简单的我们可以想象为一个队列,一个队列 就是一个topic,然后它把每个topic又分为很多个partition,这个是为了做并行的, 
在每个partition里面是有序的,相当于有序的队列,其中每个消息都有个序号（offset）,比 如0到12,从前面读往后面写,
（2）一个partition对应一个broker,一个broker可以管多个partition,比如说,topic 有6个partition,有两个broker,那每个broker就管3个partition
（3）这个partition可以很简单想象为一个文件,当数据发过来的时候它就往这个 partition上面append,追加就行,kafka和很多消息系统不一样,很多消息系统是 消费完了我就把它删掉,
而kafka是根据时间策略删除,而不是消费完就删除,在 kafka里面没有一个消费完这么个概念,只有过期这样一个概念,这个模型带来了很多个好处。
（4）这里producer自己决定往哪个partition里面去写,这里有一些的策略,譬如如果 hash就不用多个partition之间去join数据了

### 3.2 Kafka特性
生产者消费者模式
<br> **a.** 先进先出(FIFO)顺序保证，但不是强保证
<br> **b.** 可靠性保证– 自己不丢数据 – 消费者不丢数据:“至少一次,严格一次”，至少一次就是可能会有两次,会重，严格一次机制就会负责一点
<br> **c.** 吞吐量高德原因：顺序读写，可以水平扩展
<br> **d.** 消息可以保存（可配置保存天数）
<br> **e.** 在发送一条消息时，也可以指定这条消息的key，producer根据这个key个partition机制来判断将这条消息发送到那个partition
<br> **f.** 一般情况下partition的数量大于等于broker的数量，并且所有partition的leader均匀分配在broker上

### 3.3 合适的[partition数量](http://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)
&emsp;&emsp;越多的分区可以提供更高的吞吐量，但是需要打开更多的文件句柄，导致更高的不可用性，可能增加端对端的延迟，客户端需要更多的内存。
<br>&emsp;&emsp;  **a.** 在kafka中，单个partition是kafka并行操作的最小单元。在producer和broker端，向每一个分区写入数据是可以完全并行化的，
可以通过加大硬件资源的利用率来提升系统的吞吐量，例如对数据进行压缩。在consumer段，kafka只允许单个partition的数据被一个consumer线程消费。
因此，在consumer端，每一个Consumer Group内部的consumer并行度完全依赖于被消费的分区数量。
<br>&emsp;&emsp;  **b.**  同一个topic 的一个partition消息，只能被consumerGroup 的一个consumer 消费，如果 consumer大于partition的数量，则有一部分consumer 不能消费消息
综上所述，通常情况下，在一个Kafka集群中，partition的数量越多，意味着可以到达的吞吐量越大。
<br>&emsp;&emsp;  **c.**  Kafka是如何分区，（1）kafka使用默认的Partitioner: hash(key)%numPartitions,这样每次的partition num都是一样的，所以数据都落到一个分区了。
而同一consumerGroup并行消息，也是按照分区来分配的，因为只有一个分区上有数据，所以有一个consumer始终拿不到消息。
（2）Kafka会根据传进来的key计算其分区ID。但是这个Key可以不传，根据Kafka的官方文档描述：如果key为null，那么Producer将会把这条消息发送给随机的一个Partition。
在key为null的情况下，Kafka并不是每条消息都随机选择一个Partition；而是每隔topic.metadata.refresh.interval.ms才会随机选择一次！[详情请见](https://www.iteblog.com/archives/1619.html)。
（3）当producer向kafka写入基于key的消息时，kafka通过key的hash值来确定消息需要写入哪个具体的分区。通过这样的方案，kafka能够确保相同key值的数据可以写入同一个partition。
（4）kafka的这一能力对于一部分应用是极为重要的，例如对于同一个key的所有消息，consumer需要按消息的顺序进行有序消费。如果partition的数量发生改变，
那么上面的有序性保证将不复存在。为了避免上述情况发生，通常的解决办法是多分配一些分区，以满足未来的需求。通常情况下，我们需要根据未来1到2年的目标吞吐量来设计kafka的分区数量。
（5）分区解决办法：1.自定义分区函数。2.消息散列为不同的key 
<br>&emsp;&emsp;  **d.**  越多的分区可能增加端对端的延迟: （1）Kafka端对端延迟定义为producer端发布消息到consumer端接收消息所需要的时间。
即consumer接收消息的时间减去producer发布消息的时间。（2）Kafka只有在消息提交之后，才会将消息暴露给消费者。
例如，消息在所有in-sync副本列表同步复制完成之后才暴露。因此，in-sync副本复制所花时间将是kafka端对端延迟的最主要部分。
（3）在默认情况下，每个broker从其他broker节点进行数据副本复制时，该broker节点只会为此工作分配一个线程，该线程需要完成该broker所有partition数据的复制。
（4）经验显示，将1000个partition从一个broker到另一个broker所带来的时间延迟约为20ms，这意味着端对端的延迟至少是20ms。
这样的延迟对于一些实时应用需求来说显得过长。注意，上述问题可以通过增大kafka集群来进行缓解。
（5）例如，将1000个分区leader放到一个broker节点和放到10个broker节点，他们之间的延迟是存在差异的。
在10个broker节点的集群中，每个broker节点平均需要处理100个分区的数据复制。此时，端对端的延迟将会从原来的数十毫秒变为仅仅需要几毫秒。
<br>&emsp;&emsp;  **e.**  越多的partition意味着需要客户端需要更多的内存。
（1）在最新发布的0.8.2版本的kafka中，我们开发了一个更加高效的Java producer。新版producer拥有一个比较好的特征，他允许用户为待接入消息存储空间设置内存大小上限。
在内部实现层面，producer按照每一个partition来缓存消息。在数据积累到一定大小或者足够的时间时，积累的消息将会从缓存中移除并发往broker节点。
（2）如果partition的数量增加，消息将会在producer端按更多的partition进行积累。众多的partition所消耗的内存汇集起来，有可能会超过设置的内容大小限制。
当这种情况发生时，producer必须通过消息堵塞或者丢失一些新消息的方式解决上述问题，但是这两种做法都不理想。为了避免这种情况发生，我们必须重新将produder的内存设置得更大一些。
（3）根据经验，为了达到较好的吞吐量，我们必须在producer端为每个分区分配至少几十KB的内存，并且在分区数量显著增加时调整可以使用的内存数量。
（4）类似的事情对于consumer端依然有效。Consumer端每次从kafka按每个分区取出一批消息进行消费。消费的分区数越多，需要的内存数量越大。尽管如此，上述方式主要运用于非实时的应用场景。
<br>&emsp;&emsp;  **f.**  通常情况下，kafka集群中越多的partition会带来越高的吞吐量。但是，我们必须意识到集群的partition总量过大或者单个broker节点partition过多，都会对系统的可用性和消息延迟带来潜在的影响。
### 3.4 kafka producer
#### 3.4.1 producer选择分区
&emsp;&emsp;  producer 将消息发布到它指定的topic中，并负责决定发布到哪个分区。通常简单的由负载均衡机制随机选择分区，但也可以通过特定的分区函数（key值）选择分区。一般第二种比较常见。
#### 3.4.2 Sync Producer & Async Producer
&emsp;&emsp; Sync 同步模式：生产者生成消息时，只有等消费者者消费了消息才能继续生产。
<br>&emsp;&emsp; Async 异步模式：生产者不感应消费者，生产者只负责不断生产，消费者只负责不断消费。
### 3.5 kafka consumer/consumerGroup 
每一个consumer实例都属于一个consumerGroup,每一条消息只会被同一个consumerGroup里的一个consumer实例消费，不同的consumerGroup可以同时消费同一天消息。如下图：
![avatar](https://ws3.sinaimg.cn/large/006tNbRwly1fwikxgacxmj30vg0j4gog.jpg)
![avatar](https://ws2.sinaimg.cn/large/006tNbRwly1fwilf9ou0ej30g2086dgl.jpg)
### 3.6 kafka push/pull
&emsp;&emsp; 作为一个messaging system，Kafka遵循了传统的方式，选择由producer向broker push消息并由consumer从broker pull消息。
一些logging-centric system，比如Facebook的Scribe和Cloudera的Flume,采用非常不同的push模式。事实上，push模式和pull模式各有优劣。
<br>&emsp;&emsp; push模式很难适应消费速率不同的消费者，因为消息发送速率是由broker决定的。push模式的目标是尽可能以最快速度传递消息，
但是这样很容易造成consumer来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而pull模式则可以根据consumer的消费能力以适当的速率消费消息。
### 3.7 kafka 高可用保证
&emsp;&emsp; **a.** zookeeper
<br>&emsp;&emsp; **b.** replication & leader election -->failover 机制
#### 3.7.1 replication 副本机制
$KAFKA_HOME/config/server.properties ==> default.replication.factor = 1
<br>&emsp;&emsp; ***Replication***:该 Replication与leader election配合提供了自动的failover机制。replication对Kafka的吞吐率是有一定影响的，但极大的增强了可用性。
默认情况下，Kafka的replication数量为1，并且replication小于等于broker的数量。每个partition都有一个唯一的leader，所有的读写操作都在leader上完成，follower 批量从leader上pull数据。
<br>&emsp;&emsp; ***副本***：一般情况下partition的数量大于等于broker的数量，并且所有partition的leader均匀分布在broker上。follower上的日志和其leader上的完全一样。
Kafka允许topic的分区拥有若干副本，这个数量是可以配置的，你可以为每个 topic 配置副本的数量。Kafka会自动在每个副本上备份数据，所以当一个节点down掉时数据依然是可用的。 
<br>&emsp;&emsp; Kafka的副本功能不是必须的，你可以配置只有一个副本，这样其实就相当于只有一份数据。 
<br>&emsp;&emsp; ***leader***：创建副本的单位是topic的分区，每个分区都有一个leader和零或多个followers.所有的读写操作都由leader处理，一般分区的数量都比broker的数量多的多，
各分区的leader均匀的分布在brokers中。所有的followers都复制leader的日志，日志中的消息和顺序都和leader中的一致。flowers向普通的consumer那样从leader那里拉取消息并保存在自己的日志文件中。 
![avatar](https://ws3.sinaimg.cn/large/006tNbRwly1fwm0821ixcj30v80e8dht.jpg)
图注:
<br>Kafka 每个topic的partition有N个副本，其中N是topic的复制因子。Kafka通过多副本机制实现故障转移，当kafka中的一个broker失效时仍然保证服务可用。
在kafka中发生复制时确保partition的预写日志有序的写到其他节点，其他都为follower，leader处理partition所有读写请求，与此同时，follower会定期去复制leader上的数据。
leader宕机，flower中的一台则会自动成为leader。集群中的每个服务（broker）都会同时扮演两个角色，作为它所持有的一部分分区的leader，同时作为其他分区的flowers，这样集群就会具有较好的负载均衡。
图中，红色方块就是leader，绿色就是follower。
<br>&emsp;&emsp; ***心跳***：许多分布式的消息系统自动的处理失败的请求，它们对一个节点是否存活着（alive）”有着清晰的定义。Kafka判断一个节点是否活着有两个条件：
（1）节点必须可以维护和ZooKeeper的连接，Zookeeper通过心跳机制检查每个节点的连接。 
（2）如果节点是个follower,他必须能及时的同步leader的写操作，延时不能太久。 
<br>&emsp;&emsp; ***同步***：Leader会追踪所有“同步中”的节点，一旦一个down掉了，或是卡住了，或是延时太久，leader就会把它移除。
至于延时多久算是“太久”，是由参数replica.lag.max.messages决定的，怎样算是卡住了，是由参数replica.lag.time.max.ms决定的。 
leader会track “in sync”的node list（ISR）。如果一个follower宕机，或者落后太多，leader将把它从”in sync” list中移除。
这里所描述的“落后太多”指follower复制的消息落后于leader后的条数超过预定值，该在$KAFKA_HOME/config/server.properties中配置
replica.lag.max.messages=4000 、replica.lag.time.max.ms=10000
<br>&emsp;&emsp; ***同步/异步复制***：一条消息只有被“in sync” list里的所有follower都从leader复制过去才会被认为已提交。
这样就避免了部分数据被写进了leader，还没来得及被任何follower复制就宕机了，而造成数据丢失（consumer无法消费这些数据）。
而对于producer而言，它可以选择是否等待消息commit，这可以通过request.required.acks来设置。
这种机制确保了只要“in sync” list有一个或以上的 follower，一条被commit的消息就不会丢失。
<br>&emsp;&emsp; 这里的复制机制即不是同步复制，也不是单纯的异步复制。
事实上，同步复制要求“活着的”follower都复制完，这条消息才会被认为commit，这种复制方式极大的影响了吞吐率（高吞吐率是Kafka非常重要的一个特性）。
而异步复制方式下，follower异步的从leader复制数据，数据只要被leader写入log就被认为已经commit，这种情况下如果follower都落后于leader，而leader突然宕机，则会丢失数据。
而Kafka的这种使用“in sync” list的方式则很好的均衡了确保数据不丢失以及吞吐率。follower可以批量的从leader复制数据，
这样极大的提高复制性能（批量写磁盘），极大减少了follower与leader的差距（前文有说到，只要follower落后leader不太远，则被认为在“in sync” list里）。
<br>&emsp;&emsp; ***负载均衡***：Kafka尽量将所有的Partition均匀分配到整个集群上。一个典型的部署方式是一个Topic的Partition数量大于Broker的数量。
同时为了提高Kafka的容错能力，也需要将同一个Partition的Replica尽量分散到不同的机器。实际上，如果所有的Replica都在同一个Broker上，那一旦该Broker宕机，
该Partition的所有Replica都无法工作，也就达不到HA的效果。同时，如果某个Broker宕机了，需要保证它上面的负载可以被均匀的分配到其它幸存的所有Broker上。
#### 3.7.2 leader election机制
&emsp;&emsp;  上文中，副本机制能保证一个broker挂掉后kafka依旧能正常使用，但是当leader宕机后，kafka如何正常使用，在kafka也有leader election机制，来保证leader。
<br> &emsp;&emsp;  因为follower可能落后许多或者crash了，所以必须确保选择“最新”的follower作为新的leader。一个基本的原则就是，如果leader不在了，
新的leader必须拥有原来的leader commit的所有消息。这就需要作一个折衷，如果leader在标明一条消息被commit前等待更多的follower确认，
那在它die之后就有更多的follower可以作为新的leader，但这也会造成吞吐率的下降。
<br>&emsp;&emsp;  一种非常常用的选举leader的方式是“majority vote”（“少数服从多数”），但Kafka并未采用这种方式。这种模式下，如果我们有2f+1个replica（包含leader和follower），
那在commit之前必须保证有f+1个replica复制完消息，为了保证正确选出新的leader，fail的replica不能超过f个。因为在剩下的任意f+1个replica里，
至少有一个replica包含有最新的所有消息。这种方式有个很大的优势，系统的latency只取决于最快的几台server，也就是说，如果replication factor是3，
那latency就取决于最快的那个follower而非最慢那个。majority vote也有一些劣势，为了保证leader election的正常进行，它所能容忍的fail的follower个数比较少。
如果要容忍1个follower挂掉，必须要有3个以上的replica，如果要容忍2个follower挂掉，必须要有5个以上的replica。也就是说，在生产环境下为了保证较高的容错程度，
必须要有大量的replica，而大量的replica又会在大数据量下导致性能的急剧下降。这就是这种算法更多用在Zookeeper这种共享集群配置的系统中而很少在需要存储大量数据的系统中使用的原因。
例如HDFS的HA feature是基于majority-vote-based journal，但是它的数据存储并没有使用这种expensive的方式。
<br>&emsp;&emsp;  实际上，leader election算法非常多，比如Zookeper的Zab, Raft和Viewstamped Replication。
而Kafka所使用的leader election算法更像微软的PacificA算法。
<br>&emsp;&emsp;  Kafka在Zookeeper中动态维护了一个ISR（in-sync replicas） set，这个set里的所有replica都跟上了leader，
只有ISR里的成员才有被选为leader的可能。在这种模式下，对于f+1个replica，一个Kafka topic能在保证不丢失已经ommit的消息的前提下容忍f个replica的失败。
在大多数使用场景中，这种模式是非常有利的。事实上，为了容忍f个replica的失败，majority vote和ISR在commit前需要等待的replica数量是一样的，
但是ISR需要的总的replica的个数几乎是majority vote的一半
<br>&emsp;&emsp;  虽然majority vote与ISR相比有不需等待最慢的server这一优势，但是Kafka作者认为Kafka可以通过producer选择是否被commit阻塞来改善这一问题，并且节省下来的replica和磁盘使得ISR模式仍然值得。
<br>&emsp;&emsp;  上文提到，在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某一个partition的所有replica都挂了，就无法保证数据不丢失了。这种情况下有两种可行的方案：
（1）等待ISR中的任一个replica“活”过来，并且选它作为leader （2）选择第一个“活”过来的replica（不一定是ISR中的）作为leader
<br>&emsp;&emsp;  这就需要在可用性和一致性当中作出一个简单的平衡。如果一定要等待ISR中的replica“活”过来，那不可用的时间就可能会相对较长。
而且如果ISR中的所有replica都无法“活”过来了，或者数据都丢失了，这个partition将永远不可用。选择第一个“活”过来的replica作为leader，而这个replica不是ISR中的replica，
那即使它并不保证已经包含了所有已commit的消息，它也会成为leader而作为consumer的数据源（前文有说明，所有读写都由leader完成）。
Kafka0.8.*使用了第二种方式。根据Kafka的文档，在以后的版本中，Kafka支持用户通过配置选择这两种方式中的一种，从而根据不同的使用场景选择高可用性还是强一致性。
<br>&emsp;&emsp;  上文说明了一个partition的replication过程，然尔Kafka集群需要管理成百上千个partition，Kafka通过round-robin的方式来平衡partition从而避免大量partition集中在了少数几个节点上。
同时Kafka也需要平衡leader的分布，尽可能的让所有partition的leader均匀分布在不同broker上。另一方面，优化leadership election的过程也是很重要的，
毕竟这段时间相应的partition处于不可用状态。一种简单的实现是暂停宕机的broker上的所有partition，并为之选举leader。实际上，Kafka选举一个broker作为controller，
这个controller通过watch Zookeeper检测所有的broker failure，并负责为所有受影响的partition选举leader，再将相应的leader调整命令发送至受影响的broker，过程如下图所示。
![avatar](https://ws1.sinaimg.cn/large/006tNbRwly1fwlys8hvn8j313w0ro77p.jpg)
<br>&emsp;&emsp;  这样做的好处是，可以批量的通知leadership的变化，从而使得选举过程成本更低，尤其对大量的partition而言。如果controller失败了，幸存的所有broker都会尝试在Zookeeper中创建/controller->{this broker id}，
如果创建成功（只可能有一个创建成功），则该broker会成为controller，若创建不成功，则该broker会等待新controller的命令。
#### 3.7.3 controller failover
&emsp;&emsp;  当 controller 宕机时会触发 controller failover。每个 broker 都会在 zookeeper 的 "/controller" 节点注册 watcher，
当 controller 宕机时 zookeeper 中的临时节点消失，所有存活的 broker 收到 fire 的通知，每个 broker 都尝试创建新的 controller path，只有一个竞选成功并当选为 controller。
### 3.8 kafka 日志
<br>&emsp;&emsp;  a. 默认情况在log目录下，每个broker中的各个partition会产生一个log目录--（文件夹）
<br>&emsp;&emsp;  a. partition log 目录的名称是 topic名称-partition编号
<br>&emsp;&emsp;  c. log目录里面会有一个 .log文件(二进制文件)和 .index文件  --segment file
<br>&emsp;&emsp;  d. log 文件中存放实际的数据（其实.log 文件的名称本身就是一个offset，起始的offset）， .index文件存放offset信息。
<br>&emsp;&emsp;  e. 取数据是，先用consumer的offset 减去 .log 文件的起始offset，得到一个offset，在用这个值在 .index 文件中查找需要的记录在log文件中的物理偏移地址，直接到index文件中物理地址向下找，而不用开始从前往后一直读。
<br>&emsp;&emsp;  f. message 的物理结构。（.log 文件）
![avatar](https://ws3.sinaimg.cn/large/006tNbRwly1fwm0k3v9jbj30lm082aah.jpg)
图注: kafka 主题中的每个partition有一个预写日志文件，每个partition都由一系列有序的、不可变得消息组成，这些消息被联系追加到partition中，
partition中的每个消息都有一个连续的序号叫做offset，确定在分区日志文件中唯一的位置。