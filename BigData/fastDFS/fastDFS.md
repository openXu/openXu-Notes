
[官方网站](https://github.com/happyfish100/fastdfs)

## 1.小文件系统

### 引言

业务中用到的富媒体（图片、音视频、Office文档等），我们很少会去存储到数据库中，更多的时候我们会把它们放在文件系统里。但是单机时代诞生的文件系统，真的是最适合存储这些富媒体数据的么？不，文件系统需要改变，因为：

1. 伸缩性。单机文件系统的第一个问题是单机容量有限，在存储规模超过一台机器可管理的时候，应该怎么办。
2. 性能瓶颈。通常，单机文件系统在文件数目达到临界点后，性能会快速下降。在4TB的大容量磁盘越来越普及的今天，这个临界点相当容易到达。
3. 可靠性要求。单机文件系统通常只是单副本的方案，但是今天单副本的存储早已无法满足业务的可靠性要求。数据需要有冗余（比较经典的做法是3副本），并且在磁盘损坏时及早修复丢失的数据，以避免所有的副本损坏造成数据丢失。
4. 可用性要求。单机文件系统通常只是单副本的方案，在该机器宕机后，数据就不可读取，也不可写入。

在分布式存储系统出现前，有一些基于单机文件系统的改良版本被一些应用采纳。比如在单机文件系统上加 RAID5 做数据冗余，来解决单机文件系统的可靠性问题。假设 RAID5 的数据修复时间是1天（实际上往往做不到，尤其是业务系统本身压力比较大的情况下，留给 RAID 修复用的磁盘读写带宽很有限），这种方案单机的可靠性大概是100年丢失一次数据（即可靠性是2个9）。看起来尚可？但是你得小心两种情况。一种是你的集群规模变大，你仍然沿用这个土方法，比如你现在有 100 台这样的机器，那么就会变成1年就丢失一次数据。另一种情况是如果实际数据修复时间是 3 天，那么单机的可靠性就直降至4年丢失一次数据，100台就会是15天丢失一次数据。这个数字显然无法让人接受。

　　Google GFS 是很多人阅读的第一份分布式存储的论文，这篇论文奠定了 3 副本在分布式存储系统里的地位。随后 Hadoop 参考此论文实现了开源版的 GFS —— HDFS。但关于 Hadoop 的 HDFS 实际上业界有不少误区。GFS 的设计有很强的业务背景特征，本身是用来做搜索引擎的。HDFS 更适合做日志存储和日志分析（数据挖掘），而不是存储海量的富媒体文件。因为：

1. HDFS 的 block 大小为 64M，如果文件不足 64M 也会占用 64M。而富媒体文件大部分仍然很小，比如图片常规尺寸在 100K 左右。有人可能会说我可以调小 block 的尺寸来适应，但这是不正确的做法，HDFS 的架构是为大文件而设计的，不可能简单通过调整 block 大小就可以满足海量小文件存储的需求。
2. HDFS 是单 Master 结构，这决定了它能够存储的元数据条目数有限，伸缩性存在问题。当然作为大文件日志型存储，这个瓶颈会非常晚才遇到；但是如果作为海量小文件的存储，这个瓶颈很快就会碰上。
3. HDFS 仍然沿用文件系统的 API 形式，比如它有目录这样的概念。在分布式系统中维护文件系统的目录树结构，会遭遇诸多难题。所以 HDFS 想把 Master 扩展为分布式的元数据集群并不容易。

　　分布式存储最容易处理的问题域还是单键值的存储，也就是所谓的 Key-Value 存储。只有一个 Key，就意味着我们可以通过对 Key 做 Hash，或者对 Key 做分区，都能够让请求快速定位到特定某一台存储机器上，从而转化为单机问题。这也是为什么在数据库之后，会冒出来那么多 NoSQL 数据库。因为数据库和文件系统一样，最早都是单机的，在伸缩性、性能瓶颈（在单机数据量太大时）、可靠性、可用性上遇到了相同的麻烦。NoSQL 数据库的名字其实并不恰当，他们更多的不是去 SQL，而是去关系（我们知道数据库更完整的称呼是关系型数据库）。有关系意味着有多个索引，也就是有多个 Key，而这对数据库转为分布式存储系统来说非常不利。

### 选型

目前流行的分布式文件系统有许多，如MooseFS、FastDFS、GlusterFS、Ceph、MogileFS等，**常见的分布式存储对比**如下：

- **FastDFS**：一个开源的轻量级分布式文件系统，是纯C语言开发的。它对文件进行管理，功能包括：文件存储、文件同步、文件访问（文件上传、文件下载）等，解决了大容量存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站、视频网站等等。FastDFS 针对大量小文件存储有优势。
- **GlusterFS**：主要应用在集群系统中，具有很好的可扩展性。软件的结构设计良好，易于扩展和配置，通过各个模块的灵活搭配以得到针对性的解决方案。GlusterFS适合大文件，小文件性能相对较差。
- **MooseFS**：比较接近GoogleFS的c++实现，通过fuse支持了标准的posix，支持FUSE，相对比较轻量级，对master服务器有单点依赖，用perl编写，算是通用的文件系统，可惜社区不是太活跃，性能相对其他几个来说较差，国内用的人比较多。
- **Ceph**：C++编写，性能很高，支持Fuse，并且没有单点故障依赖；Ceph 是一种全新的存储方法，对应于 Swift 对象存储。在对象存储中，应用程序不会写入文件系统，而是使用存储中的直接 API 访问写入存储。因此，应用程序能够绕过操作系统的功能和限制。在openstack社区比较火，做虚机块存储用的很多！
- **GoogleFS**：性能十分好，可扩展性强，可靠性强。用于大型的、分布式的、对大数据进行访问的应用。运用在廉价的硬件上。

**针对FPC119系统业务平台媒体数据特点——多图片、短视频（大小不超过50M，均为小文件），采用FastDFS构建系统分布式文件系统**

### FastDFS 介绍

 FastDFS是用c语言编写的一款开源的分布式文件系统。FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

**FastDFS架构**

 FastDFS架构包括 Tracker server和Storage server。客户端请求Tracker server进行文件上传、下载，通过Trackerserver调度最终由Storage server完成文件上传和下载。Trackerserver作用是负载均衡和调度，通过Trackerserver在文件上传时可以根据一些策略找到Storageserver提供文件上传服务。可以将tracker称为追踪服务器或调度服务器。Storageserver作用是文件存储，客户端上传的文件最终存储在Storage服务器上，Storage server没有实现自己的文件系统而是利用操作系统 的文件系统来管理文件。可以将storage称为存储服务器。

如下图：

![img](https://img-blog.csdn.net/20180516113227670?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

2.1  Tracker 集群
         FastDFS集群中的Tracker server可以有多台，Trackerserver之间是相互平等关系同时提供服务，Trackerserver不存在单点故障。客户端请求Trackerserver采用轮询方式，如果请求的tracker无法提供服务则换另一个tracker。

2.2  Storage集群
         Storage集群采用了分组存储方式。storage集群由一个或多个组构成，集群存储总容量为集群中所有组的存储容量之和。一个组由一台或多台存储服务器组成，组内的Storage server之间是平等关系，不同组的Storageserver之间不会相互通信，同组内的Storageserver之间会相互连接进行文件同步，从而保证同组内每个storage上的文件完全一致的。一个组的存储容量为该组内存储服务器容量最小的那个，由此可见组内存储服务器的软硬件配置最好是一致的。

 采用分组存储方式的好处是灵活、可控性较强。比如上传文件时，可以由客户端直接指定上传到的组也可以由tracker进行调度选择。一个分组的存储服务器访问压力较大时，以可在该组增加存储服务器来扩充服务能力（纵向扩容）。当系统容量不足时，可以增加组来扩充存储容量（横向扩容）。

2.3  Storage状态收集
         Storage server会连接集群中所有的Tracker server，定时向他们报告自己的状态，包括磁盘剩余空间、文件同步状况、文件上传下载次数等统计信息。

2.4  文件上传流程

![img](https://img-blog.csdn.net/20180516113258902?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

​     客户端上传文件后存储服务器将文件ID返回给客户端，此文件ID用于以后访问该文件的索引信息。文件索引信息包括：组名，虚拟磁盘路径，数据两级目录，文件名。

![img](https://img-blog.csdn.net/20180516113315796?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

n 组名：文件上传后所在的storage组名称，在文件上传成功后有storage服务器返回，需要客户端自行保存。

n 虚拟磁盘路径：storage配置的虚拟路径，与磁盘选项store_path*对应。如果配置了store_path0则是M00，如果配置了store_path1则是M01，以此类推。

n 数据两级目录：storage服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。

n 文件名：与文件上传时不同。是由存储服务器根据特定信息生成，文件名包含：源存储服务器IP地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。

1.2.5  文件下载流程

![img](https://img-blog.csdn.net/20180516113332205?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

tracker根据请求的文件路径即文件ID 来快速定义文件。

比如请求下边的文件：

![img](https://img-blog.csdn.net/2018051611335055?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

1.通过组名tr端访acker能够很快的定位到客户端需要访问的存储服务器组是group1，并选择合适的存储服务器提供客户问。 

2.存储服务器根据“文件存储虚拟磁盘路径”和“数据文件两级目录”可以很快定位到文件所在目录，并根据文件名找到客户端需要访问的文件。


## 2.存储资源分配

**1. Tracker** : master(192.168.85.200), ...

**2. Storage**: master(192.168.85.200), node1(192.168.85.201)

**3. Nginx** : node1(92.168.85.201)

## 3.部署

[官方安装指南](https://github.com/happyfish100/fastdfs/blob/master/INSTALL)

### 1.安装libfastcommon & FastDFS 

FastDFS Server仅支持unix系统，在Linux和FreeBSD测试通过。V5.0以前的版本依赖**libevent**(V5.0以后不再依赖)；
v5.04开始依赖**libfastcommon**，[github地址](https://github.com/happyfish100/libfastcommon)
v5版本从v5.05开始才是稳定版本，请使用v5版本的同学尽快升级到v5.05或更新的版本，建议升级到v5.12。

[libfastcommon下载](https://github.com/happyfish100/libfastcommon/releases)
[FastDFS下载](https://github.com/happyfish100/fastdfs/releases)

若出现libfastcommon版本不匹配问题，请执行如下命令：/bin/rm -rf /usr/local/lib/libfastcommon.so /usr/local/include/fastcommon

```xml
#### ★创建软件存放目录 /soft/FastDFS/, 上传软件包并解压
root@Master:~# cd /soft/
root@Master:/soft# mkdir FastDFS
root@Master:/soft# mv /home/openxu/Desktop/fastdfs-6.06.tar.gz /soft/FastDFS/fastdfs-6.06.tar.gz
root@Master:/soft# mv /home/openxu/Desktop/libfastcommon-1.0.43.tar.gz /soft/FastDFS/libfastcommon-1.0.43.tar.gz
root@Master:/soft# tar -xzvf libfastcommon-1.0.43.tar.gz
root@Master:/soft# tar -xzvf FastDFS/fastdfs-6.06.tar.gz 

#### 编译安装
root@Master:/soft# cd FastDFS
root@Master:/soft/FastDFS# cd libfastcommon-1.0.43
root@Master:/soft/FastDFS/libfastcommon-1.0.43# ll
	drwxrwxr-x 5 root root  4096 12月 25 20:35 ./
	drwxr-xr-x 4 root root  4096 2月  11 15:20 ../
	drwxrwxr-x 2 root root  4096 12月 25 20:35 doc/
	-rw-rw-r-- 1 root root  1357 12月 25 20:35 .gitignore
	-rw-rw-r-- 1 root root 10301 12月 25 20:35 HISTORY
	-rw-rw-r-- 1 root root   674 12月 25 20:35 INSTALL
	-rw-rw-r-- 1 root root  1607 12月 25 20:35 libfastcommon.spec
	-rwxrwxr-x 1 root root  3253 12月 25 20:35 make.sh*
	drwxrwxr-x 2 root root  4096 12月 25 20:35 php-fastcommon/
	-rw-rw-r-- 1 root root  2776 12月 25 20:35 README
	drwxrwxr-x 3 root root  4096 12月 25 20:35 src/
root@Master:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
#### 编译报错，原因系统没有安装make命令
	./make.sh: line 14: gcc: command not found
	./make.sh: line 15: ./a.out: No such file or directory
	./make.sh: line 158: make: command not found
root@Master:/soft/FastDFS/libfastcommon-1.0.43# make
Command 'make' not found, but can be installed with:
apt install make      
apt install make-guile
#### 安装make命令
root@Master:/soft/FastDFS/libfastcommon-1.0.43# apt install make
#### 编译报错，原因是缺少GCC的开发环境
root@Master:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
./make.sh: line 14: gcc: command not found
./make.sh: line 15: ./a.out: No such file or directory
cc -Wall -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -g -O3 -c -o hash.o hash.c  
make: cc: Command not found
Makefile:62: recipe for target 'hash.o' failed
make: *** [hash.o] Error 127
#### 安装GCC的开发环境
root@Master:/soft/FastDFS/libfastcommon-1.0.43# apt-get install build-essential

#### ★重新编译安装
root@Master:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
root@Master:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh install
root@Master:/soft/FastDFS/libfastcommon-1.0.43# cd ..
root@Master:/soft/FastDFS# cd fastdfs-6.06/
root@Master:/soft/FastDFS/fastdfs-6.06# ./make.sh
root@Master:/soft/FastDFS/fastdfs-6.06# ./make.sh install

#### 安装成功后，/etc下会生成一个fdfs文件夹，下面有4个配置文件示例
root@Master:/soft/FastDFS/fastdfs-6.06# cd /etc/fdfs/
root@Master:/etc/fdfs# ls
client.conf.sample   storage_ids.conf.sample
storage.conf.sample  tracker.conf.sample
#### 拷贝配置文件
root@Master:/etc/fdfs# cp client.conf.sample client.conf
root@Master:/etc/fdfs# cp storage_ids.conf.sample storage_ids.conf
root@Master:/etc/fdfs# cp storage.conf.sample storage.conf
root@Master:/etc/fdfs# cp tracker.conf.sample tracker.conf

```

### 2.启动Tracker

**Tracker 集群:**FastDFS集群中的Tracker server可以有多台，Trackerserver之间是相互平等关系同时提供服务，Trackerserver
不存在单点故障。客户端请求Trackerserver采用轮询方式，如果请求的tracker无法提供服务则换另一个tracker。

```xml
#### 创建数据及log存放目录
root@Master:/etc/fdfs# cd /soft/FastDFS/
root@Master:/soft/FastDFS# mkdir -pv data/tracker
#### 修改tracker.conf文件
root@Master:/soft/FastDFS# vim /etc/fdfs/tracker.conf

	disabled=false ####默认关闭
	port=22122 ####默认端口
	base_path = /soft/FastDFS/data/tracker ####数据存放目录
	http.server_port=8501 ####设置http连接端口

#### 启动Tracker服务
root@Master:/soft/FastDFS# /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf start
#### 验证是否启动（查看系统进程ps -ef|grep fdfs）
root@Master:/soft/FastDFS# netstat -antp|grep trackerd
tcp        0      0 0.0.0.0:22122           0.0.0.0:*               LISTEN      35212/fdfs_trackerd 

#### 设置开机自启，修改/etc/rc.local文件在文件末尾加入

service fdfs_trackerd start

```

### 3.启动Storage

**Storage集群:**采用了分组存储方式。storage集群由一个或多个组构成，集群存储总容量为集群中所有组的存储容量之和。
一个组由一台或多台存储服务器组成，组内的Storage server之间是平等关系，不同组的Storageserver之间不会相互通信，
同组内的Storageserver之间会相互连接进行文件同步，从而保证同组内每个storage上的文件完全一致的。一个组的存储容量
为该组内存储服务器容量最小的那个，由此可见组内存储服务器的软硬件配置最好是一致的。

**Storage集群、组、存储器、存储路径的关系**
Storage集群
	|
	|--group1
		 |--server1
		 |     |----store_path0
		 |     |----store_path1
		 |--server2
		 |     |----store_path0
	|--group2
	     |--server2
		 |     |----store_path0
```xml
#### 创建数据及log存放目录
root@Master:/soft/FastDFS# mkdir -pv data/storage
#### 修改storage.conf文件
root@Master:/soft/FastDFS# vim /etc/fdfs/storage.conf

	disabled=false ####默认关闭
	group_name=group1 ####修改组名
	port=23000 ####默认端口号，同一组内端口号必须一致
	####storage服务数据存放路径
	base_path=/soft/FastDFS/data/storage 
	store_path_count=2 ####存储路径个数，需要和 store_path 个数匹配
	####文件存放目录，如果存储路径只有一个，可以默认为base_path；此处可设置多个目录。
	####当storage服务启动后，会自动在store_pathx配置的目录下创建data文件夹用于存储文件
	####下面配置的目录path0、path1必须先创建好
	store_path0=/soft/FastDFS/data/storage/path0
	store_path1=/soft/FastDFS/data/storage/path1
	tracker_server=192.168.85.200:22122 ####tracker地址
	http.server_port=8502 ####设置http连接端口号

#### 启动storage服务
root@Master:/soft/FastDFS# /usr/bin/fdfs_storaged /etc/fdfs/storage.conf start

#### 设置开机自启，修改/etc/rc.local文件在文件末尾加入
service fdfs_storaged start
 修改/etc/rc.local文件在文件末尾加入：`service fdfs_storaged start`

      修改rc.local文件执行权限: `chmod +x /etc/rc.local`
```


### 4.(可选)运行监视程序

```xml
root@Master:/soft/FastDFS# /usr/bin/fdfs_monitor /etc/fdfs/storage.conf
[2020-02-11 16:57:56] DEBUG - base_path=/soft/FastDFS/data/storage, connect_timeout=5, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=1, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

server_count=1, server_index=0

tracker server is 192.168.85.200:22122

group count: 1

Group 1:
group name = group1
disk total space = 50,267 MB
disk free space = 23,400 MB
trunk free space = 0 MB
storage server count = 1
active server count = 1
storage server port = 23000
storage HTTP port = 8502
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

	Storage 1:
		id = 192.168.85.200
		ip_addr = 192.168.85.200  ACTIVE
		http domain = 
		version = 6.06
		join time = 2020-02-11 16:57:52
		up time = 2020-02-11 16:57:52
		total storage = 50,267 MB
		free storage = 23,400 MB
		upload priority = 10
		store_path_count = 1
		subdir_count_per_path = 256
		storage_port = 23000
		storage_http_port = 8502
 ...
```

### 5.(可选)运行测试程序

```xml
#### 创建数据及log存放目录
root@Master:/soft/FastDFS# mkdir -pv data/client
#### 修改client.conf文件
root@Master:/soft/FastDFS# vim /etc/fdfs/client.conf

	base_path=/soft/FastDFS/data/client ####数据存放路径
	tracker_server=192.168.85.200:22122 ####tracker地址
	http.server_port=8503 ####设置http连接端口号

#### 上传文件测试，返回文件路径: 组名/虚拟磁盘路径/数据两级目录/文件名
root@Master:/soft/FastDFS# /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /home/openxu/Desktop/test.jar 

	group1/M00/00/00/wKhVyF5Cb3KAVk8hBYA4aq3gGlo156.jar

```



## 4.Nginx

我们在使用FastDFS部署一个分布式文件系统的时候，通过FastDFS的客户端API来进行文件的上传、
下载、删除等操作。同时通过FastDFS的HTTP服务器来提供HTTP服务。但是FastDFS的HTTP服务较
为简单，无法提供负载均衡等高性能的服务，所以FastDFS的开发者——淘宝的架构师余庆，为我们
提供了Nginx上使用的FastDFS模块（也可以叫FastDFS的Nginx模块）。

**FastDFS-Nignx-Module**

[下载](https://github.com/happyfish100/fastdfs-nginx-module/releases)
[安装指南](https://github.com/happyfish100/fastdfs-nginx-module/blob/master/INSTALL)

**Nginx**

[安装指南](https://docs.nginx.com/nginx-controller/admin-guide/installing-nginx-controller/)
[下载](https://github.com/happyfish100/fastdfs-nginx-module/releases)

```xml
#### ★上传安装包解压
root@Master:/soft/FastDFS# mv /home/openxu/Desktop/fastdfs-nginx-module-1.22.tar.gz /soft/FastDFS/fastdfs-nginx-module-1.22.tar.gz
root@Master:/soft/FastDFS# tar -zxvf fastdfs-nginx-module-1.22.tar.gz 

root@Master:/soft/FastDFS# mv /home/openxu/Desktop/nginx-1.17.8.tar.gz /soft/FastDFS/nginx-1.17.8.tar.gz
root@Master:/soft/FastDFS# tar -zxvf nginx-1.17.8.tar.gz

#### 进行nginx编译配置，设置prefix——编译后软件存放地址，增加模块 --add-module 增加fastdfs-nginx-module
root@Master:/soft/FastDFS/nginx-1.17.8# ./configure --prefix=/usr/local/nginx-fastdfs --add-module=/soft/FastDFS/fastdfs-nginx-module-1.22/src

#### ★编译、安装
root@Master:/soft/FastDFS/nginx-1.17.8# make
#### 报错，需要安装一些其他支持，然后再配置编译、安装
	make: *** No rule to make target 'build', needed by 'default'.  Stop.
	
	apt-get install gcc libpcre3 libpcre3-dev zlib1g zlib1g-dev openssl
	
	root@Master:/soft/FastDFS/nginx-1.17.8# apt-get install gcc
	root@Master:/soft/FastDFS/nginx-1.17.8# apt-get install libpcre3 libpcre3-dev
	root@Master:/soft/FastDFS/nginx-1.17.8# apt-get install zlib1g zlib1g-dev
	root@Master:/soft/FastDFS/nginx-1.17.8# apt-get install openssl
	
root@Master:/soft/FastDFS/nginx-1.17.8# make install

#### ★★★1.修改nginx配置
root@Master:/soft/FastDFS/nginx-1.17.8# cd /usr/local/nginx-fastdfs/
root@Master:/usr/local/nginx-fastdfs# ll
total 24
drwxr-xr-x  6 root root 4096 2月  12 17:35 ./
drwxr-xr-x 11 root root 4096 2月  12 17:35 ../
drwxr-xr-x  2 root root 4096 2月  12 17:35 conf/
drwxr-xr-x  2 root root 4096 2月  12 17:35 html/
drwxr-xr-x  2 root root 4096 2月  12 17:35 logs/
drwxr-xr-x  2 root root 4096 2月  12 17:35 sbin/

root@Master:/usr/local/nginx-fastdfs# vim conf/nginx.conf

    server {
        #### listen 8502：监听端口8502。
		#### ???不用 与/etc/fdfs/storage.conf 中的http.server_port=8502相对应
        listen       8502;
        #### 域名localhost
        server_name  localhost;

		#### 将链接映射出去，http://ip:8502/group[0-9]...相关的连接都会去找ngx_fastdfs_module
 ####		location ~/group[0-9]/M00/ { 只映射M00 http://ip:8502/group[0-9]/M00/...
        location ~/group[0-9]/ {
           ####  root   /soft/FastDFS/data/storage/data;
            ngx_fastdfs_module;
        }
        ...
    }

#### ★进入fastdfs安装目录，拷贝http.conf及mime.types至/etc/fdfs目录下
root@Master:/soft/FastDFS/fastdfs-6.06/conf# ll
total 100
drwxrwxr-x  2 root root  4096 12月 31 07:36 ./
drwxrwxr-x 12 root root  4096 12月 31 07:36 ../
-rw-rw-r--  1 root root 23981 12月 31 07:36 anti-steal.jpg
-rw-rw-r--  1 root root  1909 12月 31 07:36 client.conf
-rw-rw-r--  1 root root   965 12月 31 07:36 http.conf
-rw-rw-r--  1 root root 31172 12月 31 07:36 mime.types
-rw-rw-r--  1 root root 10246 12月 31 07:36 storage.conf
-rw-rw-r--  1 root root   620 12月 31 07:36 storage_ids.conf
-rw-rw-r--  1 root root  9138 12月 31 07:36 tracker.conf

root@Master:/soft/FastDFS/fastdfs-6.06/conf# cp http.conf mime.types /etc/fdfs/

#### ★将fastdfs-nginx-module安装目录中src下的mod_fastdfs.conf也拷贝至/etc/fdfs下并修改
root@Master:/soft/FastDFS/fastdfs-6.06/conf# cp /soft/FastDFS/fastdfs-nginx-module-1.22/src/mod_fastdfs.conf /etc/fdfs/
root@Master:/soft/FastDFS/fastdfs-6.06/conf# cd /etc/fdfs/
root@Master:/etc/fdfs# ls
client.conf         storage.conf.sample
client.conf.sample  storage_ids.conf
http.conf           storage_ids.conf.sample
mime.types          tracker.conf
mod_fastdfs.conf    tracker.conf.sample
storage.conf

#### ★★★2.修改mod_fastdfs.conf 
root@Master:/etc/fdfs# vim mod_fastdfs.conf 

	####保存日志目录
   	base_path=/soft/FastDFS/data/storage/logs 
	####★tracker服务器的IP地址以及端口号
   	tracker_server=192.168.85.200:22122 
	####storage服务器的端口号
   	storage_server_port=23000 
	####文件 url 中是否有 group 名
   	url_have_group_name = true
	####★当前storage server上的分组
	group_name=group1
	####★配置文件的存储路径
	store_path0=/soft/FastDFS/data/storage/path0  
	
	####★如果该Storage Service上只有一个组，下面可以不用配置了，一下用于配置一台存储器上有多个组的情况
	####(多组)设置组的个数，如果该Storage Service上只有一个组，默认为0
	group_count = 1 
    ####(多租)如果该Storage Service上有多个组，在文件的最后，设置group
   	[group1]   #### 多组情况下，组的索引，送group1开始
   	group_name=group1   #### 第一个组的组名
   	store_path_count=1
   	store_path0=/soft/FastDFS/data/storage/path0   ####存储路径
    ####store_path1=/soft/FastDFS/data/storage/path1   ####存储路径


#### ??★★★3.创建M00\M01至storage存储目录的软连接：
make a symbol link ${fastdfs_base_path}/data/M00 to ${fastdfs_base_path}/data,
         command line such as:
		 
ln -s /soft/FastDFS/data/storage/path0  /soft/FastDFS/data/storage/path0/M00
ln -s /soft/FastDFS/data/storage/path1  /soft/FastDFS/data/storage/path1/M01

#### 启动nginx
root@Master:/etc/fdfs# /usr/local/nginx-fastdfs/sbin/nginx 
ngx_http_fastdfs_set pid=43220

#### 杀死
root@Master:/usr/local/nginx-fastdfs/sbin# killall -9 nginx
root@Master:/usr/local/nginx-fastdfs/sbin# /usr/local/nginx-fastdfs/sbin/nginx -s stop
root@Master:/usr/local/nginx-fastdfs/sbin# /usr/local/nginx-fastdfs/sbin/nginx 

	ngx_http_fastdfs_set pid=44948

   
```

浏览器输入链接查看/下载已上传图片/文档:

http://192.168.85.200:8502/group1/M01/00/00/wKhVyF5FAg2AKdIdAAD2LMn7PBQ284.jpg
  


## 5.扩展


Storage集群
	|
	|--group1
		 |--server1 (Master : 192.168.85.200:23000)  Tracker+Storage+Nginx+fast-nginx-module   ngx_cache_purge
		 |     |----store_path0
		 |     |----store_path1
		 |--server2 (Node1 : 192.168.85.201:23000)   Storage+Nginx+fast-nginx-module
		 |     |----store_path0
		 |     |----store_path1
	|--group2
	     |--server2 (Node2 : 192.168.85.202:23000)  Storage+Nginx+fast-nginx-module
		 |     |----store_path0

### 纵向扩展:为分组新增Storage Server

```xml
#### 复制/soft/FastDFS和/etc/fdfs目录到节点上
root@Master:/soft# scp -r /soft/FastDFS/ root@192.168.85.201:/soft/FastDFS/
root@Master:/soft# scp -r /etc/fdfs/ root@192.168.85.201:/etc/fdfs

#### 编译安装
root@Node1:/soft# cd FastDFS
root@Node1:/soft/FastDFS# cd libfastcommon-1.0.43
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# ll
	drwxrwxr-x 5 root root  4096 12月 25 20:35 ./
	drwxr-xr-x 4 root root  4096 2月  11 15:20 ../
	drwxrwxr-x 2 root root  4096 12月 25 20:35 doc/
	-rw-rw-r-- 1 root root  1357 12月 25 20:35 .gitignore
	-rw-rw-r-- 1 root root 10301 12月 25 20:35 HISTORY
	-rw-rw-r-- 1 root root   674 12月 25 20:35 INSTALL
	-rw-rw-r-- 1 root root  1607 12月 25 20:35 libfastcommon.spec
	-rwxrwxr-x 1 root root  3253 12月 25 20:35 make.sh*
	drwxrwxr-x 2 root root  4096 12月 25 20:35 php-fastcommon/
	-rw-rw-r-- 1 root root  2776 12月 25 20:35 README
	drwxrwxr-x 3 root root  4096 12月 25 20:35 src/
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
#### 编译报错，原因系统没有安装make命令
	./make.sh: line 14: gcc: command not found
	./make.sh: line 15: ./a.out: No such file or directory
	./make.sh: line 158: make: command not found
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# make
Command 'make' not found, but can be installed with:
apt install make      
apt install make-guile
#### 安装make命令
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# apt install make
#### 编译报错，原因是缺少GCC的开发环境
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
./make.sh: line 14: gcc: command not found
./make.sh: line 15: ./a.out: No such file or directory
cc -Wall -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -g -O3 -c -o hash.o hash.c  
make: cc: Command not found
Makefile:62: recipe for target 'hash.o' failed
make: *** [hash.o] Error 127
#### 安装GCC的开发环境
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# apt-get install build-essential

#### ★重新编译安装
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh install
root@Node1:/soft/FastDFS/libfastcommon-1.0.43# cd ..
root@Node1:/soft/FastDFS# cd fastdfs-6.06/
root@Node1:/soft/FastDFS/fastdfs-6.06# ./make.sh
root@Node1:/soft/FastDFS/fastdfs-6.06# ./make.sh install

#### ★启动新服务器storage服务
root@Node1:/soft/FastDFS# /usr/bin/fdfs_storaged /etc/fdfs/storage.conf start
#### 设置开机自启，修改/etc/rc.local文件在文件末尾加入
service fdfs_storaged start

#### 查看
root@Node1:/soft/FastDFS# /usr/bin/fdfs_monitor /etc/fdfs/storage.conf

	[2020-02-13 15:54:01] DEBUG - base_path=/soft/FastDFS/data/storage, connect_timeout=5, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=1, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0
	server_count=1, server_index=0
	tracker server is 192.168.85.200:22122
	group count: 1

	Group 1:
	group name = group1
	disk total space = 40,056 MB
	disk free space = 12,578 MB
	trunk free space = 0 MB
	storage server count = 2
	active server count = 2
	storage server port = 23000
	storage HTTP port = 8502
	store path count = 2
	subdir count per path = 5
	current write server index = 0
	current trunk file id = 0

		Storage 1:
			id = 192.168.85.200
			ip_addr = 192.168.85.200  ACTIVE
			http domain = 
			version = 6.06
			join time = 2020-02-13 13:37:52
			...
		Storage 2:
			id = 192.168.85.201
			ip_addr = 192.168.85.201  ACTIVE
			http domain = 
			version = 6.06
			join time = 2020-02-13 13:37:52
			...

```



### 横向扩展:新增分组Storage Server

```xml
#### 复制/soft/FastDFS和/etc/fdfs目录到节点上
root@Master:/soft# scp -r /soft/FastDFS/ root@192.168.85.201:/soft/FastDFS/
root@Master:/soft# scp -r /etc/fdfs/ root@192.168.85.201:/etc/fdfs

#### ★删除没用的目录
root@Node2:~# cd /soft/FastDFS/data/storage
root@Node2:/soft/FastDFS/data/storage# rm -r path0
root@Node2:/soft/FastDFS/data/storage# rm -r path1
root@Node2:/soft/FastDFS/data/storage# rm -r logs/
root@Node2:/soft/FastDFS/data/storage# rm -r data/
#### ★创建存储文件的目录
root@Node2:/soft/FastDFS/data/storage# mkdir path0


#### 编译安装
root@Node2:/soft/FastDFS/data/storage# cd /soft/FastDFS/libfastcommon-1.0.43
#### 安装make命令
root@Node2:/soft/FastDFS/libfastcommon-1.0.43# apt install make
#### 安装GCC的开发环境
root@Node2:/soft/FastDFS/libfastcommon-1.0.43# apt-get install build-essential

#### ★编译安装
root@Node2:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh 
root@Node2:/soft/FastDFS/libfastcommon-1.0.43# ./make.sh install
root@Node2:/soft/FastDFS/libfastcommon-1.0.43# cd ..
root@Node2:/soft/FastDFS# cd fastdfs-6.06/
root@Node2:/soft/FastDFS/fastdfs-6.06# ./make.sh
root@Node2:/soft/FastDFS/fastdfs-6.06# ./make.sh install

#### ★修改/etc/storage.conf
root@Node2:/soft/FastDFS/fastdfs-6.06# vim /etc/fdfs/storage.conf

	disabled=false 
	group_name=group2 ####★修改组名
	port=23000 ####默认端口号，同一组内端口号必须一致
	####storage服务数据存放路径
	base_path=/soft/FastDFS/data/storage 
	store_path_count=1 ####★存储路径个数，需要和 store_path 个数匹配
	####文件存放目录，如果存储路径只有一个，可以默认为base_path；此处可设置多个目录。
	####当storage服务启动后，会自动在store_pathx配置的目录下创建data文件夹用于存储文件
	####★下面配置的目录path0、path1必须先创建好
	store_path0=/soft/FastDFS/data/storage/path0
	tracker_server=192.168.85.200:22122 ####tracker地址
	http.server_port=8502 ####设置http连接端口号

#### ★启动新服务器storage服务
root@Node2:/soft/FastDFS/fastdfs-6.06# /usr/bin/fdfs_storaged /etc/fdfs/storage.conf start

#### 设置开机自启，修改/etc/rc.local文件在文件末尾加入
service fdfs_storaged start

#### 查看
root@Node2:/soft/FastDFS/fastdfs-6.06# /usr/bin/fdfs_monitor /etc/fdfs/storage.conf
	
	[2020-02-13 00:25:17] DEBUG - base_path=/soft/FastDFS/data/storage, connect_timeout=5, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=1, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0
	server_count=1, server_index=0
	tracker server is 192.168.85.200:22122
	group count: 2

	Group 1:
	group name = group1
	disk total space = 40,056 MB
	disk free space = 12,574 MB
	trunk free space = 0 MB
	storage server count = 2
	active server count = 2
	storage server port = 23000
	storage HTTP port = 8502
	store path count = 2
	subdir count per path = 5
	current write server index = 1
	current trunk file id = 0

		Storage 1:
			id = 192.168.85.200
			ip_addr = 192.168.85.200  ACTIVE
			http domain = 
			version = 6.06
			join time = 2020-02-12 21:37:52
			up time = 2020-02-12 21:57:17
			total storage = 100,534 MB
			free storage = 46,810 MB
		Storage 2:
			id = 192.168.85.201
			ip_addr = 192.168.85.201  ACTIVE
			http domain = 
			version = 6.06
			join time = 2020-02-12 21:37:52
	Group 2:
	group name = group2
	disk total space = 20,028 MB
	disk free space = 11,296 MB
	trunk free space = 0 MB
	storage server count = 1
	active server count = 1
	storage server port = 23000
	storage HTTP port = 8502
	store path count = 1
	subdir count per path = 5
	current write server index = 0
	current trunk file id = 0
		Storage 1:
			id = 192.168.85.202
			ip_addr = 192.168.85.202  ACTIVE
			http domain = 
			version = 6.06
			join time = 2020-02-13 00:25:09
			up time = 2020-02-13 00:25:09
			total storage = 20,028 MB
			

#### 上传到指定分组的Storage Service
root@Master:/soft# /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /home/openxu/Desktop/22222.png 192.168.85.202:23000
group2/M00/00/00/wKhVyl5FHKOAZvA0AABImDYKwM8574.png

```


## 6.Tracker配置

```xml
vim /etc/fdfs/tracker.conf

	# 该配置文件是否不可用
	disabled = false
	# ★★★tracker server端口
	port = 22122
	# 连接超时时间，默认30s，如果是局域网，2s就够了
	connect_timeout = 5
	# 发送和接受数据超时时间，默认30s
	network_timeout = 60
	# ★★★存储数据和日志的根目录 the base path to store data and log files
	base_path = /soft/FastDFS/data/tracker
    #此服务器支持的最大并发连接数，应该把这个参数设置得更大些，例如10240。默认值为256
	max_connections = 1024
	# 接受的线程计数，建议默认值为1，从v4.07开始
	accept_threads = 1
	# 用于处理网络io的工作线程计数，默认值为4，从V2.00开始 
	work_threads = 4
	# 最小网络buff大小，默认值8kb
	min_buff_size = 8KB
	# 最大网络buff大小，默认值128KB
	max_buff_size = 128KB

	# ★★★选择上传文件组的方法
	# 0: 循环选择组上传
	# 1: 指定特定的组，指定之后，所有的文件只会上传到该组对应的storage server上
	# 2: 负载平衡，选择要上载文件的最大可用空间组
	store_lookup = 0

	# 指定要上传文件的组，当store_lookup=1时，必须在此指定组名
	store_group = group2
	
	# 指定要上载文件的存储服务器
	# 0:循环（默认）
	# 1:按ip地址排列的第一个服务器
	# 2:按优先级排列的第一个服务器顺序（最小）
	# 注意：如果use_trunk_file设置为true，则必须将store_server设置为1或2
	store_server = 0
	# 指定要上载文件的存储服务器的路径（表示磁盘或装入点）
	# 0:循环
	# 2:负载平衡，选择要上载文件的最大可用空间路径
	store_path = 0

	# 指定要下载文件的存储服务器
	# 0:循环（默认）
	# 1:当前文件上传到的源存储服务器
	download_server = 0

	# 为系统或其他应用程序保留的存储空间
	# 如果一个组内的某些storage server可用存储空间<=该值，无法将任何文件上载到此组。
	# 单位可为以下之一:G or g \ M or m \ K or k \ 无单位默认为b
	# 可用百分数 XX.XX% : 例如 10% 预留存储空间=10%
	reserved_storage_space = 20%

	# 日志级别，不区分大小写:
	### emerg for emergency
	### alert
	### crit for critical
	### error
	### warn for warning
	### notice
	### info
	### debug
	log_level = info

	# 运行此程序的unix组名, 未设置（空）表示由当前用户组运行
	run_by_group=
	# 运行此程序的unix用户名，未设置（空）表示当前用户运行
	run_by_user =

	# allow_hosts can ocur more than once, host can be hostname or ip address,
	# "*" (only one asterisk) means match all ip addresses
	# we can use CIDR ips like 192.168.5.64/26
	# and also use range like these: 10.0.1.[0-254] and host[01-08,20-25].domain.com
	# for example:
	# allow_hosts=10.0.1.[1-15,20]
	# allow_hosts=host[01-08,20-25].domain.com
	# allow_hosts=192.168.5.64/26
	allow_hosts = *

	# sync log buff to disk every interval seconds
	# default value is 10 seconds
	sync_log_buff_interval = 1

	# check storage server alive interval seconds
	check_active_interval = 120

	# thread stack size, should >= 64KB
	# default value is 256KB
	thread_stack_size = 256KB

	# auto adjust when the ip address of the storage server changed
	# default value is true
	storage_ip_changed_auto_adjust = true

	# storage sync file max delay seconds
	# default value is 86400 seconds (one day)
	# since V2.00
	storage_sync_file_max_delay = 86400

	# the max time of storage sync a file
	# default value is 300 seconds
	# since V2.00
	storage_sync_file_max_time = 300

	# if use a trunk file to store several small files
	# default value is false
	# since V3.00
	use_trunk_file = false 

	# the min slot size, should <= 4KB
	# default value is 256 bytes
	# since V3.00
	slot_min_size = 256

	# the max slot size, should > slot_min_size
	# store the upload file to trunk file when it's size <=  this value
	# default value is 16MB
	# since V3.00
	slot_max_size = 1MB

	# the alignment size to allocate the trunk space
	# default value is 0 (never align)
	# since V6.05
	# NOTE: the larger the alignment size, the less likely of disk
	#       fragmentation, but the more space is wasted.
	trunk_alloc_alignment_size = 256

	# if merge contiguous free spaces of trunk file
	# default value is false
	# since V6.05
	trunk_free_space_merge = true

	# if delete / reclaim the unused trunk files
	# default value is false
	# since V6.05
	delete_unused_trunk_files = false

	# the trunk file size, should >= 4MB
	# default value is 64MB
	# since V3.00
	trunk_file_size = 64MB

	# if create trunk file advancely
	# default value is false
	# since V3.06
	trunk_create_file_advance = false

	# the time base to create trunk file
	# the time format: HH:MM
	# default value is 02:00
	# since V3.06
	trunk_create_file_time_base = 02:00

	# the interval of create trunk file, unit: second
	# default value is 38400 (one day)
	# since V3.06
	trunk_create_file_interval = 86400

	# the threshold to create trunk file
	# when the free trunk file size less than the threshold,
	# will create he trunk files
	# default value is 0
	# since V3.06
	trunk_create_file_space_threshold = 20G

	# if check trunk space occupying when loading trunk free spaces
	# the occupied spaces will be ignored
	# default value is false
	# since V3.09
	# NOTICE: set this parameter to true will slow the loading of trunk spaces 
	# when startup. you should set this parameter to true when neccessary.
	trunk_init_check_occupying = false

	# if ignore storage_trunk.dat, reload from trunk binlog
	# default value is false
	# since V3.10
	# set to true once for version upgrade when your version less than V3.10
	trunk_init_reload_from_binlog = false

	# the min interval for compressing the trunk binlog file
	# unit: second, 0 means never compress
	# FastDFS compress the trunk binlog when trunk init and trunk destroy
	# recommand to set this parameter to 86400 (one day)
	# default value is 0
	# since V5.01
	trunk_compress_binlog_min_interval = 86400

	# the interval for compressing the trunk binlog file
	# unit: second, 0 means never compress
	# recommand to set this parameter to 86400 (one day)
	# default value is 0
	# since V6.05
	trunk_compress_binlog_interval = 86400

	# compress the trunk binlog time base, time format: Hour:Minute
	# Hour from 0 to 23, Minute from 0 to 59
	# default value is 03:00
	# since V6.05
	trunk_compress_binlog_time_base = 03:00

	# max backups for the trunk binlog file
	# default value is 0 (never backup)
	# since V6.05
	trunk_binlog_max_backups = 7

	# if use storage server ID instead of IP address
	# if you want to use dual IPs for storage server, you MUST set
	# this parameter to true, and configure the dual IPs in the file
	# configured by following item "storage_ids_filename", such as storage_ids.conf
	# default value is false
	# since V4.00
	use_storage_id = false

	# specify storage ids filename, can use relative or absolute path
	# this parameter is valid only when use_storage_id set to true
	# since V4.00
	storage_ids_filename = storage_ids.conf

	# id type of the storage server in the filename, values are:
	## ip: the ip address of the storage server
	## id: the server id of the storage server
	# this paramter is valid only when use_storage_id set to true
	# default value is ip
	# since V4.03
	id_type_in_filename = id

	# if store slave file use symbol link
	# default value is false
	# since V4.01
	store_slave_file_use_link = false

	# if rotate the error log every day
	# default value is false
	# since V4.02
	rotate_error_log = false

	# rotate error log time base, time format: Hour:Minute
	# Hour from 0 to 23, Minute from 0 to 59
	# default value is 00:00
	# since V4.02
	error_log_rotate_time = 00:00

	# if compress the old error log by gzip
	# default value is false
	# since V6.04
	compress_old_error_log = false

	# compress the error log days before
	# default value is 1
	# since V6.04
	compress_error_log_days_before = 7

	# rotate error log when the log file exceeds this size
	# 0 means never rotates log file by log file size
	# default value is 0
	# since V4.02
	rotate_error_log_size = 0

	# keep days of the log files
	# 0 means do not delete old log files
	# default value is 0
	log_file_keep_days = 0

	# if use connection pool
	# default value is false
	# since V4.05
	use_connection_pool = true

	# connections whose the idle time exceeds this time will be closed
	# unit: second
	# default value is 3600
	# since V4.05
	connection_pool_max_idle_time = 3600

	# HTTP port on this tracker server
	http.server_port = 8501

	# check storage HTTP server alive interval seconds
	# <= 0 for never check
	# default value is 30
	http.check_alive_interval = 30

	# check storage HTTP server alive type, values are:
	#   tcp : connect to the storge server with HTTP port only, 
	#        do not request and get response
	#   http: storage check alive url must return http status 200
	# default value is tcp
	http.check_alive_type = tcp

	# check storage HTTP server alive uri/url
	# NOTE: storage embed HTTP server support uri: /status.html
	http.check_alive_uri = /status.html



```



## 7.master节点部署nginx进行反向代理及负载均衡

[下载](http://labs.frickle.com/files/)nginx缓存模块，ngx_cache_purge-2.3.tar.gz

```xml
#### 上传解压
root@Master:~# mv /home/openxu/Desktop/ngx_cache_purge-2.3.tar.gz /soft/FastDFS/ngx_cache_purge-2.3.tar.gz
root@Master:~# cd /soft/FastDFS/
root@Master:/soft/FastDFS# tar -zxvf /soft/FastDFS/ngx_cache_purge-2.3.tar.gz 

#### 配置、编译、安装
#### 设置prefix——编译后软件存放地址，增加模块 --add-module 增加fastdfs-nginx-module
root@Master:/soft/FastDFS/nginx-1.17.8# cd /soft/FastDFS/nginx-1.17.8/

root@Master:/soft/FastDFS/nginx-1.17.8# ./configure --prefix=/usr/local/nginx-fdfs --add-module=/soft/FastDFS/ngx_cache_purge-2.3

root@Master:/soft/FastDFS/nginx-1.17.8# ./configure --prefix=/usr/local/fdfs-nginx-cache --add-module=/soft/FastDFS/ngx_cache_purge-2.3
root@Master:/soft/FastDFS/nginx-1.17.8# make 
root@Master:/soft/FastDFS/nginx-1.17.8# make install

#### 配置nginx进行反向代理及缓存设置
root@Master:/soft/FastDFS/nginx-1.17.8# cd /usr/local/fdfs-nginx-cache/
root@Master:/usr/local/fdfs-nginx-cache# ll
total 24
drwxr-xr-x  6 root root 4096 2月  17 14:19 ./
drwxr-xr-x 12 root root 4096 2月  17 14:19 ../
drwxr-xr-x  2 root root 4096 2月  17 14:19 conf/
drwxr-xr-x  2 root root 4096 2月  17 14:19 html/
drwxr-xr-x  2 root root 4096 2月  17 14:19 logs/
drwxr-xr-x  2 root root 4096 2月  17 14:19 sbin/

#### 创建缓存文件夹
root@Master:/usr/local/fdfs-nginx-cache# mkdir tmp
root@Master:/usr/local/fdfs-nginx-cache# mkdir cache

#### 修改配置文件

		#user  nobody;
		worker_processes  1;

		error_log  /usr/local/fdfs-nginx-cache/logs/error.log; 
		error_log  /usr/local/fdfs-nginx-cache/logs/error.log  notice;
		error_log  /usr/local/fdfs-nginx-cache/logs/error.log  info;

		#pid        logs/nginx.pid;

		events {
			worker_connections  1024;
			use epoll;
		}


		http {
			include       mime.types;
			default_type  application/octet-stream;

			#log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
			#                  '$status $body_bytes_sent "$http_referer" '
			#                  '"$http_user_agent" "$http_x_forwarded_for"';

			#access_log  logs/access.log  main;

			sendfile        on;
			#tcp_nopush     on;

			#keepalive_timeout  0;
			keepalive_timeout  65;

			#gzip  on;

			#★★★设置缓存
			server_names_hash_bucket_size  128;
			client_header_buffer_size  32k;
			large_client_header_buffers  4  32k;
			client_max_body_size  300m;
		   
			proxy_redirect  off;
			proxy_set_header  Host  $http_host;
			proxy_set_header  X-Real-IP  $remote_addr;
			proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
			proxy_connect_timeout  900;
			proxy_send_timeout  900;
			proxy_read_timeout  900;
			proxy_buffer_size  16k;
			proxy_buffers  4 64k;
			proxy_busy_buffers_size  128k;
			proxy_temp_file_write_size  128k;
			#设置缓存存储路径、存储方式、内存大小、磁盘空间大小、缓存时间等
			proxy_cache_path  /usr/local/fdfs-nginx-cache/cache  levels=1:2
			keys_zone=http-cache:200m
			max_size=1g #最大缓存内存——重要！
			inactive=7d;
			proxy_temp_path /usr/local/fdfs-nginx-cache/tmp;
		   
			#★★★设置group1代理（反向代理）
			upstream fdfs_group1 {
				server 192.168.85.200:8502  weight=1  max_fails=2  fail_timeout=30s;
				server 192.168.85.201:8502  weight=1  max_fails=2  fail_timeout=30s;
			}
			#★★★设置group1代理（反向代理）
			upstream fdfs_group2 {
				server 192.168.85.202:80  weight=1  max_fails=2  fail_timeout=30s;
			}
			server {
				listen       8888;
				server_name  localhost;

				#charset koi8-r;

				#access_log  logs/host.access.log  main;

				location / {
					root   html;
					index  index.html index.htm;
				}

		#       location ~/group[0-9]/M00/ {
		#	   location ~/group[0-9] {
		#            root /soft/FastDFS/data/storage/path0/M00;
		#			ngx_fastdfs_module;
		#		}
				#★★★group1的负载均衡配置
				location ~/group1 {
					proxy_next_upstream  http_502  http_504  error timeout invalid_header;
					# 对应group1的服务设置
					proxy_pass  http://fdfs_group1;
					#### 放开下面几个配置的注释即可开启缓存
				#    proxy_cache  http-cache;
			   #     proxy_cache_valid  200  304  12h;
				#    proxy_cache_key  $uri$is_args$args;
				 #   expires 30d;
		         #no cache   ？？？？
				#	add_header Cache-Control no-store;
				}
				# ★★★
				location ~/group2 {
					proxy_next_upstream  http_502  http_504  error timeout invalid_header;
				  #  proxy_cache  http-cache;
				   # proxy_cache_valid  200  304  12h;
					#proxy_cache_key  $uri$is_args$args;
					proxy_pass  http://fdfs_group2;
				   # expires 30d;
				}
				#purge配置，清除缓存的访问权限
			  #  location ~/purge(/.*) {
			   #     allow 127.0.0.1;
				#    allow 192.168.85.200/24;
				 #   deny all;
				  #  proxy_cache_purge  http-cache  $1$is_args$args;
			   # }
						
			}

		}



#### 启动nginx
root@Master:/usr/local/fdfs-nginx-cache# /usr/local/fdfs-nginx-cache/sbin/nginx 

#### 访问总代理，查看nginx访问日志，如果访问的是group1，可以看到master、node1节点轮询访问

http://192.168.85.200:8888/group1/M00/00/00/wKhVyF5LmhiAYEE5AAAAHIfvZ2c731.txt
http://192.168.85.200:8888/group2/M00/00/00/wKhVyl5LmhmAfC3yAAAAHIfvZ2c477.txt

tail -f /usr/local/nginx-fastdfs/logs/access/log


```






