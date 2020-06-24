

[集群配置](https://clickhouse.tech/docs/zh/single/#cluster-deployment)

[ClickHouse高可用集群的配置](https://www.cnblogs.com/freeweb/p/9352947.html)

分片的副本不能与其他分片在同一个节点的同一个端口上，ClickHouse在查询的时候会查询每个分片的一个副本数据，但是如果一个节点端口上设置了一个分片个另一个分片的副本，它们将公用本地表，这样会导致查询数据重复。

比如，3个分片，每个分片2个副本，就需要6台服务器，副本与分片需要分开，如果Master分片设置的副本在Node1上，而Node1上又有一个分片，那么Node1上的分片将与Master副本共享本地表，查询数据时会将分片和副本的数据全部查询出来，
这就导致Master分片的数据查了两份导致数据重复。

**解决办法**

- 使用多个服务集群，一个集群中的节点当作数据分片，另两个集群中的节点分别为副本(官方推荐)
- 在一台服务器上启用多个端口实例

ClickHouse群集是同质群集。设置步骤：

> 1. 在群集的所有计算机上安装ClickHouse服务器
> 2. 在配置文件中设置集群配置
> 3. 在每个实例上创建本地表
> 4. 创建一个分布式表

分布式表实际上是ClickHouse群集本地表的一种“视图”。来自分布式表的SELECT查询将使用所有群集分片的资源来执行。您可以为多个集群指定配置，并创建多个分布式表以提供对不同集群的视图。

# 1 集群配置示例：单集群（3分片0副本）


## 1.1 在集群每台节点上安装ClickHouse-server

```xml
//将安装包发送到其他节点Node1
root@Master-fzy-> scp /var/cache/apt/archives/clickhouse-client_20.3.4.10_all.deb root@Node1-fzy:/var/cache/apt/archives/clickhouse-client_20.3.4.10_all.deb
root@Master-fzy-> scp /var/cache/apt/archives/clickhouse-common-static_20.3.4.10_amd64.deb  root@Node1-fzy:/var/cache/apt/archives/clickhouse-common-static_20.3.4.10_amd64.deb
root@Master-fzy-> scp /var/cache/apt/archives/clickhouse-server_20.3.4.10_all.deb  root@Node1-fzy:/var/cache/apt/archives/clickhouse-server_20.3.4.10_all.deb
//将安装包发送到其他节点Node2
root@Master-fzy-> scp /var/cache/apt/archives/clickhouse-client_20.3.4.10_all.deb root@Node2-fzy:/var/cache/apt/archives/clickhouse-client_20.3.4.10_all.deb
root@Master-fzy-> scp /var/cache/apt/archives/clickhouse-common-static_20.3.4.10_amd64.deb  root@Node2-fzy:/var/cache/apt/archives/clickhouse-common-static_20.3.4.10_amd64.deb
root@Master-fzy-> scp /var/cache/apt/archives/clickhouse-server_20.3.4.10_all.deb  root@Node2-fzy:/var/cache/apt/archives/clickhouse-server_20.3.4.10_all.deb

//Node1和Node2分别执行下面三个命令安装
root@Node1-fzy-> dpkg -i /var/cache/apt/archives/clickhouse-common-static_20.3.4.10_amd64.deb
root@Node1-fzy-> dpkg -i /var/cache/apt/archives/clickhouse-server_20.3.4.10_all.deb
root@Node1-fzy-> dpkg -i /var/cache/apt/archives/clickhouse-client_20.3.4.10_all.deb
...                   
```

## 1.2 users.xml同步用户设置

```xml
#### 将Master节点上的users.xml发送到Node1、Node2上，该配置上有两个用户，<default: > <fpcAdmin:3Y4V0Acx>

root@Master-fzy-> scp /etc/clickhouse-server/users.xml root@Node1-fzy:/etc/clickhouse-server/users.xml
root@Master-fzy-> scp /etc/clickhouse-server/users.xml root@Node2-fzy:/etc/clickhouse-server/users.xml

#### 验证是否配置成功
root@Node1-fzy-> clickhouse-client --port 9011 --user fpcAdmin --password 3Y4V0Acx
root@Node2-fzy-> clickhouse-client --port 9011 --user fpcAdmin --password 3Y4V0Acx

```

## 1.3 config.xml引入集群配置文件

`/etc/clickhouse-server/config.xml`配置文件中有一个`<remote_servers>`标签的配置如下，这就是集群配置，可以直接在此处配置，也可以提出来配置到扩展文件中。`incl`属性表示可从外部文件中获取节点名为**clickhouse_remote_servers**的配置内容。

```xml
<yandex>
    <!-- 大概70行，放开http连接ip -->
    <listen_host>::</listen_host>
	<!-- ★大概112行，设置数据存储目录，此目录需要先创建-->
	<path>/soft/clickhouse/</path>
    <!-- 大概在第155行，设置时区为东八区-->
    <timezone>Asia/Shanghai</timezone>

    <!-- ★设置扩展配置文件的路径-->
    <include_from>/etc/clickhouse-server/metrika.xml</include_from>

    <!-- 大概在160附近，★注释其中配置的用于测试分布式存储的分片配置-->
	<remote_servers incl="clickhouse_remote_servers">
	    <!-- 用于测试分布式存储的分片配置，需要注释掉 -->
	    <!-- <test_shard_localhost>
	    ……
	    </test_unavailable_shard> -->
	</remote_servers>
	
	<!-- 大概在265附近，★使用incl属性引入外部配置中的zookeeper-servers标签，用于配置-->
	<zookeeper incl="zookeeper-servers" optional="true" />
	
</yandex>
```

## 1.4 metrika.xml配置集群信息

通过`vim /etc/clickhouse-server/metrika.xml`命令创建一个配置文件，内容如下：

- `clickhouse_remote_servers`标签中用于配置集群的host和port，其中`user`和`password`分别是对应节点上ClickHouse-server的用户名密码，每个节点用户名密码可以不一样，但是为了方便推荐设置为一样的。
- `zookeeper-servers`用于配置zookeeper集群信息，在config.xml中被引用到了
- `<macros>`用于配置当前节点分片自己的主机名或者ip，每个节点上这个配置是不同的，需要单独修改

可创建一个metrika.xml文件，将下面的配置内容复制粘贴（注意配置文件中如果有中文注释应该删除，或者保存编码为**Unicode**，避免编码错乱导致不能解析xml配置文件），然后移动到目录下：

```root@Master-fzy-> mv /tmp/guest-dB7CsM/Desktop/metrika.xml /etc/clickhouse-server/metrika.xml```

```xml
<yandex>
	<clickhouse_remote_servers>    <!-- clickhouse_remote_servers 和 config.xml中的incl属性值对应 -->
		<!-- cluster_3shards_1replicas是集群名(3分片1备份)，可以随便取名, 共设置3个分片，每个分片只有1个副本 -->
		<cluster_3shards_1replicas>
			<!-- 数据分片1  -->
			<shard>
				<!-- replica用于给分片添加副本，可以有多个副本 -->
				<replica>
					<host>Master-fzy</host>
					<port>9011</port>
					<!-- 配置Master-fzy上ClickHouse-server的用户名密码-->
					<user>fpcAdmin</user>
					<password>3Y4V0Acx</password>
				</replica>
				 <!-- 为第一个分区添加一个副本，注意此处不能与其他分片公用服务器端口，会导致数据重复 -->
				 <!-- 由于Node1-fzy在下面配置了分片，Master-fzy副本应该重新选择一台空的节点，或者使用Node1-fzy的其他端口实例 -->
				<!-- <replica>  
					<host>Node1-fzy</host>
					<port>9012</port>
					<user>fpcAdmin</user>
					<password>3Y4V0Acx</password>
				</replica> -->
			</shard>
			<!-- 数据分片2  -->
			<shard>
				<replica>
					<host>Node1-fzy</host>
					<port>9011</port>
					<user>fpcAdmin</user>
					<password>3Y4V0Acx</password>
				</replica>
			</shard>
			<!-- 数据分片3  -->
			<shard>
				<replica>
					<host>Node2-fzy</host>
					<port>9011</port>
					<user>fpcAdmin</user>
					<password>3Y4V0Acx</password>
				</replica>
			</shard>
		</cluster_3shards_1replicas>
	</clickhouse_remote_servers>
	
	<!--zookeeper相关配置-->
	<zookeeper-servers>
		<node index="1">
			<host>Master-fzy</host>
			<port>2181</port>
		</node>
		<node index="2">
			<host>Node1-fzy</host>
			<port>2181</port>
		</node>
		<node index="3">
			<host>Node2-fzy</host>
			<port>2181</port>
		</node>
	</zookeeper-servers>

	<!-- 改为当前节点的主机名 或者ip-->
	<macros>
		<replica>Master-fzy</replica>
	</macros>
	
	<networks>
		<ip>::/0</ip>
	</networks>
	
	<clickhouse_compression>
		<case>
			<min_part_size>10000000000</min_part_size>
			<min_part_size_ratio>0.01</min_part_size_ratio>
			<method>lz4</method>
		</case>
	</clickhouse_compression>
	
</yandex>
```

## 1.5 其他节点配置

```xml
#### 将配置发送到其他节点
root@Master-fzy-> scp /etc/clickhouse-server/config.xml root@Node1-fzy:/etc/clickhouse-server/config.xml
root@Master-fzy-> scp /etc/clickhouse-server/metrika.xml root@Node1-fzy:/etc/clickhouse-server/metrika.xml
root@Master-fzy-> scp /etc/clickhouse-server/config.xml root@Node2-fzy:/etc/clickhouse-server/config.xml
root@Master-fzy-> scp /etc/clickhouse-server/metrika.xml root@Node2-fzy:/etc/clickhouse-server/metrika.xml

#### 修改metrika.xml中的macros标签，改为自己的主机名
root@Node1-fzy-> vim /etc/clickhouse-server/metrika.xml
root@Node2-fzy-> vim /etc/clickhouse-server/metrika.xml

#### 重启服务（如果配置没有出错，其实可以不用重启）
root@Master-fzy-> service clickhouse-server restart
root@Node1-fzy-> service clickhouse-server restart
root@Node2-fzy-> service clickhouse-server restart
```

## 1.6 查看集群

```xml
#### 启动client
root@Master-fzy-> clickhouse-client --port 9011 --user fpcAdmin --password 3Y4V0Acx
ClickHouse client version 20.3.4.10 (official build).
Connecting to localhost:9011 as user default.
Connected to ClickHouse server version 20.3.4 revision 54433.

#### ★查看集群信息
Master-fzy :) SELECT * FROM system.clusters;
SELECT *
FROM system.clusters

┌─cluster───────────────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name──┬─host_address──┬─port─┬─is_local─┬─user─────┬─default_database─┬─errors_count─┬─estimated_recovery_time─┐
│ cluster_3shards_1replicas │         1 │            1 │           1 │ Master-fzy │ 192.168.1.190 │ 9011 │        1 │ fpcAdmin │                  │            0 │                       0 │
│ cluster_3shards_1replicas │         2 │            1 │           1 │ Node1-fzy  │ 192.168.1.191 │ 9011 │        0 │ fpcAdmin │                  │            0 │                       0 │
│ cluster_3shards_1replicas │         3 │            1 │           1 │ Node2-fzy  │ 192.168.1.192 │ 9011 │        0 │ fpcAdmin │                  │            0 │                       0 │
└───────────────────────────┴───────────┴──────────────┴─────────────┴────────────┴───────────────┴──────┴──────────┴──────────┴──────────────────┴──────────────┴─────────────────────────┘

```

# 2 集群配置示例：多集群方式（3分片，每个分片配置1个副本）



# 3 集群配置示例：单集群多端口实例（3分片，每个分片配置1个副本）

