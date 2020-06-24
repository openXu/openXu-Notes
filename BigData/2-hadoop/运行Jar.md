```xml
### 前台运行jar
root@Node3-fzy-> java -jar DataSync.jar

### 无窗口运行jar
root@Node3-fzy-> nohup java -jar /mnt/APP/DataSync/DataSync.jar &


### 创建批处理文件
root@Node3-fzy-> vim startup.sh
	
	#!/bin/bash
	nohup java -jar /mnt/APP/DataSync/DataSync.jar > /dev/null 2>&1 &

### 修改执行权限
root@Node3-fzy-> chmod 755 startup.sh

### 执行批处理文件
root@Node3-fzy-> ./startup.sh

### 查看运行的程序
root@Node3-fzy-> jps
76449 DataSync.jar
76935 Jps
(base) [/mnt/APP/DataSync]
### 杀死程序
root@Node3-fzy-> kill -9 76449
(base) [/mnt/APP/DataSync]

### 查看文件内容
root@Node3-fzy-> tail -f log/DataSync/DataSync_info.log
```

## Shell Java 程序部署

###  启动、停止程序sh

1. 启动：

   编写shell脚本，通过判断进程号是否存在来启动程序

   ```sehll
   #!/bin/bash
   
   echo "Preparing to start jar 'DataApi'..."
   # 进程名——jar包名
   PROC_NAME=DataApi 
   # 进程运行状态，是否运行 1：运行，0：不存在
   PROC_BOOL=`ps -ef|grep -w $PROC_NAME|grep -v grep|wc -l`
   # 判断如果进程正在运行
   if [ $PROC_BOOL -eq 1 ];then
       echo "Pid existed! Please check if your application is running."
   # 如未运行启动程序，输出pid号
   else
   	nohup java -jar /mnt/APP/Api/DataApi.jar >/dev/null 2>&1 &  
       echo "Jar started successfully! Pid num is $!"
   fi
   ```

2. 停止：

   编写shell脚本，查看进程号是否存在，如存在则停止，否则打印进程不存在

   ```shell
   #!/bin/bash
   # 进程名
   PROC_NAME=DataApi
   # 进程运行状态
   PROC_BOOL=`ps -ef|grep -w $PROC_NAME|grep -v grep|wc -l`
   # 进程号
   PROC_NUM=`ps gaux | grep $PROC_NAME | grep -v grep | awk '{print $2}'`
   # 如果进程存在
   if [ $PROC_BOOL -eq 1 ];then
       echo "Pid existed. Stopping application..."
      # 停止进程
   	kill -9 $PROC_NUM
   	echo "Application stopped successfully!"
   else
   	echo "Pid not exist! Application may not be running!"
   fi
   
   
   
   ```

   

