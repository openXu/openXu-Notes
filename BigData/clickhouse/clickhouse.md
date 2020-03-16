

## 1. 安装

[快速入门](https://clickhouse.tech/#quick-start)

```xml
apt-get install dirmngr
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E0C56BD4

#### 添加源
echo "deb http://repo.yandex.ru/clickhouse/deb/stable/ main/" | tee /etc/apt/sources.list.d/clickhouse.list
apt-get update

#### 安装
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

## 2. 配置讲解

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


## 3. mysql增量同步

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

## 4. spark clickhouse

**配置clickhouse-service**

```xml
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
	<tcp_port>9011</tcp_port>
	<http_port>8123</http_port>
	放开注释
	<listen_host>0.0.0.0</listen_host>

#### 重启
root@Master:~# service clickhouse-server restart

#### 查看日志，是否启动成功
root@Master:~# cat /var/log/clickhouse-server/clickhouse-server.log

#### 指定端口启动client，也可修改client配置文件中默认端口后不指定端口启动
root@Master:~# clickhouse-client --password --port 9011

```

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



## ClickHouse文档

[ClickHouse文档](https://clickhouse.tech/docs/en/single/?query=internal_replication#ontime)

### SQL支持

ClickHouse支持一种基于SQL的声明性查询语言，该语言在许多情况下都与SQL标准相同。支持的查询包括GROUP BY，ORDER BY，FROM，
IN和JOIN子句中的子查询以及标量子查询。不支持相关的子查询和窗口功能。


## SQL语法

**数据库**

```xml
#### 创建数据库[IF NOT EXISTS]可选，以下带[]都是可选的意思
Master :) CREATE DATABASE [IF NOT EXISTS] openxu

CREATE DATABASE IF NOT EXISTS openxu

Ok.

0 rows in set. Elapsed: 0.043 sec. 
#### 查看数据库
Master :) show databases

SHOW DATABASES

┌─name────┐
│ default │
│ openxu  │
│ system  │
└─────────┘

3 rows in set. Elapsed: 0.059 sec. 

#### 创建表
Master :) CREATE TABLE IF NOT EXISTS openxu.user(id UInt8,name String,age UInt8) ENGINE = MergeTree() PARTITION BY age ORDER BY name

#### 插入数据
Master :) INSERT INTO openxu.user (id,name,age) VALUES (1,'andy',26),(2,'sanzen',30)

INSERT INTO openxu.user (id, name, age) VALUES

Ok.

2 rows in set. Elapsed: 0.032 sec. 


#### 查询



    private String taskId;
    private String taskName;
    private String companyId;
    private String companyName;
    private int taskStatus;
    private String createdDate;
    private String taskStartTime;
    private String taskEndTime;
    private String taskRemindTime;
    private int completeItemCount;
    private int normalItemCount;
    private String modifiedDate;
    private int totalItemCount;




public static final String SAVE_ORGANISEUNIT_INFO = "INSERT INTO test.simu_org(,,," +
            "CompanyType,,) VALUES(?,?,?,?,?,?)";
    public static final String SAVE_TASK_INFO = "INSERT INTO test.simu_task(," +
            "TaskStatus,CreatedDate,TaskStartTime,TaskEndTime,TaskRemindTime,CompleteItemCount,NormalItemCount," +
            ") VALUES(?,?,?,?,?,?,?,?,?,?,?,?)";


```

**表操作**

```xml
#### 创建表
CREATE TABLE IF NOT EXISTS mysql_service.OrgUnit(OrganiseUnitID String,OrganiseUnitName String,OrganiseUnitClass UInt16,CompanyType UInt8,CreatedDate String,ModifiedDate String) ENGINE = MergeTree() PARTITION BY CreatedDate ORDER BY (ModifiedDate,OrganiseUnitID);

CREATE TABLE IF NOT EXISTS mysql_service.Task(TaskID String,TaskName String,CompanyID String,TaskStatus UInt8,CreatedDate String,TaskStartTime String,TaskEndTime String,TaskRemindTime String, CompleteItemCount UInt8,NormalItemCount UInt8,TotalItemCount UInt8,ModifiedDate String)ENGINE = MergeTree() PARTITION BY CreatedDate ORDER BY (ModifiedDate,TaskID);

#### 删除表
DROP table db.本地表 ON CLUSTER cluster_name;

#### 删除数据
alter table OrgUnit delete where OrganiseUnitID !='';
alter table Task delete where TaskID !='';


    
```

**查询**

```xml

select OrganiseUnitID,OrganiseUnitName,ModifiedDate from OrgUnit;

select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount from (select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate, any(CompleteItemCount) as CompleteItemCount from Task group by TaskID order by ModifiedDate DESC) t1 left join (select OrganiseUnitID,OrganiseUnitName from OrgUnit) t2 on t1.CompanyID=t2.OrganiseUnitID;
 
 
select * FROM sdms_organiseunit WHERE OrganiseUnitID = '0005dc7e-c08f-11e8-bf3a-fa163e4635ff';

alter table OrgUnit delete where OrganiseUnitID !='';
alter table Task delete where TaskID !='';

select count(*) from OrgUnit;

select count(*) from Task;

--查询某个单位的所有任务
select * from Task where CompanyID = '0000_1_100';


--错误
select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount from (
		select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate, 
		any(CompleteItemCount) as CompleteItemCount from Task group by TaskID order by ModifiedDate DESC) t1 
	left join (
		select OrganiseUnitID,OrganiseUnitName from OrgUnit) t2
	on t1.CompanyID=t2.OrganiseUnitID;

--查询某个单位所有最新任务
select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from (select * from Task order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from OrgUnit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID where CompanyID = '0000_1_100';


--查询单位所有最新任务
select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
	from (
		select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
		any(CompleteItemCount) as CompleteItemCount from (select * from Task order by ModifiedDate DESC) group by TaskID 
	) t1 
	left join (
		select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from OrgUnit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
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
any(CompleteItemCount) as CompleteItemCount from (select * from Task order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from OrgUnit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID where CompanyID = '0000_1_100') GROUP by OrganiseUnitID; 
 
select 
COUNT(TaskID) as TotalTaskCount,
OrganiseUnitID,
any(OrganiseUnitName) as OrganiseUnitName,
SUM(CompleteItemCount) as CompleteItemCount
from(select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from Task group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from OrgUnit group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID where CompanyID = '0000_1_100') GROUP by OrganiseUnitID;

--统计单位任务
select 
COUNT(TaskID) as TotalTaskCount,
OrganiseUnitID,
any(OrganiseUnitName) as OrganiseUnitName,
SUM(CompleteItemCount) as CompleteItemCount
from(select TaskID, TaskName, OrganiseUnitID, OrganiseUnitName, ModifiedDate, CompleteItemCount 
from (
select any(CompanyID) as CompanyID, TaskID, any(TaskName) as TaskName, MAX(ModifiedDate) as ModifiedDate,
any(CompleteItemCount) as CompleteItemCount from (select * from Task order by ModifiedDate DESC) group by TaskID 
) t1 
left join (
select OrganiseUnitID ,any(OrganiseUnitName) as OrganiseUnitName from (select * from OrgUnit order by ModifiedDate DESC) group by OrganiseUnitID) t2 
on t1.CompanyID=t2.OrganiseUnitID) GROUP by OrganiseUnitID order by OrganiseUnitID; 


```















