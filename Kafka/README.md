![](images/kafka.png)

kafka学习笔记

目录

- [Kafka-工作流程文件存储机制](/Kafka/Kafka-工作流程文件存储机制/)

  本文主要包括三个部分：Kafka 架构、Kafka-工作流程、Kafkas文件存储机制、数据可靠性

- [Kafka-数据可靠性保证](/Kafka/Kafka-数据可靠性保证/)
  
  怎么保证数据不丢？怎么保证数据不重？

  不丢：消费者手动维护offset、ack机制(ack=-1)，isr(副本、同步队列)，hw机制、生产者事务

  不重：[Kafka-生产者幂等性和事务](/Kafka/Kafka-数据可靠性保证/如何保证数据不重-幂等性和事务/)  

  幂等性 + (ack=-1) + 事务 = EXACTLY ONCE

- [Kafka-为什么Kafka读写性能这么高](/Kafka/Kafka-为什么Kafka读写性能这么高/)

- [Kafka-如何优雅的分区和消费分区](/Kafka/Kafka-如何优雅的分区和消费分区/)

- [Kafka-API-生产者](/Kafka/Kafka-API-生产者/)

  内容包括：消息发送流程、异步发送、同步发送、自定义拦截器、自定义分区器
  
- [Kafka-API-消费者](/Kafka/Kafka-API-消费者/)

- [Kafka-Zookeeper的作用](/Kafka/Kafka-Zookeeper的作用/)
