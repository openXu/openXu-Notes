# 1 创建分布式表

[分布式引擎Distributed](https://clickhouse.tech/docs/zh/operations/table_engines/distributed/)

分布表（Distributed）本身不存储数据，相当于路由，在创建时需要指定集群名、数据库名、数据表名、分片KEY，这里分片用rand()函数，表示随机分片。查询分布式表会根据集群配置信息，路由到具体的数据表，再把结果进行合并。

```xml
#### ★1. 创建每个分片的本地表（以下创建数据库&创建本地表分别需要在每个节点上执行一遍）
root@Master-fzy-> clickhouse-client --port 9011 --user fpcAdmin --password 3Y4V0Acx
#### 1.1 创建数据库
Master-fzy :) CREATE DATABASE IF NOT EXISTS mysql_service;
#### 1.2创建单位表
Master-fzy :) CREATE TABLE IF NOT EXISTS mysql_service.unit_local(
OrganiseUnitID String,
OrganiseUnitName String,
OrganiseUnitClass UInt16,
CompanyType UInt8,
CreatedDate DateTime,
ModifiedDate DateTime)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(CreatedDate)
ORDER BY (ModifiedDate,OrganiseUnitID);
#### 1.3 创建任务表
Master-fzy :) CREATE TABLE IF NOT EXISTS mysql_service.task_local(
TaskID String,
TaskName String,
CompanyID String,
TaskStatus UInt8,
CreatedDate DateTime,
TaskStartTime DateTime,
TaskEndTime DateTime,
TaskRemindTime DateTime, 
CompleteItemCount UInt8,
NormalItemCount UInt8,
TotalItemCount UInt8,
ModifiedDate DateTime)
ENGINE = MergeTree() 
PARTITION BY toYYYYMM(CreatedDate) 
ORDER BY (ModifiedDate,TaskID);

#### ★2.创建分布式表，我这里选择Master节点（随意）
#### 分布式引擎Distributed参数(集群名，远程数据库名，远程表名，数据分片键（可选）)
Master-fzy :) CREATE TABLE unit_all AS unit_local
ENGINE = Distributed(cluster_3shards_1replicas, mysql_service, unit_local, rand());

Master-fzy :) CREATE TABLE task_all AS task_local
ENGINE = Distributed(cluster_3shards_1replicas, mysql_service, task_local, rand());
#### 查看Master节点的表
Master-fzy :) use mysql_service;
Master-fzy :) show tables;
SHOW TABLES
┌─name───────┐
│ task_all   │
│ task_local │
│ unit_all   │
│ unit_local │
└────────────┘

```

# 2 填充测试数据

CREATE TABLE IF NOT EXISTS mysql_service.streamset_unit(
OrganiseUnitID String,
OrganiseUnitName String,
OrganiseUnitClass UInt16,
CompanyType UInt8,
CreatedDate DateTime,
ModifiedDate DateTime)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(CreatedDate)
ORDER BY (ModifiedDate,OrganiseUnitID);


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
100000 rows in set. Elapsed: 0.872 sec. Processed 431.07 thousand rows, 19.63 MB (494.31 thousand rows/s., 22.51 MB/s.)
100000 rows in set. Elapsed: 0.950 sec. Processed 300.00 thousand rows, 13.68 MB (315.71 thousand rows/s., 14.40 MB/s.)












