
## Redis服务器安装
Redis项目本身不支持windows，但是Microsoft开放技术小组开发和维护这个windows端口（针对Win64），所以我们可以在网络上下载Redis的
windows版本。

[Redis官网](https://redis.io/download)

**1. 下载安装**

[Redis for Windows](https://github.com/MicrosoftArchive/redis/releases)，点击release，可看到很多版本的安装包，选择
最新正式版`Redis-x64-3.0.504.zip`。

下载完成后解压，打开`redis-server.exe`启动Redis服务器，然后找到Redis客户端程序`redis-cli.exe`打开。

**2. Redis使用命令缓存测试**

使用Redis客户端对Redis的集中数据类型做基本的增删改查操作练习：

- 服务器相关
```xml
//连接到本地的 redis 服务
root@Node3-fzy-> redis-cli -h 127.0.0.1 -p 6666
//操作被拒绝，需要输入密码
127.0.0.1:6666> set name 'openxu'
(error) NOAUTH Authentication required.
//输入密码认证
127.0.0.1:6666> auth 123456
//选择数据库（数据库编号0-15）
127.0.0.1:6666> select 4
OK
# 获得服务的信息与统计
127.0.0.1:6666[4]> info
# Server
redis_version:3.2.12
redis_git_sha1:00000000
--------省略
＃实时监控
127.0.0.1:6666[4]> monitor
127.0.0.1:6666[4]> config get＃获得服务配置
127.0.0.1:6666[4]> flushdb＃删除当前选择的数据库中的key
127.0.0.1:6666[4]> flushall＃删除所有数据库中的键

 
```


- String类型数据操作
```xml
# 增加一个值key为name，value为openxu
127.0.0.1:6379> set name 'openxu'
OK
# 查询name的值
127.0.0.1:6379> get name
"openxu"
# 更新name值
127.0.0.1:6379> set name 'openXu'
OK
127.0.0.1:6379> get name
"openXu"
# 删除name的值
127.0.0.1:6379> del name
(integer) 1
# 查询是否存在name,0代表不存在
127.0.0.1:6379> exists name
(integer) 0
```

- List集合增删改查

```xml
# 添加key为userList，value为'openXu' 'Sandy'的List集合
127.0.0.1:6379> lpush userList 'openXu' 'Sandy'
(integer) 2
# 查询key为userList的集合
127.0.0.1:6379> lrange userList 0 -1
1) "Sandy"
2) "openXu"
# 往list尾部添加'Jonathan Lee'
127.0.0.1:6379> rpush userList 'Jonathan Lee'
(integer) 3
# 往list头部添加'Jay'
127.0.0.1:6379> lpush userList 'Jay'
(integer) 4
127.0.0.1:6379> lrange userList 0 -1
1) "Jay"
2) "Sandy"
3) "openXu"
4) "Jonathan Lee"
# 更新index为0的值
127.0.0.1:6379> lset userList 0 'open-xu'
OK
# 删除index为0的值
127.0.0.1:6379> lrem userList 0 'open-xu'
(integer) 1
127.0.0.1:6379> lrange userList 0 -1
1) "Sandy"
2) "openXu"
3) "Jonathan Lee"
```

- Set集合增删该查

```xml
# 添加key为userSet的set集合
127.0.0.1:6379> sadd userSet "openxu" "Sandy" "Jonathan Lee"
(integer) 3
# 查询key为userSet的集合
127.0.0.1:6379> smembers userSet
1) "openxu"
2) "Sandy"
3) "Jonathan Lee"
# 删除value为Sandy，返回1表示成功
127.0.0.1:6379> srem userSet 'Sandy'
(integer) 1
# 添加sandy元素，set没有顺序
127.0.0.1:6379> sadd userSet "sandy"
(integer) 1
127.0.0.1:6379> smembers userSet
1) "sandy"
2) "openxu"
3) "Jonathan Lee"
# 添加sandy元素，返回0表示失败，因为已经存在
127.0.0.1:6379> sadd userSet "sandy"
(integer) 0
127.0.0.1:6379>
```

- 清除数据库

```xml
127.0.0.1:6379> flushdb
OK
```

- Hash集合的增删改查

```xml
# 创建hash，key为userHash，字段为openxu 值为man
127.0.0.1:6379> hset userHash "openxu" "man"
(integer) 1
127.0.0.1:6379> hset userHash "Sandy" "woman"
(integer) 1
# 查询userHash字段长度
127.0.0.1:6379> hlen userHash
(integer) 2
# 查询userHash所有字段
127.0.0.1:6379> hkeys userHash
1) "openxu"
2) "Sandy"
# 查询userHash所有值
127.0.0.1:6379> hvals userHash
1) "man"
2) "woman"
# 查询字段openxu的值
127.0.0.1:6379> hget userHash "openxu"
"man"
# 获取key为userHash的哈希集合的所有字段和值
127.0.0.1:6379> hgetall userHash
1) "openxu"
2) "man"
3) "Sandy"
4) "woman"
# 更新字段openxu的值为woman
127.0.0.1:6379> hset userHash "openxu" "woman"
(integer) 0
127.0.0.1:6379> hgetall userHash
1) "openxu"
2) "woman"
3) "Sandy"
4) "woman"
# 删除userHash中的Sandy字段以及其值
127.0.0.1:6379> hdel userHash "Sandy"
(integer) 1
127.0.0.1:6379> hgetall userHash
1) "openxu"
2) "woman"
127.0.0.1:6379>

```

- SortedSet集合的增删改查

```xml
# 为userZset集合添加a1元素，分数为1
127.0.0.1:6379> zadd userZset 1 "a1"
(integer) 1
127.0.0.1:6379> zadd userZset 2 "a2"
(integer) 1
# 为userZset集合添加a3元素，分数为3
127.0.0.1:6379> zadd userZset 3 "a3"
(integer) 1
# 按照分数由小到大查询userZset的所有元素
127.0.0.1:6379> zrange userZset 0 -1
1) "a1"
2) "a2"
3) "a3"
# 按照分数由大到小查询userZset的所有元素
127.0.0.1:6379> zrevrange userZset 0 -1
1) "a3"
2) "a2"
3) "a1"
# 查询元素a2的分数值
127.0.0.1:6379> zscore userZset "a2"
"2"
```


























