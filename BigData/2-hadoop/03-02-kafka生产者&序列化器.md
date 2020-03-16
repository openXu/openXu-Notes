

生产者用于向Kafka写入数据，消费者从Kafka读取数据。在入门示例中，我们通过命令执行
(内置客户端.bat)实现了生产者和消费者的演示，这篇文章我们重点介绍Kafka的生产者，顺便
讲一下序列化器和反序列化器。

# kafka生产者

```xml
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka-clients</artifactId>
	<version>2.4.0</version>
</dependency>
```

## 同步异步发送消息

下面通过Kafka内置的客户端Java API来实现生产者。我们需要清楚的是Kafka提供了二进制连接协议（基于TCP），
也就是说我们直接向Kafka网络端口发送适当的字节序列就可以实现从Kafka读取或写入消息。所以Kafka虽然是用
Java写的但是不局限于在Java中使用，还有很多其他语言实现了Kafka客户端，比如C++、Python、Go语言等，这些
客户端不属于Kafka项目。

```Java
public static void main(String[] args) throws ExecutionException, InterruptedException {
	Properties kafkaProps = new Properties();
	kafkaProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.85.200:9092,192.168.85.201:9092");
//        kafkaProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092, localhost:9093");
	kafkaProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
	kafkaProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
	//2.创建生产者对象
	KafkaProducer producer = new KafkaProducer<String, String>(kafkaProps);
	//3.创建生产记录对象
	ProducerRecord<String, String> record = new ProducerRecord<String, String>(
			"topicTest", "msgkey", "11111111111111111111111111111");
	/**同步发送消息*/
	//4.发送消息
//        Future<RecordMetadata> future = producer.send(record);
//        //5.同步等待发送结果。如果没有发生错误，我们会得到RecordMetadata对象，可以用它获取消息的偏移量
//        RecordMetadata recordMetadata = future.get();
//         System.out.println("发送消息成功："+recordMetadata.partition()+"-"+recordMetadata.topic()+"-"+recordMetadata.offset());

	/**异步发送消息*/
	//4.发送消息
	System.out.println("异步发送");
	try {
		Future<RecordMetadata> future = producer.send(record, new Callback() {
			public void onCompletion(RecordMetadata recordMetadata, Exception e) {
				System.out.println("发送消息成功："+recordMetadata.partition()+"-"+recordMetadata.topic()+"-"+recordMetadata.offset());
			}
		});
	} catch (Exception e) {
		e.printStackTrace();
	}
	//异步发送消息时，消息还在缓冲区没发送出去，代码执行成功程序就退出了，导致缓冲区的数据没发送出去就丢掉了，所以等待2s
	Thread.sleep(2000);
	System.out.println("发送完成");
}
```

**生产者发送消息过程**

1. 创建消息对象: 创建一个ProducerRecord对象，该对象包含目标主题和要发送的消息内容。还可以指定键和分区
2. 序列化消息: 生产者KafkaProducer将键和消息值序列化成字节数组（这样才能在网络上传输）
3. 获取分区(broker): 将数据传给分区器，如果ProducerRecord指定了分区，分区器不做任何事情直接把指定的分区返回，如果没指定则会根据键来选择一个分区。选好分区后，生产者就知道该往哪个分区上的哪个主题发送这条记录了
4. 将消息记录添加到批次: 一个批次里的消息会被发送到相同的主题和分区上，有一个独立线程负责把这些记录批次发送到相应的broker上
5. 发送批次: 当一个批次满了则会将批次的消息一起发送给kafka
6. 服务器响应: 服务器收到这些消息会返回一个响应。如果消息成功写入kafka会返回一个RecordMetadata对象，该对象记录了主题和分区信息，以及记录在分区的偏移量。如果失败会返回一个错误，生产者收到错误之后会尝试重新发送消息，重试几次还失败就返回错误信息。


## 生产者配置

在上面的生产者示例中我们为生产者配置了三个参数：

1. `bootstrap.servers` 配置broker地址
2. `key.serializer` 配置key的序列化器
3. `value.serializer` 配置value的序列化器

生产者还有很多可配置的参数，在Kafka文档里都有说明，它们大部分都有合理的默认值，所以没必要去修改它。
不过有几个参数在内存使用、性能和可靠性方面对生产者影响比较大，下面一一说明：

4. `acks` 指定必须有多少个分区副本收到消息，生产者才会任务消息写入是成功的。这个参数对消息丢失的可能性
有重要影响。
   - acks=0，生产者在成功写入消息之前不会等待任何来自服务器的响应，所以它可以以网络能支持的最大速度发送消息，从而达到很高的吞吐量
   - acks=1，只要集群的首领节点收到消息，生产者就会收到来自服务器成功响应。如果消息无法到达首领节点（比如首领节点崩溃而新首领还没被选举出来），生产者会收到一个错误响应，
为了避免数据丢失，生产者会重发消息。不过如果一个没有收到消息的节点成为新首领，消息还是会丢失，这时候吞吐量取决于使用的是同步发送还是异步发送。如果让客户端等待服务器
响应（调用Future.get()）显然会增加延迟。如果客户端使用异步回调，延迟问题就可以得到缓解，不过吞吐量还是会受发送中消息数量的限制（比如生产者在收到服务器响应之前可以发送多少个消息）
   - acks=all，只有当前参与复制的节点全部收到消息时，才会收到成功响应。这种模式最安全，它可以保证不止一个服务器收到消息，就算有服务器崩溃，整个集群仍然可以运行，不过它的延迟比acks=1时更高，
因为我们要等待不止一个服务器节点接受消息
5. `buffer.memory` 设置生产者内存缓冲区大小，生产者用它缓冲要发送到服务器的消息。如果程序发送消息的速度超过发送到服务器的速度，会导致生产者空间不足，这时候send()方法要么被阻塞，要么抛异常，
取决于如何设置`block.on.buffer.full`参数（在0.9.0.0版本被替换成`max.block.ms`，表示在抛出异常之前可以阻塞一段时间）
6. `compression.type` 默认情况下，消息发送时不会被压缩。该参数可设置为snappy、gzip或lz4，指定消息发送个broker之前使用哪一种压缩算法压缩。使用压缩可降低网络传输开销和存储开销，而这往往是向Kafka发送消息的瓶颈所在
   - snappy 压缩算法由Google发明，占用较少CPU，却能提供较好的性能和相当客观的压缩比，如果比较关注性能和网络宽带，可使用这种算法
   - gzip 一般会占用较多CPU，但提供更高的压缩比，如果带宽有限选择它
7. `retries` 设置生产者可以重发消息次数，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下生产者会在每次重试之间等待100ms，不过可以通过`retry.backoff.ms`参数改变这个时间间隔。建议在设置重试次数和重试时间间隔之前，
先测试一下恢复一个崩溃节点需要多少时间，让总的重试时间比Kafka集群从崩溃中恢复（选举新首领）的时间长，否则生产者会过早放弃重试
8. `batch.size` 当有多个消息需要被发送到同一个分区时，生产者会把他们放到同一个批次里，该参数指定一个批次可以使用的内存大小，按字节数计算。当批次被填满，批次里的所有消息会被发送出去。不过生产者并不一定都会等批次被填满才发送，
半满甚至只包含一条消息的批次也有可能被发送，所以就算把批次设置得很大也不会造成延迟，只会占用更多内存。但是如果设置得太小，生产者需要更频繁的发送消息，会增加一些额外的开销
9. `linger.ms` 指定生产者再发送批次之前等待更多消息加如批次的时间。KafkaProduce会在批次填满或linger.ms达到上限时把批次发送出去。默认情况下只要有可用的线程，生产者就会把消息发送出去，就算批次里只有一个消息。linger.ms>0让生
产者再发送批次之前等待一会儿，使更多消息加入批次，虽然会增加延迟，但会提高吞吐量
10. `clint.id` 该参数可以是任意字符串，服务器会用它来识别消息来源，还可以用在日志和配额指标里
11. `max.in.flight.requests.pre.connection` 指定生产者在接受服务器响应之前可以发送多少个消息。它的值越高就会占用越多的内存，不过也会提高吞吐量。设置为1可保证消息按照发送的顺序写入服务器（即使发生了重试）
12. `request.timeout.ms` 指定生产者再发送数据时等待服务器返回响应的时间
13. `metadata.fetch.timeout.ms`  指定生产者在获取元数据（比如目标分区首领是谁）时等待服务器返回响应的时间。如果等待超时，要么重试发送，要么返回一个错误
14. `timeout.ms` 指定broker等待同步副本返回消息确认的时间，与asks的配置相匹配。如果在指定时间没有收到同步副本的确认，那么broker就会返回一个错误
15. `max.block.ms` 指定在调用send()或使用partitionsFor()方法获取元数据时生产者的阻塞时间。
16. `max.request.size` 控制生产者的请求大小，它可以指能发送的单个消息的最大值，也可以是单个请求里所有消息的总大小。比如该值1MB，那单个消息最大为1MB，或者发送的同一个批次中多条消息总大小为1MB。
另外broker对可接受的消息的消息最大值也有自己的限制`message.max.bytes`，所以两边的配置最好可以匹配，避免生产者发送的消息被broker拒绝
17. `receive.buffer.bytes`和`send.buffer.bytes` 分别指定TCP socket接受和发送数据包的缓冲区大小。如果设为-1表示使用操作系统默认值。如果生产者或消费者与broker处于不同的数据中心，那么可以适当增大这些值，因为跨数据中心的网络一般都有比较高的延迟和较低的带宽

[更多生产者配置](http://kafka.apache.org/documentation/#producerconfigs)


**顺序保证**

Kafka可以保证同一个分区里的消息是有序的，也就是说，如果生产者按照一定的顺序发送消息，broker就会按照这个顺序把他们写入分区，消费者也会按照同样的顺序读取。在某些情况下顺序非常重要，而有些场景对顺序不是很敏感。
如果把retries设为2，同时max.in.flight.requests.pre.connection>1，第一个批次消息写入失败，而第二个写入成功，broker会重试写入第一个批次，如果重试成功，那么两个批次的顺序就反过来了。

一般来说如果某些场景要求消息是有序的，那么消息是否写入成功也是很关键的，所以不建议吧retries设为0。可以把max.in.flight.requests.pre.connection设为1，这样生产者尝试发送第一批消息时，就不会有其他消息发送给broker，
不过这会严重影响生产者的吞吐量，所以只有在对消息顺序有严格要求的情况下才这么做。




# 序列化器

在上面的示例中，创建一个生产者对象必须指定序列化器，我们已经知道如何使用默认的字符串序列化器。
kafka还提供了整形和字节数组序列化器，不过他们还不足以满足大部分场景的需求。到最后我们需要序列
化的记录类型会越来越多。接下来我们开发自己的序列化器，并介绍Avro序列化器作为推荐的备选方案。

## 自定义序列化器

如果我们发送到Kafka的对象不是简单的字符串或者整数，那么可以使用序列化框架来创建消息记录，
如Avro、Thrift或Protobuf，或者使用自定义序列化器，下面我们看一下怎样自定义一个序列化器: 

```Java
public class User {
    private String name;
    private int age;
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}

import org.apache.kafka.common.serialization.Serializer;
import java.nio.ByteBuffer;
import java.util.Map;
/**
 * 为了将我们自定义的User类发送到kafka，需要使用该自定义序列化器
 */
public class UserSerializer implements Serializer<User> {
    private String encoding = "UTF8";
    public void configure(Map<String, ?> configs, boolean isKey) {
        String propertyName = isKey ? "key.serializer.encoding" : "value.serializer.encoding";
        Object encodingValue = configs.get(propertyName);
        if (encodingValue == null) {
            encodingValue = configs.get("serializer.encoding");
        }
        if (encodingValue != null && encodingValue instanceof String) {
            this.encoding = (String)encodingValue;
        }
    }

    public byte[] serialize(String topic, User user) {
        try{
            if(user == null)
                return null;
            byte[] serializedName;
            int nameSize;
            if(user.getName() != null){
                serializedName = user.getName().getBytes(encoding);
            }else{
                serializedName = new byte[0];
            }
            nameSize = serializedName.length;
            ByteBuffer buffer = ByteBuffer.allocate(4+4+nameSize);
            //序列化name字段
            buffer.putInt(nameSize);
            buffer.put(serializedName);
            //序列化age字段
            buffer.putInt(user.getAge());
            return buffer.array();
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
    public void close() {
        //不需要关闭任何东西
    }
}


/**
 * 发送User对象
 */
public static void main(String[] args) {
	Properties kafkaProps = new Properties();
	kafkaProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.85.200:9092,192.168.85.201:9092");
	kafkaProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
	kafkaProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "com.openxu.test.serializer.UserSerializer");
	//2.创建生产者对象
	KafkaProducer producer = new KafkaProducer<String, User>(kafkaProps);
	//3.创建生产记录对象
	User user = new User("openXu", 30);
	//只要使用这个UserSerializer，就可以把消息记录定义成`ProducerRecord<String, User>`，并且可以直接把User对象传给生产者
	ProducerRecord<String, User> record = new ProducerRecord<String, User>("topicTest", "msgkey", user);

	Future<RecordMetadata> future = producer.send(record);
	//5.同步等待发送结果。如果没有发生错误，我们会得到RecordMetadata对象，可以用它获取消息的偏移量
	try {
		RecordMetadata recordMetadata = future.get();
		System.out.println("发送消息成功："+recordMetadata.partition()+"-"+recordMetadata.topic()+"-"+recordMetadata.offset());
	} catch (Exception e) {
		e.printStackTrace();
	}
}
```

## 自定义反序列化器

生产者需要序列化器将对象转换成字节数组才能发送给Kafka，消费者则需要反序列化器把从Kafka接受到的
字节数组转换成Java对象。在之前发送字符串消息时，我们使用了自带的`org.apache.kafka.common.serialization.StringDeserializer`
，这个反序列化器不能将字节数组转换为`User`对象，序列化器和反序列化器应该是一一对应的，如果不对应
则会出现不可预测的结果。下面我们自定义一个反序列化器:

```Java

import org.apache.kafka.common.serialization.Deserializer;
import java.nio.ByteBuffer;
import java.util.Map;
/**
 * 反序列化器
 */
public class UserDeserializer implements Deserializer<User> {
    private String encoding = "UTF8";
    public void configure(Map<String, ?> configs, boolean isKey) {
        String propertyName = isKey ? "key.deserializer.encoding" : "value.deserializer.encoding";
        Object encodingValue = configs.get(propertyName);
        if (encodingValue == null) {
            encodingValue = configs.get("deserializer.encoding");
        }
        if (encodingValue != null && encodingValue instanceof String) {
            this.encoding = (String)encodingValue;
        }
    }
    public User deserialize(String topic, byte[] bytes) {
        if(bytes==null)
            return null;
        try {
            ByteBuffer buffer = ByteBuffer.wrap(bytes);
            int nameSize = buffer.getInt();
            byte[] nameBytes = new byte[nameSize];
            buffer.get(nameBytes);
            String name = new String(nameBytes, encoding);
            int age = buffer.getInt();
            return new User(name, age);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
    public void close() {
    }
}


/**
 * 消费者
 */
public static void main(String[] args) {
	//1. 配置
	Properties kafkaProps = new Properties();
	kafkaProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,  "192.168.85.200:9092,192.168.85.201:9092");
	kafkaProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
//        kafkaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
	kafkaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "com.openxu.test.serializer.UserDeserializer");
	kafkaProps.put("group.id", "test");   //指定消费者所属群组
	//2. 创建一个KafkaConsumer 对象
	KafkaConsumer<String, User> consumer = new KafkaConsumer<String, User>(kafkaProps);
	//3. 订阅主题，接受一个集合对象，可以订阅多个主题
	consumer.subscribe(Collections.singletonList("topicTest"));
	//4. 轮询
	try {
		while (true) {
			//消费者必须持续对Kafka进行轮询，负责会被认为已经死亡，它的分区会被移交给群组里的其他消费者
			//poll()接受一个超时时间，用于控制阻塞的时间（在消费者的冲区里没有可用数据会发生阻塞）
			ConsumerRecords<String, User> records = consumer.poll(2000);
			for (ConsumerRecord<String, User> record : records) {
				System.out.println("=========消费消息==========");
				System.out.println("主题:" + record.topic() + "  分区:" + record.partition());
				System.out.println("key:" + record.key() + "  value" + record.value());
			}
		}
	}catch (Exception e){
		e.printStackTrace();
	}finally {
		consumer.close(); //关闭消费者
	}
}
```


## 使用Avro序列化

自定义序列化和反序列化器很简单，但是存在很多问题:

- 比如修改了User中的某个字段、或者增建字段，就会出现新旧消息的兼容性问题
- 自定义序列化器吧生产者和消费者耦合在一起，代码很脆弱

所以不建议使用自定义序列化器，上面的示例只是为了让我们明白序列化器的原理。推荐使用已有的序列化器和反序列化器，
比如JSON、Avro、Thrift、Protobuf。下面我们介绍Avro，延时如何序列化Avro记录并发送给kafka。

Apache Avro是一种与编程语言无关的序列化格式。Avro数据通过与语言无关的schema来定义，schema通过JSON来描述。
Avro有一个很有意思的特性，当负责写消息的应用程序使用了新的schema，负责读消息的应用程序可以继续处理消息而不
需做任何改动，这个特性使得他特别适合用在kafka这样的消息系统上。

信息发送之后消费者需要通过对应的schema反序列化，所以Avro的数据文件里包含了整个schema，这样的开销是可接受的。
但是如果每条kafka记录里都嵌入schema就会让记录大小成倍增加。不管怎样，在读取记录时要用到schema，可以通过schema
注册表来达到目的。schema注册表不属于kafka但有一些开源的实现，比如我们下面用到的`Confluent Schema Registry`。
有了注册表，消息记录中就只需要携带schema ID即可。

### 安装schema注册表

[下载Confluent Schema Registry](https://www.confluent.io/confirmation)，我这里使用的是**confluent-5.4.0-2.12.tar.gz**。
下载完成后，解压、修改配置:

```xml
#### 解压
root@Master:/soft# tar -xzvf confluent-5.4.0-2.12.tar.gz
root@Master:/soft# cd confluent-5.4.0

//修改配置
root@Master:/soft/confluent-5.4.0# vim etc/schema-registry/schema-registry.properties

#### Confluent Schema Registry 服务的访问IP和端口
listeners=http://192.168.85.200:8081
#### Kafka集群所使用的zookeeper地址，如果不配置，会使用Confluent内置的Zookeeper地址(localhost:2181)
##### kafkastore.connection.url=192.168.42.89:2181/kafka-1.1.0-cluster
##### Kafka集群的地址(上一个参数和这个参数配置一个就可以了)
kafkastore.bootstrap.servers=192.168.85.200:9092,192.168.85.201 :9092
#### 存储 schema 的 topic
kafkastore.topic=_schemas
#### 其余保持默认即可

#### 启动 Confluent Schema Registry
root@Master:/soft/confluent-5.4.0# bin/schema-registry-start etc/schema-registry/schema-registry.properties 

#### schema格式示例
#### {
####     "schema": "{
####         \"type\": \"record\",
####         \"name\": \"User\",
####         \"fields\": [
####             {\"name\": \"age\", \"type\": \"int\"},
####             {\"name\": \"name\",  \"type\": \"string\"}
####         ]
####     }"
#### }

#### 注册 User 的 schema 注册到对应的 topic （topic001） 下
root@Master:/soft/confluent-5.4.0# curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
> --data '{"schema": "{\"type\": \"record\",\"name\": \"User\",\"fields\": [{\"name\": \"age\", \"type\": \"int\"},{\"name\": \"name\",  \"type\": \"string\"}]}"}' \
> http://192.168.85.200:8081/subjects/topic001-value/versions
#### 注册成功返回schema的id
{"id":1}

```

### 使用Avro

```Java
//生产者
public static void main(String[] args) {

	final String USER_SCHEMA = "{" +
			"\"namespace\":\"userManager.avro\"," +
			"\"type\":\"record\"," +
			"\"name\":\"User\"," +
			"\"fields\":[" +
			"{\"name\":\"name\", \"type\":\"string\"}," +
			"{\"name\":\"age\", \"type\":\"int\"}" +
			"]" +
			"}";
	Properties kafkaProps = new Properties();
	kafkaProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.85.200:9092,192.168.85.201:9092");
	kafkaProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
//        kafkaProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "com.openxu.test.serializer.UserSerializer");
	kafkaProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "io.confluent.kafka.serializers.KafkaAvroSerializer");
	// 添加schema服务的地址（查看上一步的配置），用于获取schema
	kafkaProps.put("schema.registry.url", "http://192.168.85.200:8081");
	//2.创建生产者对象
	KafkaProducer producer = new KafkaProducer<String, GenericRecord>(kafkaProps);
	//3.创建生产记录对象
	Schema schema = new Schema.Parser().parse(USER_SCHEMA);
	GenericRecord user = new GenericData.Record(schema);
	user.put("age", 30);
	user.put("name", "openXu");

	ProducerRecord<String, GenericRecord> record = new ProducerRecord<String, GenericRecord>("topicTest", "msgkey", user);

	Future<RecordMetadata> future = producer.send(record);
	try {
		RecordMetadata recordMetadata = future.get();
		System.out.println("发送消息成功："+recordMetadata.partition()+"-"+recordMetadata.topic()+"-"+recordMetadata.offset());
	} catch (Exception e) {
		e.printStackTrace();
	}
}


//消费者
public static void main(String[] args) {
	//1. 配置
	Properties kafkaProps = new Properties();
	kafkaProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,  "192.168.85.200:9092,192.168.85.201:9092");
	kafkaProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
//        kafkaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
//        kafkaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "com.openxu.test.serializer.UserDeserializer");
	kafkaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "io.confluent.kafka.serializers.KafkaAvroDeserializer");
	// 添加schema服务的地址，用于获取schema
	kafkaProps.put("schema.registry.url", "http://192.168.85.200:8081");
	kafkaProps.put("group.id", "test");   //指定消费者所属群组
	KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<String, GenericRecord>(kafkaProps);
	consumer.subscribe(Collections.singletonList("topicTest"));
	//4. 轮询
	try {
		while (true) {
			ConsumerRecords<String, GenericRecord> records = consumer.poll(2000);
			for (ConsumerRecord<String, GenericRecord> record : records) {
				System.out.println("=========消费消息==========");
				System.out.println("主题:" + record.topic() + "  分区:" + record.partition());
				System.out.println("key:" + record.key() + "  value" + record.value());
			}
		}
	}catch (Exception e){
		e.printStackTrace();
	}finally {
		consumer.close(); 
	}
}

```






Avro的数据文件里包含整个schema，如果在每条Kafka记录都嵌入schema，会让记录的大小成倍增加。所以使用“schema注册表”来解决这个问题。schema注册表并不属于Kafka，现在
已经有一些开源的schema注册表实现。
































