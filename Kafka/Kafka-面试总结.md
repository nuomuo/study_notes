请你说出你所知道的关于Kafka 的一切

思路 从架构---> 生产者--->分区---->消费者---->数据一致性保证(Flink --> )

集群规模数据量量要清楚

 

## Kafka 面试总结

Kafka我们从三个方面向您介绍，框架、碰到的问题和优化 

### 框架

#### 0. 说一下 Kafka 的架构？

kafka 版本？

kafka 有 4 个组件：生产者 、消费者 、 brokers 和 zk

~~Zk 保存着消费者的 offset 和 brokers 的 ids 等信息~~

#### 1. 集群大概的数据量

1. 数据量：

   日常每天 80-100g，每天产生1亿条日志，大概平均的数据生产速度 1M/s，不过到了峰值时间，晚上7-9点时间段，最高达到了20MB/S，大概有2W人同时在线

2. **kafka 的台数**

   通过 kafak 自带的压测工具，测试生产峰值速度，我们当时选择了 3 台 kafka。

3. 分区的数量

   分区数量 = 期望的吞吐量 / min(生产峰值速度，消费最大的速度)，我们当时设置的是5个分区

4. 存储大小

   Kafka数据默认保存7天，我们调整到3天，因为正常情况下，我们是可以跑完当天的数据，当出现异常时，第二天可以重跑一次，所以我们保留3天 是足够的。

   我们给 kakfa 硬盘的大小为：每天的数据量 *副本数 * 保留的天数 / buffer(0.7) ，大概是0.8T，我们预留了1T



#### 2. zookeeper的作用

其他的提问形式：zookeeper中存储的kafka中的信息的格式？

[Kafka-Zookeeper的作用]()

1、broker注册 2、topic 注册 3、分区注册 4、消费者注册、5、消费者组与消费者关系



### 碰到的问题

#### 0. Kafka单条日志传输大小

#### 1. Kafka 某个broker挂掉了怎么办？

某个broker挂掉，肯定会有主题的分区（leader）会挂掉，有的是（follower）会挂掉

对于follor没有影响。如果正好该主题分区的leader挂掉了，会有一个lsr中选举leader的过程，如果没有设置ack = -1 ，由于hw机制，该主题数据可能会丢失。

如果没有设置副本机制，也就是唯一的一个分区突然没有了，那么数据丢失影响显而易见。



结合项目：我们给分区设置了2个副本，所以不会出现分区丢失。所以短期是没事的。



如果分区没了，

kafkaproducer 可以动态发现分区的变化，Producer可以在最多topic.metadata.refresh.interval.ms的时间之后感知到。

kafkacomsumer：离线中 flume的kafkasource

​								实时中 flink可以配置动态发现



#### 2. 数据会丢失吗？ack 机制

```ack = -1``` 可以保证生产者传入数据不丢失，但不能保证数据不重复。

> 回答这个问题应该从生产者、消费者、Kafka内部 3个方面分别介绍对数据不丢的保证

Kafka内部的副本机制，且是落盘的。

消费者精准一次消费。下文讲。

对于数据丢失，这个重要看 ack 的配置，ack 有0,1-1三种,0表示 leader 一收到数据就回复 ack，可能丢失数据，企业中已经不使用，1 表示 leader 落盘以后回复 ack，可靠性相对可靠，-1 表示所有的副本都落盘以后再回复 ack，可靠性高，性能相对较慢。

在项目中，我们传输的是日志数据，所以采用了 **ack=1**的方式。

Consumer：offset提交在前，消费数据在后，可能会导致数据丢失。

#### 3. 数据会重吗？幂等性+事务

[如何保证数据不重-幂等性和事务]()

对于数据重复，可以通过 事务 + 幂等性来解决这个问题，幂等性通过(生产者id，分区号，序列号)对一个分区内的数据进行标识，可以保证分区内的数据不会重复.

当生产者重启时，生产者的id号会发生改变，这样同一条数据就可能会被重复发送到kafka，通过事务将pid和事务id进行绑定，解决了这个问题，



**不过我们通过会议讨论，这样会严重影响性能，所以这里我们就不做处理，等hive的dwd层进行去重。**



#### 4. 数据积压怎么办？

同时我们还遇到了数据挤压的问题，我们做了两个优化：

一是：增加分区，同时增加下一级消费者的cpu核数。

只能增加分区，不能减少。保持 kafkasource 的并行度与分区数一致。--->可以往 flink 关于动态调优上引。



二是：通过修改 batchsize 参数，提高消费者每次拉取的数据量，默认是1000条，我们把它调整到2000，极端情况下我们也调整过到3000条/s



#### 5. 实时项目中需要保证数据有序

保证同一个表的变化,数据去往同一个分区。将表名指定为key，按key计算发往的分区号。

----

### 优化（实际生产经验）

我们通过修改参数对 kafka 进行优化：



1. 将数据保存时间由默认的 7 天修改为 3 天

   ```log.retention.hours=72```

   Kafka 数据默认保存7天，我们调整到3天，因为正常情况下，我们是可以跑完当天的数据，当出现异常时，第二天可以重跑一次，所以我们保留3天是足够的。

   

2. 副本数由默认1个修改为2个，增加可靠性

   ``` default.replication.factor : 2```

3. 增加副本拷贝过程中 leader 和 follower 之间的通信时间，避免因为网络问题，不断的拷贝同一副本。

   

4. 对producer发送的数据采用是压缩

   producer.properties

   ``` compressio.type:none\gzip\snappy\1z4```



5. 调整了 kafka 的内存大小，由 1g 调整到 4g，提高性能。生产环境最好不要超过 6g。

   kafka-server-start.sh

   ```export KAFKA_HEAP_OPTS='-Xms4g -Xmx4g'```
   
6.  kafka监控 kafkaeage 监控指标



### 其他常问的问题

#### 1. Kafka 为什么速度这么快？[Kafka-为什么Kafka读写性能这么高]()

顺序读写、分区分桶索引并发索引、pagecache零拷贝、批量读写、批量压缩



#### 2.Kafka 里的数据是有序的吗？

分区内是有序的，分区间是无序的。

分区内有序是由 幂等性实现的。







#### 3.消息队列有啥用？为什么选择Kafka而不是其他的MQ,如rabbitmq？

消息队列有啥用

解耦、削峰、缓存数据重复消费



缺点：系统可用性降低。 ---》 Kafka 高可用

数据的丢重 和 顺序问题

关于顺序问题：Kafka 的消息是分区内有序的，分区间无序的。但是我们的项目对消息的顺序要求不是很高

#### 4.Kafka 的水位线了解吗----> 实质就是问的 HW LEO

**LEO：指的是每个副本最大的 offset+1；**

**HW：指的是消费者能见到的最大的 offset，也就是ISR 队列中（多个副本中）最小的 LEO。**

**（1）Follower故障**

Follower 发生故障后会被临时踢出 ISR，待该 Follower 恢复后，Follower 会读取本地磁盘记录的上次的 HW，并将 log 文件高于 Leader HW 的部分截取掉，从 HW 开始向 leader 进行同步。等该 **follower 的 LEO 大于等于该Partition 的 HW**，即 follower追上leader之后，就可以重新加入 ISR 了。

**（2）leader故障**

leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性，其余的Follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader 同步数据。

**注意：HW只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。**

---

#### 5. Canal-->Kafka-->Spark-streaming  

Flink-cdc、--->Kafka 

flink



数据精准一次性消费:checkpoint 机制

canal --> Kafka







#### 6. Spark-streaming 消费 Kafka 数据的两种方式







#### 7. kafka的精准一次消费

生产端：flume有事务，kafka prodecer 的 exactly once，**flink-cdc 两阶段提交 flink（端到端，包含了读和写）**

kafka内部，ack，副本

消费端：flume事务，spark-streaming手动维护offset



下游幂等性框架：延迟提交偏移量offset，重复

spark-streaming：手动提交offset+保证数据到一个事务。

flink-cdc、 Flink：---->两阶段提交











#### 8. kafka如果创建大量的topic，对kafak会有什么影响？

topic太多造成partition过多。partition是kafka的最小并行单元，每个partition都会在对应的broker上有日志文件。

　　当topic过多，partition增加，日志文件数也随之增加，就需要允许打开更多的文件数。

　　partition过多在controller选举和controller重新选举partition leader的耗时会大大增加，造成kafka不可用的时间延长

#### 9. kafka中rebalance？

答：

a) 触发Rebalance的时机：

b) Rebalance的触发条件有三个：

c) （1）组员个数发生变化。例如有新的consumer实例加入该消费组或者离开组

d) （2）订阅的Topic个数发生变化

e) （3）订阅Topic的分区数发生变化

f) 消费组成员正常的添加和停掉导致rebalance，这种情况无法避免，但是在某些情况下，consumer实例

g) 会被coordinator错误的认为已停止从而被踢出group。从而导致rebalance。

h) 参数解决：

i) （1）当 Consumer Group 完成 Rebalance 之后，每个 Consumer 实例都会定期地向 Coordinator 发送心跳请求，表明它还存活着。

j) 如果某个 Consumer 实例不能及时地发送这些心跳请求，Coordinator 就会认为该 Consumer 已经 “死” 了，从而将其从 Group 中移除，然后开启新一轮 Rebalance。

k) 这个时间可以通过Consumer 端的参数 session.timeout.ms进行配置。默认值是 10 秒。

l) （2）除了这个参数，Consumer 还提供了一个控制发送心跳请求频率的参数，就是 heartbeat.interval.ms。这个值设置得越小，Consumer 实例发送心跳请求的频率就越高。

m) 频繁地发送心跳请求会额外消耗带宽资源，但好处是能够更加快速地知晓当前是否开启 Rebalance，因为，目前 Coordinator 通知各个 Consumer 实例开启 Rebalance 的方法，

n) 就是将 REBALANCE_NEEDED 标志封装进心跳请求的响应体中。

o) （3）除了以上两个参数，Consumer 端还有一个参数，用于控制 Consumer 实际消费能力对 Rebalance 的影响，即 max.poll.interval.ms 参数。

p) 它限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。它的默认值是 5 分钟，表示你的 Consumer 程序如果在 5 分钟之内无法消费完

q) poll 方法返回的消息，那么 Consumer 会主动发起 “离开组” 的请求，Coordinator 也会开启新一轮 Rebalance。