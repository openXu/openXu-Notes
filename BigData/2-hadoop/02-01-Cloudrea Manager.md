[下拉最后点击资源下载](https://www.cloudera.com/resources.html)

选择6.3.0版本

[安装向导](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/install_cm_cdh.html)

# 安装前准备

环境：VMware上有3个ubuntu虚拟机，分别为FPC-Master、FPC-Node1、FPC-Node2。
由于Cloudera-Manages分Cloudera-Manages-Server和Cloudera-Manages-Agent端，
虚拟机对应关系：
FPC-Master   :  Cloudera-Manages-Server
FPC-Node1   :  Cloudera-Manages-Agent
FPC-Node2   :  Cloudera-Manages-Agent


## 修改主机名（每台虚拟机都需要）

```xml
### 临时修改主机名
root@Node1:~$ hostname
ubuntu
root@Node1:~$ sudo hostname Master
root@Node1:~$ hostname
Master

### 使用vim永久修改主机名
root@Node1:~$ sudo vi /etc/hostname
root@Node1:~$ cat /etc/hostname
Master

### 修改域名
root@Node1:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu
root@Node1:~$ sudo vi /etc/hosts
root@Node1:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1   Master	
```
## 设置Host

所有虚拟机修改完主机名后，配置IP、主机名映射：
```xml
### 查看网关地址  route -n

### 配置静态IP(修改/etc/network/interfaces文件，添加如下内容)
root@Master:~# vi /etc/network/interfaces

auto lo
iface lo inet loopback
### 网卡名
auto ens33 #网卡名
### #设置为静态
iface ens33 inet static
### IP地址
address 192.168.85.100
### 子网掩码
netmask 255.255.255.0
### 网关
gateway 192.168.85.2
### DNS服务器
dns-nameservers 192.168.1.1
dns-nameservers 8.8.8.8

### 使用Netplan配置静态ip
root@Master:/etc/netplan# vi /etc/netplan/01-network-manager-all.yaml 
### 修改文件内容如下，三台虚拟机分别这是addresses为192.168.85.200、192.168.85.201、192.168.85.202
network:
  ethernets:
      ens33:
          addresses: [192.168.85.200/24]
          gateway4: 192.168.85.2
          dhcp4: no
          nameservers:
              addresses: [8.8.8.8]
              search: []
  version: 2
  renderer: NetworkManager
### managed=false   改为true
root@Master:~# vi /etc/NetworkManager/NetworkManager.conf 
managed=false   改为true
### 重启网络
root@Master:~# netplan apply
root@Master:~# service network-manager restart
### ping
root@Master:/# ping www.baidu.com




```


```xml
### 查看每台虚拟机ip地址（分别在各自Terminal窗口执行）
root@Master:~# ifconfig
        inet 192.168.85.200  netmask 255.255.255.0  broadcast 192.168.85.255
root@Node1:~# ifconfig
        inet 192.168.85.201  netmask 255.255.255.0  broadcast 192.168.85.255
root@Node2:~# ifconfig
        inet 192.168.85.202  netmask 255.255.255.0  broadcast 192.168.85.255


### 每个虚拟机都修改host，往/etc/hosts中追加后面三行
root@Master:~# vi /etc/hosts
127.0.0.1       localhost
127.0.1.1       Master

192.168.85.200 Master
192.168.85.201 Node1
192.168.85.202 Node2

### 验证
root@Master:~# ping Node1
```

## 修改系统时间（每台虚拟机都需要）

每台虚拟机系统时间不要相差太大

```xml
openxu@Master:/$ date -R
Tue, 03 Dec 2019 23:10:35 -0800
### ★修改时区
openxu@Master:/$ tzselect
Please identify a location so that time zone rules can be set correctly.
Please select a continent, ocean, "coord", or "TZ".
 1) Africa
 2) Americas
 3) Antarctica
 4) Asia
 5) Atlantic Ocean
 6) Australia
 7) Europe
 8) Indian Ocean
 9) Pacific Ocean
10) coord - I want to use geographical coordinates.
11) TZ - I want to specify the time zone using the Posix TZ format.
### 选择亚洲
#? 4
Please select a country whose clocks agree with yours.
 1) Afghanistan		  18) Israel		    35) Palestine
 2) Armenia		  19) Japan		    36) Philippines
 3) Azerbaijan		  20) Jordan		    37) Qatar
 4) Bahrain		  21) Kazakhstan	    38) Russia
 5) Bangladesh		  22) Korea (North)	    39) Saudi Arabia
 6) Bhutan		  23) Korea (South)	    40) Singapore
 7) Brunei		  24) Kuwait		    41) Sri Lanka
 8) Cambodia		  25) Kyrgyzstan	    42) Syria
 9) China		  26) Laos		    43) Taiwan
10) Cyprus		  27) Lebanon		    44) Tajikistan
11) East Timor		  28) Macau		    45) Thailand
12) Georgia		  29) Malaysia		    46) Turkmenistan
13) Hong Kong		  30) Mongolia		    47) United Arab Emirates
14) India		  31) Myanmar (Burma)	    48) Uzbekistan
15) Indonesia		  32) Nepal		    49) Vietnam
16) Iran		  33) Oman		    50) Yemen
17) Iraq		  34) Pakistan
### 选中中国
#? 9
Please select one of the following time zone regions.
1) Beijing Time
2) Xinjiang Time
### 选择北京时间
#? 1

The following information has been given:

	China
	Beijing Time

Therefore TZ='Asia/Shanghai' will be used.
Selected time is now:	Wed Dec  4 15:25:00 CST 2019.
Universal Time is now:	Wed Dec  4 07:25:00 UTC 2019.
Is the above information OK?
1) Yes
2) No
### 确定
#? 1

You can make this change permanent for yourself by appending the line
	TZ='Asia/Shanghai'; export TZ
to the file '.profile' in your home directory; then log out and log in again.

Here is that TZ value again, this time on standard output so that you
can use the /usr/bin/tzselect command in shell scripts:
Asia/Shanghai
### ★复制时区文件替换系统时区文件
openxu@Master:/$ sudo cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
openxu@Master:/$ date -R
Wed, 04 Dec 2019 15:27:11 +0800

```

## 关闭防火墙（每台虚拟机都需要）

注意：Centos和ubuntu命令不同，本文章是基于ubuntu版本的。

```xml
### 查看防火墙状态
root@Node1:~$ sudo ufw status
Status: inactive
### 防火墙版本
root@Node1:~$ sudo ufw version
ufw 0.36
Copyright 2008-2015 Canonical Ltd.
### 启用防火墙
root@Node1:~$ sudo ufw enable
Firewall is active and enabled on system startup
### 禁用防火墙
root@Node1:~$ sudo ufw disable
Firewall stopped and disabled on system startup
### 重启
root@Node1:~$ reboot
```



## 启用NTP服务（Network Time Protocol 用来使计算机时间同步化的一种协议）

```xml

### 安装ntp包
root@Master:~# apt-get install ntp
### 编辑/etc/ntp.conf文件，以添加NTP服务器
root@Master:~# vi /etc/ntp.conf

server 0.pool.ntp.org
server 1.pool.ntp.org
server 2.pool.ntp.org

### 启动ntpd服务
root@Master:~# sudo service ntpd start
### 配置ntpd在启动时运行的服务

### 将系统始终同步到NTP服务器

### 将硬件始终与系统时钟同步

```


## SSH免密登陆

SSH（Secure Shell 安全外壳协议）是一种网络安全协议，专为远程登陆会话和其他网络服务提供安全性的
协议。通过使用SSH，可以把传输的数据进行加密，有效防止远程管理过程中的信息泄露问题。

从客户端来看，有两种登陆验证方式;

- **基于密码**
   1. 客户端将密码发送给服务端，服务端验证是否正确，这种方式存在密码泄露问题，所以出现了SSH协议（基于非对称加密）。
   2. 客户端向服务端发起ssh请求，服务端收到请求后发送公钥给客户端，客户端输入用户密码通过公钥加密后发送给服务端

- **基于密钥**
   1. 客户端生成一对密钥；
   2. 将公钥拷贝给服务端，重命名为authorized_keys；
   3. 客户端发送请求，服务端收到后使用对应的公钥加密一个字符串后发送给客户端；
   4. 客户端收到密文后使用私钥解密；
   5. 然后将解密后的字符串发给服务端；
   6. 服务端收到字符串对比正确，验证通过。


三台虚拟机FPC-Master（主节点）、FPC-Node1（从节点）、FPC-Node2（从节点），一般需要配置主节点到从节点的免密登陆，
以及主节点自己的免密登陆。也就是说在主节点上能免密登陆从节点node1和node2以及它自己，因为一般我们都是操作主节点。

> FPC-Master --> FPC-Master
> FPC-Master --> FPC-Node1
> FPC-Master --> FPC-Node2

接下来演示`FPC-Master --> FPC-Node1`的免密登陆，其他两个按步骤走就行。
注意以下命令应该在对应的虚拟机上执行，看注解标识

**ssh环境配置（每台虚拟机都需要）**
```xml
### 查看ssh服务是否运行sshd
root@Master:~# ps -e|grep ssh
  1410 ?        00:00:00 ssh-agent
  2314 ?        00:00:00 ssh-agent
  
### 如果sshd没有运行，需要安装openssh-server
root@Master:~# apt-get install openssh-server

### 启动
root@Master:~# service ssh start
### 查看ssh服务是否运行sshd
root@Master:~# ps -e|grep ssh
  1410 ?        00:00:00 ssh-agent
  2314 ?        00:00:00 ssh-agent
  5465 ?        00:00:00 sshd

### 如果不修改配置，拷贝公钥输入root密码会出现（Permission denied, please try again.）错误
### 修改sshservice配置（注意每台虚拟机上都需要做如下修改）
root@Master:~# vi /etc/ssh/sshd_config
### 将下面两个值改为yes后保存
PermitRootLogin yes
PubkeyAuthentication yes

### 修改配置文件后重启ssh服务
root@Master:~# /etc/init.d/ssh restart
```

**生成ssh密钥对，将公钥发送到服务端**

```xml
### ★★★1. 生成ssh密钥对，-t参数为加密算法（rsa或者dsa）
openxu@Master:/$ ssh-keygen -t rsa   (按4个回车)
Generating public/private rsa key pair.
Enter file in which to save the key (/home/openxu/.ssh/id_rsa): 
Created directory '/home/openxu/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/openxu/.ssh/id_rsa.
Your public key has been saved in /home/openxu/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:Oq/BqVwVCM2sh7z1LE5wsj1fJeVMcYJxH55932/wbNE openxu@Master
The key's randomart image is:
+---[RSA 2048]----+
|    .+    .o+.o  |
|     .+.  ..o= + |
|   . o. .  =  + o|
|    * +  .. +   =|
|     X oS  o  ..E|
|    o.=+o .    +o|
|     oB+ .      *|
|   . o.+.      o |
|    o ...        |
+----[SHA256]-----+

### 查看生成的密钥对
openxu@Master:~$ cd .ssh
root@Master:~/.ssh# ll
total 20
drwx------ 2 root root 4096 Dec  4 17:06 ./
drwx------ 6 root root 4096 Dec  4 16:26 ../
-rw------- 1 root root 1679 Dec  4 17:07 id_rsa
-rw-r--r-- 1 root root  397 Dec  4 17:07 id_rsa.pub
-rw-r--r-- 1 root root  666 Dec  4 17:06 known_hosts

### ★★★2. 将Master生成的公钥发送给Node1
root@Master:~/.ssh# ssh-copy-id FPC-Node1
root@node1's password: 
Number of key(s) added: 1

###↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓服务端Node1上查看发送过来的公钥↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓#########
root@Node1:~# cd .ssh
### 发现Master的公钥已经发送过来了，并且命名为authorized_keys
root@Node1:~/.ssh# ls
authorized_keys  known_hosts
###↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑服务端Node1上查看发送过来的公钥↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑#########

### ★★★3. Master作为客户端发送ssh请求到Node1，不用输入密码即可验证成功
root@Master:~/.ssh# ssh FPC-Node1
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 5.0.0-23-generic x86_64)
### 退出登陆
root@Node1:~/.ssh# exit
logout
Connection to fpc-node1 closed.
root@Master:~/.ssh#
```



# 安装向导

## 为Cloudera Manager配置存储库

安装mysql-server（只需要在主机Master上装）

```xml
### 安装
root@Master:/soft# apt-get install mysql-server
...
Created symlink /etc/systemd/system/multi-user.target.wants/mysql.service → /lib/systemd/system/mysql.service.
...
### 启动mysql服务
root@Master:/soft# service mysql start 或者
root@Master:/soft# /etc/init.d/mysql start
[ ok ] Starting mysql (via systemctl): mysql.service.

### 安装mysql JDBC驱动
root@Master:/soft# apt-get install libmysql-java

### 
root@Master:~# mysql -root -p
Enter password: 密码为空  回车就行
mysql> exit;

### 为root账户创建密码（密码root）
root@Master:~# mysqladmin -u root password
New password: 
Confirm new password: 
Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.
root@Master:~# mysql -root -p
Enter password: 
mysql> exit
Bye
root@Master:~# 

```

## 安装java环境（每台虚拟机都需要）

1. [jdk8下载](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

> 注意根据自己系统位数下载对应的压缩包

2. 将下载目录设置为vmware共享磁盘

> 该操作可以让vmware上的虚拟机访问windows宿主的某个文件夹，将其挂载到/mnt目录下。
> 虚拟机VM-->设置-->选项-->共享文件夹-->勾选总是启用-->添加共享文件夹-->确定

3. 复制jdk压缩文件到/usr/java中

```xml
### 根目录下创建/soft/java文件夹
root@Node1:~# cd ..
root@Node1:/# cd ..
root@Node1:/# mkdir soft
root@Node1:/# cd soft
root@Node1:/soft# mkdir java
root@Node1:/# cd ..
root@Node1:/# ls
bin    home   mnt ...  soft  
### 进入下载目录
root@Node1:/mnt$ ls
root@Node1:/mnt$ cd hgfs/
root@Node1:/mnt/hgfs$ ls
root@Node1:/mnt/hgfs$ cd Downloads/
root@Node1:/mnt/hgfs/Downloads$ ls
jdk-8u231-linux-x64.tar.gz
jquery-easyui-1.7.0.zip
license_freeware.txt
node-v4.4.3-x64.msi
### 复制文件
root@Node1:/mnt/hgfs/Downloads$ sudo cp jdk-8u231-linux-x64.tar.gz /soft/java/jdk.tar.gz

#进入/soft/java目录后解压
root@Node1:/soft/java$ sudo tar zxvf jdk.tar.gz
root@Node1:/soft/java$ ls -l
drwxrwxr-x 7  500  500      4096 Oct  5 03:59 jdk1.8.0_231
-rwxr-xr-x 1 root root 194773928 Dec  3 00:29 jdk.tar.gz

### 使用vim配置环境，添加如下内容
root@Node1:/soft/java$ sudo vi /etc/profile
###########
export JAVA_HOME=/soft/java/jdk1.8.0_231
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib

### 重新加载配置文件/etc/profile
root@Node1:/soft/java$ source /etc/profile
### 测试
root@Node1:/soft/java$ java -version
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)

```


# 安装Cloudera Manager和CDH

[安装Cloudera Manager和CDH](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/install_cm_cdh.html)

## 步骤1：配置存储库
## 步骤2：安装JDK

## 步骤3：安装Cloudera Manager服务器

apt-cache和apt-get是apt包的管理工具，他们根据/etc/apt/sources.list里的软件源地址列表搜索目标软件、并通过维护本地软件包列表来安装和卸载软件。

1. 添加cloudera源

[Cloudera产品文档](https://docs.cloudera.com/documentation/)中点击**版本，包装和下载信息**，继续点击**Cloudera Manager 6版本和下载信息**，
选择对应的版本，可以看到Repositories表：

|Type |	Location (baseurl)	|Repo File|
|:--|:-:|--:|
|兼容RHEL 7	|https://archive.cloudera.com/cm6/6.3.0/redhat7/yum/	|cloudera-manager.repo|
|兼容RHEL6	|https://archive.cloudera.com/cm6/6.3.0/redhat6/yum/	|cloudera-manager.repo|
|SLES 12	|https://archive.cloudera.com/cm6/6.3.0/sles12/yum/	|cloudera-manager.repo|
|Ubuntu Xenial（18.04）	|https://archive.cloudera.com/cm6/6.3.0/ubuntu1804/apt/	|cloudera-manager.list|
|Ubuntu Xenial（16.04）	|https://archive.cloudera.com/cm6/6.3.0/ubuntu1604/apt/	|cloudera-manager.list|

选择Ubuntu Xenial（18.04）对应的`cloudera-manager.list`点击下载后，复制到`/etc/apt/sources.list.d/`目录下：

[手动下载cloudera-manager安装包](https://archive.cloudera.com/cm6/6.3.0/ubuntu1804/apt/pool/contrib/e/enterprise/)


```xml
### 更新源
root@Master:/etc/apt# apt-get update
### 出现错误
Err:3 https://archive.cloudera.com/cm6/6.3.0/ubuntu1804/apt bionic-cm6.3.0 InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 73985D43B0B19C9F
### 解决错误
root@Master:/etc/apt# apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 73985D43B0B19C9F

### 安装cloudera-manager
apt-get install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
```

2. 安装

```xml
### 在主节点上安装daemons、agent、server
apt-get install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server

### 等待下载完成，并自动安装

### 主节点安装成功之后，将daemons、agent包发给从节点
root@Master:~# scp /var/cache/apt/archives/cloudera-manager-agent_6.3.0~1281944.ubuntu1804_amd64.deb openxu@Node1:/home/openxu/Desktop/cloudera-manager-agent_6.3.0~1281944.ubuntu1804_amd64.deb
root@Master:~# scp /var/cache/apt/archives/cloudera-manager-daemons_6.3.0~1281944.ubuntu1804_all.deb  openxu@Node1:/home/openxu/Desktop/cloudera-manager-daemons_6.3.0~1281944.ubuntu1804_all.deb

### 从节点安装daemons、agent
root@Node1:~# dpkg -i /home/openxu/Desktop/cloudera-manager-daemons_6.3.0~1281944.ubuntu1804_all.deb 
root@Node1:~# dpkg -i /home/openxu/Desktop/cloudera-manager-agent_6.3.0~1281944.ubuntu1804_amd64.deb 
### 如果出现错误：dependency problems
dpkg: dependency problems prevent configuration of cloudera-manager-agent:
dpkg: error processing package cloudera-manager-agent (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 cloudera-manager-agent
### 解决错误： -f 参数表示 'fix-broken'，这条命令会试图修复依赖问题。
root@Node1:~# apt-get install -f
### dpkg重新安装即可

```

## 步骤4：安装数据库

Cloudera Manager使用各种数据库和数据存储来存储有关Cloudera Manager配置的信息以及诸如系统运行状况或任务进度之类的信息。
尽管您可以在单个环境中部署不同类型的数据库，但是这样做会带来意想不到的复杂性。Cloudera建议为所有Cloudera数据库选择一个
受支持的数据库提供程序。

Cloudera建议将数据库安装在服务之外的其他主机上。将数据库与服务分离可以帮助将潜在的影响与故障或资源争用隔离开来。它还可
以简化具有专门的数据库管理员的组织中的管理。您可以将自己的PostgreSQL，MariaDB，MySQL或Oracle数据库用于Cloudera Manager
 Server和其他使用数据库的服务。

这里选择MySql

```xml
### 安装MySQL服务器(如果已经安装，可跳过)
apt-get install mysql-server
### 配置和启动MySQL服务器
#### 如果要对现有数据库进行更改，请确保在继续之前停止使用该数据库的所有服务。
service mysql stop
#### 备份
root@Master:~# cp /var/lib/mysql/ib_logfile0 /home/openxu/Desktop/backup/ib_logfile0
root@Master:~# cp /var/lib/mysql/ib_logfile1 /home/openxu/Desktop/backup/ib_logfile1
#### 修改my.cnf 使其符合以下要求
READ-COMMITTED   # 为防止死锁，请将隔离级别设置为 已读
innodb_flush_method = O_DIRECT   # 大多数发行版中，MySQL安装中的默认设置使用保守的缓冲区大小和内存使用量。Cloudera Management Service角色需要较高的写吞吐量，因为它们可能在数据库中插入许多记录
max_connections    # 具体取决于集群的大小
max_allowed_packet = 16M   # 如果群集具有超过1000个主机，请设置 max_allowed_pa​​cket=16M，否则群集可能无法启动：com.mysql.jdbc.PacketTooBigException
具体配置请看下方代码块
#### 确保MySQL服务器在启动时启动（ chkconfig在最新的Ubuntu版本中可能不可用。您可能需要使用Upstart将MySQL配置为在系统启动时自动启动）
chkconfig mysql on （不成功）
#### 启动mysql 
root@Master:~# service mysql start
root@Master:~# ps -e|grep mysql
  1054 ?        00:02:00 mysqld
#### 设置MySQL root密码和其他与安全性相关的设置

### 安装MySQL JDBC驱动程序
root@Master:~# apt-get install libmysql-java

### 为Cloudera软件创建数据库
root@Master:~# mysql -u root -p
Enter password: 
#### 为集群中部署的每个服务创建数据库
#### 设置密码校验机制：长度校验，长度为4，只需要>4位即可
mysql> set global validate_password_policy=0;
mysql> set global validate_password_length=4;
#### 使用以下命令为集群中部署的每个服务创建数据库
CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
授权用户root使用密码passwd从任意主机连接到mysql服务器
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY '123456';
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY '123456';
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY '123456';
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY '123456';
GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY '123456';
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY '123456';
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY '123456';
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY '123456';
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY '123456';
#### 上面不起作用？
mysql> GRANT ALL PRIVILEGES ON *.* TO 'scm'@'localhost' IDENTIFIED BY '123456' WITH GRANT OPTION;

#### 查看数据库
mysql> show databases;

```

```xml
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
### to do so, uncomment this line:
symbolic-links = 0

key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
###expire_logs_days = 10
###max_binlog_size = 100M

###log_bin should be on a disk with enough free space.
###Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
###system and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

###In later versions of MySQL, if you enable the binary log and do not set
###a server_id, MySQL will not start. The server_id must be unique within
###the replicating group.
server_id=1

binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

### InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

sql_mode=STRICT_ALL_TABLES
```

## 步骤5：设置Cloudera Manager数据库

```xml
### 语法
/opt/cloudera/cm/schema/scm_prepare_database.sh [options] <databaseType> <databaseName> <databaseUser> <password>
### 初始化表
root@Master:~# /opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm
Enter SCM password: 123456
### 配置成功
All done, your SCM database is configured correctly!   

```


## 步骤6：安装CDH和其他软件

```xml
### ★启动Cloudera Manager服务器，等待几分钟，以启动Cloudera Manager Server
root@Master:~# systemctl start cloudera-scm-server

### 要观察启动过程，请在Cloudera Manager Server主机上运行以下命令
root@Master:~# tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log
#### 当您看到此日志条目时，Cloudera Manager管理控制台已准备就绪
2019-12-15 22:06:16,423 INFO WebServerImpl:org.eclipse.jetty.server.AbstractConnector: Started ServerConnector@55bec86d{HTTP/1.1,[http/1.1]}{0.0.0.0:7180}
INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.

### ★启动agent
root@Master:~# systemctl start cloudera-scm-agent
### 查看日志错误信息
root@Node1:~# tail -f /var/log/cloudera-scm-agent/cloudera-scm-agent.log
Traceback (most recent call last):
  File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/cmf/agent.py", line 1390, in _send_heartbeat
    self.cfg.master_port)
  File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/avro/ipc.py", line 469, in __init__
    self.conn.connect()
  File "/usr/lib/python2.7/httplib.py", line 831, in connect
    self.timeout, self.source_address)
  File "/usr/lib/python2.7/socket.py", line 575, in create_connection
    raise err
error: [Errno 111] Connection refused
### 查看配置文件
root@Node1:~# cat -e|grep "server_" /etc/cloudera-scm-agent/config.ini
server_host=localhost
server_port=7182

修改为：
server_host=192.168.85.200（server主机ip 或 主机名）
server_port=7182

### ★使用浏览器打开链接http://<server_host>:7180
http://192.168.85.200:7180

```

使用admin\admin登录后，安装向导将启动。以下各节指导您完成安装向导的每个步骤：

**安装步骤**


[安装Cloudera Manager和CDH](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/install_cm_cdh.html)


1. Welcome（欢迎）
   - 点击“Continue”

2. 接受许可
   - 勾选后，点击“Continue”

3. 选择版
   - Cloudera Express，不需要许可证，但提供了一组有限的功能
   - Cloudera Enterprise Cloudera Enterprise试用版，不需要许可证，但在60天后到期，并且无法续订。
   - 如果选择Cloudera Enterprise，请安装许可证
   
> 选择要安装的Cloudera Manager的版本，这里选择Cloudera Express，点击“Continue”

4. 欢迎（添加群集-安装）
   - 指定集群名称

5. 集群基础
6. 设置自动TLS
7. 指定主机
   - 指定主机，在“Currently Managed Hosts”栏中勾选主机（server、agent都必须启动起来），以形成集群。
	
|Hostname |	IP Address 	Rack |	CDH Version |	Cores |
|:--|:--:|:--:|--:|
|Master |127.0.1.1 |	/default |	None 	|2|
|Node1 	|192.168.85.201 	|/default 	|None 	2|

8. 选择存储库

> 可以指定Cloudera Manager Agent和CDH及其他软件的存储库。
   - Install Method:选择 “Use Parcels (Recommended)”，在联网的情况下会自动显示出可匹配的CDH版本
   - 勾选“CDH-6.3.2-1.cdh6.3.2.p0.1605554”
   - 点击“Continue”

PS：如果您的主机没有联网，请参照
[配置本地包裹存储库](https://docs.cloudera.com/documentation/enterprise/latest/topics/cm_ig_create_local_package_repo.html)
[CDH6.3.2下载](https://archive.cloudera.com/cdh6/6.3.2/parcels/)

```xml
#### ★将下载的.parcel、.sha1、manifest.json拷贝到parcel-repo中
root@Master:/opt/cloudera/parcel-repo# cp /home/openxu/Desktop/backup/CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel /opt/cloudera/parcel-repo/CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel
root@Master:/opt/cloudera/parcel-repo# cp /home/openxu/Desktop/backup/CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha1 /opt/cloudera/parcel-repo/CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha1
root@Master:/opt/cloudera/parcel-repo# cp /home/openxu/Desktop/backup/manifest.json /opt/cloudera/parcel-repo/manifest.json

root@Master:/opt/cloudera/parcel-repo# ll
total 1950828
drwxr-xr-x 2 cloudera-scm cloudera-scm       4096 Dec 16 14:33 ./
drwxr-xr-x 8 cloudera-scm cloudera-scm       4096 Dec 11 13:25 ../
-rw------- 1 root         root         1997590979 Dec 16 14:22 CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel
-rwxr--r-- 1 root         root                 40 Dec 16 14:33 CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha1*
-rwxr--r-- 1 root         root              33887 Dec 16 14:33 manifest.json*
#### 将.sha1重命名为.sha，否则，系统会重新下载.sha1文件
root@Master:/opt/cloudera/parcel-repo# mv CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha1 CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha
root@Master:/opt/cloudera/parcel-repo# cat CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha 
a92bfc104f7cb42cea8a4e8342b9ced05dcdf4ce
#### ★为添加的 parcel 创建一个 SHA1 哈希并保存到文件
root@Master:/opt/cloudera/parcel-repo# sha1sum CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel | awk '{print $1 }' > CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha
#### 将包和哈希文件的所有权更改为cloudera-scm
sudo chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/*

#### ★重启cloudera-scm-server（重要）
root@Master:/opt/cloudera/parcel-repo# cloudera-scm-server restart

#### ★使用浏览器打开链接http://<server_host>:7180，
http://192.168.85.200:7180

```

然后重新执行上面的安装步骤，由于这里手动下载了安装包，所以Downloaded过程直接过了。接下来DIstributed(分发)、Unpacked(解压)、Activated(激活)等步骤，大概几分钟。

9. 检查群集

您已经创建了一个新的空集群。Cloudera建议您运行以下检查。为了精确测量，Cloudera建议按顺序执行。

两步检测完毕没有错误，勾选“我了解风险，让我继续集群设置”， Continue


**添加群集-配置Add Cluster - Configuration**

1. Select Services(选择服务)

选择“Custom Services”，勾选HDFS、Spark、YARN (MR2 Included)、ZooKeeper，继续

2. Assign Roles(分配角色)

选择namenode等等，继续

3. Review Changes(查看变化)

继续


Command Details
Summary



9. 接受JDK许可证

之前配置了jdk跳过

10. 输入登录凭证
11. 安装代理
12. 安装包裹
13. 检查集群






























