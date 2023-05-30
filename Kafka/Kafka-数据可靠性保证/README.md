# Kafka 数据可靠性保证

数据不丢，ack = -1，但可能会重复，isr队列，副本机制，hw leo

**数据不重，幂等性保证单会话单分区的数据不重不乱序，事务保证了跨会话跨分区的数据不重。**

> 分区内不乱序。



## 数据不丢失

### Kafka内部保证

比较常见的一个场景，就是 kafka 的某个 broker 宕机了，然后重新选举 partition 的 leader 时。如果此时 follower 还没来得及同步数据，leader 就挂了，然后某个 follower 成为了 leader，他就少了一部分数据。怎么办？

一般要求设置4个参数来保证消息不丢失：
①给topic设置 **replication.factor**参数：这个值必须大于1，表示要求每个partition必须至少有2个副本。

②在kafka服务端设置**min.isync.replicas**参数：这个值必须大于1，表示 要求一个leader至少感知到有至少一个follower在跟自己保持联系正常同步数据，这样才能保证leader挂了之后还有一个follower。

③在生产者端设置**acks=all**：表示 要求每条每条数据，必须是写入所有replica副本之后，才能认为是写入成功了

④在生产者端设置**retries=MAX**(很大的一个值，表示无限重试)：表示 这个是要求一旦写入事变，就无限重试



### 消费者丢失数据

消费者消费到了这个数据，然后消费之自动提交了offset，让kafka知道你已经消费了这个消息，当你准备处理这个消息时，自己挂掉了，那么这条消息就丢了。

关闭自动提交offset，在自己处理完毕之后手动提交offset，这样就不会丢失数据。





### 生产者弄丢了数据
生产者没有设置相应的策略，发送过程中丢失数据。

如果按照上面设置了ack=all，则一定不会丢失数据，要求是，你的leader接收到消息，所有的follower都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试，重试无限次。

## 数据不重

## 幂等性

在0.11.0.0版本引入了创建幂等性Producer的功能。仅需要设置props.put(“enable.idempotence”，true)，或props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG,true)。

enable.idempotence设置成true后，Producer自动升级成幂等性Producer。Kafka会自动去重。Broker会多保存一些字段。当Producer发送了相同字段值的消息后，Broker能够自动知晓这些消息已经重复了。

作用范围：

1. 只能保证单分区上的幂等性，即一个幂等性Producer能够保证某个主题的一个分区上不出现重复消息。
2. 只能实现单回话上的幂等性，这里的会话指的是Producer进程的一次运行。当重启了Producer进程之后，幂等性不保证。

## 事务

Kafka在0.11版本开始提供对事务的支持，提供是read committed隔离级别的事务。保证多条消息原子性地写入到目标分区，同时也能保证Consumer只能看到事务成功提交的消息。

### 事务性Producer

保证多条消息原子性地写入到多个分区中。这批消息要么全部成功，要不全部失败。事务性Producer也不惧进程重启。

Producer端的设置：

1. 开启`enable.idempotence = true`
2. 设置Producer端参数 `transactional.id`

除此之外，还要加上调用事务API，如initTransaction、beginTransaction、commitTransaction和abortTransaction，分别应对事务的初始化、事务开始、事务提交以及事务终止。
如下：

```
Copyproducer.initTransactions();
try {
            producer.beginTransaction();
            producer.send(record1);
            producer.send(record2);
            producer.commitTransaction();
} catch (KafkaException e) {
            producer.abortTransaction();
}
```

这段代码能保证record1和record2被当做一个事务同一提交到Kafka，要么全部成功，要么全部写入失败。

Consumer端的设置：
设置isolation.level参数，目前有两个取值：

1. read_uncommitted:默认值表明Consumer端无论事务型Producer提交事务还是终止事务，其写入的消息都可以读取。
2. read_committed:表明Consumer只会读取事务型Producer成功提交事务写入的消息。注意，非事务型Producer写入的所有消息都能看到。

