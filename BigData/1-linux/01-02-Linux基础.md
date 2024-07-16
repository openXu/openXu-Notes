
FPC-Node1 ：openXu  123456

# Linux目录结构

- bin  存放二进制可执行文件(ls,cat,mkdir等)  
- boot  存放用于系统引导时使用的各种文件  
- dev 用于存放设备文件  
- etc  存放系统配置文件  
- home 存放所有用户文件的根目录  
- lib  存放跟文件系统中的程序运行所需要的共享库及内核模块  
- mnt  系统管理员安装临时文件系统的安装点  
- opt  额外安装的可选应用程序包所放置的位置  
- proc  虚拟文件系统，存放当前内存的映射  
- root  超级用户目录  
- sbin存放二进制可执行文件，只有root才能访问  
- tmp   sbin用于存放各种临时文件  
- usr  用于存放系统应用程序，比较重要的目录/usr/local 本地管理员软件安装目录  
- var  用于存放运行时需要改变数据的文件


# Linux命令

[Linux 命令大全](https://www.runoob.com/linux/linux-command-manual.html)

**文件目录操作**

```xml

openxu@ubuntu:~$ hostname
openxu@ubuntu:~$ whoami

cat 连接文件并打印到标准输出设备上
sudo 以系统管理者的身份执行指令

ls 显示文件和目录列表 -l 列出文件的详细信息 -a 列出当前目录所有文件，包含隐藏文件
mkdir 创建目录 　 -p 父目录不存在情况下先生成父目录
cd 切换目录
touch 生成一个空文件
echo 生成一个带内容文件
cat、tac 显示文本文件内容
cp 复制文件或目录
rm 删除文件	-r 同时删除该目录下的所有文件	-f 强制删除文件或目录

mv 移动文件或目录、文件或 mv  aaa bbb 将aaa改名为bbb
find 在文件系统中查找指定的文件 -name  文件名
wc 统计文本文档的行数，字数，字符数
grep 在指定的文本文件中查找指定的字符串
rmdir 删除空目录
tree 显示目录目录改名树 
pwd 显示当前工作目录 
ln 建立链接文件
more、less 分页显示文本文件内容
Head、tail分别显示文件开头和结尾内容 
```

**系统管理命令**
```xml
stat 显示指定文件的相关信息,比ls命令显示内容更多 
who、w 显示在线登录用户 
whoami 显示用户自己的身份 
hostname 显示主机名称 
uname显示系统信息 
top 显示当前系统中耗费资源最多的进程 
ps 显示瞬间的进程状态
du 显示指定的文件（目录）已使用的磁盘空间的总量 
df 显示文件系统磁盘空间的使用情况 
free 显示当前内存和交换空间的使用情况 
ifconfig 显示网络接口信息 
ping 测试网络的连通性 
netstat 显示网络状态信息 
man 命令帮助信息查询
Alias 设置命令别名      alias [别名]=[“指令名”]
Clear 清屏
Kill 杀死进程

shutdown系统关机 -r 关机后立即重启，-h 关机后不重新启动 -now 立即关机
halt 关机后关闭电源 
reboot 重新启动
```

**备份压缩命令**
```xml
gzip 压缩（解压）文件或目录，压缩文件后缀为gz 
bzip2 压缩（解压）文件或目录，压缩文件后缀为bz2 
tar 文件、目录打（解）包


```

# VIM 命令

linux中的vi相当于windows中的记事本，下面的命令就是操作某个文件的命令，相当于
记事本上的菜单封装。

```xml
## 使用vim命令打开文件
vi log.txt

## 编辑模式：等待编辑命令输入
## 插入模式：编辑模式下，输入 i 进入插入模式，插入文本信息。Esc键返回编辑模式
## 命令模式：在编辑模式下，输入 “:” 进行命令模式

dd 删除当前行
x 删除当前光标位置字符

:q 直接退出vi
:wq 保存后退出vi ，并可以新建文件
:q! 强制退出
:w file 将当前内容保存成某个文件
:set number 在编辑文件显示行号
:set nonumber	在编辑文件不显示行号
详见《vi命令.docx》
```

# 账户系统文件

**1. /etc/passwd   用户表**

> 每行定义一个用户账户，此文件对所有用户可读。每行账户包含如下信息：

```xml
用户名:口令:用户标示号:组标示号:注释:宿主目录:命令解释器   
root:x:0:0:root:/root:/bin/bash
openxu:x:1000:1000:FPC-Node1,,,:/home/openxu:/bin/bash
```

- 口令是X: 说明用户的口令是被/etc/shadow文件保护的	
- 用户标识号: 系统内唯一，root用户的UID为0，普通用户从1000开始，1-999是系统的标准账户	
- 宿主目录: 用户登录系统后所进入的目录	
- 命令解释器: 指定该用户使用的shell ，默认的是/bin/bash

**2. /etc/shadow   用户密码表**

 > 为了增加系统的安全性，用户口令通常用shadow passwords保护。只有root可读。每行包含如下信息： 

```xml
用户名: 口令: 最后一次修改时间: 最小时间间隔: 最大时间间隔: 警告时间: 不活动时间: 失效时间: 标志
root: $6$PV6nehFt$RQ9AjF5tQvlYYNhJS.XtatnNaiEiy41: 18233: 0: 99999: 7: : :
openxu:$1$l$tmuhNeOK7cU7xoBExekdj0:18232:0:99999:7:::
```

- 最后一次修改时间: 从1970-1-1起，到用户最后一次更改口令的天数	
- 最小时间间隔: 从1970-1-1起，到用户可以更改口令的天数	
- 最大时间间隔: 从1970-1-1起，必须更改的口令天数	
- 警告时间: 在口令过期之前几天通知	
- 不活动时间: 在用户口令过期后到禁用账户的天数

**3. /etc/group   组表**

> 将用户进行分组时Linux对用户进行管理及控制访问权限的一种手段。一个组中可以有多个用户，一个用户可以同时属于多个组。该文件对所有用户可读。

```xml
组名：组口令：gid：组成员	
root:x:0:root
```

**4. /etc/gshadow   组密码表**

> 该文件用户定义用户组口令，组管理员等信息只有root用户可读

```xml
root:::root
```


## 使用命令行工具管理账户

**用户**

- useradd 用户名(必须是小写字母、数字、_，不能用大写字母)
- useradd –u（UID号）
- useradd –p（口令）
- useradd –g（分组）
- useradd –s（SHELL）
- useradd –d（用户目录）
- usermod –u（新UID）
- usermod –d（用户目录）
- usermod –g（组名）
- usermod –s（SHELL）
- usermod –p（新口令）
- usermod –l（新登录名）
- usermod –L (锁定用户账号密码)
- usermod –U (解锁用户账号)
- userdel 用户名 (删除用户账号)
- userdel –r 删除账号时同时删除目录

上面的参数太多，麻烦，可使用 `adduser 用户名`设置向导
```xml
openxu@ubuntu:~/test$ sudo adduser jonthlee
Adding user `jonthlee' ...
Adding new group `jonthlee' (1002) ...
Adding new user `jonthlee' (1002) with group `jonthlee' ...
Creating home directory `/home/jonthlee' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
Changing the user information for jonthlee
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] y

openxu@ubuntu:~$ pwd
/home/openxu
openxu@ubuntu:~$ cd ..
openxu@ubuntu:/home$ ls
jonthlee  openxu

```

**组账户维护命令**

- groupadd 组账户名 (创建新组)
- groupadd –g 指定组GID
- groupmod –g 更改组的GID
- groupmod –n 更改组账户名
- groupdel 组账户名 (删除指定组账户)

**口令维护命令**

- passwd 用户账户名 (设置用户口令)
- passwd –l 用户账户名 (锁定用户账户)
- passwd –u 用户账户名 (解锁用户账户)
- passwd –d 用户账户名 (删除账户口令)
- gpasswd –a 用户账户名 组账户名 (将指定用户添加到指定组)
- gpasswd –d 用户账户名 组账户名 (将用户从指定组中删除)
- gpasswd –A 用户账户名 组账户名 (将用户指定为组的管理员)


## 用户和组状态命令

- su 用户名: 切换用户账户 su root / su - root   （带-会加载用户目录下的.bash_profile环境变量，并切换到用户目录下）
- id 用户名: 显示用户的UID，GID
- whoami: 显示当前用户名称
- groups: 显示用户所属组
- sudo cat /etc/sudoers

## 配置文件

当我们登录Linux shell时，shell会执行一系列初始化动作，其中就包括读取配置文件，
然后根据配置文件来设置环境信息。事实上，在登录shell时会读取两个配置文件：/etc/profile
和用户目录下的配置文件（以.开头的隐藏文件.bash_profile）

**1./etc/profile**	

> 为系统的每个用户设置环境信息，对所有用户的登录shell都有效（全局配置文件）。
> 此文件中设定的变量（全局）可以作用于任何用户，而.bash_profile和.bashrc中
> 设定的变量（局部）只能作用于当前登录用户。/etc/profile和.bash_profile、
> .bashrc的关系类似于父子关系，具有继承特性。

**2. .bash_profile**	

> 为当前用户设置环境信息，仅对当前用户的登录shell有效（局部配置文件）。

**3. .bashrc**		

> .bash_profile只被登录shell读取并仅仅执行一次，如果在命令行上键入bash启动一个新的shell，
> 这个新shell读取的是.bashrc而不是.bash_profile，将登录shell和运行一个子shell所需的配置
> 文件分开可以获取非常灵活的配置策略，从而满足不同的场景。

**4. .bash_history**	

> 操作bash的历史记录

**5. .bash_logout**	

> 当每次退出shell时，该文件被读取并执行，主要做一些扫尾的工作，比如：删除帐号内临时文件
> 或记录登录系统所化时间等信息。

**6./etc/bashrc**	

> 和.bashrc的含义一样，只不过适用于所有的用户（全局）。

在登录Linux时，执行文件的顺序如下：
	登录Linux ---> /etc/profile ---> /etc/profile.d/*.sh 
	---> $HOME/{.bash_profile | .bash_login | .profile} 
	---> $HOME/.bashrc ---> /etc/bashrc


# 文件和目录的权限

```xml
openxu@ubuntu:~/test$ ls -l
total 12
## 文件类型: -表示普通文件，d表示目录，l表示链接文件（快捷方式）
## 文件权限rw-r--r--，分别表示用户权限、用户组权限、其他人的权限
## 文件类型: 文件权限: 目录的子目录数or文件硬链接数 : 文件属主: 文件所属组: 文件大小: 文件创建时间: 文件名
    -       rw-r--r--           1                openxu      openxu    39     Dec  2 19:14   bank.txt
-rw-r--r-- 1 openxu openxu    0 Dec  2 01:42 log1.txt
-rw-r--r-- 1 openxu openxu   26 Dec  2 02:08 log.txt
## room为目录，每个目录下至少包含2个子文件夹（. 和 ..）
drwxrwxr-x 2 openxu openxu 4096 Dec  2 22:57 room

openxu@ubuntu:~/test$ ls -a
.  ..  bank.txt  log1.txt  log.txt  room
```

## 更改操作权限

**chmod命令**: 更改操作权限
> 格式: chmod 谁 加减 权限名称 文件目录名

```xml
### 为其他用户对bank.txt 添加w(写)权限
openxu@ubuntu:~/test$ chmod o-x bank.txt 
### 文件夹room及其子目录权限改为rwxr-xrwx
openxu@ubuntu:~/test$ chmod -R 757 room
```

**-R参数** 下面的子目录做相同权限操作

- u 属主 g 所属组用户 o 其他用户 a 所有用户
- + 加权限 – 减权限 =加权限同时将原有权限删除
- rwx 权限名称

**使用数字表示权限**

r 4 w 2 x 1 

- rwx权限则4+2+1=7
- rw-权限则4+2=6
- r-x权限则4+1=5

## 更改属主及属组

**chown命令**: 更改与文件关联的所有者或组
> 格式: chown [ -R ] Owner [ :Group ] { File ... | Directory ... }

```xml
openxu@ubuntu:~/test$ ls -l
total 12
-rw-rw-rw- 1 openxu openxu   37 Dec  2 23:27 bank.txt
-rw-r--r-- 1 openxu openxu    0 Dec  2 01:42 log1.txt
-rw-r--r-- 1 openxu openxu   26 Dec  2 02:08 log.txt
drwsr-srwx 2 openxu openxu 4096 Dec  2 22:57 room

### 将log1.txt的属主改为jonthlee
openxu@ubuntu:~/test$ sudo chown jonthlee log1.txt
[sudo] password for openxu: 
openxu@ubuntu:~/test$ ls -l
total 12
-rw-rw-rw- 1 openxu   openxu   37 Dec  2 23:27 bank.txt
-rw-r--r-- 1 jonthlee openxu    0 Dec  2 01:42 log1.txt
-rw-r--r-- 1 openxu   openxu   26 Dec  2 02:08 log.txt
drwsr-srwx 2 openxu   openxu 4096 Dec  2 22:57 room

### 将log1.txt的属主改为openxu，属组为jonthlee (只改组的时候也需要将属主带上)
openxu@ubuntu:~/test$ sudo chown openxu:jonthlee log1.txt
openxu@ubuntu:~/test$ ls -l
total 12
-rw-rw-rw- 1 openxu openxu     37 Dec  2 23:27 bank.txt
-rw-r--r-- 1 openxu jonthlee    0 Dec  2 01:42 log1.txt
-rw-r--r-- 1 openxu openxu     26 Dec  2 02:08 log.txt
drwsr-srwx 2 openxu openxu   4096 Dec  2 22:57 room
openxu@ubuntu:~/test$ 

```

**Chgrp命令**:变更文件或目录所属群组（上面替换组的时候比较麻烦，可以使用该命令）

> 格式: chgrp [ -R ] Group { File ... | Directory ... }

```xml
### 将log1.txt 的属组改为openxu
openxu@ubuntu:~/test$ chgrp openxu log1.txt 
openxu@ubuntu:~/test$ ls -l
total 12
-rw-rw-rw- 1 openxu openxu   37 Dec  2 23:27 bank.txt
-rw-r--r-- 1 openxu openxu    0 Dec  2 01:42 log1.txt
-rw-r--r-- 1 openxu openxu   26 Dec  2 02:08 log.txt
drwsr-srwx 2 openxu openxu 4096 Dec  2 22:57 room
```


# 安装java

1. [jdk8下载](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

> 注意根据自己系统位数下载对应的压缩包

2. 将下载目录设置为vmware共享磁盘

> 该操作可以让vmware上的虚拟机访问windows宿主的某个文件夹，将其挂载到/mnt目录下。
> 虚拟机VM-->设置-->选项-->共享文件夹-->勾选总是启用-->添加共享文件夹-->确定

3. 复制jdk压缩文件到/usr/java中

```xml
## 根目录
openxu@ubuntu:/$ ls
bin  mnt   ... home   usr
## 进入下载目录
openxu@ubuntu:/mnt$ ls
openxu@ubuntu:/mnt$ cd hgfs/
openxu@ubuntu:/mnt/hgfs$ ls
openxu@ubuntu:/mnt/hgfs$ cd Downloads/
openxu@ubuntu:/mnt/hgfs/Downloads$ ls
jdk-8u231-linux-x64.tar.gz
jquery-easyui-1.7.0.zip
license_freeware.txt
node-v4.4.3-x64.msi
## 复制文件
openxu@ubuntu:/mnt/hgfs/Downloads$ sudo cp jdk-8u231-linux-x64.tar.gz /usr/java/jdk-8u231-linux-x64.tar.gz

#进入/usr/java目录后解压
openxu@ubuntu:/usr/java$ sudo tar zxvf jdk-8u231-linux-x64.tar.gz 
openxu@ubuntu:/usr/java$ ls -l
drwxrwxr-x 7  500  500      4096 Oct  5 03:59 jdk1.8.0_231
-rwxr-xr-x 1 root root 194773928 Dec  3 00:29 jdk-8u231-linux-x64.tar.gz

## 使用vim配置环境，添加如下内容
openxu@ubuntu:/usr/java$ sudo vi /etc/profile
###########
export JAVA_HOME=/usr/java/jdk1.8.0_231
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib

## 重新加载配置文件/etc/profile
openxu@ubuntu:/usr/java$ source /etc/profile
## 测试
openxu@ubuntu:/usr/java$ java -version
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)

```

