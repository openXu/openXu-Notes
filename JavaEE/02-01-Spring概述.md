## 一 - Spring概述

Spring是分层的Java SE/EE应用full-stack轻量级开源框架，以Ioc(Inverse Of Control控制反转)
和AOP(Aspect Oriented Programming面向切面编程)为内核，提供了展现层Spring MVC和持久层Spring
JDBC以及业务层事务管理等众多的企业级应用技术，还能整合开源世界众多著名的三方框架和类库，逐渐成为
使用最多的Java EE企业应用开源框架

[spring-framework github](https://github.com/spring-projects/spring-framework)

### Spring的发展

**1. Spring1.x**
在Spring1.x时代，都是通过xml文件配置bean，随着项目的不断扩大，
需要将xml配置分放到不同的配置文件中，需要频繁的在java类和xml配置文件中切换。

**2. Spring2.x**
随着JDK 1.5带来的注解支持，Spring2.x可以使用注解对Bean进行申明和注入，
大大的减少了xml配置文件，同时也大大简化了项目的开发。

那么，问题来了，究竟是应该使用xml还是注解呢？

最佳实践：
- 应用的基本配置用xml，比如：数据源、资源文件等；
- 业务开发用注解，比如：Service中注入bean等；

**3. Spring3.x到Spring4.x**
从Spring3.x开始提供了Java配置方式，使用Java配置方式可以更好的理解你配置的Bean，
现在我们就处于这个时代，并且Spring4.x和Spring boot都推荐使用java配置的方式。
Java配置是Spring4.x推荐的配置方式，可以完全替代xml配置。

Spring的Java配置方式是通过 @Configuration 和 @Bean 这两个注解实现的：
- @Configuration 作用于类上，相当于一个xml配置文件；
- @Bean 作用于方法上，相当于xml配置中的<bean>；

### 程序间的依赖关系

**1. 程序的耦合**

调用者和被调用者的依赖关系，我们在开发中应该遵循**编译时不依赖，运行时才依赖**的原则。
如果编译时就依赖，可能导致多人开发时调用者必须等待被调用者开发完成后才能开始编码。可以使用
**反射创建**解决此问题，但是会引发新的问题：**类的全名写死了以后修改需要改源码**。
那就通过读取配置文件获取类名创建对象。

```Java
//方式1(编译时依赖):必须等CustomerDaoImpl开发完成才能调用
ICustomerDao customerDao = new CustomerDaoImpl();

//方式2:通过配置文件及反射获取实现类对象，避免了编译时依赖，运行时才会报找不到类错误
//通过配置文件配置 实现类
ResourceBundle bundle = ResourceBundle.getBundle("bean");
//通过反射获取实现类对象
ICustomerDao customerDao = (ICustomerDao)Class.forName(bundle.getString("DAO_IMPL"));
```


**2. ResourceBundle读取配置文件**

在开发web应用时，如果没有重写`HttpServlet`的`doGet()`方法，调用接口会报`405:HTTP method GET is not supported by this URL`错误，
这个错误提示就是通过配置文件的方式获取的，其中用到了`ResourceBundle`类

需要注意的是：
- ResourceBundle只能读取`properties`文件，别的文件格式读不了
- 只暴露了读取方法，不能写入
- 只能将配置文件放在src(类路径)目录及子目录下，不在类路径读取不了

```Java
//方法1 通过ResourceBundle读取
//通过 包名.类名 的方式获取配置文件（不用加.properties后缀）
private static final ResourceBundle lStrings = ResourceBundle.getBundle("javax.servlet.http.LocalStrings");
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
	String protocol = req.getProtocol();
	//从配置文件中读取对应的错误提示
	String msg = lStrings.getString("http.method_get_not_supported");
	if (protocol.endsWith("1.1")) {
		resp.sendError(405, msg);
	} else {
		resp.sendError(400, msg);
	}
}

//方法2 通过Properties读取(ResourceBundle底层也是通过这种方式)
Properties props = new Properties();
//使用类加载器获取类路径下的配置文件
InputStream in = HelloWorld.class.getClassLoader().getResourceAsStream("javax/servlet/http/LocalStrings.propertoes");
//问题（绝对不能使用）：程序发布后src目录没有了
in = new FileInputStream("src/xx.properties");
props.load(in);
```
