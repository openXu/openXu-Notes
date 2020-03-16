## 动态代理

> 动态代理：在不改变源码的基础上对已有方法增强，它是AOP思想实现的基础

### 两种动态代理实现方式

**1、基于接口的动态代理**

```Java
public interface IActor {
    void basicAct(float money);
    void dangerAct(float money);
}

public class Actor implements IActor {
    @Override
    public void basicAct(float money){
        System.out.println("拿到钱"+money+"，开始基本表演");
    }
    @Override
    public void dangerAct(float money){
        System.out.println("拿到钱"+money+"，开始危险表演！！");
    }
}

public static void main(String[] args) {
	/**
	 * 模拟演员接活儿表演，经纪人作为代理中间拦截，抽成
	 */
	Actor actor = new Actor();
//        actor.basicAct(500f);
//        actor.dangerAct(1000f);
	/*
	  动态代理：在不改变源码的基础上对已有方法增强，它是AOP思想实现的基础
	  1、基于接口的动态代理
	  要求：被代理类最少实现一个接口
	  提供者：JDK官方
	  涉及类：Proxy
	  创建代理对象的方法：Proxy.newProxyInstance(ClassLoader, Class[], InvocationHandler)
	  参数含义：
			ClassLoader：类加载器，和被代理对象使用相同的类加载器，一般固定写法xx.
			Class[]：字节码数组，被代理类实现的接口，要求代理对象和被代理对象具有相同的行为。一般都是固定写法
			InvocationHandler：一个接口，用于我们提供增长代码的。一般都是写一个该接口的实现类，实现类可以是匿名内部类。
							   它的含义是如何代理，此处代码只能是谁用谁提供
			策略模式：数据已经有了，目的明确，达成目的的过程就是策略
	 */
	IActor proxyActor = (IActor)Proxy.newProxyInstance(actor.getClass().getClassLoader(),
			actor.getClass().getInterfaces(), new InvocationHandler() {
				/*
				 * 执行被代理对象的任何方法都会经过该方法，该方法有拦截的功能
				 * @param proxy：代理对象的引用，不一定每次都会有
				 * @param method：被代理对象当前正在执行的方法
				 * @param args：当前执行方法的参数
				 * @return ：当前执行方法的返回值
				 */
				@Override
				public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
					Object rtValue = null;
					Float money = (Float)args[0];
					if("basicAct".equals(method.getName())){
						if(money>10000){
							rtValue = method.invoke(actor, money/2);
						}
					}
					if("dangerAct".equals(method.getName())) {
						if (money > 50000) {
							rtValue = method.invoke(actor, money/2);
						}
					}
					return rtValue;
				}
			});
	proxyActor.basicAct(20000);
	proxyActor.dangerAct(60000);
}
```

**2、基于接口的动态代理**

```Java
public static void main(String[] args) {

	Actor actor = new Actor();
//        actor.basicAct(500f);
//        actor.dangerAct(1000f);
	 /*
	  2、基于子类的动态代理
	  要求：被代理类不能是final修饰
	  提供者：第三方CDLib
	  涉及类：Enhancer
	  创建代理对象的方法：create(Class, Callback)
	  参数含义：
			Class：被代理对象的字节码
			callback：它和InvocationHandler的作用是一样的，我们一般使用其子接口MethodInterceptor.
					  在使用时也是创建匿名内部类
	 */
	IActor proxyActor = (IActor) Enhancer.create(actor.getClass(), new MethodInterceptor(){
		/*
		 * 执行被代理对象的任何方法都会经过该方法，和InvocationHandler的invoke方法一样
		 * @param arg0 代理对象的引用，不一定每次都会有
		 * @param arg1 被代理对象当前正在执行的方法
		 * @param arg2 当前执行方法的参数
		 * @param arg3 当前执行方法的代理对象，一般不用
		 * @return
		 */
		@Override
		public Object intercept(Object arg0, Method arg1, Object[] arg2, MethodProxy arg3){
			Object rtValue = null;
			Float money = (Float)args[0];
			if("basicAct".equals(method.getName())){
				if(money>10000){
					rtValue = method.invoke(actor, money/2);
				}
			}
			if("dangerAct".equals(method.getName())) {
				if (money > 50000) {
					rtValue = method.invoke(actor, money/2);
				}
			}
			return rtValue;
		}
	});
	proxyActor.basicAct(20000);
	proxyActor.dangerAct(60000);
}
```

### 动态代理应用

> 三层架构模式中，view调用service，service调用dao层。service层增删改查每个方法都
> 要开启事务、提交、回滚等操作都是重复代码，怎样来实现只需要一行dao.xxx()？这时候就
> 可以写一个工具类每次返回service层的代理对象，在调用dao.xxx()前后插入事务代码

```Java
public class BeanFactory {
    public static ICustomerService getCustomerService(){
        final ICustomerService cs = new CustomerServiceImpl();
        //创建业务层的代理对象
        ICustomerService proxyCs = (ICustomerService) Proxy.newProxyInstance(
                cs.getClass().getClassLoader(), cs.getClass().getInterfaces(), new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        //事务相关重复代码使用动态代理的方式插入
                        SessionFactory sessionFactory = null;
                        Session session = null;
                        Transaction tx = null;
                        try{
                            sessionFactory = HibernateUtils.getSessionFactory();
                            session = sessionFactory.openSession();
                            tx = session.beginTransaction();
                            //被代理业务层执行（dao.xxx()）
                            Object rtValue = method.invoke(cs, args);
                            tx.commit();
                            return rtValue;
                        }catch(Exception e){
                            e.printStackTrace();
                            tx.rollback();
                        }finally{
                            session.close();
                            sessionFactory.close();
                        }
                        return null;
                    }
                }
        );
        return proxyCs;
    }
}
```

## AOP

### AOP概述

在软件业，AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，
通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，
是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。
利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，
提高程序的可重用性，同时提高了开发的效率。

简单的说就是把我们程序重复的代码抽取出来，在需要执行的时候，使用动态代理技术，在不修改源码
的基础上，对我们已有的方法进行增强。Spring AOP 模块提供拦截器来拦截一个应用程序，
例如，当执行一个方法时，你可以在方法执行之前或之后添加额外的功能。

**作用：** 在程序运行期间，不修改源码对已有方法进行增强
**优势：** 减少重复代码，提高开发效率，方便维护
**AOP实现方式：** 使用动态代理技术

### AOP术语
在我们开始使用 AOP 工作之前，让我们熟悉一下 AOP 概念和术语。
这些术语并不特定于 Spring，而是与 AOP 有关的。

|项				     |描述|		
|---			     |:--:|
|Target object目标对象|被一个或者多个方面所通知的对象，这个对象永远是一个被代理对象。也称为被通知对象。|
|Proxy代理            |一个类被AOP织入增强后就产生一个结果代理类|
|Join point连接点     |在你的应用程序中它代表一个点，说白了就是被代理对象的接口中的抽象方法|
|Pointcut切入点       |这是一组一个或多个连接点，通知应该被执行。就是需要增强的连接点（有些方法可能不需要增强）（连接点不一定是切入点，切入点一定是连接点）|
|Advice通知/增强      |拦截到Joinpoint之后所要做的事情就是通知，通知类型：前置通知、后置通知、异常通知、最终通知、环绕通知|
|Aspect切面           |切面是切入点、通知的综合|
|Introduction引介     |引用允许你添加新方法或属性到现有的类中。|
|Weaving织入          |Weaving 把方面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。这些可以在编译时，类加载时和运行时完成。|

AOP就是为Target创建Proxy来实现代码增强，Target中可以被增强的方法（接口）就是JoinPoint，
真正实现了增强的方法就是Pointcut，增强的代码就是Advice，Advice什么时候执行叫Aspect切面，
Advice的插入叫做Weaving织入。

★ 通俗了讲，可以把目标对象看作一个汉堡包，汉堡包有很多层，里面有白菜、鸡胸、芝士、面包等，每一层就是一个方法。如果不对这个
汉堡增强，我们每一口咬下去都是同样的层顺序。
★ 突然有一天我觉得这个汉堡很难吃，我想在白菜上面加点牛肉，在鸡胸下垫点生菜，按照普通人的做法，我应该先将汉堡需要改变的层整
体拆开，然后插入相应的层，最后组装好，开吃。这就好比我们对目标对象增强需要修改源码一样。
★ 而现在有一种新的办法，叫做AOP（面向切面）,它不需要对原汉堡进行改造，你不是想吃新的做法的汉堡吗？你先咬一口含在嘴里，把嘴张开，
我在咬断面上给你注入你需要的牛肉、生菜，然后你可以开始咀嚼了。这就是面向切面编程，面向切面我可以看透整个目标对象，哪里需要增强
我都能插入通知。

### 基于xml的Spring AOP

AOP的实现就是创建代理对象，在被代理对象方法执行前后插入代码，Spring中的AOP就是帮我们
创建代理对象，然后将一些结点暴露给我们，我们需要做的就是编写具体的业务实现，以及告诉Spring我要在哪里
插入什么样的代码
	
**环境搭建**

1. 配置Spring AOP仓库

```xml
<!--AOP start-->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-aop</artifactId>
	<version>${spring.version}</version>
</dependency>
<!--aspectj的runtime包(必须)-->
<dependency>
	<groupId>org.aspectj</groupId>
	<artifactId>aspectjrt</artifactId>
	<version>1.9.1</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.aspectj/aspectjweaver
Aspect的依赖包
-->
<dependency>
	<groupId>org.aspectj</groupId>
	<artifactId>aspectjweaver</artifactId>
	<version>1.9.1</version>
</dependency>
<!--AOP end-->
```

2. 连接点和切入点

```Java
public interface IUserService {
    void saveUser();
    void deleteUser(int id);
    int findUser();
}
public class UserService implements IUserService {
    @Override
    public void saveUser() {
        System.out.println("保存用户");
    }
    @Override
    public void deleteUser(int id) {
        System.out.println("删除用户："+id);
    }
    @Override
    public int findUser() {
        System.out.println("查询用户");
        int i = 1/0;
        return 0;
    }
}
```

3. 通知类

```Java
public class Logger {
    /**前置通知*/
    public void logPrintStart(){
        System.out.println(">>>增强日志工具开始记录日志了。。。");
    }
    /**后置通知（返回通知）*/
    public void logPrintReturn(){
        System.out.println("---增强日志返回。。。");
    }
    /**异常通知*/
    public void logPrintThrowing(){
        System.out.println("!!!增强异常通知。。。");
    }
    /**最终通知*/
    public void logPrintEnd(){
        System.out.println("<<<增强日志工具记录日志结束了。。。");
    }
    /**
     * 环绕通知
     * 问题：当我们配置了环绕通知后，切入点方法没有执行，而环绕通知里的代码执行了
     * 分析：由动态代理可知，环绕通知指的是invoke()方法，并且里面有明确的切入点方法调用，
     *       而我们现在的环绕通知没有明确切入点方法调用
     * 解决：spring为我们提供了ProceedingJoinPoint接口，作为环绕通知的方法参数来使用，
     *       在程序运行时，spring框架会为我们提供该接口的实现类供我们使用，该接口中有
     *       一个proceed()方法，等同于method.invoke()，就是明确调用业务层核心方法（切入点方法）
     *
     * 结论：环绕通知是spring框架为我们提供的一种可以在代码中手动控制通知方法什么时候执行的方式
     */
    public void logPrintAround(ProceedingJoinPoint pjp){
        System.out.println("*******环绕通知开始*******");
        try {
            System.out.println("前置通知");
            pjp.proceed();
            System.out.println("后置通知");
        } catch (Throwable throwable) {
            System.out.println("异常通知");
            throwable.printStackTrace();
        } finally {
            System.out.println("最终通知");
        }
        System.out.println("*******环绕通知结束*******");
    }
}
```

4. xml配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
                http://www.springframework.org/schema/aop
                http://www.springframework.org/schema/aop/spring-aop.xsd">
    <!--配置Service的bean-->
    <bean id="userService" class="com.openxu.aop.service.impl.UserService"></bean>

    <!--基于xml的aop配置步骤-->
    <!--1. 把通知类(增强代码类)交给spring来管理-->
    <bean id="logger" class="com.openxu.aop.Logger"></bean>
    <!--2.导入aop名称空间，并且使用aop:config开始aop的配置-->
    <aop:config>
        <!--5.定义通用切入点表达式，如果在aop:aspect标签外部，则表示素有切面都可用-->
        <aop:pointcut id="ptAll" expression="execution(* com.openxu.aop.service.impl.*.*(..))"/>

        <!--3.使用aop:aspect配置切面，id属性是切面的唯一标识，ref属性用于配置通知类bean的id-->
        <aop:aspect id="logAdvice" ref="logger">
            <!--4.配置通知类型，指定增强方法何时执行
                  method属性：指定增强方法名称
                  pointcut属性：指定切入点表达式
                  切入点表达式：关键字execution(表达式：访问修饰符 返回值 包名.包名...类名.方法名(参数列表))

          ★★★★★全匹配方式(只能匹配单个方法，下面列出变换过程)：
                          public void com.openxu.aop.service.impl.UserService.saveUser()
                      访问修饰符课省略：
                          void com.openxu.aop.service.impl.UserService.saveUser()
                      返回值可以使用通配符，表示任意返回值：
                          * com.openxu.aop.service.impl.UserService.saveUser()
                      包名可以使用通配符，表示任意包，但是有几级包就需要写几个* ：
                          * *.*.*.UserService.saveUser()
                      包名可以使用..表示当前包及其子包：
                          * com..UserService.saveUser()
                      类名和方法名都可使用通配符：
                          * com..*.*()
                      参数类型可以使用具体类型，来表示参数类型：
                          基本类型直接写类型名称：int
                          引用类型必须是包名.类名：java.lang.Integer
                      参数列表可以使用通配符，表示任意参数类型，但是必须有参数
                          * com..*.*(*)  有参数的方法才能被匹配增强
                      参数列表可以使用..表示有无参数都均可，有参数可以是任意类型：
                          * com..*.*(..)
                  全通配方式：* *..*.*(..)
                  实际开发中，我们一般都是对业务层方法进行增强，所以写法：
                          * com.openxu.aop.service.impl*.*(..)
                  -->
            <!--前置通知：永远在切入点方法执行之前执行-->
            <aop:before method="logPrintStart" pointcut="execution(* com.openxu.aop.service.impl.*.*(..))"/>
            <!--后置通知：切入点方法正常执行之后执行-->
            <aop:after-returning method="logPrintReturn" pointcut-ref="pt2"/>
            <!--异常通知：切入点方法产生异常后执行，和后置通知互斥-->
            <aop:after-throwing method="logPrintThrowing" pointcut-ref="pt2"/>
            <!--最终通知：无论切入点执行是否正常，都会在其后面执行-->
            <aop:after method="logPrintEnd" pointcut-ref="ptAll"/>
            <!--5. 配置通用切入点表达式：如果写在了aop:aspect标签内部，则表示只有当前切面可用-->
            <aop:pointcut id="pt2" expression="execution(* com.openxu.aop.service.impl.UserService.findUser())"/>

            <!--配置环岛通知-->
            <aop:around method="logPrintAround" pointcut="execution(* com.openxu.aop.service.impl.UserService.saveUser())"/>

        </aop:aspect>
    </aop:config>
</beans>
```

5. 测试
```Java
public static void main(String[] args) {
	ApplicationContext ac = new ClassPathXmlApplicationContext("aopbean.xml");
	IUserService service = (IUserService)ac.getBean("userService");
	service.saveUser();
	service.deleteUser(1);
	service.findUser();
}
```

### 基于注解Spring AOP

上面的通知类和目标类及接口都一样，我们需要改造的是不要在xml中配置，而是在通知类上通过注解实现AOP.

- @Aspect //配置了切面，相当于xml中aop:aspect标签
- @Pointcut("execution(* com.openxu.aop.service.impl.*.*(..))") //定义通用切入点表达式
- @Before("pt1()")  //前置通知
- @AfterReturning("pt1()")   //后置通知（返回通知）
- @AfterThrowing("pt1()")   //异常通知
- @After("pt1()")    //最终通知
- @Around("pt1()")   //环绕通知

```Java
@Service("userService")
public class UserService implements IUserService {
    @Override
    public void saveUser() {
        System.out.println("保存用户");
    }
    @Override
    public void deleteUser(int id) {
        System.out.println("删除用户："+id);
    }
    @Override
    public int findUser() {
        System.out.println("查询用户");
        int i = 1/0;
        return 0;
    }
}
@Component("logger")
@Aspect //配置了切面，相当于xml中aop:aspect标签
public class Logger {
	//定义通用切入点表达式
    @Pointcut("execution(* com.openxu.aop.service.impl.*.*(..))")
    private void pt1(){}

    @Before("pt1()")  //前置通知
    public void logPrintStart(){
        System.out.println(">>>增强日志工具开始记录日志了。。。");
    }
    @AfterReturning("pt1()")   //后置通知（返回通知）
    public void logPrintReturn(){
        System.out.println("---增强日志返回。。。");
    }
    @AfterThrowing("pt1()")   //异常通知
    public void logPrintThrowing(){
        System.out.println("!!!增强异常通知。。。");
    }
    @After("pt1()")    //最终通知
    public void logPrintEnd(){
        System.out.println("<<<增强日志工具记录日志结束了。。。");
    }
    @Around("pt1()")   //环绕通知
    public void logPrintAround(ProceedingJoinPoint pjp){
        System.out.println("*******环绕通知开始*******");
        try {
            System.out.println("前置通知");
            pjp.proceed();
            System.out.println("后置通知");
        } catch (Throwable throwable) {
            System.out.println("异常通知");
            throwable.printStackTrace();
        } finally {
            System.out.println("最终通知");
        }
        System.out.println("*******环绕通知结束*******");
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
    <!--配置Spring创建容器时要扫描的包-->
    <context:component-scan base-package="com.openxu.aop"></context:component-scan>
    <!--开启Spring对注解AOP的支持-->
    <aop:aspectj-autoproxy/>
</beans>
```

```Java
public static void main(String[] args) {
	ApplicationContext ac = new ClassPathXmlApplicationContext("aopbean.xml");
	IUserService service = (IUserService)ac.getBean("userService");
	service.saveUser();
	service.deleteUser(1);
	service.findUser();
}
```


**彻底去除xml配置**

```Java
@Configuration
@ComponentScan("com.openxu.aop")  //配置Spring容器扫描包
@EnableAspectJAutoProxy  //开启Spring对注解AOP的支持，相当于xml中<aop:aspectj-autoproxy/>
public class SpringConfig {
}

public static void main(String[] args) {
//        ApplicationContext ac = new ClassPathXmlApplicationContext("aopbean.xml");
	ApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfig.class);
	IUserService service = (IUserService)ac.getBean("userService");
	service.saveUser();
	service.deleteUser(1);
	service.findUser();
}
```















