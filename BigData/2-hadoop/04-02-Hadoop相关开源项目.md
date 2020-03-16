

# Hive

Hive是一个构建在Hadoop上的数据仓库框架，其设计目的是让精通SQL技能但Java编程技能相对弱的分析师能对Faceb
存放在HDFS中大规模数据集执行查询。Hive一般在工作站上运行，它把SQL查询转换为一系列在Hadoop集群上运行的作业。
Hive把数据组织为表，通过这种方式存储在HDFS上的数据赋予结构。存储在metastore数据库中。

## 快速入门

### 安装Hive

```xml
#### 下载Hive(http://hive/apache.org/downloads.html)，解压
% tar xzf apache-hice-x.y.z-bin.tar.gz
#### 配置环境
% export HIVE_HOME=~/soft/apache-hice-x.y.z-bin
% export PATH=$PATH:$HIVE_HOME/bin

#### 启动Hive的shell环境
% hive
hice>
```

由于我之前安装了CDH，只需要在集群中添加Hive服务即可，数据库连接配置如下：

类型 : MySQL
Use JDBC URL Override : 是
JDBC URL : jdbc:mysql://127.0.0.1:3306/hive
用户名:hive
密码：123456

### Hive的shell环境

Hive的shell环境是我们和Hive交互、发出HiveQL命令的主要方式。HiveQL是Hive的查询语言，它是SQL的一种方言。

```xml
#### 列出Hive的表(大小写不敏感)
hice> SHOW TABLES;

#### 非交互模式运行Hive shell，使用-f选项运行指定文件中的命令
% hive -f script.q

#### 运行内嵌短脚本命令
% hive -e 'SELECT * FROM user'

#### -S选项强制不显示运行过程信息
% hive -S -e 'SELECT * FROM user'

```



## Hive与传统数据库相比



## HiveQL




## 表


## 查询数据


































