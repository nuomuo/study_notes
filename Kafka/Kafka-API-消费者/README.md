## Consumer API

Consumer 消费数据时的可靠性是很容易保证的，因为数据在 Kafka 中是持久化的，故不用担心数据丢失问题。

由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故障前的位置的继续消费，所以 consumer 需要实时记录自己消费到了哪个 offset，以便故障恢复后继续消费。所以 offset 的维护是Consumer 消费数据是必须考虑的问题。

offset 存储在 Kafka 中的一个 专门存放 offset的 topic 中

### 自动提交 offset

1）编写代码

需要用到的类：

**KafkaConsumer**：需要创建一个消费者对象，用来消费数据

**ConsumerConfig**：获取所需的一系列配置参数

**ConsuemrRecord**：每条数据都要封装成一个 ConsumerRecord 对象

为了使我们能够专注于自己的业务逻辑，Kafka 提供了自动提交 offset 的功能。 

自动提交offset的相关参数：

**enable.auto.commit**：是否开启自动提交offset功能

**auto.commit.interval.ms**：自动提交offset的时间间隔

```java
public class CustomConsumer {
    public static void main(String[] args) {
        // 1. 创建kafka消费者配置类
        Properties properties = new Properties();
        // 2. 添加配置参数
        // 添加连接
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "node1:9092");
        // 配置序列化 必须
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
        "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
        "org.apache.kafka.common.serialization.StringDeserializer");
        // 配置消费者组
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test");
        // 是否自动提交offset
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        // 提交offset的时间周期
        properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        //3. 创建kafka消费者
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);
        //4. 设置消费主题  形参是列表
        consumer.subscribe(Arrays.asList("first"));
        //5. 消费数据
        while (true){
            // 每100ms消费一次数据
            ConsumerRecords<String, String> consumerRecords = 
            consumer.poll(100);
            for (ConsumerRecord<String, String> record : consumerRecords) {
                System.out.printf("partition = %d,offset = %d,key = %s,value = %s%n",record.partition(),record.offset(),record.key(),record.value());
            }
        }
    }
}
```

关键代码：

```java
 properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
```

### 重置 offset

auto.offset.reset = earliest | latest | none |
当Kafka中没有初始偏移量(消费者组第一次消费)或服务器上不再存在当前偏移量时（例如该数据已被删除），该怎么办：

1. earliest：自动将偏移量重置为最早的偏移量

2. latest(默认值)：自动将偏移量重置为最新偏移量

3. none：如果未找到消费者组的先前偏移量，则向消费者抛出异常

```java
//auto.offset.reset = earliest | latest | none |
// 如果某个消费组第一次 消费消息 zookeeper 中没有该 消费者组的偏移量
//1.  earliest：自动将偏移量重置为最早的偏移量 会获取该topic所有的消息
//2.  latest(默认值)：自动将偏移量重置为最新偏移量 不获取之前的消息
//3.  none：如果未找到消费者组的先前偏移量，则向消费者抛出异常
// org.apache.kafka.clients.consumer.NoOffsetForPartitionException: Undefined offset with no reset policy for partitions:
properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,"latest");
```

### 手动提交 offset

虽然自动提交 offset 十分简介便利，但由于其是基于规定的时间（**auto.commit.interval.ms**）提交的，开发人员难以把握 offset 提交的时机。因此 Kafka 还提供了手动提交 offset 的API。

手动提交 offset 的方法有两种：分别是 commitSync（同步提交）和 commitAsync（异步提交）。两者的相同点是，都会将**本次 poll 的一批数据最高的偏移量提交**；不同点是，commitSync 阻塞当前线程，一直到提交成功，并且会自动失败重试（由不可控因素导致，也会出现提交失败）；而 commitAsync 则没有失败重试机制，故有可能提交失败。

```java
public class CustomConsumerByHand {
    public static void main(String[] args) {
        ...
        // 是否自动提交offset
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        ...
        while (true){
            
            ConsumerRecords<String, String> consumerRecords = consumer.poll(100);
            // 同步提交offset
            consumer.commitSync();
			// 异步提交offset
            consumer.commitAsync(new OffsetCommitCallback() {
                /**
                 * 回调函数输出
                 * @param offsets   offset信息
                 * @param exception 异常
                 */
                @Override
                public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
                    // 如果出现异常打印
                    if (exception != null ){
                        System.err.println("Commit failed for " + offsets);
                    }
                }
            });
        }
    }
}
```



虽然同步提交 offset 更可靠一些，但是由于其会阻塞当前线程，直到提交成功。因此吞吐量会收到很大的影响。因此更多的情况下，会选用异步提交 offset 的方式。



**数据漏消费和重复消费分析**

无论是同步提交还是异步提交 offset，都有可能会造成数据的漏消费或者重复消费。先提交 offse t后消费，有可能造成数据的漏消费；而先消费后提交 offset，有可能会造成数据的重复消费。