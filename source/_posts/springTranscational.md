---
title: spring事务管理源码解析--加了@Transactional注解后Spring究竟为我们做了哪些事情？
date: 2018-04-19 23:16:28
tags: [Spring]
categories:
- Java
- Web
---

实现一个事务需要以下几步：1.获取数据库连接  2.执行数据库操作  3.如果2步骤发生异常就回滚，否则就提交  4.释放资源。

1、3、4步骤是所有事务所共有的逻辑，程序真正需要关心的只有第2步，spring的事务管理也正是帮我们做了1、3、4的工作。

### 准备工作

- 第一步，要配置dataSource、transactionManager
```xml
<!--启动spring注解功能 -->
<tx:annotation-driven transactionmanager="transactionManager" />

<!-- 设定transactionManager -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>

<!-- 配置数据源，这里以C3P0为例 -->
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
          destroy-method="close">
    <property name="driverClass" value="..."></property>
        ...
    <property name="checkoutTimeout" value="..."/>
</bean>
```
- 第二步，在需要事务的方法上加上注解 @Transactional 即可。

<!-- more -->

### 配置参数
```java 源码位置：org.springframework.transaction.annotation.Transactional
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

	@AliasFor("transactionManager")
	String value() default "";

	@AliasFor("value")
	String transactionManager() default "";
    
    // 事务的传播属性，默认已有事务存在就加入当前事务，否则就开启一个新事务	
	Propagation propagation() default Propagation.REQUIRED;

    // 隔离级别，默认当前所用数据库的隔离级别，下面以Mysql为例
	Isolation isolation() default Isolation.DEFAULT;

    // 超时时间
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

    // 是否只读	
	boolean readOnly() default false;

    // 发生哪些异常时回滚事务
	Class<? extends Throwable>[] rollbackFor() default {};

	String[] rollbackForClassName() default {};
    
	// 发生哪些异常时不回滚事务
	Class<? extends Throwable>[] noRollbackFor() default {};

	String[] noRollbackForClassName() default {};
}
```

这里主要关注以下三个参数：
- 传播属性
- 隔离级别
- 超时时间

前面配置的transactionManager（即DataSourceTransactionManager）继承关系图如下：

![spring_transactional_01](/uploads/spring/spring_transactional_01.png)

我们只看PlatformTransactionManager-->AbstractPlatformTransactionManager-->DataSourceTransactionManager这条线，先来看PlatformTransactionManager接口：

```java 源码位置：org.springframework.transaction.PlatformTransactionManager
public interface PlatformTransactionManager {

	TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;

}
```

它是spring事务管理器的顶层接口，这三个方法是对应前边所说的1、3两个步骤，分别为开启事务、提交、回滚。再来看它的实现类AbstractPlatformTransactionManager，主要看上边三个方法对应的子类实现，可以发现它们都是一样的套路：先做一些边界条件判断，然后调用自己内部的对应do...方法，而且每个对应的do...方法都是protected abstract 修饰的，实际上继承于AbstractPlatformTransactionManager 的类有很多，如DataSourceTransactionManager , JtaTransactionManager, HibernateTransactionManager 等，不管是哪一个种，总要有 getTransaction，commit，rollback的方法，所以这个三个方法在接口中，不过具体实现方式有所区别，所以具体的实现需要而且是必须在子类中完成。AbstractPlatformTransactionManager 规定了这三个方法的调用顺序，真正的细节在子类中以”do*”开头的方法中实现，这也正是大多数抽象类的作用所在。

我们先来看此抽象类的 getTransaction 方法：

![spring_transactional_02](/uploads/spring/spring_transactional_02.png)

重点关注我标注出来的两步，先调用doGetTransaction方法获取一个事务（新开启的或者已经存在的），然后调用doBegin方法开始执行事务。

我们直接去看具体实现类的这两个方法，这里以DataSourceTransactionManager为例。doGetTransaction方法其实就是返回我们配置的全局数据源。

![spring_transactional_03](/uploads/spring/spring_transactional_03.png)

再来看doBegin方法，先获取一个connetion，设置autoCommit为false，设置隔离级别...

![spring_transactional_04](/uploads/spring/spring_transactional_04.png)

然后调用 service 中的代码，如果没有抛出异常Spring框架将继续调用AbstractPlatformTransactionManager中的commit方法，继续看源码：

![spring_transactional_05](/uploads/spring/spring_transactional_05.png)

![spring_transactional_06](/uploads/spring/spring_transactional_06.png)

还是要看子类doCommit方法，先拿到连接（其实就是之前获取到的那个），然后commit，有异常就抛给上层。 

这里先抛出两个问题：spring事务管理如何保证同一个线程总是能获取到已开启的事务？也就是说默认传播属性下同一个线程怎么保证最多只会有一个事务？

![spring_transactional_07](/uploads/spring/spring_transactional_07.png)

现在事务已经提交了，如果发生异常的话，父类中调用方法 rollback方法：

![spring_transactional_08](/uploads/spring/spring_transactional_08.png)

继续看子类doRollback方法：

![spring_transactional_09](/uploads/spring/spring_transactional_09.png)

现在回想我刚才抛出的两个问题，不知道你有没有发现，在提交和回滚的是获取当前连接都是这样调用：

```java
Connection con = txObject.getConnectionHolder().getConnection();
```

那怎么保证每次get的是同一个连接？

来看的dobegin方法，重点关注bind the session holder这个方法

![spring_transactional_10](/uploads/spring/spring_transactional_10.png)

![spring_transactional_11](/uploads/spring/spring_transactional_11.png)

附上resource的定义，可以发现它是一个ThreadLocal。

```java
private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
```

现在明白了吧，spring在开启事务的时候会把获取到的连接绑定在事务的上下文环境中，然后把这个上下文环境又绑定在当前线程的ThreadLocal中，这样每个线程的事务就是独立的了，那同一个线程每次获取连接获取到的也就是同一个了。

最后看一下对资源释放的代码，父类中cleanupAfterCompletion方法调用了doCleanupAfterCompletion，我们直接看子类实现：

![spring_transactional_12](/uploads/spring/spring_transactional_12.png)

releaseConnection方法就是进行资源释放的工作，逻辑比较简单就不分析了。

至此 Spring 帮我们管理的事务，其主要流程和方法已经介绍完了，当然其中还有很多 private 方法和 protected 方法没有介绍。我想只要把主干捋清楚之后，其他一些方法也很好理解了。

绕了一大圈，实际上还是绕不过开启事务，提交事务，回滚事务三件事，只是这三件事情现在由Spring帮助我们在背后默默做好了。


