1 redis
===
>- github.com/go-redis/redis
>- client提供连结池，每个cpu（runtime）最多10个socket
>- 连结池设置从连接池获取client超时
>- 连结池client空置时间过久关闭idle time
>- 封装好的client设置其超时时间、addr、password
>- client对象存在各种redis操作方法
>- client负责将对应的函数对应的redis命令发送至连接conn并从conn中读取结果，返回给调用者
>- client可以再多做一层封装，可以将指针对应的值首先marshal然后再发送至conn，从conn拿到结果后首先unmarsha，然后返回
>- 带tls封装

2 mysql 
===



3 kafka
===
[kafka中文文档](http://kafka.apachecn.org/)
[kafka参考官网](https://kafka.apache.org/intro)
[kafka confluence](https://docs.confluent.io/current/)
>- cli&server为tcp协议
>- 名词解释：
>
```
queue&&publish-subscribe
topic: partition
Partiton: 有序
Producer: 发送时确定发送的partition
Consumer: 分group，可并行，instance数目不应多余partition，
每个partition被consumer中的一个消费以确保有序性(一个partition中有序);
Producer <->broker <-> consumer 生产与消费分析，数据可以入库或永久保存
数据入kafka入磁盘并可做容错处理：多副本，1个leader多个follower, log storage and replication design
```
>- 注册schema registry地址，kafka broke地址
>- kafka的保证：
>
```
high-level Kafka给予以下保证:
生产者发送到特定topic partition 的消息将按照发送的顺序处理。 也就是说，如果记录M1和记录M2由相同的生产者发送，并先发送M1记录，那么M1的偏移比M2小，并在日志中较早出现
一个消费者实例按照日志中的顺序查看记录.
对于具有N个副本的主题，我们最多容忍N-1个服务器故障，从而保证不会丢失任何提交到日志中的记录.
```
>- kafka与传统消息队列的区别：
>
```
传统的消息系统有两个模块: 队列 和 发布-订阅。 在队列中，消费者池从server读取数据，每条记录被池子中的一个消费者消费; 在发布订阅中，记录被广播到所有的消费者。两者均有优缺点。 队列的优点在于它允许你将处理数据的过程分给多个消费者实例，使你可以扩展处理过程。 不好的是，队列不是多订阅者模式的—一旦一个进程读取了数据，数据就会被丢弃。 而发布-订阅系统允许你广播数据到多个进程，但是无法进行扩展处理，因为每条消息都会发送给所有的订阅者。
```
>- Kafka 设计的更好。topic中的partition是一个并行的概念。 Kafka能够为一个消费者池提供顺序保证和负载平衡，是通过将topic中的partition分配给消费者组中的消费者来实现的， 以便每个分区由消费组中的一个消费者消耗。通过这样，我们能够确保消费者是该分区的唯一读者，并按顺序消费数据。 众多分区保证了多个消费者实例间的负载均衡。但请注意，消费者组中的消费者实例个数不能超过分区的数量。

3.0 分布式
---
>- 日志的分区partition （分布）在Kafka集群的服务器上。每个服务器在处理数据和请求时，共享这些分区。每一个分区都会在已配置的服务器上进行备份，确保容错性.
>- 每个分区都有一台 server 作为 “leader”，零台或者多台server作为 follwers 。leader server 处理一切对 partition （分区）的读写请求，而follwers只需被动的同步leader上的数据。当leader宕机了，followers 中的一台服务器会自动成为新的 leader。每台 server 都会成为某些分区的 leader 和某些分区的 follower，因此集群的负载是平衡的。

3.1 consumer
---
>- consumer需要有decode&process方法
>- consumer有配置：groupID 消费的topic offset
>- consumer向broker订阅消息
>- consumer不断循环，拉取到相关数据，对数据进行decode，然后process，process完成之后再make offset
>- 一个consumer只消费一个partition， 一个partition只供一个consumer消费，若一个partition可以供两个consumer消费容易出现重复消费(process完才更新offset）
>- 在Kafka中实现消费的方式是将日志中的分区划分到每一个消费者实例上，以便在任何时间，每个实例都是分区唯一的消费者。维护消费组中的消费关系由Kafka协议动态处理。如果新的实例加入组，他们将从组中其他成员处接管一些 partition 分区;如果一个实例消失，拥有的分区将被分发到剩余的实例。
>- Kafka 只保证分区内的记录是有序的，而不保证主题中不同分区的顺序。每个 partition 分区按照key值排序足以满足大多数应用程序的需求。但如果你需要总记录在所有记录的上面，可使用仅有一个分区的主题来实现，这意味着每个消费者组只有一个消费者进程。

3.2 producer
---
>- 生产者可以将数据发布到所选择的topic（主题）中。生产者负责将记录分配到topic的哪一个 partition（分区）中。可以使用循环的方式来简单地实现负载均衡，也可以根据某些语义分区函数(例如：记录中的key)来完成。下面会介绍更多关于分区的使用。
>- producer可以设置不同生产者类型，异步&同步，地址，注册地址等，可以产生的topic等
>- 根据topic选取producer
>- 将即将发送的msg编码成avro，并将header等组装成kafka发送topic形式
>- 根据produer是同步 or 异步发送kafka消息
>- send时可以指定partion，也可以不指定，rounbin随机
>- kafka需要单独开启线程监听异步发送消息是否真正发送，监听err并处理
>- kafka可以等到消息达到一定数量或者超过一定时间再发送消息

3.4 Kafka vs flink vs  spark
---
[kafka与其他工具对比分析](https://dzone.com/articles/streaming-in-spark-flink-and-kafka-1)

[kafka常见面试题](https://www.iteblog.com/archives/2605.html)
[consumer分区策略](https://www.iteblog.com/archives/2209.html)
>- kafka分区数量不可减少：
>
```
我们可以使用 bin/kafka-topics.sh 命令对 Kafka 增加 Kafka 的分区数据，但是 Kafka 不支持减少分区数。
Kafka 分区数据不支持减少是由很多原因的，比如减少的分区其数据放到哪里去？是删除，还是保留？删除的话，那么这些没消费的消息不就丢了。如果保留这些消息如何放到其他分区里面？追加到其他分区后面的话那么就破坏了 Kafka 单个分区的有序性。如果要保证删除分区数据插入到其他分区保证有序性，那么实现起来逻辑就会非常复杂。
```