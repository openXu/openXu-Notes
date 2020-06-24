
[Cassandra教程](https://www.w3cschool.cn/cassandra/cassandra_introduction.html)

## Cassandra简介

Apache Cassandra是一个高度可扩展的高性能分布式数据库，用于处理大量商用服务器上的大量数据，
提供高可用性，无单点故障。这是一种NoSQL类型的数据库。

**NoSQL 数据库**
NoSQL数据库（有时称为“不是唯一的SQL”）是一种数据库，它提供一种机制来存储和检索数据，而不是
关系数据库中使用的表格关系。这些数据库是无架构的，支持简单的复制，具有简单的API，最终一致，
并且可以处理大量的数据。与关系数据库相比，NoSql数据库使用不同的数据结构。它使NoSQL中的一些操作更快。

**Cassandra 架构**

Cassandra的设计目的是处理跨多个节点的大数据工作负载，而没有任何单点故障。Cassandra在其节点之间具有
对等分布式系统，并且数据分布在集群中的所有节点之间。

- 集群中的所有节点都扮演相同的角色。 每个节点是独立的，并且同时互连到其他节点。
- 集群中的每个节点都可以接受读取和写入请求，无论数据实际位于集群中的何处。
- 当节点关闭时，可以从网络中的其他节点提供读/写请求。

## Cassandra安装

将Cassandra的Apache存储库添加到中/etc/apt/sources.list.d/cassandra.sources.list，例如，以获取最新的3.11版本：
echo "deb http://www.apache.org/dist/cassandra/debian 311x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
添加Apache Cassandra存储库密钥：
curl https://www.apache.org/dist/cassandra/KEYS | sudo apt-key add -

## Cassandra CQL命令

[教程](https://www.w3cschool.cn/cassandra/cassandra_create_table.html)

**常用命令**

下面的示例中，cql命令可使用cqlsh执行，Java示例中的cql字符串也可以使用cqlsh执行。

- 查看所有键空间
   - describe keyspaces;
- 删除键空间
   - drop keyspaces xxx;
- 查询系统键空间(未通过)
   - SELECT * FROM system.schema_keyspaces;
- 使用keyspace
   - USE open_xu;
- 查看空间中所有表
   - describe tables;


**key spaces（对应关系数据库中的数据库）**

```xml
<!--cassandra-->
<dependency>
   <groupId>com.datastax.cassandra</groupId>
   <artifactId>cassandra-driver-core</artifactId>
   <version>3.5.0</version>
</dependency>
<!-- https://mvnrepository.com/artifact/com.codahale.metrics/metrics-core -->
<dependency>
	<groupId>com.codahale.metrics</groupId>
	<artifactId>metrics-core</artifactId>
	<version>3.0.2</version>
</dependency>
```

```Java
//keyspace相关
@Test
public void keySpaceTest(){
	//创建一个Cluster集群构造器
	Cluster.Builder builder = new Cluster.Builder();
	//添加联系点（节点的IP地址）"192.168.16.180:9092,192.168.16.181:9092,192.168.16.182:9092");
	builder.addContactPoint("192.168.16.180");
	Cluster cluster = builder.build();
	//创建会话对象
	Session session = cluster.connect();
	/**
	 * CREATE KEYSPACE: 创建keyspace
	 * replication: 指定副本位置策略和所需副本的数量
	 * durable_writes: 指示Cassandra是否对当前KeySpace的更新使用commitlog。可选，默认为true。
	 */
	String query = "CREATE KEYSPACE open_xu WITH replication = {'class':" +
			"'SimpleStrategy', 'replication_factor':1} AND DURABLE_WRITES = false; ";
	session.execute(query);
	//        ResultSet resultSet = session.execute("describe keyspaces;");
	/**修改keyspace*/
	query = "ALTER KEYSPACE open_xu WITH replication = {'class':'SimpleStrategy'" +
			", 'replication_factor':2} AND DURABLE_WRITES = true; ";
	session.execute(query);
	/**删除*/
	query = "DROP KEYSPACE open_xu;";
	session.execute(query);
}
```

**table操作**

```Java
@Test
public void tableTest(){
	Cluster.Builder builder = new Cluster.Builder();
	builder.addContactPoint("192.168.16.180");
	Cluster cluster = builder.build();
	Session session = cluster.connect("open_xu");
	/**创建表   cqlsh:open_xu> select * from user;验证 */
	String cql = "CREATE TABLE user(\n" +
			"   id int PRIMARY KEY,\n" +
			"   name text,\n" +
			"   city text,\n" +
			"   phone varint\n" +
			"   );";
	session.execute(cql);
	cql = "CREATE TABLE user1(\n" +
			"   id int PRIMARY KEY,\n" +
			"   name text,\n" +
			"   city text,\n" +
			"   phone varint\n" +
			"   );";
	session.execute(cql);
	/**修改表*/
	//添加列
	cql = "ALTER TABLE user1 ADD email text;";
	session.execute(cql);
	//删除列
	cql = "ALTER TABLE user1 DROP email;";
	session.execute(cql);
	/**删除表*/
	cql = "DROP TABLE user1;";
	session.execute(cql);

	/**截断表：截断表时，表的所有行都将永久删除*/
	cql = "TRUNCATE user1;";
	session.execute(cql);

}
```

**CURD操作**

```Java
@Test
public void cyrdTest(){
	Cluster.Builder builder = new Cluster.Builder();
	builder.addContactPoint("192.168.16.180");
	Cluster cluster = builder.build();
	Session session = cluster.connect("open_xu");
	/**添加数据 */
	String cql = "insert into user (id, name, city, phone)" +
			"values(1, 'openxu', '北京', 1821)";
	session.execute(cql);
	cql = "insert into user (id, name, city, phone)" +
			"values(2, 'sandy', '长沙', 378965)";
	session.execute(cql);
	cql = "insert into user (id, name, city, phone)" +
			"values(3, 'jonthons Li', '湖南', 2312232)";
	session.execute(cql);
	cql = "insert into user (id, name, city, phone)" +
			"values(4, 'zjl', '湖南', 2312232)";
	session.execute(cql);
	/**更新数据*/
	cql = "update user set city='石门', phone=2000 where id = 2;";
	session.execute(cql);
	/**删除列（一条记录某个字段）*/
	cql = "delete phone from user where id = 4";
	session.execute(cql);
	/**删除行(一条记录)*/
	cql = "delete from user where id = 4";
	session.execute(cql);
	/**查询*/
	cql = "select * from user";
	ResultSet set = session.execute(cql);
	System.out.println(set.all());

	/**批处理（一次执行多个cql）*/
	cql ="BEGIN BATCH " +
			"insert into user (id, name, city, phone)"
			+ " values(5, 'qqq', '湖南', 432);"
			+ " update user set name='open_Xu', phone=110 WHERE id = 1;"
			+ " DELETE city FROM user WHERE id = 1;"
			+ " APPLY BATCH;";
	session.execute(cql);
}
```

## CQL数据类型

|数据类型	|常量				|描述						|
|---|:--:|---:|
|ascii		|strings			|表示ASCII字符串			|
|bigint		|bigint				|表示64位有符号长			|
|blob		|blobs				|表示任意字节				|
|Boolean	|booleans			|表示true或false			|
|counter	|integers			|表示计数器列				|
|decimal	|integers, floats	|表示变量精度十进制			|
|double		|integers			|表示64位IEEE-754浮点		|
|float		|integers, floats	|表示32位IEEE-754浮点		|
|inet		|strings			|表示一个IP地址，IPv4或IPv6	|
|int		|integers			|表示32位有符号整数			|
|text		|strings			|表示UTF8编码的字符串		|
|timestamp	|integers, strings	|表示时间戳					|
|timeuuid	|uuids				|表示类型1 UUID				|
|uuid		|uuids				|表示类型1或类型4			|
|			|					|UUID						|
|varchar	|strings			|表示uTF8编码的字符串		|
|varint		|integers			|表示任意精度整数			|

|集合	|描述								|
|---|:--:|
|list	|列表是一个或多个有序元素的集合。	|
|map	|地图是键值对的集合。				|
|set	|集合是一个或多个元素的集合。		|

**自定义数据类型**

Cqlsh为用户提供了创建自己的数据类型的工具。下面给出了处理用户定义的数据类型时使用的命令。

- CREATE TYPE -创建用户定义的数据类型。
- ALTER TYPE -修改用户定义的数据类型。
- DROP TYPE -删除用户定义的数据类型。
- DESCRIBE TYPE -描述用户定义的数据类型。
- DESCRIBE TYPES -描述用户定义的数据类型。

### CQL集合

```Java
@Test
public void cqlDataTypeTest(){
	Cluster.Builder builder = new Cluster.Builder();
	builder.addContactPoint("192.168.16.180");
	Cluster cluster = builder.build();
	Session session = cluster.connect("open_xu");
	/**List集合*/
	String cql = "CREATE TABLE user2(name text PRIMARY KEY, email list<text>);";
	session.execute(cql);
	//插入
	cql = "INSERT INTO user2(name, email) VALUES ('openxu'," +
			"['abc@gmail.com','cba@yahoo.com']);";
	session.execute(cql);
	//修改  + or -
	cql = "Update user2 set email = email+['543232@qq.com'] where name = 'openxu';";
	session.execute(cql);

	/**Set集合*/
	cql = "CREATE TABLE user3(name text PRIMARY KEY, phone set<varint>);";
	session.execute(cql);
	//插入
	cql = "INSERT INTO user3(name, phone) VALUES ('openxu'," +
			"{09876,987456});";
	session.execute(cql);
	//修改 + or -
	cql = "Update user3 set phone = phone+{123789} where name = 'openxu';";
	session.execute(cql);

	/**Map集合*/
	cql = "CREATE TABLE user4(name text PRIMARY KEY, addr map<text, text>);";
	session.execute(cql);
	//插入
	cql = "INSERT INTO user4(name, addr) VALUES ('openxu'," +
			"{'home':'石门县', 'office':'北京', 'aa':'wef'});";
	session.execute(cql);
	//修改
	cql = "delete addr['aa'] from user4 where name = 'openxu';";
	session.execute(cql);
	cql = "Update user4 set addr['office']=null where name = 'openxu';";
	session.execute(cql);
}
```

### CQL自定义数据类型

CQL提供了创建和使用用户定义的数据类型的功能。您可以创建一个数据类型来处理多个字段。

```xml
# 创建自定义数据类型
cqlsh:open_xu> create type user_info(name text, phone set<int>);
# 查看所有自定义类型
cqlsh:open_xu> describe types;
user_info
# 查看指定类型详情
cqlsh:open_xu> describe type user_info;
CREATE TYPE open_xu.user_info (
    name text,
    phone set<int>
);
# 修改自定义类型
cqlsh:open_xu> alter type user_info add city text;
cqlsh:open_xu> describe type user_info;
CREATE TYPE open_xu.user_info (
    name text,
    phone set<int>,
    city text
);

# 删除自定义类型
cqlsh:open_xu> drop type user_info;
cqlsh:open_xu> describe types;
<empty>

```


