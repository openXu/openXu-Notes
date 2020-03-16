
为了更好的理解本片文章的内容，在看文章之前，先看几个问题，带着这几个问题再看具体内容：

Spark就是用来操作数据的（计算框架）

- 为什么要操作数据？
- 操作的数据从哪里来？
- 怎样操作数据？
- 操作之后的数据怎么办？

# Spark入门

[Apache Spark](https://spark.apache.org/)是用于大数据处理的集群计算框架。与其他大多数数据处理框架不同，
Spark并没有以MapReduce作为执行引擎，而是使用了它自己的分布式运行环境在集群上执行工作。Spark与Hadoop紧密
集成，它可以在YARN上运行，并支持Hadoop文件格式及其存储后端（如HDFS）

Spark最突出的表现在于它能将作业与作业之间产生的大规模的**工作数据集存储在内存中**。这种能力使得Spark在性能上
超过了等效的MapReduce工作流，通常可高出1个数量级，原因是MapReduce的数据集始终需要从磁盘中加载。

Spark提供了三种语言的API：Scala、Java和Python，Spark还为Scala和Python提供了REPL(read-eval-printloop)
交互模式，可以快速且轻松的浏览数据集。

Spark可以运行在许多中模式下，可以是本地模式下运行，也可以在分布式模式下运行（Mesos或YARN上），还可以运行在Spark发行版自带的独立
调度器上。本文入门阶段我们在本地模式下运行，后面会讲在集群上运行相关内容。

## 1. 安装Spark

我当前使用VW安装了ubuntu虚拟机，搭建了hadoop环境（通过CDH），所以有些命令或者输出可能和你们不太一样，不用太纠结。下面演示怎样在Linux下安装Spark

从[下载页面](https://spark.apache.org/downloads.html)下载一个稳定版本的Spark二进制发行包（应当选择与正在使用的Hadoop发行版匹配的版本），
然后在合适的位置解压，配置环境：

```xml
% tart xzf spark-x.y.z-bin.distro.tgz

% export SPARK_HOME=~/soft/spark-x.y.z-bin-distro
% export PATH=$PATH:$SPARK_HOME/bin
```


## 2 快速入门指南

[快速入门指南](http://spark.apache.org/docs/latest/quick-start.html)

通过spark-shell程序运行一个交互式会话来演示Spark (Spark-shell是添加了一些Spark功能的Scala REPL交互式解释器)

```xml
### ★spark-shell命令用来启动该shell环境,--master local[4]指定本地模式，4表示使用4个线程（每个线程一个core）来模拟Spark分布式计算并发执行应用程序。
root@Master:~# spark-shell --master local[4]
19/12/18 01:24:17 WARN lineage.LineageWriter: Lineage directory /var/log/spark/lineage doesn't exist or is not writable. Lineage for this application will be disabled.
Spark context Web UI available at http://Master:4040
### 已经创建了一个名为sc的Scala变量，它用于保存SparkContext实例，这是Spark的主要入口点
Spark context available as 'sc' (master = local[4], app id = local-1576661055779).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.0-cdh5.3.2
      /_/
Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_231)
Type in expressions to have them evaluated.
Type :help for more information.
scala> 

### 日志过多，可以修改/etc/spark/conf/log4j.properties
root@Master:~# vim /etc/spark/conf/log4j.properties 
	root.logger=INFO,console   ### INFO修改为WARN

### 查看变量sc
scala> sc
res0: org.apache.spark.SparkContext = org.apache.spark.SparkContext@6e95c023

### 通过本地文件创建一个名为lines的RDD
scala> val lines = sc.textFile("/test/README.md")
lines: org.apache.spark.rdd.RDD[String] = /test/README.md MapPartitionsRDD[1] at textFile at <console>:24
scala> lines.count()
### 报错，因为去hdfs中找了，如果需要从本地文件中读取，要加上"file://"
org.apache.hadoop.mapred.InvalidInputException: Input path does not exist: hdfs://Master:8020/test/README.md
  at org.apache.hadoop.mapred.FileInputFormat.singleThreadedListStatus(FileInputFormat.java:294)
  ... 49 elided
  
### ★创建一个名为lines的RDD
scala> val lines = sc.textFile("file:///test/README.md")
lines: org.apache.spark.rdd.RDD[String] = file:///test/README.md MapPartitionsRDD[3] at textFile at <console>:24
### ★统计RDD中元素个数
scala> lines.count()
res0: Long = 3
### ★这个RDD中的第一个元素，也就是README.md中第一行
scala> lines.first()
res1: String = hahahahaha

### ★传递函数的API，=>用于创建一个函数，相当于lambda表达式，filter()方法接受一个函数作为参数
scala> val sparkLine = lines.filter(line=>line.contains("spark"))
sparkLine: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[3] at filter at <console>:25
### 读取所有包含spark字符串的行中的第一行
scala> sparkLine.first()
res3: String = hello spark

### ★按Ctr + D 退出
scala> :quit
```

在 Spark 中，我们通过对分布式数据集的操作来表达我们的计算意图，这些计算会自动地
在集群上并行进行。这样的数据集被称为弹性分布式数据集（resilient distributed dataset），
简称 RDD。RDD 是 Spark 对分布式数据和计算的基本抽象

## 3 Spark核心概念

每个Spark应用都有一个驱动器程序来发起集群上的各种并行操作，驱动器程序包含应用的main函数，并定义了集群上的分布式数据集，
还对这些数据集应用了相关操作，在上面的例子中，实际的驱动其程序就是Spark shell。

驱动器程序**通过一个SparkContext对象来访问Spark，这个对象代表对计算集群的一个连接**，Shell启动时已经自动创建了一个SparkContext
对象sc变量。一旦有了SparkContext，就可以用来创建RDD，示例中调用了sc.textFie()创建一个代表文件中各行文本的RDD，然后可以在这些
行上进行各种操作。

## 4 独立应用

除了REPL交互式运行之外，Spark可以在Java、Scala或Python的独立程序中被连接使用，这与在shell中使用的主要区别在于你需要
自行初始化SparkContext对象，接下来使用的API就一样了。

连接Spark的过程在各语言中并不一样。在 Java 和 Scala 中，只需要给你的应用添加一个对于**spark-core**工件的 Maven 依赖。
[Spark Core Maven](https://mvnrepository.com/artifact/org.apache.spark/spark-core)
```xml
<dependency>
	<groupId>org.apache.spark</groupId>
	<artifactId>spark-core_2.11</artifactId>   # scala版本号
	<version>2.4.0</version>     # spark版本号
</dependency>
```

一旦完成了应用与Spark的连接，接下来就需要在程序中创建SparkContext，可以使用Java、Scala、Python语言，下面的演示中默认使用
Scala演示。在写程序之前，需要为IDE导入**scala-sdk**（File->Project Structure->Libraries->+->Scala SDK），注意，导入的ScalaSDK版本
必须与spark-core版本一致。

**Java版本**

```Java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;
import java.util.Arrays;
import java.util.Iterator;
public class SparkTest {
    public static void main(String[] args) {
        /*
         * 配置Spark集群URL 和 应用名称
         * 集群URL:告诉Spark如何连接到集群上，这里配置local这个特殊值可以让Spark运行在单机单线程上而无需连接到集群
         * 应用名：当连接到集群时，这个名称可以帮助你在集群管理其的用户界面中找到你的应用
         */
        SparkConf conf = new SparkConf()
                .setMaster("local[3]")
                .setAppName("SparkTestJava");
        //创建SparkContext实例
        JavaSparkContext sc = new JavaSparkContext(conf);
        //读取我们的输入数据
        JavaRDD<String> input = sc.textFile("file:///D:\\IDE\\scala-2.11.0\\doc\\README");
        //切分为单词
        JavaRDD<String> words = input.flatMap(new FlatMapFunction<String, String>() {
            public Iterator<String> call(String s) throws Exception {
                return Arrays.asList(s.split(" ")).iterator();
            }
        });
        //转换为键值对并计数
        JavaPairRDD<String, Integer> counts = words.mapToPair(new PairFunction<String, String, Integer>() {
            public Tuple2<String, Integer> call(String s) throws Exception {
                return new Tuple2<String, Integer>(s, 1);
            }
        }).reduceByKey(new Function2<Integer, Integer, Integer>() {
            public Integer call(Integer integer, Integer integer2) throws Exception {
                return integer+integer2;
            }
        });
        //将统计出来的单词总数存入一个文本文件，引发求值
        counts.saveAsTextFile("D:\\IDE\\scala-2.11.0\\doc\\sparktest");
        //关闭Spark
        sc.stop();
    }
}
```

**Scala版本**

```Scala
import org.apache.spark.{SparkConf, SparkContext}

object SparkScalaTest {
  def main(args: Array[String]): Unit = {
    //初始化SparkContext
    val conf = new SparkConf()
      .setMaster("local").setAppName("SparkTestScala")
    val sc = new SparkContext(conf)
    //读取输入数据
    val input = sc.textFile("file:///D:\\IDE\\scala-2.11.0\\doc\\README");
    //切分为一个个单词
    val words = input.flatMap(line => line.split(" "))
    //转换为键值对并计数
    val counts = words.map(word => (word, 1)).reduceByKey{case (x, y)=>x+y}
    //将统计出来的单词总数存入一个文本文件，引发求值
    counts.saveAsTextFile("D:\\IDE\\scala-2.11.0\\doc\\sparkScalatest")
    sc.stop()
  }
}
```





## 5 在集群上运行Spark

### 5.1 Spark运行时架构

在分布式环境下，Spark 集群采用的是主 / 从结构。在一个 Spark 集群中，有一个节点负责中央协调，调度各个分布式工作
节点。这个中央协调节点被称为**驱动器（Driver）节点**，与之对应的工作节点被称为**执行器（executor）节点**。驱
动器节点可以和大量的执行器节点进行通信，它们也都作为独立的 Java 进程运行。驱动器节点和所有的执行器节点一起被称
为一个 Spark 应用（application）。

**驱动器节点**

Spark 驱动器是执行你的程序中的 main() 方法的进程。它执行用户编写的用来创建SparkContext、创建 RDD，以及进行 RDD
的转化操作和行动操作的代码。驱动器程序在 Spark 应用中有下述两个职责：

- 把用户程序转为任务
- 为执行器节点调度任务

驱动器程序会将一些 Spark 应用的运行时的信息通过网页界面呈现出来，默认在端口4040 上。比如，在本地模式下，访问 
http://localhost:4040 就可以看到这个网页了

**执行器节点**

Spark 执行器节点是一种工作进程，负责在 Spark 作业中运行任务，任务间相互独立。Spark 应用启动时，执行器节点就被
同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。执行器进程有两大作用：

- 它们负责运行组成 Spark 应用的任务，并将结果返回给驱动器进程；
- 它们通过自身的块管理器（Block Manager）为用户程序中要求缓存的 RDD 提供内存式存储。RDD 是直接缓存在执行器进程内的，因此任务可以在运行时充分利用缓存数据加速运算。


**集群管理器**

Spark 应用通过一个叫作**集群管理器**（Cluster Manager）的外部服务在集群中的机器上启动。Spark 依赖于集群管理器来
启动执行器节点，而在某些特殊情况下，也依赖集群管理器来启动驱动器节点。集群管理器是 Spark 中的可插拔式组件。这样，
除了 Spark 自带的独立集群管理器，Spark 也可以运行在其他外部集群管理器上，比 如 YARN 和 Mesos。

**Spark应用执行过程**

1. 用户通过 spark-submit 脚本提交应用。
2. spark-submit 脚本启动驱动器程序，调用用户定义的 main() 方法。
3. 驱动器程序与集群管理器通信，申请资源以启动执行器节点。
4. 集群管理器为驱动器程序启动执行器节点。
5. 驱动器进程执行用户应用中的操作。根据程序中所定义的对 RDD 的转化操作和行动操作，驱动器节点把工作以任务的形式发送到执行器进程。
5. 任务在执行器程序中进行计算并保存结果。
7. 如果驱动器程序的 main() 方法退出，或者调用了 SparkContext.stop()，驱动器程序会终止执行器进程，并且通过集群管理器释放资源。

### 5.2 集群管理器

#### 独立集群管理器

Spark 独立集群管理器提供在集群上运行应用的简单方法。这种集群管理器由一个主节点和几个工作节点组成，各自都
分配有一定量的内存和 CPU 核心。当提交应用时，你可以配置执行器进程使用的内存量，以及所有执行器进程使用的 
CPU 核心总数。要启动独立集群管理器，你既可以通过手动启动一个主节点和多个工作节点来实现，也可以使用 Spark
 的 sbin 目录中的启动脚本来实现。

**使用集群启动脚本步骤**

1. 将编译好的 Spark 复制到所有机器的一个相同的目录下，比如 /home/yourname/spark。
2. 设置好从主节点机器到其他机器的 SSH 无密码登陆
3. 编辑主节点的 conf/slaves 文件并填上所有工作节点的主机名。
4. 在主节点上运行 sbin/start-all.sh（要在主节点上运行而不是在工作节点上）以启动集群。如果全部启动成功，你不会得到需要密码的提示符，而且可以在 http://masternode:8080看到集群管理器的网页用户界面，上面显示着所有的工作节点。
5. 要停止集群，在主节点上运行 bin/stop-all.sh。

**手动启动集群**

```xml
// 使用 Spark 的 bin/ 目录下的 spark-class 脚本分别手动启动主节点和工作节点
// 在主节点上，输入：
bin/spark-class org.apache.spark.deploy.master.Master
// 然后在工作节点上输入：
bin/spark-class org.apache.spark.deploy.worker.Worker spark://masternode:7077
// （其中 masternode 是你的主节点的主机名）。在 Windows 中，使用 \ 来代替 /。
```

#### Hadoop YARN

YARN 是在 Hadoop 2.0 中引入的集群管理器，它可以让多种数据处理框架运行在一个共享
的资源池上，并且通常安装在与 Hadoop 文件系统（简称 HDFS）相同的物理节点上。在
这样配置的 YARN 集群上运行 Spark 是很有意义的，它可以让 Spark 在存储数据的物理节
点上运行，以快速访问 HDFS 中的数据。

在 Spark 里使用 YARN 很简单：你只需要设置指向你的 Hadoop 配置目录的环境变量，然
后使用 spark-submit 向一个特殊的主节点 URL 提交作业即可。

```xml
spark-submit --master yarn yourapp

--deploy-mode 参数设置不同的模式。
--num-executors  使用固定数量的执行器节点。默认为 2
--executor-memory 设置每个执行器的内存用量
--executor-cores 设置每个执行器进程从 YARN 中占用的核心数目。
--queue 选择你的队列的名字
```

#### Apache Mesos

略

#### Amazon EC2

略

### 5.3 打包代码与依赖

如果你的程序引入了任何既不在 org.apache.spark 包内也不属于语言运行时的库的依赖，你就需要确保所有的依赖在该 Spark 应用运行时
都能被找到。

Java 和 Scala 用户也可以通过 spark-submit 的 --jars 标记提交独立的 JAR 包依赖。当只有一两个库的简单依赖，并且这些库本身不
依赖于其他库时，这种方法比较合适。但是一般 Java 和 Scala 的工程会依赖很多库。当你向 Spark 提交应用时，你必须把应用的整个依
赖传递图中的所有依赖都传给集群。你不仅要传递你直接依赖的库，还要传递这些库的依赖，以及它们的依赖的依赖，等等。手动维护和提交
全部的依赖 JAR 包是很笨的方法。事实上，**常规的做法是使用构建工具，生成单个大 JAR 包，包含应用的所有的传递依赖**。这
通常被称为**超级（uber）JAR 或者组合（assembly）JAR**，大多数 Java 或 Scala 的构建工具都支持生成这样的工件。

**使用Maven构建的用Java编写的Spark应用**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <!-- 工程相关信息 -->
    <groupId>com.databricks</groupId>
    <artifactId>example-build</artifactId>
    <name>Simple Project</name>
    <packaging>jar</packaging>
    <version>1.0</version>
    <dependencies>
        <!-- Spark依赖 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>1.2.0</version>
			<!-- 把 Spark 标记为 provided 来确保 Spark 不与应用依赖的其他工件打包在一起 -->
            <scope>provided</scope>
        </dependency>
        <!-- 第三方库 -->
        <dependency>
            <groupId>net.sf.jopt-simple</groupId>
            <artifactId>jopt-simple</artifactId>
            <version>4.3</version>
        </dependency>
        <!-- 第三方库 -->
        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>2.0</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <!-- 用来创建超级JAR包的Maven shade插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

**依赖冲突**

当用户应用与 Spark 本身依赖同一个库时可能会发生依赖冲突，导致程序崩溃。这种情况不是很常见，但是出现的时候也让人
很头疼。通常，依赖冲突表现为 Spark 作业执行过程中抛出 NoSuchMethodError、ClassNotFoundException，或其他与类加
载相关的 JVM 异常。对于这种问题，主要有两种解决方式：

1. 修改你的应用，使其使用的依赖库的版本与 Spark 所使用的相同
2. 使用通常被称为“shading”的方式打包你的应用(shading 的功能也是这个插件取名为 maven-shade-plugin 的原因)

Maven 构建工具通过**对插件进行高级配置**来支持这种打包方式。shading 可以让你以另一个命名空间保留冲突的包，并自动重写
应用的代码使得它们使用重命名后的版本。这种技术有些简单粗暴，不过对于解决运行时依赖冲突的问题非常有效。如果要了解使
用这种打包方式的具体步骤，请参阅你所使用的构建工具对应的文档。

### 5.4 spark-submit部署应用

Spark为各种集群管理器提供了统一的工具来提交作业，这个工具是 spark-submit。

`bin/spark-submit [options] <app jar | python file> [app options]`

[更多配置选项的相关信息，可以查阅 Spark 官方文档](http://spark.apache.org/docs/latest/submitting-applications.html)

**spark-submit的一些常见标记**

| 标记 | 描述 |
|:--|:--|
|--master |表示要连接的集群管理器。|
|--deploy-mode |选择在本地（客户端“client”）启动驱动器程序，还是在集群中的一台工作节点机器（集群“cluster”）上启动。在客户端模式下，spark-submit 会将驱动器程序运行在 spark-submit 被调用的这台机器上。在集群模式下，驱动器程序会被传输并执行于集群的一个工作节点上。默认是本地模式|
|--class |运行 Java 或 Scala 程序时应用的主类|
|--name |应用的显示名，会显示在 Spark 的网页用户界面中|
|--jars |需要上传并放到应用的 CLASSPATH 中的 JAR 包的列表。如果应用依赖于少量第三方的 JAR 包，可以把它们放在这个参数里|
|--files |需要放到应用工作目录中的文件的列表。这个参数一般用来放需要分发到各节点的数据文件|
|--py-files |需要添加到 PYTHONPATH 中的文件的列表。其中可以包含 .py、.egg 以及 .zip 文件|
|--executor-memory| 执行器进程使用的内存量，以字节为单位。可以使用后缀指定更大的单位，比如“512m”（512 MB）或“15g”（15 GB）|
|--driver-memory |驱动器进程使用的内存量，以字节为单位。可以使用后缀指定更大的单位，比如“512m”（512 MB）或“15g”（15 GB）|


**--master标记可以接收的值**

|值 |描述 |
|:--|:--|
|spark://host:port |连接到指定端口的 Spark 独立集群上。默认情况下 Spark 独立主节点使用7077 端口|
|mesos://host:port |连接到指定端口的 Mesos 集群上。默认情况下 Mesos 主节点监听 5050 端口|
|yarn |连接到一个 YARN 集群。当在 YARN 上运行时，需要设置环境变量 HADOOP_CONF_DIR 指向 Hadoop 配置目录，以获取集群信息|
|local |运行本地模式，使用单核|
|local[N] |运行本地模式，使用 N 个核心|
|local[*] |运行本地模式，使用尽可能多的核心|

**示例**

```xml
##### 提交应用
root@Master:~# spark-submit \
> --master yarn \
> --class com.opeu.spark.SparkScalaTest \
> --name sparkTest \
> --executor-memory 512m \
> --driver-memory 512m \
> /home/openxu/Desktop/test.jar 

root@Master:~# spark-submit --master yarn --class com.opeu.spark.SparkScalaTest 
--name sparkTest --executor-memory 512m --driver-memory 512m /home/openxu/Desktop/test.jar 

##### 查看应用
http://master:8088/cluster/apps

##### 杀掉应用
root@Master:~# yarn application -kill application_1577239615288_0001
```


### 5.5 Spark运行模式理解

**本地运行模式**

该模式被称为Local[N]模式，是用单机的多个线程来模拟Spark分布式计算，通常用来验证开发出来的应用程序逻辑上有没有问题。
其中N代表可以使用N个线程，每个线程拥有一个core。如果不指定N，则默认是1个线程（该线程有1个core）。

```xml
#### ★ 本地运行模式，使用3线程提交应用
root@Master:~# spark-submit --master local[3] /home/openxu/Desktop/test.jar 
#### 查看进程
root@Master:~# jps
#### 既是客户提交任务的Client进程、又是Spark的driver程序、还充当着Spark执行Task的Executor角色
21362 SparkSubmit   
21779 Jps
```

**本地伪集群运行模式**

单机启动多个进程来模拟集群下的分布式场景，而不像Local[N]这种多个线程只能在一个进程下委屈求全的共享资源。
提交应用程序时使用local-cluster[x,y,z]参数：x代表要生成的executor数，y和z分别代表每个executor所拥有的core和memory数。
```xml
#### ★ 本地伪集群运行模式，2个executor进程，每个进程分配2个core和512m的内存，来运行应用程序
root@Master:~# spark-submit --master local-cluster[2,2,512] /home/openxu/Desktop/test.jar  
#### 查看进程
root@Master:~# jps
#### SparkSubmit依然充当全能角色，又是Client进程，又是driver程序，还有点资源管理的作用。生成的两个CoarseGrainedExecutorBackend，就是用来并发执行程序的进程
25673 SparkSubmit
```

























# 二. RDD编程

RDD(Resilient Distributed Dataset弹性分布式数据集)是Spark对数据的核心抽象，RDD其实就是分布式的元素集合。在Spark
中对数据的所有操作不外乎创建RDD、转化已有RDD以及调用RDD操作进行求值，而在这一切背后，Spark会自动将RDD中的数据分发
到集群上，并将操作并行化执行。

## 2.1 RDD基础

Spark中的RDD就是一个不可变的分布式对象集合，每个RDD都被分为多个分区，这些分区运行在集群中的不同节点上。RDD可以包含
Python、Java、Scala中任意类型的对象，甚至是自定义对象。可以使用两种方法创建RDD:读取一个外部数据集（SparkContext.textFile()），
或在驱动器程序里分发驱动器程序中的对象集合（如list和set）

RDD支持两种类型的操作：**转化操作**和**行动操作**，转化操作会由一个RDD生成一个新的RDD，行动操作会对RDD计算出一个结果，
并将结果返回到驱动器程序中，或把结果存储到外部存储系统中

```xml
### ★创建一个名为lines的RDD
scala> val lines = sc.textFile("file:///test/README.md")
lines: org.apache.spark.rdd.RDD[String] = file:///test/README.md MapPartitionsRDD[3] at textFile at <console>:24
### ★(行动操作)统计RDD中元素个数
scala> lines.count()
res0: Long = 3
### ★(转化操作)传递函数的API，=>用于创建一个函数，相当于lambda表达式，filter()方法接受一个函数作为参数
scala> val sparkLine = lines.filter(line=>line.contains("spark"))
sparkLine: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[3] at filter at <console>:25
### ★(行动操作)读取所有包含spark字符串的行中的第一行
scala> sparkLine.first()
res3: String = hello spark
```

转化操作和行动操作的区别在于 Spark 计算 RDD 的方式不同。创建RDD时Spark只会惰性计算这些 RDD，只有第一次在一个行动操作中用到
时，才会真正计算。如果 Spark 在我们运行 lines = sc.textFile(...) 时就把文件中所有的行都读取并存储起来，就会消耗很多存储空间，
而我们马上就要筛选掉其中的很多数据。相反，一旦 Spark 了解了完整的转化操作链之后，它就可以只计算求结果时真正需要的数据。事实上，
在行动操作 first() 中，Spark 只需要扫描文件直到找到第一个匹配的行为止，而不需要读取整个文件。

默认情况下，Spark 的 RDD 会在你每次对它们进行行动操作时重新计算。如果想在多个行动操作中重用同一个 RDD，可以使用`RDD.persist()`
让 Spark 把这个RDD缓存下来。我们可以让 Spark 把数据持久化到许多不同的地方:

## 2.2 持久化（缓存）

Spark RDD是惰性求值的，而有时我们希望能多次使用同一个RDD，如果简单的对RDD调用行动操作，Spark 每次都会重算 RDD 以及它的所有依赖。
这在迭代算法中消耗额外大， 因为迭代算法常常会多次使用同一组数据：

```xml
### 转化操作得到新的RDD
val result = input.map(x => x*x) 
### 行动操作，输出计数
println(result.count()) 
### 行动操作，把RDD输出
println(result.collect().mkString(","))
```

**为了避免多次计算同一个 RDD，可以让 Spark 对数据进行持久化。**出于不同的目的，我们可以为 RDD 选择不同的持久化级别：

| 级别  |使用的空间  |CPU时间| 是否在内存中|是否在磁盘上 |备注|
|:--|:--:|:--:|:--:|:--:|--:|
|MEMORY_ONLY |高 |低 |是 |否||
|MEMORY_ONLY_SER |低 |高 |是 |否||
|MEMORY_AND_DISK |高 |中等 |部分 |部分 |如果数据在内存中放不下，则溢写到磁盘上|
|MEMORY_AND_DISK_SER| 低 |高 |部分 |部分 |如果数据在内存中放不下，则溢写到磁盘上。在内存中存放序列化后的数据
|DISK_ONLY |低 |高 |否 |是||

RDD 还有一个方法叫作 `unpersist()`，调用该方法可以手动把持久化的 RDD 从缓存中移除

## 2.3 创建RDD

- 读取外部数据集

`val lines = sc.textFile("/path/to/README.md")`

- 在驱动器程序中对一个集合进行并行化

> 这是创建RDD最简单的方式，将程序中一个已有的集合传给SparkContext的parallelize()方法，这种方式在学习Spark的时候非
> 常有用，不过在开发中用的不多，毕竟这种方式需要把你的整个数据集先放在一台机器内存中

```Scala
val lines = sc.parallelize(List("pandas", "i like pandas"))
```

```Java
JavaRDD<String> lines = sc.parallelize(Arrays.asList("pandas", "i like pandas"));
```

## 2.4 RDD操作

RDD 支持两种操作：转化操作和行动操作。RDD 的转化操作是返回一个新的 RDD 的操作，比如 map() 和 filter()，
而行动操作则是向驱动器程序返回结果或把结果写入外部系统的操作，会触发实际的计算，比如 count() 和 first()。S

**转化操作**

RDD 的转化操作是返回新 RDD 的操作，转化出来的 RDD 是惰性求值的，只有在行动操作中用到这些 RDD 时才会被计算。
惰性求值意味着当我们对 RDD 调用转化操作（例如调用 map()）时，操作不会立即执行。相反，Spark 会在内部记录下
所要求执行的操作的相关信息。我们不应该把 RDD 看作存放着特定数据的数据集，而最好把每个 RDD 当作我们通过转化
操作构建出来的、记录如何计算数据的指令列表。

通过转化操作，你从已有的 RDD 中派生出新的 RDD，Spark 会使用**谱系图**（lineage graph）来记录这些不同 RDD 之间
的依赖关系。Spark 需要用这些信息来按需计算每个 RDD，也可以依靠谱系图在持久化的 RDD 丢失部分数据时恢复所丢失的数据。

**行动操作**

行动操作是第二种类型的 RDD 操作，它们会把最终求得的结果返回到驱动器程序，或者写入外部存储系统中。由于行动操
作需要生成实际的输出，它们会**强制执行那些求值必须用到的 RDD 的转化操作**。

## 2.5 常见的转化操作

**1. 针对各个元素的转化操作**

**map()**: 接收一个函数，把这个函数用于 RDD 中的每个元素，将函数的返回结果作为结果RDD 中对应元素的值

**flatMap()**: 希望对每个输入元素生成多个输出元素。提供给 flatMap() 的函数被分别应用到了输入 RDD 的每个元素上。不
过返回的不是一个元素，而是一个返回值序列的迭代器

**filter()**: 则接收一个函数，并将 RDD 中满足该函数的元素放入新的 RDD 中返回

**2. 伪集合操作**

RDD 本身不是严格意义上的集合，但它也支持许多数学上的集合操作作，比如合并和相交操作。注意，这些操作都要求操作的
RDD 是相同数据类型的。

**distinct()**: 转化操作来生成一个只包含不同元素的新RDD。不过需要注意，distinct() 操作的开销很大，因为它需要将所有数据通过网络进行
混洗（shuffle），以确保每个元素都只有一份。后面介绍数据混洗，以及如何避免数据混洗

**union(other)**: 合并是最简单的集合操作，它会返回一个包含两个 RDD 中所有元素的 RDD

**intersection(other)**: 相交，只返回两个 RDD 中都有的元素。intersection()会去掉所有重复的元素（单个RDD内的重复元素也会一起移除）。尽管
intersection() 与 union() 的概念相似，intersection() 的性能却要差很多，因为它需要通过网络混洗数据来发现共有的元素。

**subtract(other)**: 接收另一个 RDD 作为参数，返回一个由只存在于第一个 RDD 中而不存在于第二个 RDD 中的所有元素组成的 RDD。和intersection() 一样，它也需要数据混洗

**cartesian(other)**: 计算两个 RDD 的笛卡儿积，会返回所有可能的 (a, b) 对，其中 a 是源RDD中的元素，而b则来自另一个 RDD。求大规模 RDD 的笛卡儿积开销巨大。

**示例**

对一个数据为{1, 2, 3, 3}的RDD进行基本的RDD转化操作

|函数名 |目的 |示例 |结果|
|:--|:--:|:--:|--:|
|map() |将函数应用于 RDD 中的每个元素，将返回值构成新的 RDD|rdd.map(x => x + 1)| {2, 3, 4, 4}|
|flatMap() |将函数应用于 RDD 中的每个元素，将返回的迭代器的所有内容构成新的 RDD。通常用来切分单词|rdd.flatMap(x => x.to(3))| {1, 2, 3, 2, 3, 3, 3}|
|filter()| 返回一个由通过传给 filter()|的函数的元素组成的 RDD|rdd.filter(x => x != 1)| {2, 3, 3}|
|distinct() |去重 |rdd.distinct() |{1, 2, 3}|
|sample(withReplacement, fraction, [seed])|对 RDD 采样，以及是否替换 |rdd.sample(false, 0.5)| 非确定的|

对数据分别为{1, 2, 3}和{3, 4, 5}的RDD进行针对两个RDD的转化操作

|函数名 |目的 |示例 |结果|
|:--|:--:|:--:|--:|
|union() 生成一个包含两个 RDD 中所有元素的 RDD|rdd.union(other) |{1, 2, 3, 3, 4, 5}|
|intersection() |求两个 RDD 共同的元素的 |RDD rdd.intersection(other)| {3}|
|subtract() |移除一个 RDD 中的内容（例如移除训练数据）|rdd.subtract(other) |{1, 2}|
|cartesian() |与另一个 RDD 的笛卡儿积| rdd.cartesian(other) |{(1, 3), (1, 4), ... (3, 5)}|

## 2.6 常见的行动操作

对一个数据为{1, 2, 3, 3}的RDD进行基本的RDD行动操作

|函数名 |目的 |示例 |结果|
|:--|:--:|:--:|--:|
|collect()|返回 RDD 中的所有元素 |rdd.collect()| {1, 2, 3, 3}|
|count() |RDD 中的元素个数 |rdd.count() |4|
|countByValue() |各元素在 RDD 中出现的次数 |rdd.countByValue()| {(1, 1), (2, 1), (3, 2)}|
|take(num)| 从 RDD 中返回 num 个元素| rdd.take(2) |{1, 2}|
|top(num) |从 RDD 中返回最前面的 num个元素|rdd.top(2) {3, 3}|
|takeOrdered(num)(ordering)|从 RDD 中按照提供的顺序返回最前面的 num 个元素|rdd.takeOrdered(2)(myOrdering)| {3, 3}|
|takeSample(withReplacement, num, [seed])|从 RDD 中返回任意一些元素 |rdd.takeSample(false, 1) |非确定的|
|reduce(func) |并行整合RDD中所有数据（例如 sum）|rdd.reduce((x, y) => x + y)| 9|
|fold(zero)(func) |和 reduce() 一 样，但是需要提供初始值|rdd.fold(0)((x, y) => x + y) |9|
|aggregate(zeroValue)(seqOp, combOp)|和reduce()相似， 但是通常返回不同类型的函数|rdd.aggregate((0, 0))((x, y) => (x._1 + y, x._2 + 1), (x, y) => (x._1 + y._1, x._2 + y._2))|(9,4)|
|foreach(func) |对 RDD 中的每个元素使用给定的函数|rdd.foreach(func) |无|


## 2.7 在不同RDD类型间转换

有些函数只能用于特定类型的 RDD，比如 mean() 和 variance() 只能用在数值 RDD 上，
而 join() 只能用在键值对 RDD 上。在 Scala 和 Java 中，这些函数都没有定义在标准的 RDD
类中，所以要访问这些附加功能，必须要确保获得了正确的专用 RDD 类。

**1. Scala**

Scala 中，将 RDD 转为有特定函数的 RDD（比如在 RDD[Double] 上进行数值操作）是由隐式转换来自动处理
的。我们需要加上`import org.apache.spark.SparkContext._`来使用这些隐式转换。你可以在 SparkContext 
对象的 [Scala 文档](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext$)
中查看所列出的隐式转换。这些隐式转换可以隐式地将一个 RDD 转为各种封装类，比如 DoubleRDDFunctions
（数值数据的 RDD）和 PairRDDFunctions（键值对 RDD），这样我们就有了诸如 mean() 和
variance() 之类的额外的函数。

隐式转换虽然强大，但是会让阅读代码的人感到困惑。如果你对 RDD 调用了像 mean() 这
样的函数，可能会发现 RDD 类的 [Scala 文档](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.RDD)中根本没有 mean() 函数。调用之所以能够成功，是
因为隐式转换可以把 RDD[Double] 转为 DoubleRDDFunctions。当我们在 Scala 文档中查找函
数时，不要忘了那些封装的专用类中的函数

**2. Java**

在 Java 中，各种 RDD 的特殊类型间的转换更为明确。Java 中有两个专门的类 JavaDoubleRDD
和 JavaPairRDD，来处理特殊类型的 RDD，这两个类还针对这些类型提供了额外的函数。
这让你可以更加了解所发生的一切，但是也显得有些累赘。

Java中针对专门类型的函数接口

|函数名 |等价函数 |用途|
|:--|:--:|--:|
|DoubleFlatMapFunction<T> |Function<T, Iterable<Double>> |用于 flatMapToDouble，以生成 DoubleRDD|
|DoubleFunction<T> |Function<T, Double> |用于 mapToDouble，以生成DoubleRDD|
|PairFlatMapFunction<T, K, V> |Function<T, Iterable<Tuple2<K, V>>> |用于 flatMapToPair，以生成 PairRDD<K, V>|
|PairFunction<T, K, V> |Function<T, Tuple2<K, V>> |用 于 mapToPair， 以 生 成PairRDD<K, V>|


## 2.8 数值RDD的操作

Spark 的数值操作是通过流式算法实现的，允许以每次一个元素的方式构建出模型。这些
统计数据都会在调用 stats() 时通过一次遍历数据计算出来，并以 StatsCounter 对象返
回。下面列出了 StatsCounter 上的可用方法。

|方法 |含义 |
|:--|--:|
|count() |RDD 中的元素个数|
|mean() |元素的平均值|
|sum() |总和|
|max() |最大值|
|min() |最小值|
|variance() |元素的方差|
|sampleVariance() |从采样中计算出的方差|
|stdev() |标准差|
|sampleStdev() |采样的标准差||

# 三. 键值对操作

Spark为包含键值对类型的RDD提供了一些专有的操作。这些RDD被称为pair RDD(对RDD)。Pair RDD 是很多程序的构成要素,
因为它们提供了并行操作各个键或跨节点重新进行数据分组的操作接口。例如，pair RDD 提供 reduceByKey() 方法，
可以分别归约每个键对应的数据，还有 join() 方法，可以把两个 RDD 中键相同的元素组合到一起，合并为一个 RDD。
我们通常从一个RDD中提取某些字段（例如代表事件时间、用户ID或者其他标识符的字段），并使用这些字段作为 pair RDD 操作中的键。

## Pair RDD的转化操作

Pair RDD的转化操作（以键值对集合{(1, 2), (3, 4), (3, 6)}为例）

|函数名 |目的 |示例 |结果|
|:--|:--:|:--:|--:|
|reduceByKey(func) |合并具有相同键的值 |rdd.reduceByKey((x, y) => x + y) |{(1, 2), (3, 10)}|
|groupByKey() |对具有相同键的值进行分组 |rdd.groupByKey()| {(1, [2]), (3, [4, 6])}|
|combineByKey( createCombiner,mergeValue,mergeCombiners,partitioner)|使用不同的返回类型合并具有相同键的值|||

|mapValues(func) |对 pair RDD 中的每个值应用一个函数而不改变键|rdd.mapValues(x => x+1) |{(1, 3), (3, 5), (3, 7)}|
|flatMapValues(func) |对 pair RDD 中的每个值应用一个返回迭代器的函数，然后对返回的每个元素都生成一个对应原键的键值对记录。通常用于符号化|rdd.flatMapValues(x => (x to 5)) |{(1, 2), (1, 3), (1, 4), (1, 5), (3, 4), (3,5)}|
|keys() |返回一个仅包含键的 RDD |rdd.keys() |{1, 3, 3}|
|values() |返回一个仅包含值的| RDD rdd.values() |{2, 4, 6}|
|sortByKey()| 返回一个根据键排序的| RDD rdd.sortByKey() |{(1, 2), (3, 4), (3, 6)}|

针对两个pair RDD的转化操作（rdd = {(1, 2), (3, 4), (3, 6)}other = {(3, 9)}）

|函数名 |目的 |示例 |结果|
|:--|:--:|:--:|--:|
|subtractByKey |删掉 RDD 中键与 other RDD 中的键相同的元素|rdd.subtractByKey(other)| {(1, 2)}|
|join |对两个 RDD 进行内连接 |rdd.join(other) |{(3, (4, 9)), (3, 
(6, 9))}|
|rightOuterJoin |对两个 RDD 进行连接操作，确保第一个 RDD 的键必须存在（右外连接）|rdd.rightOuterJoin(other) |{(3,(Some(4),9)), (3,(Some(6),9))}|
|leftOuterJoin |对两个 RDD 进行连接操作，确保第二个 RDD 的键必须存在（左外连接）|rdd.leftOuterJoin(other) |{(1,(2,None)), (3, (4,Some(9))), (3, (6,Some(9)))}|
|cogroup |将两个 RDD 中拥有相同键的数据分组到一起|rdd.cogroup(other) |{(1,([2],[])), (3, ([4, 6],[9]))}|


# 四. 数据读取与保存

Spark 支持很多种输入输出源，一部分原因是Spark本身是基于Hadoop生态圈而构建，特别是Spark可以通过
Hadoop MapReduce所使用的InputFormat和OutputFormat接口访问数据，而大部分常见的文件格式与存储系
统（例如 S3、HDFS、Cassandra、HBase 等）都支持这种接口。不过，基于这些原始接口构建出的高层 API
会更常用，幸运的是Spark及其生态系统提供了很多可选方案。下面会介绍以下三类常见的数据源。

## 文件格式

**文本文件**

在 Spark 中读写文本文件很容易。当我们将一个文本文件读取为 RDD 时，输入的**每一行都会成为 RDD 的一个元素**。
也可以将多个完整的文本文件一次性读取为一个 pair RDD，其中键是文件名，值是文件内容。

Spark 支持读取给定目录中的所有文件，以及在输入路径中**使用通配字符**（如 part-*.txt）。大规模数据集通常存放在
多个文件中，因此这一特性很有用，尤其是在同一目录中存在一些别的文件（比如成功标记文件）的时候。

```Scala
### 读取一个文本文件，textFile也能读取目录，会把目录下每个文件都读取到 RDD中
val input = sc.textFile("file:///home/holden/repos/spark/README.md")
### 读取一个目录，返回一个 pair RDD，其中键是输入文件的文件名。
val input = sc.wholeTextFiles("file://home/holden/salesFiles") 
### 求每个文件的平均值
val result = input.mapValues{y => 
 val nums = y.split(" ").map(x => x.toDouble) 
 nums.sum / nums.size.toDouble 
}
### 将RDD 中的内容都输入到路径对应的文件中。Spark 将传入的路径作为目录对待，会在那个目录下输出多个文件
result.saveAsTextFile(outputFile)
```

**JSON**

JSON 是一种使用较广的**半结构化数据格式**。读取 JSON 数据的最简单的方式是将数据作为文本文件读取，然后
使用JSON解析器来对RDD中的值进行映射操作。类似地，也可以使用我们喜欢的JSON序列化库来将数据转为字符串，
然后将其写出去。在Java和Scala中也可以使用一个自定义Hadoop格式来操作JSON数据。

```Scala
#### 在 Scala 中读取 JSON
import com.fasterxml.jackson.module.scala.DefaultScalaModule
import com.fasterxml.jackson.module.scala.experimental.ScalaObjectMapper
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.DeserializationFeature
...
case class Person(name: String, lovesPandas: Boolean) // 必须是顶级类
...
// 将其解析为特定的case class。使用flatMap，通过在遇到问题时返回空列表（None）
// 来处理错误，而在没有问题时返回包含一个元素的列表（Some(_)）
val result = input.flatMap(record => {
 try {
 Some(mapper.readValue(record, classOf[Person]))
 } catch {
 case e: Exception => None
 }})
 
#### 在 Scala 中保存为 JSON
 result.filter(p => P.lovesPandas).map(mapper.writeValueAsString(_))
  .saveAsTextFile(outputFile)
```

## 文件系统

**本地/“常规”文件系统**

```Scala
### 只需要指定输入为一个 file://路径，从本地文件系统	读取一个压缩的文本文件
val rdd = sc.textFile("file:///home/holden/happypandas.gz")
```

**Amazon S3**

用 Amazon S3 来存储大量数据正日益流行。当计算节点部署在 Amazon EC2 上的时候，使
用 S3 作为存储尤其快，但是在需要通过公网访问数据时性能会差很多。

```Scala
### 以 s3n:// 开头的路径，从my-Files下读取所有的txt格式的文件
val rdd = sc.textFile("s3n://bucket/my-Files/*.txt")
```

**HDFS**

```Scala
### 以 hdfs://master:port 开头的路径，从HDFS文件系统中读取数据
val rdd = sc.textFile("hdfs://master:port/path")
```

## 数据库

**Java数据库连接**

```Scala
def createConnection() = {
 Class.forName("com.mysql.jdbc.Driver").newInstance();
 DriverManager.getConnection("jdbc:mysql://localhost/test?user=holden");
}
def extractValues(r: ResultSet) = {
 (r.getInt(1), r.getString(2))
}
val data = new JdbcRDD(sc,
 createConnection, "SELECT * FROM panda WHERE ? <= id AND id <= ?",
 lowerBound = 1, upperBound = 3, numPartitions = 2, mapRow = extractValues)
println(data.collect().toList)
```

**Cassandra**

随着 DataStax 开源其用于Spark的[Cassandra 连接器](https://github.com/datastax/spark-cassandra-connector)，
Spark 对 Cassandra 的支持大大提升。这个连接器目前还不是 Spark 的一部分，因此你需要添加一些额外的依赖到你的构建文件中才能使用它。

```xml
<dependency> <!-- Cassandra -->
 <groupId>com.datastax.spark</groupId>
 <artifactId>spark-cassandra-connector</artifactId>
 <version>1.0.0-rc5</version>
</dependency>
<dependency> <!-- Cassandra -->
 <groupId>com.datastax.spark</groupId>
 <artifactId>spark-cassandra-connector-java</artifactId>
 <version>1.0.0-rc5</version>
```

```Scala
val conf = new SparkConf(true)
 .set("spark.cassandra.connection.host", "hostname")
val sc = new SparkContext(conf)

// 为SparkContext和RDD提供附加函数的隐式转换
import com.datastax.spark.connector._
// 将整张表读为一个RDD。假设你的表test的创建语句为
// CREATE TABLE test.kv(key text PRIMARY KEY, value int);
val data = sc.cassandraTable("test" , "kv")
// 打印出value字段的一些基本统计。
data.map(row => row.getInt("value")).stats()

除了读取整张表，也可以查询数据集的子集。通过在 cassandraTable() 的调用中加上 where
子句，可以限制查询的数据，例如 sc.cassandraTable(...).where("key=?", "panda")。
// Scala 中保存数据到 Cassandra
val rdd = sc.parallelize(List(Seq("moremagic", 1)))
rdd.saveToCassandra("test" , "kv", SomeColumns("key", "value"))
```

**HBase**

```Scala
从 HBase 读取数据的 Scala 示例
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client.Result
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapreduce.TableInputFormat
val conf = HBaseConfiguration.create()
conf.set(TableInputFormat.INPUT_TABLE, "tablename") // 扫描哪张表
val rdd = sc.newAPIHadoopRDD(
 conf, classOf[TableInputFormat], classOf[ImmutableBytesWritable],classOf[Result])
```

**Elasticsearch**

```Scala
在 Scala 中使用 Elasticsearch 输出
val jobConf = new JobConf(sc.hadoopConfiguration)
jobConf.set("mapred.output.format.class", "org.elasticsearch.hadoop.
mr.EsOutputFormat")
jobConf.setOutputCommitter(classOf[FileOutputCommitter])
jobConf.set(ConfigurationOptions.ES_RESOURCE_WRITE, "twitter/tweets")
jobConf.set(ConfigurationOptions.ES_NODES, "localhost")
FileOutputFormat.setOutputPath(jobConf, new Path("-"))
output.saveAsHadoopDataset(jobConf)

在 Scala 中使用 Elasticsearch 输入
def mapWritableToInput(in: MapWritable): Map[String, String] = {
 in.map{case (k, v) => (k.toString, v.toString)}.toMap
}
val jobConf = new JobConf(sc.hadoopConfiguration)
jobConf.set(ConfigurationOptions.ES_RESOURCE_READ, args(1))
jobConf.set(ConfigurationOptions.ES_NODES, args(2))
val currentTweets = sc.hadoopRDD(jobConf,
 classOf[EsInputFormat[Object, MapWritable]], classOf[Object],
 classOf[MapWritable])
// 仅提取map
// 将MapWritable[Text, Text]转为Map[String, String]
val tweets = currentTweets.map{ case (key, value) => mapWritableToInput(value) }
```

# 五. Spark编程进阶

## 累加器

通常在向 Spark 传递函数时，比如使用 map() 函数或者用 filter() 传条件时，可以使用驱动器程序中定义的变
量，但是集群中运行的每个任务都会得到这些变量的一份新的副本，更新这些副本的值也不会影响驱动器中的对应变量。
Spark 的两个共享变量，累加器与广播变量，分别为结果聚合与广播这两种常见的通信模式突破了这一限制。

累加器 提供了将工作节点中的值聚合到驱动器程序中的简单语法。累加器的一个常见用途是在调试时对作业执行过程中的事件
进行计数。例如，假设我们在从文件中读取呼号列表对应的日志，同时也想知道输入文件中有多少空行（也许不希望在
有效输入中看到很多这样的行）

当然，也可以使用 reduce() 这样的行动操作将整个 RDD 中的值都聚合到驱动器中。只是
我们有时希望使用一种更简单的方法来对那些与 RDD 本身的范围和粒度不一样的值进行
聚合。

```Scala
// Scala 中累加空行
val sc = new SparkContext(...)
val file = sc.textFile("file.txt")
// 创建出存有初始值的累加器,创建Accumulator[Int]并初始化为0
val blankLines = sc.accumulator(0) 
val callSigns = file.flatMap(line => {
 if (line == "") {
 blankLines += 1 // 累加器加1
 }
 line.split(" ")
})
callSigns.saveAsTextFile("output.txt")
//只有在运行 saveAsTextFile() 行动操作后才能看到正确的计数，因为行动操作前的转化操作flatMap() 是惰性的，
println("Blank lines: " + blankLines.value)
```


## 广播变量

广播变量其实就是类型为 spark.broadcast.Broadcast[T] 的一个对象，其中存放着类型为 T 的值。可以在任务中通过对
Broadcast 对象调用 value 来获取该对象的值。这个值只会被发送到各节点一次，**应作为只读值处理**（修改这个值不会影响到别的节点）。
使用的是一种高效的类似 BitTorrent 的通信机制。

```Scala
// 在 Scala 中使用广播变量查询国家
// 查询RDD contactCounts中的呼号的对应位置。将呼号前缀读取为国家代码来进行查询
val signPrefixes = sc.broadcast(loadCallSignTable())
val countryContactCounts = contactCounts.map{case (sign, count) =>
 val country = lookupInArray(sign, signPrefixes.value)
 (country, count)
}.reduceByKey((x, y) => x + y)
countryContactCounts.saveAsTextFile(outputDir + "/countries.txt")
```





# 七. Spark SQL

Spark SQL是Spark用来操作结构化和半结构化数据的接口，Spark SQL提供了以下三大功能：

1. Spark SQL可以从各种结构化数据源（例如 JSON、Hive、Parquet 等）中读取数据
2. Spark SQL不仅支持在Spark程序内使用SQL语句进行数据查询，也支持从类似商业智能软件 Tableau 这样的外部工具中通过标准数据库连接器（JDBC/ODBC）连接Spark SQL进行查询。
3. 当在Spark程序内使用Spark SQL时，Spark SQL支持SQL与常规的 Python/Java/Scala代码高度整合，包括连接RDD与SQL表、公开的自定义SQL函数接口等。这样一来，许多工作都更容易实现了。

为了实现这些功能，Spark SQL提供了一种特殊的RDD，叫作**SchemaRDD**。SchemaRDD是存放Row对象的RDD，每个Row对象代表一行记录。
SchemaRDD 还包含记录的结构信息（即数据字段）。SchemaRDD 看起来和普通的RDD很像，但是在内部，SchemaRDD 可以利用结构信息更加
高效地存储数据。此外，SchemaRDD还支持RDD上所没有的一些新操作，比如运行 SQL 查询。SchemaRDD 可以从外部数据源创建，也可以从
查询结果或普通 RDD 中创建。**1.3.0及后续版本中，SchemaRDD 已经被DataFrame所取代**。










# Spark Streaming

许多应用需要即时处理收到的数据，例如用来实时追踪页面访问统计的应用、训练机器学
习模型的应用，还有自动检测异常的应用。Spark Streaming 是 Spark 为这些应用而设计的
模型。它允许用户使用一套和批处理非常接近的 API 来编写流式计算应用，这样就可以大
量重用批处理应用的技术甚至代码。

Spark Streaming 使用离散化流（discretized stream）作为抽象表示，叫作 DStream。DStream 是随时间
推移而收到的数据的序列。在内部，每个时间区间收到的数据都作为 RDD 存在，而 **DStream 是由这些 RDD 所组成的序列**
（因此得名“离散化”）。DStream 可以从各种输入源创建，比如 Flume、Kafka 或者 HDFS。创建出
来的 DStream 支持两种操作，一种是转化操作（transformation），会生成一个新的DStream，另一种是输出
操作（output operation），可以把数据写入外部系统中。DStream提供了许多与 RDD 所支持的操作相类似的
操作支持，还增加了与时间相关的新操作，比如滑动窗口


## 输入源

除核心数据源外，还可以用附加数据源接收器来从一些知名数据获取系统中接收的数据，
这些接收器都作为 Spark Streaming 的组件进行独立打包了。它们仍然是 Spark 的一部分，
不过你需要在构建文件中添加额外的包才能使用它们。现有的接收器包括 Twitter、Apache 
Kafka、Amazon Kinesis、Apache Flume，以及 ZeroMQ。可以通过添加与 Spark 版本匹配
的 Maven 工件 spark-streaming-[projectname]_2.10 来引入这些附加接收器。

### Apache Kafka



