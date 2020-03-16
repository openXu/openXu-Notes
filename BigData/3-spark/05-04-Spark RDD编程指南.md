# 1 总览

每个Spark应用程序都包含一个驱动程序，该程序运行用户的main功能并在集群上执行各种并行操作。Spark提供的
主要抽象是**弹性分布式数据集（RDD）**，它是跨集群节点划分的元素的集合，可以并行操作。通过从Hadoop文
件系统（或任何其他Hadoop支持的文件系统）中的文件或驱动程序中现有的Scala集合开始并进行转换来创建RDD。
用户还可以将RDD保留在内存中，以使其能够在并行操作中有效地重用。RDD自动从节点故障中恢复。

Spark中的第二个抽象是可以在并行操作中使用的**共享变量**。默认情况下，当Spark作为一组任务在不同节点上
并行运行一个函数时，它会将函数中使用的每个变量的副本传送给每个任务。有时，需要在任务之间或任务与驱动程
序之间共享变量。Spark支持两种类型的共享变量:**广播变量**（可用于在所有节点上的内存中缓存值）和累加器
(将工作节点中的值聚合到驱动器程序中的简单语法，例如计数器和总和)

本文以Spark 2.4.0版本为基础，只提供Scala语言代码，详情请见[RDD Programming Guide](http://spark.apache.org/docs/2.4.0/rdd-programming-guide.html)

# 2 RDD创建

[SparkContext API](http://spark.apache.org/docs/2.4.0/api/scala/index.html#org.apache.spark.rdd.RDD)

```Scala
// Spark程序必须做的第一件事是创建一个SparkContext对象，该对象告诉Spark如何访问集群
Logger.getLogger("org.apache.spark").setLevel(Level.WARN)
Logger.getLogger("org.apache.kafka").setLevel(Level.WARN)
val conf = new SparkConf(true)
  .setMaster(ConstansConfig.sparkHost)
  .setAppName("ExamStream")
val sc = new SparkContext(conf)

//调用SparkContext的parallelize方法来从驱动程序中现有的Scala集合创建RDD
val data = Array(1, 2, 3, 4, 5)
val distData1 = sc.parallelize(data)
println(distData1.collect().toList)
/*
 * Spark可以从Hadoop支持的任何存储源创建分布式数据集，包括您的本地文件系统，HDFS，Cassandra，HBase，
 * Amazon S3等。Spark支持文本文件、SequenceFiles和任何其他Hadoop InputFormat。
 *
 * 可以使用`SparkContext.textFile()`方法创建文本文件RDD 。此方法需要一个URI的文件，支持以下文件系统：
 * 本地 “常规”文件系统 : val rdd = sc.textFile("file:///home/holden/happypandas.gz")
 * HDFS文件系统 ：val rdd = sc.textFile("hdfs://master:port/path")
 * Amazon S3 ： val rdd = sc.textFile("s3n://bucket/my-Files/xx.txt") 用 Amazon S3 来存储大量数据正日益流行。
 *              当计算节点部署在 Amazon EC2 上的时候，使用 S3 作为存储尤其快，但是在需要通过公网访问数据时性能会差很多。
 *
 * Spark的所有基于文件的输入方法（包括textFile）都支持目录，压缩文件和通配符(星号)。
 * 例如:
 * textFile("/my/directory")   file://可省略
 * textFile("/my/directory/星号.txt")
 * textFile("/my/directory/星号.gz")
 *
 * 该textFile方法还采用可选的第二个参数来控制文件的分区数。默认情况下，Spark为文件的每个块创建一个分区（HDFS中的块默认为128MB），
 * 但可以通过传递更大的值来请求更大数量的分区。请注意，分区不能少于块。
 */
//1 文本文件创建RDD，会将文本文件中每一行作为一个元素组成 RDD[String]
val distFile2 = sc.textFile("file:///D:/WorkSpace/IntelliJ/HadoopSimple/testFile/file1.txt")
distFile2.foreach(str => {
  println("2.文本文件每行:"+str)
})
//除了文本文件，Spark的scala API海之吃下面几种数据格式
//2 读取包含多个小文本文件的目录，将每个小文本文件作为（文件名，内容）对返回 RDD[(String文件名, String文件内容)]
val distFile3 = sc.wholeTextFiles("D:/WorkSpace/IntelliJ/HadoopSimple/testFile")   //file:/D:/WorkSpace/IntelliJ/HadoopSimple/.idea
distFile3.foreach(el=>{
  println("3.文件名:"+el._1+"   文件内容"+el._2)
})
//3 SequenceFile
val seqData = List(("ABC", 1), ("BCD", 2), ("CDE", 3), ("DEF", 4), ("FGH", 5))
val seqRdd = sc.parallelize(seqData, 1)
val dir = "file:///D:/WorkSpace/IntelliJ/HadoopSimple/testFile/sequenceFile-" + System.currentTimeMillis()
seqRdd.saveAsSequenceFile(dir)
val distFile4 = sc.sequenceFile[String, Int](dir + "/part-00000")
println(distFile4.collect().map(elem => (elem._1 + ", " + elem._2)).toList)
/*
 * 4 对于其他Hadoop InputFormat，可以使用该SparkContext.hadoopRDD()，
 * 该方法采用任意JobConf输入格式类，键类和值类。使用与使用输入源进行Hadoop作业相同的方式设置这些内容。
 * 您还可以SparkContext.newAPIHadoopRDD基于“新” MapReduce API（org.apache.hadoop.mapreduce）将其用
 * 于InputFormats 。
 */
```

# 3 弹性分布式数据集（RDD:Resilient Distributed Datasets）

[RDD API](http://spark.apache.org/docs/2.4.0/api/scala/index.html#org.apache.spark.rdd.RDD)

RDD是Spark中的基本抽象，表示可以并行操作的元素的不变分区集合。这个类包含了所有可用的RDDS的基本操作，
如map，filter和persist。另外`org.apache.spark.rdd.PairRDDFunctions`包含仅在键值对的RDD上可用的操
作，例如groupByKey和join。 `org.apache.spark.rdd.DoubleRDDFunctions`包含仅在Doubles的RDD上可用
的操作；`org.apache.spark.rdd.SequenceFileRDDFunctions`包含可以另存为SequenceFilesRDD上可用的操作。
所有操作都可以通过隐式自动在任何正确类型的RDD（例如RDD [(Int，Int)]）上使用。

## 3.3 RDD操作
### 3.3.1 基本
### 3.3.2 将函数传递给Spark
### 3.3.3 了解闭包
#### 例
#### 本地与集群模式
#### RDD的打印元素
### 3.3.4 使用键值对
### 3.3.5 转变
### 3.3.6 动作
### 3.3.7 随机操作
#### 背景
#### 绩效影响


## 4.4RDD持久性
### 选择哪个存储级别？
### 删除资料
# 5.共享变量
## 广播变量
## 蓄能器
# 6.部署到集群
# 7.从Java / Scala启动Spark作业
# 8.单元测试
# 9.从这往哪儿走