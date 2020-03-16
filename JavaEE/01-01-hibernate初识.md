### 一 - Web内容回顾

- **JavaEE三层结构**
1. web层：struts2框架
2. service层：spring框架
3. dao层：hibernate框架（数据库crud操作）

- **MVC思想**
1. M:模型
2. V:视图
3. C:控制器

### 二 - Hibernate概述

hibernate框架应用在javaee三层结构中的dao层，对数据库做crud操作，其
底层代码就是jdbc，hibernate对jdbc进行封装，不需要写复杂的jdbc代码，
也不需要写sql实现语句，它是一个开源的轻量级的框架

- **orm思想**

orm (object relational mapping)对象关系映射，让实体类（javabean）和数据库表进行
一一映射，让实体类属性和表中字段一一对应。不需要直接操作数据库表，而是操作对应实体类对象。
hibernate就是使用的这种思想。

- **JDBC回顾**
```Java
//1. 加载驱动程序
Class.forName(driverClass)
//加载MySql驱动
Class.forName("com.mysql.jdbc.Driver")
//加载Oracle驱动
Class.forName("oracle.jdbc.driver.OracleDriver")
//2. 获得数据库连接
Connection conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/imooc", "root", "root");
//3. 创建Statement\PreparedStatement对象
conn.createStatement();
PreparedStatement psmt = conn.prepareStatement(sql);
//执行sql查询
ResultSet rs = pstm.executeQuery();
//遍历结果集
while(rs.next()){
	System.out.println(rs.getString("user_name")+" 年龄："+rs.getInt("age"));
}
//释放资源
...
```

- **Hibernate与JDBC对比**
```Java
//不需要操作表，直接操作对应实体类即可，和Android的GreenDao（Android ORM for SQLite databas）一样
User user = new User();
user.setUserName("openXu");
//hibernate提供的session对象
session.save(user);
```

### 三 - Hibernate环境搭建

1. 导入hibernate的jar包
```
//必须的jar包
antlr-2.7.7.jar
dom4j-1.6.1.jar
geronimo-jta_1.1_spec-1.1.1.jar
hibernate-commons-annotations-5.0.1.Final.jar
hibernate-core-5.0.7.Final.jar
hibernate-jpa-2.1-api-1.0.0.Final.jar
jandex-2.0.0.Final.jar
javassist-3.18.1-GA.jar
jboss-logging-3.3.0.Final.jar
//规范
hibernate-entitymanager-5.0.7.Final.jar
//mysql驱动
mysql-connector-java-5.0.4-bin.jar
//日志信息输出
log4j-1.2.16.jar
slf4j-api-1.6.1.jar
slf4j-log4j12-1.7.2.jar
```

2. 创建实体类
```Java
public class User{
	private int uid;  //hibernate要求实体类有一个唯一值属性
	private String userName;
	private String password;
	...
}
```

3. 配置实体类和数据库表的映射关系

> 创建xml格式的配置文件，文件名称和位置无固定要求，建议放在实体类所在包中，名称=实体类.hbm.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 引入hibernate的dtd约束（hibernate-mapping-3.0.dtd中拷贝） -->
<!DOCTYPE hibernate-mapping PUBLIC 
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
	<!-- 1 class标签配置类和表映射    name属性：实体类全路径 ; table属性：数据库表名称-->
	<class name="cn.itcast.entity.User" table="t_user">
		<!-- 2 id标签 配置实体类id和表id对应 
			hibernate要求实体类有一个属性唯一值
			hibernate要求表有字段作为唯一值
			name属性：实体类里面id属性名称
			column属性：生成的表字段名称
		 -->
		<id name="uid" column="uid">
			<!-- generator标签设置数据库表id增长策略 
				native:生成表id值就是主键自动增长-->
			<generator class="uuid"></generator>
		</id>
		<!-- 配置其他属性和表字段对应 
			name属性：实体类属性名称
			column属性：生成表字段名称，可省略，省略后表字段和实体属性名一样
			type属性：字段类型，可省略，自动对应实体类属性类型
		-->
		<property name="userName" column="username" type="string"></property>
		<property name="passWord" column="password"></property>
	</class>
</hibernate-mapping>
```

4. 创建hibernate核心配置文件

> 核心配置文件格式为xml，但是名称（hibernate.cfg.xml）和位置(src目录下)是固定的

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- hibernate-configuration-3.0.dtd中拷贝 -->
<!DOCTYPE hibernate-configuration PUBLIC
	"-//Hibernate/Hibernate Configuration DTD 3.0//EN"
	"http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
	<session-factory>
		<!-- 第一部分： 配置数据库信息 必须的 -->
		<property name="hibernate.connection.driver_class">com.mysql.jdbc.Driver</property>
		<property name="hibernate.connection.url">jdbc:mysql:///hibernate_day01</property>
		<property name="hibernate.connection.username">root</property>
		<property name="hibernate.connection.password">root</property>
		
		<!-- 第二部分： 配置hibernate信息  可选的-->
		<!-- 输出底层sql语句 -->
		<property name="hibernate.show_sql">true</property>
		<!-- 格式化底层sql语句输出 -->
		<property name="hibernate.format_sql">true</property>
		<!-- hibernate帮创建表，但是需要配置之后 
			update: 如果已经有表，更新，如果没有，创建
		-->
		<property name="hibernate.hbm2ddl.auto">update</property>
		<!-- 配置数据库方言
			在mysql里面实现分页 关键字 limit，只能使用mysql里面
			在oracle数据库，实现分页rownum
			让hibernate框架识别不同数据库的自己特有的语句
		 -->
		<property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
		
		<!-- 第三部分： 把映射文件放到核心配置文件中 必须的-->
		<mapping resource="cn/itcast/entity/User.hbm.xml"/>
	</session-factory>
</hibernate-configuration>
```

### 四 - Hibernate 基本使用

```Java
//第一步 加载hibernate核心配置文件
//到src下面找到名称是hibernate.cfg.xml
//Configuration cfg = new Configuration();
//cfg.configure();

//第二步 创建SessionFactory对象
//读取hibernate核心配置文件内容，创建sessionFactory
//在过程中，根据映射关系，在配置数据库里面把表创建
//SessionFactory sessionFactory = cfg.buildSessionFactory();

SessionFactory sessionFactory = HibernateUtils.getSessionFactory();

//第三步 使用SessionFactory创建session对象
// 类似于连接
Session session = sessionFactory.openSession();

//第四步 开启事务
Transaction tx = session.beginTransaction();

//第五步 写具体逻辑 crud操作
//添加功能
User user = new User();
user.setUsername("小马");
user.setPassword("1314520");
user.setAddress("美国");
//调用session的方法实现添加
session.save(user);

//第六步 提交事务
tx.commit();
//第七步 关闭资源
session.close();
sessionFactory.close();
```


### 五 - Hibernate API

#### Configuration
- Configuration用于配置Hibernate环境，它会到src目录下加载名为hibernate.cfg.xml的配置文件

#### SessionFactory
- SessionFactory对象被创建时，会根据核心配置文件中的配置完成数据库的初始化操作，比如表的创建（需要配置）
- SessionFactory对象创建是特别耗资源的，建议一个项目一般只创建一个sessionFactory对象
- 具体实现思想如下：

```Java
public class HibernateUtils {
	static Configuration cfg = null;
	static SessionFactory sessionFactory = null;
	static {
		cfg = new Configuration();
		cfg.configure();
		sessionFactory = cfg.buildSessionFactory();
	}
	public static SessionFactory getSessionFactory() {
		return sessionFactory;
	}
}

@Test
public void testSave(){
	SessionFactory sessionFactory = null;
	Session session = null;
	Transaction tx = null;
	try{
		sessionFactory = HibernateUtils.getSessionFactory();
		session = sessionFactory.getCurrentSession();
		tx = session.beginTransaction();
		User user = new User();
		user.setUsername("openXu");
		user.setPassword("123456");
		user.setAddress("China");
		session.save(user);
		tx.commit();
	}catch(Exception e){
		e.printStackTrace();
		tx.rollback();
	}finally{
		session.close();
		sessionFactory.close();
	}
}
```
#### Transaction
- 事务四个特性：原子性、一致性、隔离性、持久性
```Java
Transaction tx = session.beginTransaction();//开启事务
tx.commit();    //事务提交
tx.rollback();  //事务回滚
```

#### Session
- Session类似于jdbc中的Connection，它提供了大量方法用于实现crud操作
> sace()、update()、saveOrUpdate()、delete()、get、load()、createQuery()、createSQLQuery()、createCriteria()
- Session对象为单线程对象

#### Query
> 使用Query对象不需要写sql语句，而是使用hql（hibernate query language）。
> sql操作表和表字段，而hql操作实体类和属性

```Java
//使用hql语句创建Query对象（from 实体类名称）
Query query = session.createQuery("from User");
List<User> list= query.list();
for(User user : list){
	System.out.println(user);
}
```
#### Criteria
```Java
//Criteria：使用实体类class创建Criteria
Criteria criteria = session.createCriteria(User.class);
List<User> list = criteria.list();
for(User user : list){
	System.out.println(user);
}
```
#### SQLQuery
```Java
//SQLQuery:使用sql语句创建SQLQuery对象
SQLQuery sqlQuery = session.createSQLQuery("select * from t_user");
//List<Object[]> list = sqlQuery.list();
//for(Object[] objects : list){
//	System.out.println(Arrays.toString(objects));
//}
//设置查询出的数据放到那个实体类中，如果不设置将放回Object数组集合
sqlQuery.addEntity(User.class);
List<User> list = sqlQuery.list();
for(User user : list){
	System.out.println(user);
}
```

### 六 - 配置提示
两种方式：
1. 可以上网
2. 本地配置
- Eclipse->Window->Preferences->搜索XML Catalog->Add
- Key type:URL    Key:http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd
- Location:选择对应的dtd文件（hibernate-mapping-3.0.dtd）



