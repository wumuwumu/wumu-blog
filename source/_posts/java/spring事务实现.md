---
title: spring事务实现
date: 2019-08-03 14:40:33
tags:
- java
- spring
---

# 事务概念回顾

> ## 什么是事务？

事务是逻辑上的一组操作，要么都执行，要么都不执行.

> ## 事物的特性（ACID）：

1. **原子性：** 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
2. **一致性：** 执行事务前后，数据保持一致；
3. **隔离性：** 并发访问数据库时，一个用户的事物不被其他事物所干扰，各并发事务之间数据库是独立的；
4. **持久性:**  一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

# Spring事务管理接口介绍

> ## Spring事务管理接口：

- **PlatformTransactionManager：** （平台）事务管理器
- **TransactionDefinition：** 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)
- **TransactionStatus：** 事务运行状态

**所谓事务管理，其实就是“按照给定的事务规则来执行提交或者回滚操作”。**

> ## PlatformTransactionManager接口介绍

**Spring并不直接管理事务，而是提供了多种事务管理器** ，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。 Spring事务管理器的接口是： **org.springframework.transaction.PlatformTransactionManager** ，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

### PlatformTransactionManager接口代码如下：

PlatformTransactionManager接口中定义了三个方法：

```
Public interface PlatformTransactionManager()...{  
    // Return a currently active transaction or create a new one, according to the specified propagation behavior（根据指定的传播行为，返回当前活动的事务或创建一个新事务。）
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException; 
    // Commit the given transaction, with regard to its status（使用事务目前的状态提交事务）
    Void commit(TransactionStatus status) throws TransactionException;  
    // Perform a rollback of the given transaction（对执行的事务进行回滚）
    Void rollback(TransactionStatus status) throws TransactionException;  
    } 
复制代码
```

我们刚刚也说了Spring中PlatformTransactionManager根据不同持久层框架所对应的接口实现类,几个比较常见的如下图所示

![](http://wumu.sciento.cn/img/20190803144836.png)

比如我们在使用JDBC或者iBatis（就是Mybatis）进行数据持久化操作时,我们的xml配置通常如下：

```
	<!-- 事务管理器 -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<!-- 数据源 -->
		<property name="dataSource" ref="dataSource" />
	</bean>
复制代码
```

> ## TransactionDefinition接口介绍

事务管理器接口 **PlatformTransactionManager** 通过 **getTransaction(TransactionDefinition definition)** 方法来得到一个事务，这个方法里面的参数是 **TransactionDefinition类** ，这个类就定义了一些基本的事务属性。

**那么什么是事务属性呢？**

事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。事务属性包含了5个方面。 

![](http://wumu.sciento.cn/img/20190803144913.png)



### TransactionDefinition接口中的方法如下：

TransactionDefinition接口中定义了5个方法以及一些表示事务属性的常量比如隔离级别、传播行为等等的常量。

我下面只是列出了TransactionDefinition接口中的方法而没有给出接口中定义的常量，该接口中的常量信息会在后面依次介绍到。

```
public interface TransactionDefinition {
    // 返回事务的传播行为
    int getPropagationBehavior(); 
    // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
    int getIsolationLevel(); 
    // 返回事务必须在多少秒内完成
    //返回事务的名字
    String getName()；
    int getTimeout();  
    // 返回是否优化为只读事务。
    boolean isReadOnly();
} 
复制代码
```

### （1）事务隔离级别（定义了一个事务可能受其他并发事务影响的程度）：

我们先来看一下 **并发事务带来的问题** ，然后再来介绍一下 **TransactionDefinition 接口** 中定义了五个表示隔离级别的常量。

> #### 并发事务带来的问题

在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务（多个用户对统一数据进行操作）。并发虽然是必须的，但可能会导致一下的问题。

- **脏读（Dirty read）:** 当一个事务正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。

- **丢失修改（Lost to modify）:** 指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。

  例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。

- **不可重复读（Unrepeatableread）:** 指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。

- **幻读（Phantom read）:** 幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

**不可重复度和幻读区别：**

不可重复读的重点是修改，幻读的重点在于新增或者删除。

例1（同样的条件, 你读取过的数据, 再次读取出来发现值不一样了 ）：事务1中的A先生读取自己的工资为     1000的操作还没完成，事务2中的B先生就修改了A的工资为2000，导        致A再读自己的工资时工资变为  2000；这就是不可重复读。

例2（同样的条件, 第1次和第2次读出来的记录数不一样 ）：假某工资单表中工资大于3000的有4人，事务1读取了所有工资大于3000的人，共查到4条记录，这时事务2 又插入了一条工资大于3000的记录，事务1再次读取时查到的记录就变为了5条，这样就导致了幻读。

> #### 隔离级别

TransactionDefinition 接口中定义了五个表示隔离级别的常量：

- **TransactionDefinition.ISOLATION_DEFAULT:**	使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **TransactionDefinition.ISOLATION_READ_COMMITTED:** 	允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **TransactionDefinition.ISOLATION_REPEATABLE_READ:** 	对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **TransactionDefinition.ISOLATION_SERIALIZABLE:** 	最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。



### （2）事务传播行为（为了解决业务层方法之间互相调用的事务问题）：

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。在TransactionDefinition定义中包括了如下几个表示传播行为的常量：

**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

这里需要指出的是，前面的六种事务传播行为是 Spring 从 EJB 中引入的，他们共享相同的概念。而 **PROPAGATION_NESTED** 是 Spring 所特有的。以 PROPAGATION_NESTED 启动的事务内嵌于外部事务中（如果存在外部事务的话），此时，内嵌事务并不是一个独立的事务，它依赖于外部事务的存在，只有通过外部的事务提交，才能引起内部事务的提交，嵌套的子事务不能单独提交。如果熟悉 JDBC 中的保存点（SavePoint）的概念，那嵌套事务就很容易理解了，其实嵌套的子事务就是保存点的一个应用，一个事务中可以包括多个保存点，每一个嵌套子事务。另外，外部事务的回滚也会导致嵌套子事务的回滚。

### (3) 事务超时属性(一个事务允许执行的最长时间)

所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。

### (4) 事务只读属性（对事物资源是否执行只读操作）

事务的只读属性是指，对事务性资源进行只读操作或者是读写操作。所谓事务性资源就是指那些被事务管理的资源，比如数据源、 JMS 资源，以及自定义的事务性资源等等。如果确定只对事务性资源进行只读操作，那么我们可以将事务标志为只读的，以提高事务处理的性能。在 TransactionDefinition 中以 boolean 类型来表示该事务是否只读。

### (5) 回滚规则（定义事务回滚规则）

# 例子

## 使用API

下面给出一个基于底层 API 的编程式事务管理的示例， 
基于PlatformTransactionManager、TransactionDefinition 和 TransactionStatus 三个核心接口，我们完全可以通过编程的方式来进行事务管理。

```java
public class BankServiceImpl implements BankService {
    private BankDao bankDao;
    private TransactionDefinition txDefinition;
    private PlatformTransactionManager txManager;

public boolean transfer(Long fromId， Long toId， double amount) {
    // 获取一个事务
    TransactionStatus txStatus = txManager.getTransaction(txDefinition);
    boolean result = false;
    try {
        result = bankDao.transfer(fromId， toId， amount);
        txManager.commit(txStatus);    // 事务提交
    } catch (Exception e) {
        result = false;
        txManager.rollback(txStatus);      // 事务回滚
        System.out.println("Transfer Error!");
    }
    return result;
}
相应的配置文件如下所示：
```
```xml
<bean id="bankService" class="footmark.spring.core.tx.programmatic.origin.BankServiceImpl">
    <property name="bankDao" ref="bankDao"/>
    <property name="txManager" ref="transactionManager"/>
    <property name="txDefinition">
    <bean class="org.springframework.transaction.support.DefaultTransactionDefinition">
        <property name="propagationBehaviorName" value="PROPAGATION_REQUIRED"/>
    </bean>
    </property>
</bean>如上所示，我们在BankServiceImpl类中增加了两个属性：一个是 TransactionDefinition 类型的属性，它用于定义事务的规则；另一个是 PlatformTransactionManager 类型的属性，用于执行事务管理操作。如果一个业务方法需要添加事务，我们首先需要在方法开始执行前调用PlatformTransactionManager.getTransaction(…) 方法启动一个事务；创建并启动了事务之后，便可以开始编写业务逻辑代码，然后在适当的地方执行事务的提交或者回滚。
```

## 基于 TransactionTemplate 的编程式事务管理

　　当然，除了可以使用基于底层 API 的编程式事务外，还可以使用基于 TransactionTemplate 的编程式事务管理。通过上面的示例可以发现，上述事务管理的代码散落在业务逻辑代码中，破坏了原有代码的条理性，并且每一个业务方法都包含了类似的启动事务、提交/回滚事务的样板代码。Spring 也意识到了这些，并提供了简化的方法，这就是 Spring 在数据访问层非常常见的 模板回调模式。

```java
public class BankServiceImpl implements BankService {
    private BankDao bankDao;
    private TransactionTemplate transactionTemplate;
    ......
    public boolean transfer(final Long fromId， final Long toId， final double amount) {
        return (Boolean) transactionTemplate.execute(new TransactionCallback(){
            public Object doInTransaction(TransactionStatus status) {
                Object result;
                try {
                        result = bankDao.transfer(fromId， toId， amount);
                    } catch (Exception e) {
                        status.setRollbackOnly();
                        result = false;
                        System.out.println("Transfer Error!");
                }
                return result;
            }
        });
    }
}
```

相应的配置文件如下所示：

```java
<bean id="bankService" class="footmark.spring.core.tx.programmatic.template.BankServiceImpl">
    <property name="bankDao" ref="bankDao"/>
    <property name="transactionTemplate" ref="transactionTemplate"/>
</bean>
```


TransactionTemplate 的 execute() 方法有一个 TransactionCallback 类型的参数，该接口中定义了一个 doInTransaction() 方法，通常我们以匿名内部类的方式实现 TransactionCallback 接口，并在其 doInTransaction() 方法中书写业务逻辑代码。这里可以使用默认的事务提交和回滚规则，这样在业务代码中就不需要显式调用任何事务管理的 API。doInTransaction() 方法有一个TransactionStatus 类型的参数，我们可以在方法的任何位置调用该参数的 setRollbackOnly() 方法将事务标识为回滚的，以执行事务回滚。

​    此外，TransactionCallback 接口有一个子接口 TransactionCallbackWithoutResult，该接口中定义了一个 doInTransactionWithoutResult() 方法，TransactionCallbackWithoutResult 接口主要用于事务过程中不需要返回值的情况。当然，对于不需要返回值的情况，我们仍然可以使用 TransactionCallback 接口，并在方法中返回任意值即可。



## 基于底层 API 的编程式事务管理 
　　下面给出一个基于底层 API 的编程式事务管理的示例， 
基于PlatformTransactionManager、TransactionDefinition 和 TransactionStatus 三个核心接口，我们完全可以通过编程的方式来进行事务管理。

```java
public class BankServiceImpl implements BankService {
    private BankDao bankDao;
    private TransactionDefinition txDefinition;
    private PlatformTransactionManager txManager;
    public boolean transfer(Long fromId， Long toId， double amount) {
    // 获取一个事务
    TransactionStatus txStatus = txManager.getTransaction(txDefinition);
    boolean result = false;
    try {
        result = bankDao.transfer(fromId， toId， amount);
        txManager.commit(txStatus);    // 事务提交
    } catch (Exception e) {
        result = false;
        txManager.rollback(txStatus);      // 事务回滚
        System.out.println("Transfer Error!");
    }
    return result;
}
相应的配置文件如下所示：
```

```xml
<bean id="bankService" class="footmark.spring.core.tx.programmatic.origin.BankServiceImpl">
    <property name="bankDao" ref="bankDao"/>
    <property name="txManager" ref="transactionManager"/>
    <property name="txDefinition">
    <bean class="org.springframework.transaction.support.DefaultTransactionDefinition">
        <property name="propagationBehaviorName" value="PROPAGATION_REQUIRED"/>
    </bean>
    </property>
</bean>
```


如上所示，我们在BankServiceImpl类中增加了两个属性：一个是 TransactionDefinition 类型的属性，它用于定义事务的规则；另一个是 PlatformTransactionManager 类型的属性，用于执行事务管理操作。如果一个业务方法需要添加事务，我们首先需要在方法开始执行前调用PlatformTransactionManager.getTransaction(…) 方法启动一个事务；创建并启动了事务之后，便可以开始编写业务逻辑代码，然后在适当的地方执行事务的提交或者回滚。

## 基于 TransactionTemplate 的编程式事务管理

　　当然，除了可以使用基于底层 API 的编程式事务外，还可以使用基于 TransactionTemplate 的编程式事务管理。通过上面的示例可以发现，上述事务管理的代码散落在业务逻辑代码中，破坏了原有代码的条理性，并且每一个业务方法都包含了类似的启动事务、提交/回滚事务的样板代码。Spring 也意识到了这些，并提供了简化的方法，这就是 Spring 在数据访问层非常常见的 模板回调模式。

```java
public class BankServiceImpl implements BankService {
    private BankDao bankDao;
    private TransactionTemplate transactionTemplate;
    ......
    public boolean transfer(final Long fromId， final Long toId， final double amount) {
        return (Boolean) transactionTemplate.execute(new TransactionCallback(){
            public Object doInTransaction(TransactionStatus status) {
                Object result;
                try {
                        result = bankDao.transfer(fromId， toId， amount);
                    } catch (Exception e) {
                        status.setRollbackOnly();
                        result = false;
                        System.out.println("Transfer Error!");
                }
                return result;
            }
        });
    }
}
```

相应的配置文件如下所示：

```xml
<bean id="bankService" class="footmark.spring.core.tx.programmatic.template.BankServiceImpl">
    <property name="bankDao" ref="bankDao"/>
    <property name="transactionTemplate" ref="transactionTemplate"/>
</bean>
```


TransactionTemplate 的 execute() 方法有一个 TransactionCallback 类型的参数，该接口中定义了一个 doInTransaction() 方法，通常我们以匿名内部类的方式实现 TransactionCallback 接口，并在其 doInTransaction() 方法中书写业务逻辑代码。这里可以使用默认的事务提交和回滚规则，这样在业务代码中就不需要显式调用任何事务管理的 API。doInTransaction() 方法有一个TransactionStatus 类型的参数，我们可以在方法的任何位置调用该参数的 setRollbackOnly() 方法将事务标识为回滚的，以执行事务回滚。

　　此外，TransactionCallback 接口有一个子接口 TransactionCallbackWithoutResult，该接口中定义了一个 doInTransactionWithoutResult() 方法，TransactionCallbackWithoutResult 接口主要用于事务过程中不需要返回值的情况。当然，对于不需要返回值的情况，我们仍然可以使用 TransactionCallback 接口，并在方法中返回任意值即可。

## Spring 声明式事务管理
　　Spring 的声明式事务管理是建立在 Spring AOP 机制之上的，其本质是对目标方法前后进行拦截，并在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

　　声明式事务最大的优点就是不需要通过编程的方式管理事务，这样就不需要在业务逻辑代码中掺杂事务管理的代码，只需在配置文件中作相关的事务规则声明（或通过等价的基于标注的方式），便可以将事务规则应用到业务逻辑中。总的来说，声明式事务得益于 Spring IoC容器 和 Spring AOP 机制的支持：IoC容器为声明式事务管理提供了基础设施，使得 Bean 对于 Spring 框架而言是可管理的；而由于事务管理本身就是一个典型的横切逻辑（正是 AOP 的用武之地），因此 Spring AOP 机制是声明式事务管理的直接实现者。

　　显然，声明式事务管理要优于编程式事务管理，这正是spring倡导的非侵入式的开发方式。声明式事务管理使业务代码不受污染，一个普通的POJO对象，只要在XML文件中配置或者添加注解就可以获得完全的事务支持。因此，通常情况下，笔者强烈建议在开发中使用声明式事务，不仅因为其简单，更主要是因为这样使得纯业务代码不被污染，极大方便后期的代码维护。

## 基于 <tx> 命名空间的声明式事务管理 

　　Spring 2.x 引入了 <tx> 命名空间，结合使用 <aop> 命名空间，带给开发人员配置声明式事务的全新体验，配置变得更加简单和灵活。总的来说，开发者只需基于<tx>和<aop>命名空间在XML中进行简答配置便可实现声明式事务管理。下面基于<tx>使用Hibernate事务管理的配置文件：

```xml
<!-- 配置 DataSourece -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
    destroy-method="close">
    <!-- results in a setDriverClassName(String) call -->
    <property name="driverClassName">
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property name="url">
        <value>jdbc:mysql://localhost:3306/ssh</value>
    </property>
    <property name="username">
        <value>root</value>
    </property>
    <property name="password">
        <value>root</value>
    </property>
</bean>

<!-- 配置 sessionFactory -->
<bean id="sessionFactory"
    class="org.springframework.orm.hibernate3.annotation.AnnotationSessionFactoryBean">
    <!-- 数据源的设置 -->
    <property name="dataSource" ref="dataSource" />
    <!-- 用于持久化的实体类类列表 -->
    <property name="annotatedClasses">
        <list>
            <value>cn.edu.tju.rico.model.entity.User</value>
            <value>cn.edu.tju.rico.model.entity.Log</value>
        </list>
    </property>
    <!-- Hibernate 的配置 -->
    <property name="hibernateProperties">
        <props>
            <!-- 方言设置   -->
            <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
            <!-- 显示sql -->
            <prop key="hibernate.show_sql">true</prop>
           <!-- 格式化sql -->
            <prop key="hibernate.format_sql">true</prop>
            <!-- 自动创建/更新数据表 -->
            <prop key="hibernate.hbm2ddl.auto">update</prop>
        </props>
    </property>
</bean>

<!-- 配置 TransactionManager -->
<bean id="txManager"
    class="org.springframework.orm.hibernate3.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory" />
</bean>

<!-- 配置事务增强处理的切入点，以保证其被恰当的织入 -->    
<aop:config>
    <!-- 切点 -->
    <aop:pointcut expression="execution(* cn.edu.tju.rico.service.impl.*.*(..))"
        id="bussinessService" />
    <!-- 声明式事务的切入 -->
    <aop:advisor advice-ref="txAdvice" pointcut-ref="bussinessService" />
</aop:config>

<!-- 由txAdvice切面定义事务增强处理 -->
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <!-- get打头的方法为只读方法,因此将read-only设为 true -->
        <tx:method name="get*" read-only="true" />
        <!-- 其他方法为读写方法,因此将read-only设为 false -->
        <tx:method name="*" read-only="false" propagation="REQUIRED"
            isolation="DEFAULT" />
    </tx:attributes>
</tx:advice>
```

 事实上，Spring配置文件中关于事务的配置总是由三个部分组成，即：DataSource、TransactionManager和代理机制三部分，无论哪种配置方式，一般变化的只是代理机制这部分。其中，DataSource、TransactionManager这两部分只是会根据数据访问方式有所变化，比如使用hibernate进行数据访问时，DataSource实际为SessionFactory，TransactionManager的实现为 HibernateTransactionManager。如下图所示：

## 基于 @Transactional 的声明式事务管理

　　除了基于命名空间的事务配置方式，Spring 还引入了基于 Annotation 的方式，具体主要涉及@Transactional 标注。@Transactional 可以作用于接口、接口方法、类以及类方法上：当作用于类上时，该类的所有 public 方法将都具有该类型的事务属性；当作用于方法上时，该标注来覆盖类级别的定义。如下所示：

```java
@Transactional(propagation = Propagation.REQUIRED)
public boolean transfer(Long fromId， Long toId， double amount) {
    return bankDao.transfer(fromId， toId， amount);
}
```


Spring 使用 BeanPostProcessor 来处理 Bean 中的标注，因此我们需要在配置文件中作如下声明来激活该后处理 Bean，如下所示：

```java
<tx:annotation-driven transaction-manager="transactionManager”/>
```

1 与前面相似，transaction-manager、datasource 和 sessionFactory的配置不变，只需将基于<tx>和<aop>命名空间的配置更换为上述配置即可。

## Spring 声明式事务的本质

　　就Spring 声明式事务而言，无论其基于 <tx> 命名空间的实现还是基于 @Transactional 的实现，其本质都是 Spring AOP 机制的应用：即通过以@Transactional的方式或者XML配置文件的方式向业务组件中的目标业务方法插入事务增强处理并生成相应的代理对象供应用程序(客户端)使用从而达到无污染地添加事务的目的。如下图所示：



# 参考

https://juejin.im/post/5b00c52ef265da0b95276091

https://blog.csdn.net/justloveyou_/article/details/73733278 

