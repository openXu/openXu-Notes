
## JdbcTemplate初体验

### 基本使用

配置依赖

```xml
<!--Spring jdbc-->
<!-- 数据库连接 https://mvnrepository.com/artifact/org.springframework/spring-jdbc -->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-jdbc</artifactId>
	<version>${spring.version}</version>
</dependency>
<!-- 数据库映射 https://mvnrepository.com/artifact/org.springframework/spring-orm -->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-orm</artifactId>
	<version>${spring.version}</version>
</dependency>
<!--数据库连接驱动-->
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>5.1.47</version>
</dependency>
```

创建表

```sql
create table account(
	id int primary key auto_increment,
	name varchar(40),
	money float
)character set utf8 collate utf8_general_ci;

insert into account(name, money) values ('aaa', 1000);
insert into account(name, money) values ('bbb', 1000);
insert into account(name, money) values ('ccc', 1000);

```

使用

```Java
public static void main(String[] args) {
	//1. 定义数据源
	DriverManagerDataSource ds = new DriverManagerDataSource();
	ds.setDriverClassName("com.mysql.jdbc.Driver");
	ds.setUrl("jdbc:mysql://localhost:3306/openxu");
	ds.setUsername("root");
	ds.setPassword("root");
	//2. 获取对象
	JdbcTemplate jt = new JdbcTemplate(ds);
	//3. 执行操作
	jt.execute("insert into account(name, money) values ('ddd', 1000);");
}
```

### IoC改造

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">

    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/openxu"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>
</beans>
```

```Java
public static void main(String[] args) {
	ApplicationContext ac = new ClassPathXmlApplicationContext("jdbcbean.xml");
	JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
	jt.execute("insert into account(name, money) values ('eee', 123456);");
}
```



## JdbcTemplate使用

### JdbcTemplate的CRUD

```Java
public static void main(String[] args) {
	ApplicationContext ac = new ClassPathXmlApplicationContext("jdbcbean.xml");
	JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
//        jt.execute("insert into account(name, money) values ('eee', 1000);");
	//保存
//        jt.update("insert into account(name, money) values (?, ?);", "ggg", 123456789);
	//更新
//        jt.update("update account set money = ? where id = ?", 444, 3);
	//删除
//        jt.update("delete form account where id = ?", 5);
	//查所有 BeanPropertyRowMapper将sql查询出的数据绑定到Account对象上
	List<Account> accounts = jt.query("select * from account where money > ?",
			new BeanPropertyRowMapper<>(Account.class),1000);
	for(Account account : accounts){
		System.out.println(account);
	}
	//查一个
	accounts = jt.query("select * from account where id = ?",
			new BeanPropertyRowMapper<>(Account.class),1);
	System.out.println(accounts.isEmpty()?"没有数据":accounts.get(0));
	//查询返回一行一列：聚合函数的使用
	//queryForObject是Spring 3.x之后的新方法，在spring2.x的时候是queryForInt、queryForLong、queryForShort
	Integer count1 = jt.queryForObject("select count(*) from account where money > ?", Integer.class, 2000);
	Long count2 = jt.queryForObject("select count(*) from account where money > ?", Long.class, 2000);
	System.out.println(count1);
	System.out.println(count2);
}
```

### JdbcTemplate在Dao中使用

```Java
public interface IAccountDao {
    Account fundAccountById(Integer id);
    List<Account> fundAccountByName(String name);
    void updateAccount(Account account);
}

public class AccountDao implements IAccountDao {
    private JdbcTemplate jdbcTemplate;
    public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
	
    @Override
    public Account fundAccountById(Integer id) {
        List<Account> list = jdbcTemplate.query("select * from account where id = ?",
                new BeanPropertyRowMapper(Account.class), id);
        return list.isEmpty()?null:list.get(0);
    }
    @Override
    public List<Account> fundAccountByName(String name) {
        return jdbcTemplate.query("select * from account where name = ?",
                new BeanPropertyRowMapper(Account.class), name);
    }
    @Override
    public void updateAccount(Account account) {
        jdbcTemplate.update("update account set name=?, money=? where id=?",
                account.getName(), account.getMoney(), account.getId());
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">

    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/openxu"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>

    <bean id="accountDao" class="com.openxu.jdbc.dao.impl.AccountDao">
        <property name="jdbcTemplate" ref="jdbcTemplate"></property>
    </bean>
</beans>
```

```Java
public static void main(String[] args) {
	ApplicationContext ac = new ClassPathXmlApplicationContext("jdbcbean.xml");
	IAccountDao dao = (IAccountDao) ac.getBean("accountDao");
	//根据id查询
	Account account = dao.fundAccountById(1);
	System.out.println("根据id查询："+account);
	account.setMoney(7897f);
	//修改
	dao.updateAccount(account);
	//根据名称查询
	List<Account> accountList = dao.fundAccountByName("aaa");
	System.out.println("根据名查询："+accountList);
}
```

### JdbcTemplate在Dao中使用JdbcDaoSupport

一般项目中会有很多类型的Dao，按照上面的DaoImpl中的代码，每个里面都会有JdbcTemplate对象，
以及set方法，这就产生了重复代码，SpringJdbc为我们提供了JdbcDaoSupport类，将这段代码抽取
出来，我们的Dao只需要继承该类，然后就只需要在xml中配置dataSource即可（实现课看源码）

```Java
public class AccountDao extends JdbcDaoSupport implements IAccountDao {
 /*   private JdbcTemplate jdbcTemplate;
    public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
*/
    @Override
    public Account fundAccountById(Integer id) {
        List<Account> list = getJdbcTemplate().query("select * from account where id = ?",
                new BeanPropertyRowMapper(Account.class), id);
        return list.isEmpty()?null:list.get(0);
    }
    @Override
    public List<Account> fundAccountByName(String name) {
        return getJdbcTemplate().query("select * from account where name = ?",
                new BeanPropertyRowMapper(Account.class), name);
    }
    @Override
    public void updateAccount(Account account) {
        getJdbcTemplate().update("update account set name=?, money=? where id=?",
                account.getName(), account.getMoney(), account.getId());
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">
	<!--配置Spring内置数据源 -->			
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/openxu"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>
    <bean id="accountDao" class="com.openxu.jdbc.dao.impl.AccountDao">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
</beans>
```

### 使用其他数据源

java为数据源定制了标准`javax.sql.DataSource`，不同数据源都需要实现该类，上面演示的`DriverManagerDataSource`
是Spring自带的数据源，我们也可以更换成其他数据源：

```xml
<!--配置DBCP数据源 -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
	<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
	<property name="url" value="jdbc:mysql://localhost:3306/openxu"/>
	<property name="username" value="root"/>
	<property name="password" value="root"/>
</bean>

<!--配置c3p0数据源 -->
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledataSource">
	<property name="driverClass" value="com.mysql.jdbc.Driver"/>
	<property name="jdbcUrl" value="jdbc:mysql://localhost:3306/openxu"/>
	<property name="user" value="root"/>
	<property name="password" value="root"/>
</bean>
```

## 事务

### 无事务示例

通过用户转账的例子演示没有事务控制时，转账出现转出成功，转进失败的情况

```Java
public interface IAccountDao {
    Account findAccountById(Integer id);
    Account findAccountByName(String name);
    void updateAccount(Account account);
}

public class AccountDao extends JdbcDaoSupport implements IAccountDao {
    @Override
    public Account findAccountById(Integer id) {
        List<Account> list = getJdbcTemplate().query("select * from account where id = ?",
                new BeanPropertyRowMapper(Account.class), id);
        return list.isEmpty()?null:list.get(0);
    }
    @Override
    public Account findAccountByName(String name) {
        List<Account> list = getJdbcTemplate().query("select * from account where name = ?",
                new BeanPropertyRowMapper(Account.class), name);
        if(list.isEmpty())
            return null;
        if(list.size()>1)
            throw new RuntimeException("账户查询出多个人");
        return list.get(0);
    }
    @Override
    public void updateAccount(Account account) {
        getJdbcTemplate().update("update account set name=?, money=? where id=?",
                account.getName(), account.getMoney(), account.getId());
    }
}
```

```Java
public interface IAccountService {
    Account findAccountById(Integer id);
    void transfer(String sourceName, String targetName, Float money);
}

public class AccountService implements IAccountService {
    private IAccountDao accountDao;
    public void setAccountDao(IAccountDao accountDao) {
        this.accountDao = accountDao;
    }
    @Override
    public Account findAccountById(Integer id) {
        return accountDao.findAccountById(id);
    }
    @Override
    public void transfer(String sourceName, String targetName, Float money) {
        //根据名称查询账户信息
        Account source = accountDao.findAccountByName(sourceName);
        Account target = accountDao.findAccountByName(targetName);
        //转账
        source.setMoney(source.getMoney()-money);
        target.setMoney(target.getMoney()+money);
        //更新
        accountDao.updateAccount(source);
        int i = 1/0;
        accountDao.updateAccount(target);
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">

    <bean id="accountService" class="com.openxu.jdbc.service.impl.AccountService">
        <property name="accountDao" ref="accountDao"></property>
    </bean>
    <!--为dao配置datasource，JdbcDaoSupport会自动注入JdbcTemplate-->
    <bean id="accountDao" class="com.openxu.jdbc.dao.impl.AccountDao">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/openxu"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>
</beans>
```

```Java
public static void main(String[] args) {
	ApplicationContext ac = new ClassPathXmlApplicationContext("txbean.xml");
	IAccountService service = (IAccountService) ac.getBean("accountService");
	Account account = service.findAccountById(1);
	System.out.println(account);
	//转账转一半，转出了没转进，因为没有事务控制
	service.transfer("aaa", "bbb", 1000f);
}
```

### Spring基于xml声明事务控制

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd
                http://www.springframework.org/schema/tx
                http://www.springframework.org/schema/tx/spring-tx.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">

    <bean id="accountService" class="com.openxu.jdbc.service.impl.AccountService">
        <property name="accountDao" ref="accountDao"></property>
    </bean>
    <!--为dao配置datasource，JdbcDaoSupport会自动注入JdbcTemplate-->
    <bean id="accountDao" class="com.openxu.jdbc.dao.impl.AccountDao">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/openxu"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>

    <!--1. Spring基于xml的事务声明-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"></property> <!--注入数据源-->
    </bean>
    <!--2. 配置事务的通知-->
    <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <!--4.配置事务属性
              isolation:配置事务的隔离级别。默认值：DEFAULT，使用数据库的默认隔离级别。Mysql是REPEATABEL_READ
              propagation:配置事务的传播行为，默认值：REQUIRED，一般的选择（增删改方法）。查询方法选择SUPPORTS
              timeout：指定事务的超时时间，默认值是-1，永不超时，当制定其他值时以秒为单位
              read-only:配置事务是否只读事务，默认false，读写型事务。 当为true时表示只读，只能用于查询方法
              rollback-for:用于指定一个异常，当执行产生该异常时，事务回滚。产生其他异常时，事务不回滚。没有默认值，任何异常都回滚
              no-rollback-for:用于指定一个异常，当执行产生该异常时，事务不回滚。产生其他异常时回滚。没有默认值，任何异常都回滚-->
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" read-only="false"/>
            <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        </tx:attributes>
    </tx:advice>
    <!--3. 配置aop-->
    <aop:config>
        <!--配置切入点表达式-->
        <aop:pointcut id="pt1" expression="execution(* com.openxu.jdbc.service.impl.*.*(..))"/>
        <!--配置事务通知和切入点表达式的关联-->
        <aop:advisor advice-ref="txAdvice" pointcut-ref="pt1"/>
    </aop:config>
</beans>
```

### Spring基于xml和注解组合的配置步骤

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd
                http://www.springframework.org/schema/tx
                http://www.springframework.org/schema/tx/spring-tx.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">
	...
	
    <!--Spring基于xml和注解组合的配置步骤-->
    <!--1. 配置事务管理器-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"></property> <!--注入数据源-->
    </bean>
    <!--2. 配置spring开启注解事务的支持-->
    <tx:annotation-driven transaction-manager="transactionManager"/>
    <!--3. 在需要事务的地方使用@Teansactional注解
           该注解可以写在接口上、类上、方法上
           写在接口上表示该接口所有实现类都有事务
           写在类上表示该类中所有方法都有事务
           写在方法上表示该方法有事务
           优先级：就近原则-->
</beans>
```

```Java
@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
public class AccountService implements IAccountService {
    private IAccountDao accountDao;
    public void setAccountDao(IAccountDao accountDao) {
        this.accountDao = accountDao;
    }
    @Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
    @Override
    public Account findAccountById(Integer id) {
        return accountDao.findAccountById(id);
    }
    @Override
    public void transfer(String sourceName, String targetName, Float money) {
        //根据名称查询账户信息
        Account source = accountDao.findAccountByName(sourceName);
        Account target = accountDao.findAccountByName(targetName);
        //转账
        source.setMoney(source.getMoney()-money);
        target.setMoney(target.getMoney()+money);
        //更新
        accountDao.updateAccount(source);
        int i = 1/0;
        accountDao.updateAccount(target);
    }
}
```

### Spring全注解事务配置

```Java
@Repository("accountDao")
public class AccountDao implements IAccountDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Override
    public Account findAccountById(Integer id) {
        List<Account> list = jdbcTemplate.query("select * from account where id = ?",
                new BeanPropertyRowMapper(Account.class), id);
        return list.isEmpty()?null:list.get(0);
    }
    @Override
    public Account findAccountByName(String name) {
        List<Account> list = jdbcTemplate.query("select * from account where name = ?",
                new BeanPropertyRowMapper(Account.class), name);
        if(list.isEmpty())
            return null;
        if(list.size()>1)
            throw new RuntimeException("账户查询出多个人");
        return list.get(0);
    }
    @Override
    public void updateAccount(Account account) {
        jdbcTemplate.update("update account set name=?, money=? where id=?",
                account.getName(), account.getMoney(), account.getId());
    }
}

@Service("accountService")
@Transactional(propagation = Propagation.REQUIRED, readOnly = false)
public class AccountService implements IAccountService {
    @Autowired
    private IAccountDao accountDao;

    @Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
    @Override
    public Account findAccountById(Integer id) {
        return accountDao.findAccountById(id);
    }

    @Override
    public void transfer(String sourceName, String targetName, Float money) {
        //根据名称查询账户信息
        Account source = accountDao.findAccountByName(sourceName);
        Account target = accountDao.findAccountByName(targetName);
        //转账
        source.setMoney(source.getMoney()-money);
        target.setMoney(target.getMoney()+money);
        //更新
        accountDao.updateAccount(source);
        int i = 1/0;
        accountDao.updateAccount(target);
    }
}

public class JdbcConfig {
    @Bean(name="jdbcTemplate")
    public JdbcTemplate createJdbcTemplet(DataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
    @Bean(name="dataSource")
    public DataSource createDataSource(){
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName("com.mysql.jdbc.Driver");
        ds.setUrl("jdbc:mysql://localhost:3306/openxu");
        ds.setUsername("root");
        ds.setPassword("root");
        return ds;
    }
}
public class TransactionManager {
    @Bean("transactionManager")
    public PlatformTransactionManager createTransactionManager(DataSource dataSource){
        return new DataSourceTransactionManager(dataSource);
    }
}
@Configuration
@ComponentScan("com.openxu.jdbc")
@Import({JdbcConfig.class, TransactionManager.class})   //导入其他配置类
@EnableTransactionManagement  //开启纯注解事务
public class SpringConfig {
}

public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfig.class);
        IAccountService service = (IAccountService) ac.getBean("accountService");
        Account account = service.findAccountById(1);
        System.out.println(account);
        //转账转一半，转出了没转进，因为没有事务控制
        service.transfer("aaa", "bbb", 1000f);
    }
```




