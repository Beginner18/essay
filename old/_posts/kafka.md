---
title: kafka
date: 2019-12-12 14:08:49
tags:
---
1 kafka主要设计原理
===
>- kafka： broker(leader follower, in-sync-replicas)、partiton、consumer（group、offset；range分配，robin）、produer（ack 0 1 -1）；至少一次、至多一次、恰好一次（producer、consumer、broker三个角度谈论）、磁盘读写（达到一定量或者超过一定时间才会读写）、send to file(无内核用户级切换，直接从磁盘->内核内存->内核内存的socket空间)、磁盘顺序读写类似log日志；高可用、可扩展（partition只增不减，保证单个partition有序，减少的代价过大）、一致性、高吞吐、快；connect kafka数据与其他库的交互；不做缓存利用操作系统自己的pagecache；不需要做fsync，一般通过replica机制实现持久化;序列化数据，压缩
>- 零copy，send on file,[零copy技术](https://www.jianshu.com/p/fad3339e3448)
>- [零copy解析](https://juejin.im/entry/59b740fdf265da06633d02cf)
>
[kafka中文文档](http://kafka.apachecn.org/documentation.html#topicconfigs)

2 kafka生产者配置
===
>- kafka: producer ack
>
```
此配置是 Producer 在确认一个请求发送完成之前需要收到的反馈信息的数量。 这个参数是为了保证发送请求的可靠性。以下配置方式是允许的：
acks=0 如果设置为0，则 producer 不会等待服务器的反馈。该消息会被立刻添加到 socket buffer 中并认为已经发送完成。在这种情况下，服务器是否收到请求是没法保证的，并且参数retries也不会生效（因为客户端无法获得失败信息）。每个记录返回的 offset 总是被设置为-1。
acks=1 如果设置为1，leader节点会将记录写入本地日志，并且在所有 follower 节点反馈之前就先确认成功。在这种情况下，如果 leader 节点在接收记录之后，并且在 follower 节点复制数据完成之前产生错误，则这条记录会丢失。
acks=all 如果设置为all，这就意味着 leader 节点会等待所有同步中的副本确认之后再确认这条记录是否发送完成。只要至少有一个同步副本存在，记录就不会丢失。这种方式是对请求传递的最有效保证。acks=-1与acks=all是等效的。
```
>- retries: 
>
```
若设置大于0的值，则客户端会将发送失败的记录重新发送，尽管这些记录有可能是暂时性的错误。请注意，这种 retry 与客户端收到错误信息之后重新发送记录并无区别。允许 retries 并且没有设置max.in.flight.requests.per.connection 为1时，记录的顺序可能会被改变。比如：当两个批次都被发送到同一个 partition ，第一个批次发生错误并发生 retries 而第二个批次已经成功，则第二个批次的记录就会先于第一个批次出现
```
>- kafka保证一个partition内消息顺序与producer发送时间顺序一致：
>- max.in.flight.requests.per.connection：默认为5
>
```
在发生阻塞之前，客户端的一个连接上允许出现未确认请求的最大数量。注意，如果这个设置大于1，并且有失败的发送，则消息可能会由于重试而导致重新排序(如果重试是启用的话)。
```
>- batch,分批次发送:
>
```
当将多个记录被发送到同一个分区时， Producer 将尝试将记录组合到更少的请求中。这有助于提升客户端和服务器端的性能。这个配置控制一个批次的默认大小（以字节为单位）。
当记录的大小超过了配置的字节数， Producer 将不再尝试往批次增加记录。
发送到 broker 的请求会包含多个批次的数据，每个批次对应一个 partition 的可用数据
小的 batch.size 将减少批处理，并且可能会降低吞吐量(如果 batch.size = 0的话将完全禁用批处理)。 很大的 batch.size 可能造成内存浪费，因为我们一般会在 batch.size 的基础上分配一部分缓存以应付额外的记录。
```
>- 延迟配置类似tcp nagle算法：
>
```
producer 会将两个请求发送时间间隔内到达的记录合并到一个单独的批处理请求中。通常只有当记录到达的速度超过了发送的速度时才会出现这种情况。然而，在某些场景下，即使处于可接受的负载下，客户端也希望能减少请求的数量。这个设置是通过添加少量的人为延迟来实现的&mdash；即，与其立即发送记录， producer 将等待给定的延迟时间，以便将在等待过程中到达的其他记录能合并到本批次的处理中。这可以认为是与 TCP 中的 Nagle 算法类似。这个设置为批处理的延迟提供了上限:一旦我们接受到记录超过了分区的 batch.size ，Producer 会忽略这个参数，立刻发送数据。但是如果累积的字节数少于 batch.size ，那么我们将在指定的时间内“逗留”(linger)，以等待更多的记录出现。这个设置默认为0(即没有延迟)。例如：如果设置linger.ms=5 ，则发送的请求会减少并降低部分负载，但同时会增加5毫秒的延迟。
```
>- request.timeout.ms:
>
```
客户端等待请求响应的最大时长。如果超时未收到响应，则客户端将在必要时重新发送请求，如果重试的次数达到允许的最大重试次数，则请求失败。这个参数应该比 replica.lag.time.max.ms （Broker 的一个参数）更大，以降低由于不必要的重试而导致的消息重复的可能性。
```
>- enable.idempotence允许幂等（broker仅储存一次在produer可能超时重发的情况下，若超时重发超过一定次数则发送失败):
>
```
当设置为true时， Producer 将确保每个消息在 Stream 中只写入一个副本。如果为false，由于 Broker 故障导致 Producer 进行重试之类的情况可能会导致消息重复写入到 Stream 中。请注意,启用幂等性需要确保 max.in.flight.requests.per.connection小于或等于5，retries 大于等于0，并且ack必须设置为all 。如果这些值不是由用户明确设置的，那么将自动选择合适的值。如果设置了不兼容的值，则将抛出一个ConfigException的异常。
```

3 kafka consumer配置
===
>- consumer配置：
>- fetch.min.bytes:默认1字节，基本有消息consumer可以立刻拿到无延迟，若大于1则消息不足的情况下会有延迟
>- groupID
>- heartbeat.interval.ms: The expected time between heartbeats to the consumer coordinator when using Kafka's group management facilities. Heartbeats are used to ensure that the consumer's session stays active and to facilitate rebalancing when new consumers join or leave the group. The value must be set lower than session.timeout.ms, but typically should be set no higher than 1/3 of that value. It can be adjusted even lower to control the expected time for normal rebalances.
>- max.partition.fetch.bytes: 一个partition一个请求，可以获取的最大数据量
>- session.timeout.ms
>
```
The timeout used to detect consumer failures when using Kafka's group management facility. The consumer sends periodic heartbeats to indicate its liveness to the broker. If no heartbeats are received by the broker before the expiration of this session timeout, then the broker will remove this consumer from the group and initiate a rebalance. Note that the value must be in the allowable range as configured in the broker configuration by group.min.session.timeout.ms and group.max.session.timeout.ms.
```
>- auto.offset.reset: What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted):
earliest: automatically reset the offset to the earliest offset
latest: automatically reset the offset to the latest offset
none: throw exception to the consumer if no previous offset is found for the consumer's group
anything else: throw exception to the consumer.
>- enable.auto.commit: If true the consumer's offset will be periodically committed in the background.(允许后台自动commit)
>- isolation.level: 隔离级别读未提交，读提交；
>- patition分配策略：如何将相应的partition分配至相应的同组consumer instance；
>- auto.commit.interval.ms： The frequency in milliseconds that the consumer offsets are auto-committed to Kafka if enable.auto.commit is set to true.
>
```
Range分配（均分):consumerID字典顺序，同组内；多个topic容易造成consumer实例不均匀
有一个topic:truman partition:10 consumer:consumer0,consumer1,consumer2。那么
numPartitionsPerConsumer=10/3=3，consumersWithExtraPartition=10%3=1。分配逻辑为：当
i=0,按字典顺序，先分配consumer0，start=0，length=4，即consumer1分配tp0,tp1,tp2,tp3。
同理最终方案为：
consumer0->tp0,tp1,tp2,tp3
consumer1->tp4,tp5,tp6
consumer2->tp7,tp8,tp9
RoundRobinAssignor与RangeAssignor的重大区别，就是RoundRobinAssignor是在Group中所有consumer所关注的全体topic中进行分派，而RangeAssignor则是依次对每个topic进行分派。
```
[kafka分区策略](http://trumandu.github.io/2019/06/27/kafka-consumer-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%8C%EF%BC%89%E5%88%86%E5%8C%BA%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5/)