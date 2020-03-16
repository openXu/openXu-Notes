

生产者用于向Kafka写入数据，消费者从Kafka读取数据。在入门示例中，我们通过命令执行
(内置客户端.bat)实现了生产者和消费者的演示，这里介绍Kafka API，通过api编程的方式
来实现生产者和消费者。

该篇示例中使用的语言为Java，我们需要清楚的是Kafka提供了二进制连接协议（基于TCP），
只需要向Kafka网络端口发送只当的字节序列就可以实现从Kafka读取或写入消息。所以Kafka
虽然是用Java写的但是不局限于在Java中使用，还有很多其他语言实现了Kafka客户端，比如C++、
Python、Go语言等。



# Kafka消费者



## 相关概念

**消费者和消费者群组**

Kafka消费者从属于消费者群组，一个群组里的消费者订阅的是同一个主题，每个消费
接收主题一部分分区的消息。当消费者要加入群组时，它会向群组协调器发送JoinGroup请求，
第一个加入群组的消费者将成为“群主”。群主从协调器那里获得群组的成员列表（列表中包含
了所有最近发送过心跳的消费者，它们被认为是活跃的），并负责给每一个消费者分配分区。
它使用一个实现了PartitionAssignor接口的类来决定哪些分区应该被分配给哪个消费者。

不要让消费者的数量超过主题分区的数量，多余的消费者只会被闲置。

**分区再均衡**
分区的所有权从一个消费者转移到另一个消费者，这样的行为被称为再均衡。它为消费
者群组带来了高可用性和伸缩性（我们可以放心地添加或移除消费者），不过在正常情
况下，我们并不希望发生这样的行为。在再均衡期间，消费者无法读取消息，造成整个
群组一小段时间的不可用。

消费者通过向被指派为群组协调器的broker不同的群组可以有不同的协调器）发送心跳
来维持它们和群组的从属关系以及它们对分区的所有权关系。只要消费者以正常的时间
间隔发送心跳，就被认为是活跃的，说明它还在读取分区里的消息。消费者会在轮询消息
（为了获取消息）或提交偏移量时发送心跳。如果消费者停止发送心跳的时间足够长，会
话就会过期，群组协调器认为它已经死亡，就会触发一次再均衡。

## 创建Kafka消费者

```Java
//1. 配置
Properties kafkaProps = new Properties();
kafkaProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092, localhost:9093");
kafkaProps.put("group.id", "test");   //指定消费者所属群组
kafkaProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
kafkaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
//2. 创建一个KafkaConsumer 对象
KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(kafkaProps);
//3. 订阅主题，接受一个集合对象，可以订阅多个主题
consumer.subscribe(Collections.singletonList("openxu_topic"));
try {
	//4. 轮询
	while (true) {
		//消费者必须持续对Kafka进行轮询，负责会被认为已经死亡，它的分区会被移交给群组里的其他消费者
		//poll()接受一个超时时间，用于控制阻塞的时间（在消费者的冲区里没有可用数据会发生阻塞）
		ConsumerRecords<String, String> records = consumer.poll(100);
		for (ConsumerRecord<String, String> record : records) {
			System.out.println("=========消费消息==========");
			System.out.println("主题:" + record.topic() + "  分区:" + record.partition());
			System.out.println("key:" + record.key() + "  value" + record.value());
		}
	}
}catch (Exception e){
	e.printStackTrace();
}finally {
	//5. 关闭消费者
	consumer.close(); 
}
```

## 消费者配置

[更多消费者配置](http://kafka.apache.org/documentation/#consumerconfigs)








