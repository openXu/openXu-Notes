
# 1. 单节点

[快速入门](https://clickhouse.tech/#quick-start)

```xml
apt-get install dirmngr
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E0C56BD4

#### 添加源
echo "deb http://repo.yandex.ru/clickhouse/deb/stable/ main/" | tee /etc/apt/sources.list.d/clickhouse.list
apt-get update

#### 安装，默认下载的文件在/var/cache/apt/archives/目录下
apt-get install -y clickhouse-server clickhouse-client

	设置默认用户密码：123456

#### 启动clickhouse服务，自动使用/etc/clickhouse-server/config.xml作为配置文件
#### 也可以手动启动clickhouse-server --config=/etc/clickhouse-server/config.xml
service clickhouse-server start

#### 使用客户端
clickhouse-client --password
	
	输入默认用户密码：123456
	ClickHouse client version 20.1.6.30 (official build).
	Password for user (default): 
	Connecting to localhost:9000 as user default.
	Connected to ClickHouse server version 20.1.6 revision 54431.

Master :) select 1
SELECT 1
┌─1─┐
│ 1 │
└───┘
1 rows in set. Elapsed: 0.052 sec. 

#### 退出
exit
```

# 2 配置讲解

我们在已安装的软件包中有什么:

**clickhouse-client** 该软件包包含clickhouse-client应用程序，交互式ClickHouse控制台客户端。
**clickhouse-common** 程序包包含ClickHouse可执行文件。
**clickhouse-server** 软件包包含用于将ClickHouse作为服务器运行的配置文件

服务器配置文件位于中`/etc/clickhouse-server/`，`config.xml`的`<path>`元素确定数据存储的路径，
因此它应位于具有大磁盘容量的卷上，默认值为`/var/lib/clickhouse/`。如果要调整配置，直接编辑
`config.xml`文件并不是很方便，因为将来的软件包更新可能会重写该文件。推荐的覆盖config元素的方法
是在`config.d`目录中创建文件，这些文件充当config.xml的“补丁”。

您可能已经注意到，clickhouse-server软件包安装后不会自动启动，更新后也不会自动重新启动。启动服务
器的方式取决于您的init系统，通常是`sudo service clickhouse-server start`或者`/etc/init.d/clickhouse-server start`

服务器日志的默认位置是`/var/log/clickhouse-server/`。Ready for connections记录消息后，服务器将准备好处理客户端连接。

一旦clickhouse-server启动并运行，我们就可以clickhouse-client用来连接到服务器并运行一些测试查询，例如SELECT "Hello, world!";。

# 3 可视化工具dbeaver

打开dbeaver，创建一个ClickHouse连接，设置JDBC连接的主机名ip和端口8123，然后连接，如果连接不上，需要修改ClickHouse-server的配置文件，开放ClickHouse服务的ip和端口：

```xml
#### 修改配置文件
root@Master-fzy-> vim /etc/clickhouse-server/config.xml

	<!-- Listen specified host. use :: (wildcard IPv6 address), if you want to accept connections both with IPv4 and IPv6 from everywhere. -->
	<listen_host>::</listen_host>   <!--放开这段配置注释-->
	<!-- Same for hosts with disabled ipv6: -->
	<!-- <listen_host>0.0.0.0</listen_host> -->
	
#### 重启
root@Master:~# service clickhouse-server restart
```
# 4 修改TCP端口

```xml
PS:端口冲突导致启动失败，可用下面的方法排障

#### clickhouse-server默认tcp端口为9000，先看一下9000端口有没有被占用，如果被占用，可修改<tcp_port>9011</tcp_port>
root@Master:~# netstat -anlp|grep 9000
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      73641/clickhouse-se 
tcp        0      0 192.168.85.200:9000     0.0.0.0:*               LISTEN      72062/python2       
tcp6       0      0 ::1:9000                :::*                    LISTEN      73641/clickhouse-se 
#### 尝试杀掉被占用的端口程序（可选方案）
root@Master:~# kill -9 72062
#### 修改配置文件
root@Master:~# vim /etc/clickhouse-server/config.xml
	
	修改端口
	<tcp_port>9011</tcp_port>    //clickhouse-client就是通过tcp端口访问
	<http_port>8123</http_port>  //JDBC等连接是通过http端口访问

#### 重启
root@Master:~# service clickhouse-server restart

#### 查看日志，是否启动成功
root@Master:~# cat /var/log/clickhouse-server/clickhouse-server.log

#### 使用client会报错，因为它会去连接默认的tcp端口9000
root@Master:~# clickhouse-client --password
	ClickHouse client version 20.3.4.10 (official build).
	Connecting to localhost:9000 as user default.
	Code: 210. DB::NetException: Connection refused (localhost:9000)

#### 指定端口启动client，也可修改client配置文件中默认端口后不指定端口启动
root@Master:~# clickhouse-client --password --port 9011

#### 如果想要不指定--port参数，我们需要修改clickhouse-client默认端口

```
# 5. 用户名密码

[ss](https://blog.csdn.net/github_39577257/article/details/103066747#3.1)
为了不让ClickHouse裸奔，现在我们配置一下用户认证部分。密码配置有两种方式，一种是明文方式，一种是密文方式（sha256sum的Hash值）。
官方推荐使用密文作为密码配置，在Master、Node1、Node2三个机器上配置`/etc/clickhouse-server/users.xml`文件。用户名和密码的配置
主要是在`users`标签中，下面的配置文件中配置了两个用户，一个是默认用户default，就是如果未指明用户时默认使用的用户，其密码配置的
为sha256密文方式，第二个用户是ck，为一个只读用户，即只能查看数据，无法建表修改数据等操作，其密码直接采用的明文方式进行配置。

在`users.xml`配置文件`<users>`标签中添加`<fpcAdmin>`表示配置一个名为`fpcAdmin`的新用户，在里面配置密码，和用户配置信息，如下：

注意配置文件保存编码为**Unicode**

```xml
<?xml version="1.0"?>
<yandex>
    <!-- Profiles of settings. -->
    <profiles>
        <!-- Default settings. -->
        <default>
            <!-- Maximum memory usage for processing single query, in bytes. -->
            <max_memory_usage>10000000000</max_memory_usage>
            <!-- Use cache of uncompressed blocks of data. Meaningful only for processing many of very short queries. -->
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <!-- How to choose between replicas during distributed query processing.
                 random - choose random replica from set of replicas with minimum number of errors
                 nearest_hostname - from set of replicas with minimum number of errors, choose replica
                  with minimum number of different symbols between replica's hostname and local hostname
                  (Hamming distance).
                 in_order - first live replica is chosen in specified order.
                 first_or_random - if first replica one has higher number of errors, pick a random one from replicas with minimum number of errors.
            -->
            <load_balancing>random</load_balancing>
        </default>
        <!-- Profile that allows only read queries. -->
        <readonly>
            <readonly>1</readonly>
        </readonly>
    </profiles>

    <!-- Users and ACL. -->
    <users>
        <!-- 如果没有特殊指定用户, 将默认使用default用户. -->
        <default>
			<!-- Password could be specified in plaintext or in SHA256 (in hex format).
		
		                 If you want to specify password in plaintext (not recommended), place it in 'password' element.
		                 Example: <password>qwerty</password>.
		                 Password could be empty.
		
		                 If you want to specify SHA256, place it in 'password_sha256_hex' element.
		                 Example: <password_sha256_hex>65e84be33532fb784c48129675f9eff3a682b27168c0ea744b2cf58ee02337c5</password_sha256_hex>
		                 Restrictions of SHA256: impossibility to connect to ClickHouse using MySQL JS client (as of July 2019).
		
		                 If you want to specify double SHA1, place it in 'password_double_sha1_hex' element.
		                 Example: <password_double_sha1_hex>e395796d6546b1b65db9d665cd43f0e858dd4303</password_double_sha1_hex>
		
		                 How to generate decent password:
		                 Execute: PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
		                 In first line will be password and in second - corresponding SHA256.
		
		                 How to generate double SHA1:
		                 Execute: PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha1sum | tr -d '-' | xxd -r -p | sha1sum | tr -d '-'
		                 In first line will be password and in second - corresponding double SHA1.
		            -->
            <!-- 密码可以用明文或SHA256（十六进制格式）指定.

                 如果您想用纯文本（不推荐）指定密码，请将其放在“password”元素中。
                 示例：<password>qwerty</password>。
                 密码可能为空。

                 如果要指定SHA256，请将其放在“password_sha256_hex”元素中。
				 示例：<password_sha256_hex>65e...337c5</password_sha256_hex>
                 SHA256的限制：无法使用MySQL JS客户端连接到ClickHouse（截至2019年7月）。

                 如果要指定double SHA1，请将其放入“password_double_sha1_hex”元素。
				 示例：<password_double_sha1_hex>e395796d6546b1b65db9d665cd43f0e858dd4303</password_double_sha1_hex>


				如何生成合适的密码：
				执行：PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
				第一行是密码，第二行是SHA256。

				如何生成双SHA1：
				执行：PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha1sum | tr -d '-' | xxd -r -p | sha1sum | tr -d '-'
				第一行是密码，第二行是对应的双SHA1。
            -->
            <password></password>

            <!-- List of networks with open access.
                 To open access from everywhere, specify:
                    <ip>::/0</ip>
                 To open access only from localhost, specify:
                    <ip>::1</ip>
                    <ip>127.0.0.1</ip>
                 Each element of list has one of the following forms:
                 <ip> IP-address or network mask. Examples: 213.180.204.3 or 10.0.0.1/8 or 10.0.0.1/255.255.255.0
                     2a02:6b8::3 or 2a02:6b8::3/64 or 2a02:6b8::3/ffff:ffff:ffff:ffff::.
                 <host> Hostname. Example: server01.yandex.ru.
                     To check access, DNS query is performed, and all received addresses compared to peer address.
                 <host_regexp> Regular expression for host names. Example, ^server\d\d-\d\d-\d\.yandex\.ru$
                     To check access, DNS PTR query is performed for peer address and then regexp is applied.
                     Then, for result of PTR query, another DNS query is performed and all received addresses compared to peer address.
                     Strongly recommended that regexp is ends with $
                 All results of DNS requests are cached till server restart.
            -->
            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>

            <!-- Settings profile for user. -->
            <profile>default</profile>

            <!-- Quota for user. -->
            <quota>default</quota>
        </default>
		<!-- ★配置新用户 -->
        <fpcAdmin>
        	<!-- 设置sha256密码，明文为 3Y4V0Acx -->
            <password_sha256_hex>fe2188eda898c46880ae21f0ca06906f265730ff16914f515420adb06f86d087</password_sha256_hex>
            <!-- 具有开放访问权限的网络列表。要从任何地方打开访问，请指定::/0 -->
			<networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>

            <!-- 设置用户配置文件，对应上面的profiles标签中的子标签. -->
            <profile>default</profile>

            <!-- Quota for user. -->
            <quota>default</quota>
            
        </fpcAdmin>

    </users>

    <!-- Quotas. -->
    <quotas>
        <!-- Name of quota. -->
        <default>
            <!-- Limits for time interval. You could specify many intervals with different limits. -->
            <interval>
                <!-- Length of interval. -->
                <duration>3600</duration>

                <!-- No limits. Just calculate resource usage for time interval. -->
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <read_rows>0</read_rows>
                <execution_time>0</execution_time>
            </interval>
        </default>
    </quotas>
</yandex>
```

使用新用户登陆Client:

```xml
root@Master-fzy-> clickhouse-client --port 9011 --user fpcAdmin --password 
输入密码：3Y4V0Acx
```



















# 3. mysql增量同步

```xml

clickhouse-client -u default --password
123456
#### 创建数据库
Master :) CREATE DATABASE IF NOT EXISTS mysql_service;
#### 同步组织机构表
Master :) CREATE TABLE IF NOT EXISTS mysql_service.sdms_organiseunit ENGINE = MergeTree ORDER BY OrganiseUnitCode AS SELECT * FROM mysql('114.115.144.251:33061','iipplat_shuishan','sdms_organiseunit','root','111aaa***');
Master :) SELECT * FROM mysql_service.sdms_organiseunit;
#### 同步任务表
Master :) CREATE TABLE IF NOT EXISTS mysql_service.exam_examinetask ENGINE = MergeTree ORDER BY TaskCode AS SELECT * FROM mysql('114.115.144.251:33061','iipplat_shuishan','exam_examinetask','root','111aaa***');
Master :) SELECT * FROM mysql_service.exam_examinetask;
#### 连表查询
Master :) use mysql_service;
Master :) select CompanyID, CompanyName, TaskNum from 
				(select CompanyID, COUNT(1) AS TaskNum from exam_examinetask group by CompanyID) t1 
				left join 
				(select OrganiseUnitName as CompanyName, OrganiseUnitID from sdms_organiseunit) t2 
				on t1.CompanyID=t2.OrganiseUnitID;

┌─CompanyID────────────────────────────┬─CompanyName────────────────────┬─TaskNum─┐
│ 857bd450-35a9-11e9-8592-fa163e4635ff │ 齐北辆场院维修库               │      19 │
│ 47b6dfe0-34b8-11e9-8592-fa163e4635ff │ 首尔大厦                       │       2 │
│ 423f507f-04bb-11e8-814d-fa163e4635ff │ 123测试                        │     432 │
...
│ 7a36dde1-1585-11e9-bf3a-fa163e4635ff │ 法之运维保单位                 │      10 │
└──────────────────────────────────────┴────────────────────────────────┴─────────┘
共54排。已用时间：0.383秒。已处理5779 000行，2.77 MB（15089 000行/秒，7.23 MB/秒）

```

# 4. spark clickhouse

**maven依赖**

```xml
	<properties>
        <spark.version>2.4.0</spark.version>
        <scala.version>2.11</scala.version>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <!--lombok getter\setter log-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.20</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-sql -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-streaming -->
<!--        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_${scala.version}</artifactId>
            <version>${spark.version}</version>
            &lt;!&ndash;   <scope>provided</scope>&ndash;&gt;
        </dependency>-->
        <!--mysql连接驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <!--java.lang.ClassNotFoundException: org.codehaus.janino.InternalCompilerException-->
        <dependency>
            <groupId>org.codehaus.janino</groupId>
            <artifactId>janino</artifactId>
            <version>3.0.8</version>
        </dependency>
        <!--ClickHouse驱动包-->
        <dependency>
            <groupId>ru.yandex.clickhouse</groupId>
            <artifactId>clickhouse-jdbc</artifactId>
            <version>0.1.40</version>
        </dependency>
    </dependencies>
```

**通过Spark将mysql中数据插入到clickhouse**

```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.{SaveMode, SparkSession}
object SparkClickHouse {
  def main(args: Array[String]): Unit = {
    readMysql()
  }

  def readMysql(): Unit ={
    val spark = SparkSession.builder
      .master("local")
      .appName("SparkClickHouse")
      .getOrCreate()
    val jdbcDF = spark.read
      .format("jdbc")
      .option("url", "jdbc:mysql://114.115.144.251:33061/iipplat_shuishan?useUnicode=true&characterEncoding=UTF-8&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC")
      .option("driver", "com.mysql.jdbc.Driver")
      .option("dbtable", "(select sdms_organiseunit.OrganiseUnitID,sdms_organiseunit.OrganiseUnitName from sdms_organiseunit) T")
      .option("user", "root")
      .option("password", "111aaa***")
      .load()
//      .show()

    // Saving data to a JDBC source
    jdbcDF.write
      .format("jdbc")
      .option("url", "jdbc:clickhouse://192.168.85.200:8123/mysql_service")
      .option("driver", "ru.yandex.clickhouse.ClickHouseDriver")
      .option("dbtable", "sdms_organiseunit")
      .option("user", "default")
      .option("password", "123456")
      .mode(SaveMode.Append)
      .option("batchsize", "50000")   //默认写入批次是2w，可以调大至5w
      .option("isolationLevel", "NONE")  // 设置事务  clickhouse 不支持事务
      .option("numPartitions", "1")  // 设置并发
      .save()
  }
}

```


# ClickHouse文档

[ClickHouse文档](https://clickhouse.tech/docs/en/single/?query=internal_replication#ontime)



## SQL支持

ClickHouse支持一种基于SQL的声明性查询语言，该语言在许多情况下都与SQL标准相同。支持的查询包括GROUP BY，ORDER BY，FROM，
IN和JOIN子句中的子查询以及标量子查询。不支持相关的子查询和窗口功能。

**数据库**

```xml
#### 创建数据库[IF NOT EXISTS]可选，以下带[]都是可选的意思
CREATE DATABASE [IF NOT EXISTS] openxu
#### 查看数据库
show databases

#### 创建表
CREATE TABLE IF NOT EXISTS openxu.user(id UInt8,name String,age UInt8) ENGINE = MergeTree() PARTITION BY age ORDER BY name
#### 插入数据
INSERT INTO openxu.user (id,name,age) VALUES (1,'andy',26),(2,'sanzen',30)
```

**表操作**

```xml
#### 创建表
CREATE TABLE IF NOT EXISTS mysql_service.unit(OrganiseUnitID String,OrganiseUnitName String,OrganiseUnitClass UInt16,CompanyType UInt8,CreatedDate String,ModifiedDate String) ENGINE = MergeTree() PARTITION BY toYYYYMM(CreatedDate) ORDER BY (ModifiedDate,OrganiseUnitID);
#### 删除表
DROP table db.本地表 ON CLUSTER cluster_name;
#### 删除数据
alter table unit_local delete where OrganiseUnitID !='';
alter table task_local delete where TaskID !='';
#### 清空表
truncate table unit_local;  
```

**查询**

```xml

--查询某个单位的所有任务
select * from mysql_service.task where CompanyID = '0000_11_100';
select * from mysql_service.task_all where CompanyID = '0000_11_100';

--查询某个单位所有最新任务
select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from (select * from mysql_service.task order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from mysql_service.unit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID where CompanyID = '0000_1_100';


--查询单位所有最新任务
select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
	from (
		select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
		any(CompleteItemCount) as CompleteItemCount from (select * from mysql_service.task order by ModifiedDate DESC) group by TaskID 
	) t1 
	left join (
		select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from mysql_service.unit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
	on t1.CompanyID=t2.OrganiseUnitID ORDER BY TaskID ;

--统计某个单位任务
select 
COUNT(TaskID) as TotalTaskCount,
OrganiseUnitID,
any(OrganiseUnitName) as OrganiseUnitName,
SUM(CompleteItemCount) as CompleteItemCount
from(select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from (select * from mysql_service.task order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from mysql_service.unit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID where CompanyID = '0000_1_100') GROUP by OrganiseUnitID; 
 
select 
COUNT(TaskID) as TotalTaskCount,
OrganiseUnitID,
any(OrganiseUnitName) as OrganiseUnitName,
SUM(CompleteItemCount) as CompleteItemCount
from(select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from mysql_service.task group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from mysql_service.unit group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID where CompanyID = '0000_1_100') GROUP by OrganiseUnitID;

--统计单位任务(单节点)
select 
COUNT(TaskID) as TotalTaskCount,
OrganiseUnitID,
any(OrganiseUnitName) as OrganiseUnitName,
SUM(CompleteItemCount) as CompleteItemCount
from(select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from (select * from mysql_service.task order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from mysql_service.unit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID) GROUP by OrganiseUnitID order by OrganiseUnitID; 


--统计单位任务(集群)
select 
COUNT(TaskID) as TotalTaskCount,
OrganiseUnitID,
any(OrganiseUnitName) as OrganiseUnitName,
SUM(CompleteItemCount) as CompleteItemCount
from(select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from (select * from mysql_service.task_all order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from mysql_service.unit_all order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID) GROUP by OrganiseUnitID order by OrganiseUnitID; 


```















