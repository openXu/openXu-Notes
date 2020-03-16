## IOC的概念和实现原理

控制反转（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计原则，
可以用来减低计算机代码之间的耦合度。其中最常见的方式叫做**依赖注入**（Dependency Injection，简称DI），
还有一种方式叫**依赖查找**（Dependency Lookup）。通过控制反转，对象在被创建的时候，
由一个调控系统内所有对象的外界实体将其所依赖的对象的引用传递给它。也可以说，依赖被注入到对象中。

**技术描述**

Class A中用到了Class B的对象b，一般情况下，需要在A的代码中显式的new一个B的对象。

采用依赖注入技术之后，A的代码只需要定义一个私有的B对象，不需要直接new来获得这个对象，
而是通过相关的容器控制程序来将B对象在外部new出来并注入到A类里的引用中。
而具体获取的方法、对象被获取时的状态由配置文件（如XML）来指定。

## Spring体验
开发工具为IntelliJ IDEA，使用maven构建。

- maven依赖配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.openxu.spring</groupId>
    <artifactId>SpringTest</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <spring.version>4.3.18.RELEASE</spring.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-core -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/commons-logging/commons-logging -->
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.1.1</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/log4j/log4j -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>

    </dependencies>

    <build>
        <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
            <plugins>
                <plugin>
                    <artifactId>maven-clean-plugin</artifactId>
                    <version>3.0.0</version>
                </plugin>
                <!-- see http://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_jar_packaging -->
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>3.0.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.7.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>2.20.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-jar-plugin</artifactId>
                    <version>3.0.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-install-plugin</artifactId>
                    <version>2.5.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-deploy-plugin</artifactId>
                    <version>2.8.2</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

- bean.xml配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="customerService" class="com.openxu.service.CustomerServiceImpl"/>
    <bean id="customerDao" class="com.openxu.dao.CustomerDaoImpl"/>
</beans>
```

- 测试
```Java
public static void main(String[] args) {
	/*
	 * ClassPathXmlApplicationContext 只能加载类路径下的配置文件（用这个）
	 * FileSystemXmlApplicationContext 可以加载磁盘任意位置的配置文件
	 *
	 * Bean创建的两种规则：
	 * BeanFactory: 延迟加载的思想创建bean对象，bean对象使用的时候才创建
	 * ApplicationContext: 立即加载的思想创建bean对象，只要一解析完配置文件，bean对象就创建了
	 */
	//1、ApplicationContext预加载
	//获取容器
	//ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
	//System.out.println("获取对象");
	//根据bean的key获取对象
	//ICustomerService cs = (ICustomerService) ac.getBean("customerService");
	//ICustomerDao cd = (ICustomerDao) ac.getBean("customerDao");
	//cs.saveCustomer();
	//cd.saveCustomer();

	//2、BeanFactory懒加载
	BeanFactory factory = new XmlBeanFactory(new ClassPathResource("bean.xml"));
	System.out.println("获取对象");
	ICustomerService cs = (ICustomerService) factory.getBean("customerService");
	ICustomerDao cd = (ICustomerDao) factory.getBean("customerDao");
	cs.saveCustomer();
	cd.saveCustomer();
}
```

## Spring IOC

### Bean的三种创建方式

- (开发中这种方式最常用)委托spring容器创建，将调用默认无参构造方法创建对象，如果没有无参构造函数则创建失败抛异常

- 使用静态工厂方法创建对象，需要配置`factory-method`属性指定静态工厂中创建对象的方法

- 使用实例工厂中的方法创建，需要配置工厂实例对象，然后配置`factory-method`属性

```xml
 <!--方式1：spring创建对象，默认调用class对应类的无参构造函数创建对象-->
<bean id="customerService" class="com.openxu.service.CustomerServiceImpl"/>
<bean id="customerDao" class="com.openxu.dao.CustomerDaoImpl"/>

<!--方式2：使用静态工厂创建bean对象，需要配置具体的创建方法-->
<bean id="staticCustomerService" class="com.openxu.factory.StaticFactory" factory-method="getCustomerService"/>

<!--方式3：使用实例工厂创建bean对象，需要配置工厂实例，还需要配置创建方法-->
<bean id="instanceFactory" class="com.openxu.factory.InstanceFactory"/>
<bean id="instanceCustomerService" factory-bean="instanceFactory" factory-method="getCustomerService"/>
```

### Bean生命周期

**Bean的作用范围**

可以通过bean标签的**scope**属性配置bean对象的作用范围，该属性取值：
- singleton:单例（默认值）
- prototype:多例（当我们让spring接管struts2的action创建时，action必须配置此值）
- request:作用范围是一次请求，和当前请求的转发
- session:作用范围是一次回话
- globalsession:作用范围是一次全局会话

**Bean的生命周期**

通过bean标签的**init-method**、**destroy-method**属性监听对象的创建和销毁

- singleton单例
> `ApplicationContext`容器创建对象就被创建了，只要容器在，对象就一直存在，直到容器销毁，对象就消亡

- prototype多例
> 每次使用时创建对象，只要对象在使用就会存活，当长时间不用且没有被引用，由java垃圾回收机制回收

### Spring的依赖注入IOC

spring有三种依赖注入方式：

**1. 使用构造函数注入**
```xml
<!--构造函数注入（constructor-arg标签）:
		type:指定参数类型
		index:指定参数索引位置，从0开始
		name:指定参数名称
		value:指定基本数据类型或String类型的数据
		ref:指定其他bean类型数据-->
<bean id="customerService" class="com.openxu.service.CustomerServiceImpl">
	<!--String类型-->
	<constructor-arg name="driver" value="com.mysql.jdbc.Driver"></constructor-arg>
	<!--Integer类型-->
	<constructor-arg name="port" value="3306"></constructor-arg>
	<!--Date类型-->
	<constructor-arg name="today" ref="now"></constructor-arg>
</bean>
<bean id="now" class="java.util.Date"></bean>
```

**2. set方法注入**
```xml
<!--set方法注入（property标签）:
		name:指定参数set方法名称
		value:指定基本数据类型或String类型的数据
		ref:指定其他bean类型数据-->
<bean id="customerService" class="com.openxu.service.CustomerServiceImpl">
	<!--String类型-->
	<property name="driver" value="com.mysql.jdbc.Driver"/>
	<!--Integer类型-->
	<property name="port" value="3306"/>
	<!--Date类型-->
	<property name="today" ref="now"/>
</bean>
<bean id="now" class="java.util.Date"></bean>
```

**3. set复杂数据类型注入**
```xml
<!--复杂数据类型注入-->
<bean id="customerService" class="com.openxu.service.CustomerServiceImpl">
	<!--结构相同，标签可以互换array、list、set-->
	<property name="myStr">  <!--String[]-->
		<array>
			<value>AAA</value>
			<value>BBB</value>
		</array>
	</property>
	<property name="myList">  <!--List<String>-->
		<list>
			<value>AAA</value>
			<value>BBB</value>
		</list>
	</property>
	<property name="mySet">  <!--Set<String>-->
		<set>
			<value>AAA</value>
			<value>BBB</value>
		</set>
	</property>
	<!--结构相同，标签可以互换map-->
	<property name="myMap">  <!--Map<String,String>-->
		<map>
			<entry key="A" value="AAA"></entry>
			<entry key="B">
				<value>BBB</value>
			</entry>
		</map>
	</property>
	<property name="myProps">  <!--Properties-->
		<map>
			<entry key="A" value="AAA"></entry>
			<entry key="B" value="BBB"></entry>
		</map>
	</property>
</bean>
```

### Spring IOC注解

Spring依赖注入可以通过xml配置的方式实现，这种方式很麻烦。然后就出现了注解的实现方式。

**基本使用**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans  xmlns="http://www.springframework.org/schema/beans"
        xmlns:context="http://www.springframework.org/schema/context"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd">
    <!--告知Spring在创创建容器时要扫描的包，当配置此标签后，spring回去指定的包及子包下找对应的注解
        标签是在一个context的命名空间里，所以必须导入context名称空间-->
    <context:component-scan base-package="com.openxu"></context:component-scan>
    <!--再也不需要配置bean标签了 <bean id="customerDao" class="com.openxu.dao.CustomerDaoImpl"/>-->
</beans>
```

```Java
/**
 * 1、用于创建bean对象的注解
 *    @Component ：定义在类上，相当于配置了一个bean标签
 *              value属性：指定bean的id，不写默认值为当前类的短名首字母小写
 *    该注解衍生的三个注解：
 *      @Controller 一般用于表现层的注解
 *      @Service 一般用于业务层
 *      @Repository 一般用于持久层
 *    这三个注解和@Component作用和属性一模一样，三个注解继承Component
 */
@Component
public class CustomerServiceImpl1 implements ICustomerService {
    public void saveCustomer() {
    }
}
```

```Java
//通过注解获取bean对象
ApplicationContext ac = new ClassPathXmlApplicationContext("bean1.xml");
ICustomerService cd = (ICustomerService) ac.getBean("customerServiceImpl1");
cd.saveCustomer();
```

**常用注解**
```Java
/**
 * 1、用于创建bean对象的注解
 *    @Component ：定义在类上，相当于配置了一个bean标签
 *              value属性：指定bean的id，不写默认值为当前类的短名首字母小写
 *    该注解衍生的三个注解：
 *      @Controller 一般用于表现层的注解
 *      @Service 一般用于业务层
 *      @Repository 一般用于持久层
 *    这三个注解和@Component作用和属性一模一样，三个注解继承Component
 *
 * 2、用于注入数据
 *    @Autowired :自动按照类型注入.只要有唯一的类型匹配就能注入成功
 *                当注入的bean在容器中类型不唯一时,它会吧变量名称作为bean的id,在容器中查找对应的对象
 *                当没有找到一致的bean的id,就报错
 *                当我们使用注解注入时,set方法就不是必须的了
 *    @Qualifier :在自动按照注解类型注入的基础上,再按照bean的id注入
 *                它在给类成员注入数据时,不能独立使用,需要同时加上 @Autowired
 *                但是在给方法的形参注入数据时,可以单独使用
 *                value属性:用于指定bean的id
 *    @Resource :直接按照bean的id注入
 *                name属性:用于指定bean的id
 *    以上三个注解用于注入其他bean类型的,基本类型需要使用Value注解,集合等复杂类型不能使用注解注入
 *    @Value :用于注入基本类型和String类型,它可以借助spring的el表达式读取properties文件中的配置
 *              value属性:用于指定要注入的数据  @Value("openXu")
 *
 * 3、用于改变作用范围
 *    @Scope :改变bean的作用范围
 *            value属性 : 用于指定范围的取值,取值和xml中scope属性取值一样
 *                       singleton:（默认值单例）prototype:(多例)  request:(一次请求)
 *                       session:(一次会话) globalsession:(一次全局会话)
 *
 * 4、声明周期
 *    @PostConstruct    @PostDestroy
 */

@Component
public class CustomerServiceImpl1 implements ICustomerService {

    @Autowired
//    @Qualifier("customerDaoImpl1")
//    @Resource(name="customerDaoImpl1")
    private ICustomerDao customerDao;

    public void saveCustomer() {
        customerDao.saveCustomer();
    }
}
```

**新注解**

上面的示例中，我们还需要用到xml配置Spring扫描包，和一些jdbc的bean。下面的一些注解课帮助我们彻底摆脱xml。

- @Configuration: 将当前类看成是spring的配置类（可用可不用）
- @ComponentScan：告诉spring需要扫描的文件夹@ComponentScan("com.openxu")
- @Import: 导入其他配置类@Import(JdbcConfig.class)
- @PropertySource：导入配置文件，可以通过el表达式加载配置文件中的数据@PropertySource("classpath:config/jsbcConfig.properties")
- @Bean :向spring容器中存入该方法的返回值bean
- @Qualifier("xx"): 为形参指定需要id为xx的对象



```Java
1、jdbc配置文件jsbcConfig.properties
jdbc.driver = com.mysql.jdbc.Driver
jdbc.url = jdbc:mysql://localhost:3306/openxu
jdbc.user = root
jdbc.password = root

2、JdbcConfig
/**
 * @Bean :向spring容器中存入该方法的返回值bean，
 *        name属性：指定bean的id，当不指定时默认值为方法名称
 */
public class JdbcConfig {
    //通过"${xxx}"注入jsbcConfig.properties配置文件中的数据
    @Value("${jdbc.driver}")
    private String driver;
    @Value("${jdbc.url}")
    private String jdbcurl;
    @Value("${jdbc.user}")
    private String user;
    @Value("${jdbc.password}")
    private String password;

    /**
     * @param dataSource 通过类型匹配直接在容器中找到DataSource的对象，
     *                   如果有两个方法注入了DataSource对象（Spring容器中有多个DataSource对象），
     *                   需要通过@Qualifier("xx")指定需要id为xx的对象
     * @return
     */
    @Bean(name="runner")
    public QueryRunner createQueryRunner(@Qualifier("ds1")DataSource dataSource){
        return new QueryRunner(dataSource);
    }
    @Bean(name="ds1")
    public DataSource createDataSource(){
        try{
            ComboPooledDataSource ds = new ComboPooledDataSource();
            ds.setDriverClass(driver);
            ds.setJdbcUrl(jdbcurl);
            ds.setUser(user);
            ds.setPassword(password);
            return ds;
        }catch (Exception e){
            throw new RuntimeException(e);
        }
    }
    @Bean(name="ds2")
    public DataSource createDataSource(){
        try{
            ComboPooledDataSource ds = new ComboPooledDataSource();
            ds.setDriverClass(driver);
            ds.setJdbcUrl(jdbcurl);
            ds.setUser(user);
            ds.setPassword(password);
            return ds;
        }catch (Exception e){
            throw new RuntimeException(e);
        }
    }
}

3、SpringConfiguration
/**
 * @Configuration: 将当前类看成是spring的配置类（可用可不用）
 *
 * @ComponentScan：告诉spring需要扫描的文件夹，相当于在xml中配置。value属性接受一个String数组{"", ""}
 *          这个注解随便放在那个类上都行，使用AnnotationConfigApplicationContext加载
 *      <context:component-scan base-package="com.openxu"></context:component-scan>
 *
 * @Import: 导入其他配置类，比如JdbcConfig中两个方法是为了给spring容器注入bean的，完全托管给Spring就行，
 *           为JdbcConfig加@Component注解有不合适（不需要JdbcConfig对象），所以使用@Import导入为配置类
 *
 * @PropertySource：导入配置文件，可以通过el表达式加载配置文件中的数据，通过@Value注入
 *                  4.3.5以前的版本不支持el解析器，需要主动加载资源占位符解析器用于解析${}：
 *                  @Bean
 *                  public static PropertySourcesPlaceholderConfigurer createPropertySourcesPlaceholderConfigurer(){
 *                      return new PropertySourcesPlaceholderConfigurer();
 *                  }
 *
 */
@Configuration
@ComponentScan("com.openxu")
@Import(JdbcConfig.class)
@PropertySource("classpath:config/jsbcConfig.properties")
public class SpringConfiguration {
    //资源占位符解析器，4.3.5版本以后自带了，不需要这段代码也能解析${}
    @Bean
    public static PropertySourcesPlaceholderConfigurer createPropertySourcesPlaceholderConfigurer(){
        return new PropertySourcesPlaceholderConfigurer();
    }
}

4、AnnotationConfigApplicationContext使用
ApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
ICustomerService cd = (ICustomerService) ac.getBean("customerServiceImpl1");
cd.saveCustomer();
```
















