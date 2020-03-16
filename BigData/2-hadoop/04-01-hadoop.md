 
# 1. 初识Hadoop

古代用牛拉重物，当一头牛拉不动一根圆木时，从来没考虑要培育出一种更强壮的牛。
数据大爆炸的时代，不该想方设法打造超级计算机，而应该千方百计综合利用更多计
算机来解决问题。

## 1.1 数据

国际数据公司（IDC）预测2020年全球数据总量为44ZB=10亿TB。

大数据胜于好算法：不论算法有多牛，基于小数据的推荐效果都不如基于大量可用数据
的一般算法推荐效果。

## 1.2 数据的存储与分析

**文件分块，并行读取**
> 硬盘存储容量不断提升，访问速度（硬盘读取速度）却没有与时俱进。如果将一个文件
> 分割成100份，放在100个硬盘上，同时并行读取，那就很快读完所有数据。而且每个硬
> 盘可存放很多个文件的部分数据。

**数据复本，避免数据丢失**
> 一旦使用多个硬盘就可能个别硬盘发生故障，为了避免数据丢失，最常见的做法就是复制：
> 保存数据的复本，一旦系统发生故障就可以使用复本。冗余硬盘阵列（RAID）就是这个原理。
> Hadoop文件系统**(Hadoop Distributed FileSystem，HDFS)**也是一类，不过他采取的方法
> 稍有不同。

**分散的数据分析模型**
> 大多数分析任务需要结合大部分数据来共同完成分析，也就是说可能需要很多硬盘中读取
> 的数据结合使用。**MapReduce**提出一个编程模型，该模型抽象出这些硬盘读/写问题并将
> 其转换为对一个数据集（键值对组成）的计算，这样的计算由map和reduce两部分组成，只
> 有在这两部分提供对外的接口。与HDFS类似，MapReduce自身也有很高的可靠性。

简而言之，Hadoop为我们提供一个可靠且可扩展的**存储和分析平台**。Hadoop可靠(运行在
商用硬件上)且成本可控(开源)。

**HDFS负责将数据均匀的存储在计算机集群硬盘上，按照MapReduce编程模型编写数据分析的程序，
YARN负责将程序任务调度到有存储数据的各台机器节点上运行。**


# 2. 关于MapReduce

`MapReduce`是一种可用于数据处理的编程模型。MapReduce程序本质上是并行运行的，可将大规
模的数据分析任务分发给任何一个拥有足够多机器的数据中。MapReduce的优势在于处理大规模数
据集。

## 2.1 气象数据集

分布在全球各地的气象传感器每隔一小时收集气象数据和大量日志数据，这些数据是半结构化的形
式按行存储，一行就是一条记录，该记录中按顺序记录了时间、温度、经纬度等信息。数据文件按
照日期和气象站进行组织，从1901年到2001年每年都有一个目录，每个目录中包含各个气象站该年
气象数据的打包文件。

**一行记录**

`0063701199099999**195001010300(#时间)**4...9999999N9**+00001(正负温度)**+9999999999...`

**1990年气象数据目录：**
```xml
	|-- 010010-99999-1991.gz
	|-- 010014-99999-1991.gz
	...
	|-- 010100-99999-1991.gz
	|-- 010101-99999-1991.gz
```
	
鉴于Hadoop对少量的大文件的处理更容易高效，所以将每个年度的数据解压缩到一个文件中，并以
年份命名（可由一个MapReduce程序完成），数据整理后结果复制到HDFS中:
```xml
s3n://hadoopbook/ncdc/raw/isd-1901.tar.bz2
s3n://hadoopbook/ncdc/raw/isd-1902.tar.bz2
...
s3n://hadoopbook/ncdc/raw/isd-2001.tar.bz2
```

## 2.2 使用Unix工具分析数据

对于上面的气象数据集，现在要计算出每年全球气温的最高记录是多少？使用传统处理按行存储数据
的工具**awk**，下面的脚本用于计算每年最高气温：

```JavaScript
#!/usr/bin/env bash
//循环遍历按年压缩的数据文件
for year in all/*   
do
	//显示年份
    echo -ne 'basename $year .gz'"\t"
	gunzip -c $year | \
		//使用awk处理每个文件，提取气温和质量代码
		awk'{temp = substr($0, 88, 5) + 0;
			q = substr($0, 93, 1);
			//气温值是否有效&&气温值是否有疑问或者错误
			if( temp!=9999 && q ~ /[01459]/ && temp > max)}
				//如果数据读取正确，将最大值更新
				max = temp
			END{print max}'
	done
```
运行结果
```JavaScript
% ./max_temperature.sh
1901   31
1902   24
1903   28
1904   25
...
```

为了加快处理速度，需要并行处理程序来进行数据分析。理论上讲很简单，每个线程负责处理
不同年分的数据，但是存在一些问题：

- 将**任务划分成大小相同**的作业不是容易的事（每个压缩包里的数据量可能不一样），总运行时间取决与处理最大文件所需时间
- 解决上面的问题需要把压缩包文件分割成小块后再计算，计算之后还需要**合并某一年的数据**得出最大值
- 每个线程运行结果还需要**按年份排序**输出
- 受限于单台计算机的处理能力，如果用多台计算机，大环境其他因素就会相互影响，**协调性和可靠性**不可控

## 2.3 使用Hadoop分析数据

上面使用并行的方式是可行的，但实际上很麻烦，Hadoop帮我们将并行产生的问题解决了。

- HDFS分布式存储：将数据均匀分布在各个节点
- MapReduce数据分析编程模型：解决数据计算、合并、排序
- YARN资源调度：负责程序在各节点上的调度

**map和reduce**

MapReduce任务过程分为两个处理阶段：map阶段和reduce阶段，每个阶段都**以键值对作为输入和输出**。
程序员需要写两个函数：map函数和reduce函数。

**MapReduce框架数据流**

```xml
### 原始数据
0063701199099999**195001010300(#时间)**4...9999999N9**+00021(正负温度)**+9999999999...
0063701199099999**195001010400(#时间)**4...9999999N9**+00022(正负温度)**+9999999999...
...
0063701199099999**195101020300(#时间)**4...9999999N9**+00025(正负温度)**+9999999999...
0063701199099999**195101020400(#时间)**4...9999999N9**+00023(正负温度)**+9999999999...
...

### map函数提取数据：
(1950, 21)
(1950, 22)
(1951, 25)
(1951, 23)
...

### sort排序：
(1950, [21, 22])
(1951, [23, 25])

### reduce函数遍历列表找出最大值：
1950, 22
1951, 25

```

**Java代码实现**

Hadoop提供了MapReduce的API，我们可以使用非Java语言编写map和reduce函数。Hadoop Streaming(数据流)
使用Unix标准流作为Hadoop和应用程序之间的接口，所以我们可以使用任何编程语言通过标准输入/输出来写
MapReduce程序。下面贴上Java的实现：

```Java
/****************1、map函数****************/
//Hadoop提供一套可优化网络序列化传输的基本类型，而不直接使用Java内嵌类
import org.apache.hadoop.oi.IntWritable;    //相当于Java中Integer
import org.apache.hadoop.oi.LongWritable;   //相当于Java中Long
import org.apache.hadoop.oi.Text;           //相当于Java中String
import org.apache.hadoop.mapreduce.Mapper;
/**
 * Mapper接口四个泛型分别代表map函数:
 * 输入Key(偏移量)
 * 输入value(一行记录文本)
 * 输出Key(年分)
 * 输出value(温度值)
 */
public class MaxTemperatureMapper extends MapReduceBase 
	implements Mapper<LongWritable, Text, Text, IntWritable>{
	@Override
	public void map(LongWritable key, Text value, Context context){
		String line = value.toString;    
		String year = line.substring(15, 19);  //截取年份
		int airTemperature;                    //截取温度值
		if(line.charAt(87)=='+'){
			airTemperature = Integer.parseInt(line.substring(88, 92));
		}else{
			airTemperature = Integer.parseInt(line.substring(87, 92));
		}
		// 检验气温值是否有效
		String quality = line.substring(92, 93);
		if(airTemperature != MISSING && quality.matches("[01459]")){
			// 将合法的年份气温值写入标准输出
			context.writ(new Text(year), new IntWritable(airTemperature));
			//输出(1950, 21)(1950, 22)...(1951, 25)(1951, 23)...
		}
	}	
}

/****************2、reduce函数****************/
import java.io.IOException;
import org.apache.hadoop.oi.IntWritable;
import org.apache.hadoop.oi.Text;
import org.apache.hadoop.mapreduce.Reducer;
public class MaxTemperatureReducer extends 
	Reducer<Text, IntWritable, Text, IntWritable>{
	//reduce函数输入为(1950, [21, 22...])
	@Override
	public void reduce(Text key, Iterable<IntWritable> values, Context context){
		int maxValue = Integer.MIN_VALUE;
		//遍历年份中的温度值，求最大值
		for(IntWritable value : values)
			maxValue = Math.Max(maxValue, value.get());
		context.write(key, new IntWritable(maxValue));
	}
}

/****************3、运行MapReduce作业****************/
import java.io.IOException;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.oi.IntWritable;
import org.apache.hadoop.oi.Text;
import org.apache.hadoop.mapreduce.Jab;
import org.apache.hadoop.mapreduce.input.FileInputFormat;
import org.apache.hadoop.mapreduce.input.FileOutputFormat;
public class MaxTemperature {
    public static void main(String[] args) {
        if(args.length!=2){
            System.err.println("请传入输入输出参数");
            System.exit(-1);
        }
        //Job对象指定作业执行规范，用于控制整个作业运行
        Job job = new Job();
        //传递需要执行的类，Hadoop会利用这个类查找包含它的JAR文件
        job.setJarByClass(MaxTemperature.class);
        job.setJobName("Max Temperature");
        // 指定输入和输出数据的路径，输入路径和调用多次实现多路径输入
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        //指定map类型和reduce类型
        job.setMapperClass(MaxTemperatureMapper.class);
        job.setReducerClass(MaxTemperatureReducer.class);
        //设置map函数输出类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        System.exit(job.waitForCompletion(true)?0:1);
    }
}

/****************4、运行jar****************/
% export HADOOP_CLASSPATH=hadoop-examples.jar
%hadoop MaxTemperature input/ncdc/sample.txt output
输出数据写入output目录，每个reducer都有一个输出文件，现在我们只在一个节点运行，所以只有一个输出：
% cat output/part-r-00000
1950   23
1951   25
...
```

## 2.4 横向扩展

上面的示例介绍了MapReduce针对少量输入数据是如何工作的，为了实现横向扩展，需要把数据存储在
分布式文件系统（典型为HDFS），通过使用Hadoop资源管理系统YARN，Hadoop可以将MapReduce计算转移
到存储有部分数据的各台机器上，我们看看具体过程。

- Hadoop将作业（MapRedyce作业Job是执行的一个工作单元）分成若干个任务（task）执行，其中包含两类任务（map任务和reduce任务），这些任务运行在集群节点上，通过YARN调度
- Hadoop将MapReduce的输入数据划分成等长的小数据块（分片），通常趋向于HDFS的一个块的大小默认128MB
- 为每个分片构建一个map任务，由该任务运行用户定义的map函数处理分片中的每条记录
- YARN调度map任务在有存储输入数据的节点上运行，以获得最佳新能，无需使用宝贵的集群带宽资源
- 所有节点上map任务输出数据通过网络传输发送到运行reduce任务节点作为输入（reduce任务也可能有多个），执行reduce任务
- 将reduce输出存储在HDFS中实现可靠存储


# 3. Hadoop分布式文件系统
 
## HDFS概念

**数据块**

HDFS块默认为128MB，与单一磁盘上的文件系统相似，HDFS上的文件也被划分为块大小的多个分块最为独立存储单元。
与面向单一磁盘文件系统不同的是，HDFS中小于一个块大小的文件不会占用整个块的空间。

HDFS块比磁盘的块大，目的是为了最小化寻址开销。但是也不能设置得过大，MapReduce中map任务通常一次只
处理一个块中的数据，如果任务数太少（少于节点数量），作业的运行速度就会比较慢。

**namenode和datanode**

HDFS集群有两类节点，一个namenode(管理节点)和多个datanode(工作节点)。

namenode管理文件系统命名空间，维护文件系统树及整棵树内所有的文件和目录，这些信息以两个文件（命名空间镜像文件、编辑日志文件）
形式永久保存在本地磁盘上。namenode也记录这每个文件中各个块所在的数据节点信息，这些信息会在系统启动时根据节点信息重建。

datanode是文件系统的工作节点，它们根据需要存储并检索数据块（受客户端或namenode调度），并且定期向namenode发送它们所存储的块的列表。

没有namenode，文件系统将无法使用，如果运行namenode服务的机器毁坏，文件系统上所有的文件将丢失，因为不知道如何根据
datanode的块重建文件。因此对namenode是西安容错非常重要，Hadoop为此提供两种机制：

- 备份那些组成文件系统元数据持久状态的文件。一般的配置是，将持久状态写入本地磁盘的同时，写入一个远程挂载的网络文件系统（NFS）
- 运行一个辅助namenode，但他不能被用作namenode，而是定期合并编辑日志与命名空间镜像，以防止编辑日志过大。一般把存储在NFS上的namenode元数据复制到辅助namenode并作为新的主namenode运行

**块缓存**

通常datanode从磁盘中读取块，但对于访问频繁的文件，其对应的块可能被显式地缓存在datanode的内存中，
以**堆外块缓存**的形式存在。作业调度器通过在缓存块的datanode上运行任务，**可以利用快缓存的优势提高读操作的性能**。

默认情况下一个块仅缓存在一个datanode内存中，当然可以针对每个文件配置datanode的数量。用户或应用通过
缓存池（cache pool）中增加一个cache directive来告诉namenode需要缓存那些文件及存多久。缓存池是一个
用于管理缓存权限和资源使用的管理性分组。

**联邦HDFS**

namenode在内存中保存文件系统中每个文件和每个数据块的引用关系，这意味着对于一个拥有大量文件的超大集群来说，
**内存将成为限制系统横向扩展的瓶颈**。在2.x发行版本系列中引入的联邦HDFS允许系统通过添加namenode实现扩展，
其中每个namenode管理文件系统命名空间的一部分。

在联邦环境下，每个namenode伟华一个命名空间卷，由命名空间元数据和一个数据块池组成，数据块池包含该命名空间下
文件的所有数据块。命名空间卷之间是相互独立的，两两之间并不相互通信，一个namenode的失效不会影响其他namenode
伟华的命名空间可用性。

**HDFS的高可用性**

Hadoop2针对上述问题增加了对HDFS高可用性的支持。在这一实现中，配置了一对活动-备用namenode。当活动namenode失效，
备用namenode就会接管它的任务并开始服务于来自客户端的请求，不会有明显中断。

系统中有一个称为**故障转移控制器**的新实体，管理着将活动namenode转移为备用namenode的转换过程。有多种故障转移
控制器，但默认的一种是使用了ZooKeeper来确保有且仅有一个活动namenode。每个namenode运行着一个轻量级的故障转移控
制器，其工作就是监视宿主namenode是否失效（简单心跳机制实现）并在namenode失效时进行故障切换。

## 命令行接口

```xml
### 伪分布配置有两个属性项解释
#### 设置Hadoop的默认文件系统，默认端口8082 上运行namenode
fs.defaultFS = hdfs://localhost/ 
#### HDFS设置文件系统块复本数量，默认为3
dfs.replication = 1   

### 文件系统基本操作
#### 从本地文件系统将一个文件复制到HDFS系统的user/tom目录下
% hadoop fs -copyFromLocal input/docs/quangle.txt \ hdfs://localhost/user/tom/quangle.txt
#### 查看HDFS文件列表
% hadoop fs -mkdir books
% hadoop fs -ls .
Found 2 items
drwxr-xr-x - tom supergroup 0 2014-10-04 13:22 books
-rw-r--r-- 1 tom supergroup 119 2014-10-04 13:21 quangle.txt
```

默认情况下，Hadoop运行时安全措施处于停用模式，客户端身份是没有经过认证的。为防止用户或自动工具及程序
意外修改或删除文件系统的重要部分，启用权限控制还是很重要的（参见dfs.permission.enabled属性）

有一个超级用户的概念，超级用户是namenode进程的标识，对于超级用户，系统不会执行任何权限检查。

## Hadoop文件系统

Hadoop对文件系统提供了许多接口，它一般使用URI方案来选取合适的文件系统实例进行交互。

```xml
### 列出本地文件系统根目录下的文件
% hadoop fs -ls file:///

```


# 4. 关于YARN



# 第5章 Hadoop的I/O操作



# 第2章 关于MapReduce

