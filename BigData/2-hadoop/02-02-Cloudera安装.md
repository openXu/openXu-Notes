[原文](https://www.cnblogs.com/lion.net/p/5477899.html)

# CDH介绍

### 什么是CDH和CM?
　　CDH一个对Apache Hadoop的集成环境的封装，可以使用Cloudera Manager进行自动化安装。
　　Cloudera-Managerceql(本文中简称CM)是一个工具，CM能够管理一个大的Hadoop cluster并不是一只要下载tar files什么压缩并启动services这么简单。后续有非常多设定、监控等麻烦的事要处理，CM都能够做到，有些类似Casti。Cloudera Manager整合了一列的功能让系统管理者能更方便的维护Hadoop。
　　

### CDH的主要功能?
管理
监控
诊断
集成
### CDH版本衍化
　　Hadoop是一个开源项目，所以很多公司在这个基础进行商业化，Cloudera对hadoop做了相应的改变。
　　Cloudera公司的发行版，我们将该版本称为CDH（Cloudera Distribution Hadoop）。截至目前为止，CDH共有5个版本，其中，前两个已经不再更新，最近的两个，分别是CDH4在Apache Hadoop 2.0.0版本基础上演化而来的），CDH5，它们每隔一段时间便会更新一次。
　　Cloudera以Patch Level划分小版本，比如Patch Level为923.142表示在原生态Apache Hadoop 0.20.2基础上添加了1065个Patch（这些Patch是各个公司或者个人贡献的，在Hadoop jira上均有记录），其中923个是最后一个Beta版本添加的Patch，而142个是稳定版发行后新添加的Patch。由此可见，Patch Level越高，功能越完备且解决的Bug越多。
　　Cloudera版本层次更加清晰，且它提供了适用于各种操作系统的Hadoop安装包，可直接使用apt-get或者yum命令进行安装，更加省事。
　　在CDH5以上的版本中，已经加入了Hadoop2.0的HA单点故障解决方案。可以参考《ubuntu12.04+hadoop2.2.0+zookeeper3.4.5+hbase0.96.2+hive0.13.1分布式环境部署》中的单点故障测试。

### CDH5.1.2支持的主要组件简要介绍
- HTTPFS
　　1：Httpfs是Cloudera公司提供的一个Hadoop Hdfs的一个Http接口，通过WebHDFS REST API 可以对hdfs进行读写等访问
　　2：与WebHDFS的区别是不需要客户端可以访问Hadoop集群的每一个节点，通过Httpfs可以访问放置在防火墙后面的Hadoop集群
　　3：Httpfs是一个Web应用,部署在内嵌的Tomcat中

- HBASE
　　Hbase是Bigtable的开源山寨版本。是建立的Hdfs之上，提供高可靠性、高性能、列存储、可伸缩、实时读写的数据库系统。
　　它介于Bosql和RDBMS之间，仅能通过主键(row key)和主键的Range来检索数据，仅支持单行事务(可通过Hive支持来实现多表Join等复杂操作)。主要用来存储非结构化和半结构化的松散数据。
　　与Hadoop一样，Hbase目标主要依靠横向扩展，通过不断增加廉价的商用服务器，来增加计算和存储能力。

HDFS
　　Hadoop分布式文件系统(HDFS)被设计成适合运行在通用硬件(commodity hardware)上的分布式文件系统。它和现有的分布式文件系统有很多共同点。但同时，它和其他的分布式文件系统的区别也是很明显的。HDFS是一个高度容错性的系统，适合部署在廉价的机器上。HDFS能提供高吞吐量的数据访问，非常适合大规模数据集上的应用。HDFS放宽了一部分POSIX约束，来实现流式读取文件系统数据的目的。HDFS在最开始是作为Apache Nutch搜索引擎项目的基础架构而开发的。HDFS是Apache Hadoop Core项目的一部分。

HIVE
　　Hive 是一个基于 Hadoop 的开源数据仓库工具，用于存储和处理海量结构化数据。它把海量数据存储于 Hadoop 文件系统，而不是数据库，但提供了一套类数据库的数据存储和处理机制，并采用 HQL （类 SQL ）语言对这些数据进行自动化管理和处理。我们可以把 Hive 中海量结构化数据看成一个个的表，而实际上这些数据是分布式存储在 HDFS 中的。 Hive 经过对语句进行解析和转换，最终生成一系列基于 hadoop 的 Map/Reduce 任务，通过执行这些任务完成数据处理。

HUE
　　Hue是CDH专门的一套WEB管理器，它包括3个部分Hue Ui，Hue Server，Hue db。Hue提供所有的CDH组件的Shell界面的接口。你可以在Hue编写MR，查看修改HDFS的文件，管理Hive的元数据，运行Sqoop，编写Oozie工作流等大量工作。

Impala
　　Impala对你存储在Apache Hadoop在HDFS，HBase的数据提供直接查询互动的SQL。除了像Hive使用相同的统一存储平台，Impala也使用相同的元数据，SQL语法（Hive SQL），ODBC驱动程序和用户界面（Hue Beeswax）。Impala还提供了一个熟悉的面向批量或实时查询和统一平台。

MapReduce
　　MapReduce是一种编程模型，用于大规模数据集（大于1TB）的并行运算。概念"Map（映射）"和"Reduce（归约）"，和他们的主要思想，都是从函数式编程语言里借来的，还有从矢量编程语言里借来的特性。他极大地方便了编程人员在不会分布式并行编程的情况下，将自己的程序运行在分布式系统上。 当前的软件实现是指定一个Map（映射）函数，用来把一组键值对映射成一组新的键值对，指定并发的Reduce（归约）函数，用来保证所有映射的键值对中的每一个共享相同的键组。MapReduce更多运行于离线系统，而实时计算，可以使用Storm。关于Sotrm的使用和介绍，可以参考这篇文章《ubuntu12.04+storm0.9.2分布式集群的搭建》。

Oozie
　　Oozie是Yahoo针对Apache Hadoop开发的一个开源工作流引擎。用于管理和协调运行在Hadoop平台上（包括：HDFS、Pig和MapReduce）的Jobs。Oozie是专为雅虎的全球大规模复杂工作流程和数据管道而设计。

Solr
　　Solr是一个基于Lucene的Java搜索引擎服务器。Solr 提供了层面搜索、命中醒目显示并且支持多种输出格式（包括 XML/XSLT 和 JSON 格式）。它易于安装和配置，而且附带了一个基于 HTTP 的管理界面。Solr已经在众多大型的网站中使用，较为成熟和稳定。Solr 包装并扩展了 Lucene，所以Solr的基本上沿用了Lucene的相关术语。更重要的是，Solr 创建的索引与 Lucene 搜索引擎库完全兼容。通过对 Solr 进行适当的配置，某些情况下可能需要进行编码，Solr 可以阅读和使用构建到其他 Lucene 应用程序中的索引。此外，很多 Lucene 工具（如Nutch、 Luke）也可以使用 Solr 创建的索引。

Spark
　　Spark是UC Berkeley AMP lab所开源的类Hadoop MapReduce的通用的并行计算框架，Spark基于map reduce算法实现的分布式计算，拥有Hadoop MapReduce所具有的优点；但不同于MapReduce的是Job中间输出结果可以保存在内存中，从而不再需要读写HDFS，因此Spark能更好地适用于数据挖掘与机器学习等需要迭代的map reduce的算法。
　　Spark和Storm类似，都是基于内存的运行，不确定哪种方式在数据吞吐量上要具优势，不过Storm计算时间延迟要小。关于Sotrm的使用和介绍，可以参考这篇文章《ubuntu12.04+storm0.9.2分布式集群的搭建》。

Sqoop
　　Sqoop中一大亮点就是可以通过hadoop的mapreduce把数据从关系型数据库中导入数据到HDFS。sqoop架构非常简单，其整合了Hive、Hbase和Oozie，通过map-reduce任务来传输数据，从而提供并发特性和容错。sqoop主要通过JDBC和关系数据库进行交互。理论上支持JDBC的database都可以使用sqoop和hdfs进行数据交互。
　　
YARN
　　YARN可以理解为是Hadoop MapReduceV2版本，YARN重构根本的思想是将 JobTracker 两个主要的功能分离成单独的组件，这两个功能是资源管理和任务调度 / 监控。新的资源管理器全局管理所有应用程序计算资源的分配，每一个应用的 ApplicationMaster 负责相应的调度和协调。一个应用程序无非是一个单独的传统的 MapReduce 任务或者是一个 DAG( 有向无环图 ) 任务。ResourceManager 和每一台机器的节点管理服务器能够管理用户在那台机器上的进程并能对计算进行组织。
　　事实上，每一个应用的 ApplicationMaster 是一个详细的框架库，它结合从 ResourceManager 获得的资源和 NodeManager 协同工作来运行和监控任务。

Zookeeper
　　Zookeeper 分布式服务框架是 Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。
　　Zookeeper 作为一个分布式的服务框架，主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储，但是 Zookeeper 并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化。通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。

2、CDH的官网在哪里？
　　http://www.cloudera.com/

3、CDH在哪里下载?
　　由于CDH有多个版本，作者不建议单独下载安装，可以通过cloudera-manager-daemons、cloudera-manager-server、cloudera-manager-agent来安装，本文后面会有介绍。

[](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/install_cm_cdh.html)
[下拉最后点击资源下载](https://www.cloudera.com/resources.html)

选择6.3.0版本

[安装向导](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/install_cm_cdh.html)





