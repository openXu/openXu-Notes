# SpringBoot简介

Spring通过IoC(控制反转：通过依赖注入实现)和AOP（面向切面编程）等技术，方便我们
管理和整合众多优秀的框架，让我们将更多的精力放在业务开放上。但是Spring的配置还是
有些繁琐的，在Spring特性配置和我们业务问题之间需要进行思维切换，配置占据了我们的
不少开发时间。除此之外，项目的依赖管理也是吃力不讨好的事，决定项目里需要那些库就已
经够头痛了，你还要知道这些库的哪个版本和其他库会不会有冲突。依赖管理也是一种损耗，
添加依赖一旦选错了版本，随之而来的不兼容问题毫无疑问是生产力杀手。

SpringBoot让这一切成为了过去。

SpringBoot是独立于Spring Framework之外的一个的独立框架，简化了基于Spring的应用开发，
只需要`run`就能创建一个独立的、生产级别的Spring应用。Spring Boot为Spring平台及第三方
库提供**开箱即用**的设置（提供默认设置），这样我们就可以简单的开始，多数Spring Boot应用只需
很少的Spring配置。

我们可以使用SpringBoot创建java应用，并使用java -jar启动它，或采用传统war部署方式。

SpringBoot主要目标：
- 为所有Spring的开发提供一个从根本上更快的入门体验
- 开箱即用，但通过自己设置参数，既可快速摆脱这种方式
- 提供一些大型项目中常见的非功能性特性，如内嵌服务器、安全、指标，健康检测、外部化配置等
- 绝对没有代码生成，也无需xml配置

# SpringBoot入门

[Spring Boot Guide](https://spring.io/guides/gs/spring-boot/)

**1. 创建Maven工程，导入依赖**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.springframework</groupId>
    <artifactId>gs-spring-boot</artifactId>
    <version>0.1.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**2. 创建启动类**

```Java
import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
/*
 @SpringBootApplication ：表明当前类是SpringBoot的引导类，是项目启动注解，它是一个组合注解，包含
            @EnableAutoConfiguration：启用自动配置
            @ComponentScan：指定Spring需要扫描的包（默认为当前注解类所在包及子包）
            @SpringBootConfiguration：相当于Spring中的Configuration，表示该类为Spring的配置类
            如果我们使用这三个注项目依旧可以启动起来，只是过于烦琐，@SpringBootApplication进行简化
 */
@SpringBootApplication
public class SpringBootTest {

    public static void main(String[] args) {
        //应用程序开始运行的方法(默认启动方式)
        SpringApplication.run(SpringBootTest.class, args);

        //改变启动模式（去掉启动输出图标）
        //SpringApplication application = new SpringApplication(SpringBootTest.class);
        //application.setBannerMode(Banner.Mode.OFF);
        //application.run(args);
    }
}
```

**3. 基于SpringBoot的SpringMVC配置**

```Java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
/*
 @RestController：表示该类为控制器，意味着Spring MVC已准备好使用该类来处理Web请求
            相当于Spring中的两个注解组合：
            @Controller：当前类为表现层，往Spring容器中注入一个接口对象
            @ResponseBody：返回内容而不是网页
 */
@RestController
public class HelloController {
    /*
      http://127.0.0.1:8080/hello
      @RequestMapping：（支持类型级别和方法级别）将Web请求映射请求处理类中的方法
            value属性：为此映射指定一个名称
     */
    @RequestMapping("/hello")
    public String index() {
        return "Hello Spring Boot!";
    }
}
```

# 整合spring-data-jpa

JPA(Java Persistence API Java持久性API)是Sun官方提出的Java持久化规范，JPA的主要实现有Hibernate、EclipseLink、OpenJPA等。
Spring Data JPA是Spring Data的一个子项目，通过提供基于JPA的Respository极大的减少了JPA作为数据访问方案
的代码量，通过Spring Data JPA框架，开发者可以省略实现持久层业务逻辑的工作，唯一要做的就是声明持久层的接口，
其他的都交给他来完成。

示例：使用Spring Boot + Spring MVC + Spring Data JPA + EasyUI框架组合实现数据库表查询及网页展示

**1. 导入依赖**

```xml
<!--JPA
	★★★下载失败（包红线）有可能是因为jpa所依赖的某些包没能下载下来，可能下载超时，可以在maven的
	setting.xml中添加阿里云镜像，然后到maven本地仓库找到对应报错的jar包，将其删除，然后重新下载
-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!--mysql连接驱动-->
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
```

**2. 创建表**

```xml
create table account(
	id int primary key auto_increment,
	name varchar(40),
	money float
)character set utf8 collate utf8_general_ci;

insert into account(name, money) values ('openxu', 1000);
insert into account(name, money) values ('Jonsthan Lee', 500000);
insert into account(name, money) values ('Sandy', 1000000);
```

**3. 创建业务层、持久层、表现层**

```Java
//实体类
@Entity
@Table(name="account")
@Data
public class Account {
    @Id
    @Column(name="id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column(name = "name") //windows系统下mysql不区分大小写，Linux下严格区分
    private String name;
    @Column(name = "money")
    private String money;
}


/**业务层*/
public interface IAccountService {
    List<Account> findAll();
}
@Service("accountService")
public class AccountServiceImpl implements IAccountService {
    @Autowired
    private IAccountDao accountDao;
    @Override
    public List<Account> findAll() {
        return accountDao.findAll();
    }
}

/**
 * 持久层
 * Spring Data JPA的顶层接口Repository，它的子类有:
 *      CrudRepository: 提供了基本的增删改查等接口
 *          PagingAndSortingRepository：提供了基本的分页和排序等接口
 * JpaRepository继承了他们，所以在项目中我们都是通过实现JpaRepository或者其子类进行基本的数据库操作。
 *
 * Spring Data JPA 为我们约定了 系列规范，只要按照规范编写代码， Spring Data JPA 就会根据代码翻译成相关的 SQL语句，
 * 进行数据库查询，比如可以使用findBy、Like、In等关键字，其中 findBy可以用 read、readBy、query、queryBy、get、getBy 来代替。
 * 关于查询关键字的更多内容，可以到官方网站：
 * https://docs.spring.io/spring-data/data-jpa/docs/current/reference/html/#reference
 */
@Repository("accountDao")
public interface IAccountDao extends JpaRepository<Account, Integer> {
    /**findBy关键字：根据字段查询*/
    List<Account> findByName(String name);
    /**Like关键字：根据字段查询模糊查询*/
    List<Account> findByNameLike(String name);
    /**In关键字：通过id集合查询*/
    List<Account> findByIdIn(Collection<Integer> ids);
}

/**表现层*/
@RestController
@RequestMapping("/account")
public class AccountController {
    @Autowired
    private IAccountService accountService;
	//http://127.0.0.1:8080/account/list
    @RequestMapping("/list")
    public List<Account> findAllAccount(){
        List<Account> accounts = accountService.findAll();
        return accounts;
    }
}
```

**4. JDBC配置**

```xml
#DB Configuration:
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/openxu
spring.datasource.username=root
spring.datasource.password=root

#JPA Configuration
spring.jpa.database=MySQL
spring.jpa.show-sql=true
spring.jpa.generate-ddl=true
spring.jpa.hibernate.ddl-auto=update
```

**5. EasyUI**

[Jquery EasyUI下载](http://www.jeasyui.net/download/jquery.html)，解压后放在`resources/static/`下,
然后创建一个`account.html`页面：

```html
<!DOCTYPE html>
<html lang="zh_CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>人员信息</title>
    <link rel="stylesheet" type="text/css"
          href="ui/themes/default/easyui.css">
    <link rel="stylesheet" type="text/css" href="ui/themes/icon.css">
    <script type="text/javascript" src="ui/jquery.min.js"/>
    <script type="text/javascript" src="ui/jquery.easyui.min.js"/>
    <script type="text/javascript" src="ui/locale/easyui-lang-zh_CN.js"/>
    <script type="text/javascript">
        $(function () {
            alert("哈哈哈")
            $('#grid').datagrid({
                url : 'account/list',   //当访问account.html时表示对应http://127.0.0.1:8080/account/list
                columns:[[{
                    field:"id",
                    title:"编号",
                    width:50
                },{
                    field:"name",
                    title:"姓名",
                    width:100
                },{
                    field:"money",
                    title:"存款",
                    width:100
                }]]
            });
        });
    </script>

<head/>

<body>
    <h1>人员信息</h1>
    <table id="grid"></table>
</body>
<html/>
```

# 整合MyBatis

MyBatis是一款优秀的持久层框架，支持定制化SQL、存储过程及高级映射。避免了几乎所有JDBC代码
和手动设置参数以及获取结果集。

```xml
<!--整合MyBatis-->
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>1.3.1</version>
</dependency>
```

```Java
/**
 * 使用MyBatis实现对数据库的操作接口
 * @Mapper:MyBatis 根据接口定义与 Mapper 文件中的 SQL 语句动态创建接口实现
 */
@Mapper  //要求MyBatis版本3.3以上才能使用
public interface IAccountMapper {

    @Select("select * from account where name like '%${value}%'")
    public List<Account> queryAccountByName(String name);
}


/**业务层*/
public interface IAccountService {
    List<Account> findUserByName(String name);
}
@Service("accountService")
public class AccountServiceImpl implements IAccountService {
    @Autowired
    private IAccountMapper accountMapper;
    @Override
    public List<Account> findUserByName(String name) {
        return accountMapper.queryAccountByName(name);
    }
}


@RestController
@RequestMapping("/account")
public class AccountController {
    @Autowired
    private IAccountService accountService;
    @RequestMapping("/findByName/{name}")   //http://127.0.0.1:8080/account/findByName/openXu
    public List<Account> findAllAccount(@PathVariable("name") String name){
        List<Account> accounts = accountService.findUserByName(name);
        return accounts;
    }
}
```

# 整合Redis

Redis是一个基于内存的单线程高性能key-value型数据库，独写性能优异。与Memcached缓存相比，Redis支持丰富的数据类型，
包括string(字符串)、list(链表)、set(集合)、zset(sorted set有序集合)和hash(哈希类型)。因此，Redis在企业中被广泛使用。

## Redis服务器安装
Redis项目本身不支持windows，但是Microsoft开放技术小组开发和维护这个windows端口（针对Win64），所以我们可以在网络上下载Redis的
windows版本。

[Redis官网](https://redis.io/download)

**1. 下载安装**

[Redis for Windows](https://github.com/MicrosoftArchive/redis/releases)，点击release，可看到很多版本的安装包，选择
最新正式版`Redis-x64-3.0.504.zip`。

下载完成后解压，打开`redis-server.exe`启动Redis服务器，然后找到Redis客户端程序`redis-cli.exe`打开。

**2. Redis使用命令缓存测试**

使用Redis客户端对Redis的集中数据类型做基本的增删改查操作练习（请查看[Redis操作](G:/openXu/Blog/blog_javaee/03-02-Redis操作.md)）

## SpringBoot集成Redis

Spring Boot提供了强大的基于注解的缓存支持，可以低侵入的给原有Spring应用增加缓存功能，提高数据访问性能，我们可以根据具体的项目要求使用相应的缓存技术。

Spring Boot支持许多类型的缓存，比如EhCache、JCache、Redis等，在不添加任何额外配置的情况下，Spring Boot默认使用SimpleCacheConfiguration。

**1. 引入依赖**
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**2. application.properties配置**

```xml
### redis 缓存配置
###默认 redis 数据库为 db0
spring.redis.database=0
###服务器地址，默认为 localhost
spring.redis.host=localhost
###链接端口，默认为 6379
spring.redis.port=6379 
### redis 密码默认为空
spring.redis.password=
```

**3. 测试**

```Java
@Test
@Resource
private RedisTemplate redisTemplate;
@Resource
private StringRedisTemplate stringRedisTemplate;
private static final String ALL_USER = "ALL_USER_LIST";

public void redisTest(){
	//增：key:name   value:openxu
	redisTemplate.opsForValue().set("name", "openxu");
	String name = (String)redisTemplate.opsForValue().get("name");
	System.out.println("name="+name);
	//删除
	redisTemplate.delete("name");
	//更新
	redisTemplate.opsForValue().set("name", "sandy");
	//查询
	name = stringRedisTemplate.opsForValue().get("name");
	System.out.println("name="+name);
	
	/*
	  RedisTemplate和StringRedisTemplate出了提供opsForValue()方法用来操作简单
	  属性数据之外，还提供以下数据访问方法：
	  opsForList()  操作含有list的数据
	  opsForSet()   操作含有set的数据
	  opsForZSet()  操作含有ZSet(有序set)的数据
	  opsForHash()  操作含有hash的数据

	  数据存放到Redis，key和value都是通过Spring提供的Serializer序列化到数据库的。
	  RedisTemplate默认使用JdkSerializationRedisSerializer，StringRedisTemplate
	  默认使用StringRedisSerializer
	 */
	  //查询所有用户
		List<Account> list = accountService.findAll();
		for(Account account : list){
			log.info("从数据库查询 : "+account);
		}
		//清除缓存中的用户数据
		redisTemplate.delete(ALL_USER);
		//将用户数据存放到Redis缓存中。leftPushAll()查询缓存中所有用户数据，若键不存在会创建键及于其关联的List，之后在将参数中的集合从左到右依次插入
		redisTemplate.opsForList().leftPushAll(ALL_USER, list);
		//从redis中查询所有用户数据，range()去链表中所有元素，0表示第一个元素，-1表示最后一个元素
		list = redisTemplate.opsForList().range(ALL_USER, 0 , -1);
		for(Account account : list){
			log.info("从redis查询 : "+account);
		}
}
```

**4. Spring Boot MVC整合Redis**

```Java
@SpringBootApplication
//★1、开启Spring对缓存的支持
@EnableCaching   
public class SpringBootTest {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootTest.class, args);
    }
}


//Dao
/**使用MyBatis实现对数据库的操作接口*/
@Mapper  //要求MyBatis版本3.3以上才能使用
public interface IAccountMapper {
    @Select("select * from account")
    public List<Account> queryAll();
}

//Service
@Service("accountService")
public class AccountServiceImpl implements IAccountService {
    @Autowired
    private IAccountMapper accountMapper;
    @Override
    /*
      ★2、@Cacheable表示当前方法使用缓存，并存入redis数据库中
        key：指定方法执行返回值的key，该属性是Spring用的，不写也有默认值
        value:表示存入redis数据库的key
     */
    @Cacheable(value="findAllCache", key="'account.findAll'")
    public List<Account> findAll() {
        //访问接口，发现只有第一次回去数据库中查询
        System.out.println("执行去数据库查询");
        return accountMapper.queryAll();
    }
}

//Controller
@RestController
@RequestMapping("/account")
public class AccountController {
    @Autowired
    private IAccountService accountService;

    @RequestMapping("/findAll")   //http://127.0.0.1:8080/account/findAll
    public List<Account> findAllAccount(){
        List<Account> accounts = accountService.findAll();
        return accounts;
    }
}

//启动程序后，浏览器访问http://127.0.0.1:8080/account/findAll
//发现"执行去数据库查询"只会打印一次

```

# Spring Boot整合





















