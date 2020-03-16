


```xml

### 创建主题
root@Master:~# kafka-topics --create --bootstrap-server 192.168.85.200:9092 --replication-factor 1 --partitions 1 --topic topictest1
### 查看主题
root@Master:~# kafka-topics --list --bootstrap-server 192.168.85.200:9092

发送消息
root@Master:~# kafka-console-producer --broker-list 192.168.85.200:9092 --topic topictest1
### (新终端窗口)启动消费者
root@Master:~# kafka-console-consumer --bootstrap-server 192.168.85.200:9092 --topic topictest1 --from-beginning

```






















