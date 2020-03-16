### 一 - 一对多操作

**1.一对多映射配置**

> 创建两个实体类，公司和员工，一个公司有多个员工，一个员工只属于一个公司

```Java
public class Company {
	private Integer cid;
	private String name;
	private String adress;
	//hibernate要求使用set集合（无序、无重复元素）表示多的数据
	private Set<Worker> workers = new HashSet<Worker>();
	...
}

public class Worker {

	private Integer wid;
	private String wname;
	private String wsex;
	
	private Company company;
	...
}
```
> 实体映射文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-mapping PUBLIC 
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
	<class name="com.openxu.hbt.entity.Company" table="t_company">
		<id name="cid" column="cid">
			<generator class="native"></generator>
		</id>
		<property name="name"></property>
		<property name="adress"></property>
		<!-- set标签表示所有员工的集合 -->
		<set name="workers" >
			<!-- 外键:hibernate双向维护外键机制， 在一和多的配置文件中都要配置外键-->
			<key column="cid"></key>
			<!-- 一对多对应的实体类 -->
			<one-to-many class="com.openxu.hbt.entity.Worker"></one-to-many>
		</set>
	</class>
</hibernate-mapping>

<hibernate-mapping>
	<class name="com.openxu.hbt.entity.Worker" table="t_worker">
		<id name="wid" column="wid">
			<generator class="native"></generator>
		</id>
		<property name="wname" ></property>
		<property name="wsex" ></property>
		<!-- 多对一对应的实体类 -->
		<many-to-one name="company" class="com.openxu.hbt.entity.Worker" column="cid"></many-to-one>
	</class>
</hibernate-mapping>
```

> 核心配置文件
```xml
<!-- 第三部分： 把映射文件放到核心配置文件中 必须的-->
<mapping resource="com/openxu/hbt/entity/Company.hbm.xml"/>
<mapping resource="com/openxu/hbt/entity/Worker.hbm.xml"/>
```

**2.级联保存**

> 添加一个公司，为这个公司同事添加多个员工
		
```Java
	//一对多级联添加
	//1、创建公司和员工对象
	Company company = new Company();
	company.setName("法之运");
	company.setAdress("北京市昌平区回龙观东大街");
	
	Worker worker = new Worker();
	worker.setWname("许开放");
	worker.setWsex("男");
	//2、把员工放到公司的set集合里面
	company.getWorkers().add(worker);
	//3、为员工设置所属公司
	worker.setCompany(company);
	//4、级联保存到数据库
	session.save(company);
	session.save(worker);
```

> 简化写法
```xml
<!-- cascade="save-update"表示级联保存 -->
<set name="workers" cascade="save-update">
	<key column="cid"></key>
	<one-to-many class="com.openxu.hbt.entity.Worker"></one-to-many>
</set>
```
```Java
Company company = new Company();
company.setName("法之运");
company.setAdress("北京市昌平区回龙观东大街");

Worker worker = new Worker();
worker.setWname("许开放");
worker.setWsex("男");

company.getWorkers().add(worker);

session.save(company);
```

**3.级联删除**

> 删除一个公司，将这个公司里所有员工也删除。
> 通过数据库客户端sql删除公司时不支持级联删除，如果公司中有员工（有外键关联），则这个公司删不掉
```xml
<!-- cascade="delete"表示级联删除，多个值用,分隔 -->
<set name="workers" cascade="save-update,delete">
	<key column="cid"></key>
	<one-to-many class="com.openxu.hbt.entity.Worker"></one-to-many>
</set>
```
```Java
Company company = session.get(Company.class, 3);
session.delete(company);
```

**4.级联修改**

```Java
//根据id查询公司、员工，然后设置级联关系，持久态对象会自动保存数据库
Company company = session.get(Company.class, 1);
Worker worker = session.get(Worker.class, 3);
company.getWorkers().add(worker);
worker.setCompany(company);
```

**4.1.inverse属性**

> 因为hibernate双向维护外键，在客户和联系人里面都需要维护外键，修改客户时候修改一次外键，
> 修改联系人时候也修改一次外键，造成效率问题

解决方式:
> 一对多里面，让其中一的一方映射文件中配置inverse属性，使其放弃外键维护
> 一个国家有总统，国家有很多人，总统不能认识国家所有人，国家所有人可以认识总统
```xml
<!-- cascade="delete"表示级联删除，多个值用,分隔 -->
<set name="workers" inverse="true">
	<key column="cid"></key>
	<one-to-many class="com.openxu.hbt.entity.Worker"></one-to-many>
</set>
```

### 一 - 多对多操作

**1.多对多映射配置**

> 一个角色有多个人，一个人有多个角色

```Java
public class Role {
  	private Integer role_id;//角色id
  	private String role_name;//角色名称
  	private String role_memo;//角色描述
  	
  	// 一个角色有多个用户
  	private Set<User> setUser = new HashSet<User>();
  	
	public Set<User> getSetUser() {
		return setUser;
	}
	public void setSetUser(Set<User> setUser) {
		this.setUser = setUser;
	}
	...
}

public class User {
  	private Integer user_id;//用户id
  	private String user_name;//用户名称
  	private String user_password;//用户密码
  	
  	//一个用户可以有多个角色
  	private Set<Role> setRole = new HashSet<Role>();
  	
	public Set<Role> getSetRole() {
		return setRole;
	}
	public void setSetRole(Set<Role> setRole) {
		this.setRole = setRole;
	}
	...
}
```
```xml
<hibernate-mapping>
	<class name="cn.itcast.manytomany.Role" table="t_role">
		<id name="role_id" column="role_id">
			<generator class="native"></generator>
		</id>
		<property name="role_name" column="role_name"></property>
		<property name="role_memo" column="role_memo"></property>
		
		<!-- 在角色里面表示所有用户，使用set标签 -->
		<set name="setUser" table="user_role">
			<!-- 角色在第三张表外键 -->
			<key column="roleid"></key>
			<many-to-many class="cn.itcast.manytomany.User" column="userid"></many-to-many>
		</set>
	</class>
</hibernate-mapping>

<hibernate-mapping>
	<class name="cn.itcast.manytomany.User" table="t_user">
		<id name="user_id" column="user_id">
			<generator class="native"></generator>
		</id>
		<property name="user_name" column="user_name"></property>
		<property name="user_password" column="user_password"></property>
		<!-- 在用户里面表示所有角色，使用set标签 
			name属性：角色set集合名称
			table属性：第三张表名称
		-->
		<set name="setRole" table="user_role" cascade="save-update,delete">
			<!-- key标签里面配置
				配置当前映射文件在第三张表外键名称
			 -->
			<key column="userid"></key>
			<!-- class：角色实体类全路径
			     column：角色在第三张表外键名称
			 -->
			<many-to-many class="cn.itcast.manytomany.Role" column="roleid"></many-to-many>
		</set>
	</class>
</hibernate-mapping>
```

**2.级联保存**

> 添加一个公司，为这个公司同事添加多个员工
		
```Java
//添加两个用户，为每个用户添加两个角色
Actor actor1 = new Actor();
actor1.setAname("張嘉譯");
actor1.setApassword("1023456");

Actor actor2 = new Actor();
actor2.setAname("張國榮");
actor2.setApassword("123456");

Role role1 = new Role();
role1.setRname("总经理");
role1.setRmemo("总经理");

Role role2 = new Role();
role2.setRname("霸王別姬");
role2.setRmemo("霸王別姬");

Role role3 = new Role();
role3.setRname("秘书");
role3.setRmemo("秘书");

actor1.getSetRole().add(role1);
actor1.getSetRole().add(role2);
actor2.getSetRole().add(role2);
actor2.getSetRole().add(role3);

session.save(actor1);
session.save(actor2);
```

**3.级联删除**

```Java
//这样操作是有问题的，会将id为1的用户相关的所有信息删除，包含用户所有的角色和这些角色相关的数据（可能将别人的角色关系表也删除了）
Actor actor = session.get(Actor.class, 1);
session.delete(actor);

//多对多级联删除需要操作第三张关系表，不能直接删除非关系表的数据
//上面的示例应该这样做
Actor actor = session.get(Actor.class, 1);
actor.getSetRole().clear();
session.delete(actor);

```


**4.级联修改**
```Java
ctor actor = session.get(Actor.class, 1);
System.out.println(actor);
Role role = session.get(Role.class, 3);
actor.getSetRole().add(role);
```










