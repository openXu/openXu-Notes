
[alibaba canal GitHub](https://github.com/alibaba/canal)

canal(译:水道/管道/沟渠)是阿里巴巴开源的mysql binlog增量订阅&消费组件，主要用途是基于Mysql数据库增量日志解析，
提供增量数据订阅和消费。简单的说，canal就是一个中间件，用它可以监听mysql的变化，canal将数据库变化事件转换为消息，
然后我们可以订阅这些消息后做一些工作，比如数据库同步等等。

![canal工作流](pic/1001_canal_pro.jpg)

# 工作原理

**MySQL主备复制原理**

- MySQL master 将数据变更写入二进制日志( binary log, 其中记录叫做二进制日志事件binary log events，可以通过 show binlog events 进行查看)
- MySQL slave 将 master 的 binary log events 拷贝到它的中继日志(relay log)
- MySQL slave 重放 relay log 中事件，将数据变更反映它自己的数据

**canal 工作原理**

- canal 模拟 MySQL slave的交互协议，伪装自己为MySQL slave，向MySQL master发送dump协议
- MySQL master 收到 dump 请求，开始推送binary log给slave(即canal)
- canal 解析binary log对象(原始为byte流)

# 快速开始

[QuickStart](https://github.com/alibaba/canal/wiki/QuickStart)

## 1. 准备

需要先开启Binlog写入功能，配置binlog-format为ROW模式，my.cnf(windows安装包为my.ini)中配置如下:

```xml
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式

server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```

授权canal连接MySQL账号具有作为MySQL slave的权限, 如果已有账户可直接grant:

```xml
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

## 2. canal安装

下载canal, 访问release页面, 选择需要的包下载, 如以 1.1.4 版本为例:

[下载](https://github.com/alibaba/canal/releases)

```xml
#### 下载压缩包
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.deployer-1.1.4.tar.gz

#### 或者直接在网页下载后拷贝过去
root@Master:~# mv /home/openxu/Desktop/canal.deployer-1.1.4.tar.gz /soft/canal.deployer-1.1.4.tar.gz

#### 解压
root@Master:~# cd /soft/
root@Master:/soft# mkdir canal
root@Master:/soft# tar zxvf canal.deployer-1.1.4.tar.gz -C canal
root@Master:/soft# cd canal/
root@Master:/soft/canal# ll
total 24
drwxr-xr-x 6 root root 4096 2月  20 15:24 ./
drwxr-xr-x 9 root root 4096 2月  20 15:24 ../
drwxr-xr-x 2 root root 4096 2月  20 15:24 bin/
drwxr-xr-x 5 root root 4096 2月  20 15:24 conf/
drwxr-xr-x 2 root root 4096 2月  20 15:24 lib/
drwxrwxrwx 2 root root 4096 9月   2 15:26 logs/

#### 配置修改
root@Master:/soft/canal# vim conf/example/instance.properties 

	#### mysql serverId
	canal.instance.mysql.slaveId = 1234
	####position info，需要改成自己的数据库信息
	canal.instance.master.address = 192.168.93.1:3306 
	canal.instance.master.journal.name = 
	canal.instance.master.position = 
	canal.instance.master.timestamp = 
	####canal.instance.standby.address = 
	####canal.instance.standby.journal.name =
	####canal.instance.standby.position = 
	####canal.instance.standby.timestamp = 
	####username/password，需要改成自己的数据库信息
	canal.instance.dbUsername = canal  
	canal.instance.dbPassword = canal
	canal.instance.defaultDatabaseName =
	#### 代表数据库的编码方式对应到 java 中的编码类型，比如 UTF-8，GBK , ISO-8859-1
	canal.instance.connectionCharset = UTF-8  
	#### table regex
	canal.instance.filter.regex = .\*\\\\..\*
	#### 如果系统是1个 cpu，需要将 canal.instance.parser.parallel 设置为 false
	canal.instance.parser.parallel = false

#### 启动
root@Master:/soft/canal# sh bin/startup.sh 

#### 查看 server 日志
root@Master:/soft/canal# cat logs/canal/canal.log

	3:32:45.868 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## set default uncaught exception handler
	2020-02-19 23:32:45.902 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## load canal configurations
	2020-02-19 23:32:45.911 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## start the canal server.
	2020-02-19 23:32:45.939 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[192.168.85.200(192.168.85.200):11111]
	2020-02-19 23:32:46.915 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## the canal server is running now ......

#### 查看 instance 的日志
root@Master:/soft/canal# cat logs/example/example.log



#### 关闭
root@Master:/soft/canal# sh bin/stop.sh

```

# Canal Kafka QuickStart

## Kafka

```xml
#### 创建主题，共canal使用
root@Master:~# kafka-topics --create --bootstrap-server 192.168.85.200:9092 --replication-factor 1 --partitions 2 --topic canal
#### 查看主题
root@Master:~# kafka-topics --list --bootstrap-server 192.168.85.200:9092

#### 开启消费窗口
kafka-console-consumer --bootstrap-server 192.168.85.200:9092 --topic canal --from-beginning

#### 发送消息测试
kafka-console-producer --broker-list 192.168.85.200:9092 --topic canal

```

## canal配置

### instance.properties

```xml
#### 修改instance 配置文件 conf/example/instance.properties

	####  按需修改成自己的数据库信息
	#################################################
	...
	canal.instance.master.address=192.168.93.1:3306
	#### username/password,数据库的用户名和密码
	...
	canal.instance.dbUsername = canal
	canal.instance.dbPassword = canal
	...
	#### mq config####
	#### mq里的topic名
	canal.mq.topic=canal
	#### 针对库名或者表名发送动态topic
	####canal.mq.dynamicTopic=mytest,.*,mytest.user,mytest\\..*,.*\\..*
	canal.mq.partition=0
	#### hash partition config
	####canal.mq.partitionsNum=3
	####库名.表名: 唯一主键，多个表之间用逗号分隔
	####canal.mq.partitionHash=mytest.person:id,mytest.role:id
	#################################################


```

### canal.properties

```xml

root@Master:/soft/canal# vim conf/canal.properties

	#################################################
	#########               common argument         ############# 
	#################################################
	...
	 
	★ 可选项: tcp(默认), kafka, RocketMQ
	canal.serverMode = kafka

	....

	##################################################
	#########                    MQ                      #############
	##################################################

	#### kafka/rocketmq 集群配置: 192.168.1.117:9092,192.168.1.118:9092,192.168.1.119:9092 
	canal.mq.servers = 192.168.85.200:9092,192.168.85.201:9092,192.168.85.202:9092
	#### 发送失败重试次数
	canal.mq.retries = 0
	#### kafka为ProducerConfig.BATCH_SIZE_CONFIGrocketMQ无意义。flagMessage模式下可以调大该值, 但不要超过MQ消息体大小上限
	canal.mq.batchSize = 16384
	canal.mq.maxRequestSize = 1048576
	#### flatMessage模式下请将该值改大, 建议50-200
	canal.mq.lingerMs = 1
	canal.mq.bufferMemory = 33554432
	#### Canal的batch size, 默认50K, 由于kafka最大消息体限制请勿超过1M(900K以下)
	canal.mq.canalBatchSize = 50
	#### Canal get数据的超时时间, 单位: 毫秒, 空为不限超时
	canal.mq.canalGetTimeout = 100
	#### 是否为flat json格式对象
	canal.mq.flatMessage = false
	canal.mq.compressionType = none
	canal.mq.acks = all
	#### kafka消息投递是否使用事务
	canal.mq.transaction = false
	
```


## 修改数据库查看kafka消息

```xml
#### 启动
root@Master:/soft/canal# sh bin/startup.sh 
#### 关闭
root@Master:/soft/canal# sh bin/stop.sh
```

在mysql上执行插入语句: 
`insert INTO account (id, name, money) VALUES (7, 'haha', 20);`

kafka消费窗口收到消息:  

```json
{
	"data": [{
		"id": "4",
		"name": "haha",
		"money": "20.0"
	}],
	"database": "openxu",
	"es": 1582189292000,
	"id": 1,
	"isDdl": false,
	"mysqlType": {
		"id": "int(11)",
		"name": "varchar(40)",
		"money": "float"
	},
	"old": null,
	"pkNames": ["id"],
	"sql": "",
	"sqlType": {
		"id": 4,
		"name": 12,
		"money": 7
	},
	"table": "account",
	"ts": 1582189293606,
	"type": "INSERT"
}
```










