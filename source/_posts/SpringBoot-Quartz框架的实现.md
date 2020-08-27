---
title: SpringBoot+Quartz框架的实现
date: 2020-08-14 18:39:29
tags:
- springboot
- quartz
---

定时任务 想必做程序的都或多或少的接触过,以便于我们以某个特定的 时间/频率 去执行所需要的程序,Quartz 是一个优秀的框架,可以根据我们的配置将 定时任务的执行 时间/频率 持久化至数据库, 我们通过修改数据库中的任务下次执行时间,达到不需要等到任务配置执行的原始 时间/频率,随时地运行定时任务; 并且可以看到任务的运行状态 WATING BLOCKING等

   1.导入依赖

   quartz自定义配置的数据源会使用C3P0创建连接,所以要引入C3P0依赖

```java
 <!-- Quartz定时任务 -->
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-quartz</artifactId>
   </dependency>
<!--C3P0 -->
   <dependency>
       <groupId>com.mchange</groupId>
       <artifactId>c3p0</artifactId>
       <version>0.9.5.5</version>
   </dependency>
```

2.quartz 配置文件,yml方式

创建定时任务表的sql太长,这里就不贴了,我会将sql上传至GitHub,文末我会贴地址

```java
## quartz定时任务
spring:
  quartz:
    #jdbc 采用数据库方式  memory 采用内存方式
    job-store-type: jdbc  
    initialize-schema: embedded
    #设置自动启动，默认为 true
    auto-startup: true
    #启动时更新己存在的Job
    overwrite-existing-jobs: true
    properties:
      org:
        quartz:
          scheduler:
            instanceName: MyScheduler
            instanceId: AUTO
          jobStore:
            #指定使用的JobStore
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            #数据库前缀
            tablePrefix: QRTZ_
            #是否为集群
            isClustered: false
            #检测任务执行时间的间隔  毫秒
            misfireThreshold: 5000
            clusterCheckinInterval: 10000
            #数据源名称
            dataSource: myDS
          #线程池配置
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool
            threadCount: 20
            threadPriority: 5
            threadsInheritContextClassLoaderOfInitializingThread: true
          #数据源
          dataSource:
            myDS:
              driver: com.mysql.cj.jdbc.Driver
              URL: jdbc:mysql://localhost:3306/test?characterEncoding=UTF-8&useUnicode=true&useSSL=false&tinyInt1isBit=false&serverTimezone=Asia/Shanghai
              user: root
              password: root
              maxConnections: 5
```

有同学可能会问了,配置文件是配置好了,是在哪引用的呢? 别急, 且听我娓娓道来

spring-boot-starter-quartz (为方便诉说,下文中使用 bootquartz代替) 这个包下的QuartzProperties会帮我们自动加载配置文件,且看以下部分截图

![img](https://img-blog.csdnimg.cn/20200617111853517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RlbW9fTGl1,size_16,color_FFFFFF,t_70)

可以看到, QuartzProperties 使用了 @ConfigurationProperties 加载了 spring.quartz 前缀的配置,也就是上面我们的配置文件中的配置;加载之后呢, bootquartz包下有 类 QuartzAutoConfiguration, 看名字就可以知道,这个就是自动配置 quartz的类了.

所以我们不需要再去通过代码去配置 SchedulerFactoryBean 了,这是后话

QuartzAutoConfiguration 类注释

![img](https://img-blog.csdnimg.cn/20200617113632769.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RlbW9fTGl1,size_16,color_FFFFFF,t_70)

通过上面的截图我们发现,这里引用了 QuartzProperties

其中的 quartzScheduler()方法帮助我们创建了 SchedulerFactoryBean 并使用了** **QuartzProperties 中的自定义配置,以下是quartzScheduler()部分代码

```java
	@Bean
	@ConditionalOnMissingBean
	public SchedulerFactoryBean quartzScheduler() {
		SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
		if (!this.properties.getProperties().isEmpty()) {
			schedulerFactoryBean
	.setQuartzProperties(asProperties(this.properties.getProperties()));
		}
		customize(schedulerFactoryBean);
		return schedulerFactoryBean;
	}
```

姑且一提,方法中调用了 customize(SchedulerFactoryBean  schedulerFactoryBean) 方法,这个方法会寻找实现了 SchedulerFactoryBeanCustomizer 接口的配置类,在其实现方法 customize(SchedulerFactoryBean  schedulerFactoryBean)中 可对 SchedulerFactoryBean  使用代码自定义配置

那么到这里结束了吗?不! 这里还有本文中最大的一个坑,作者深受其扰,扒了两天的源码才找到这个问题!!!

如果我们的项目中有其它的默认数据源,那么quartz会忽略配置文件中自定义数据源,使用默认数据源,原因看以下源码

首先是 QuartzAutoConfiguration 中的 静态内部类 JdbcStoreTypeConfiguration

```java
	@Configuration
	@ConditionalOnSingleCandidate(DataSource.class)
	protected static class JdbcStoreTypeConfiguration {
		@Bean
		@Order(0)
		public SchedulerFactoryBeanCustomizer dataSourceCustomizer(
				QuartzProperties properties, DataSource dataSource,
				@QuartzDataSource ObjectProvider<DataSource> quartzDataSource,
				ObjectProvider<PlatformTransactionManager> transactionManager) {
			return (schedulerFactoryBean) -> {
				if (properties.getJobStoreType() == JobStoreType.JDBC) {
                              //重点在这里 begin
					DataSource dataSourceToUse = getDataSource(dataSource,	quartzDataSource);
					schedulerFactoryBean.setDataSource(dataSourceToUse);
                              //重点在这里 end
					PlatformTransactionManager txManager = transactionManager.getIfUnique();
					if (txManager != null) {
schedulerFactoryBean.setTransactionManager(txManager);
					}
				}
			};
		}

       private DataSource getDataSource(DataSource dataSource,
				ObjectProvider<DataSource> quartzDataSource) {
			DataSource dataSourceIfAvailable = quartzDataSource.getIfAvailable();
			return (dataSourceIfAvailable != null) ? dataSourceIfAvailable : dataSource;
		}
```

其中的getDataSource 方法判断了我们项目中的 quartzDataSource是否为空,如果为空,那么就使用默认的数据源;quartzDataSource怎么才能不为空呢? 可以看到dataSourceCustomizer 方法参数中有 @QuartzDataSource 注解, 这个注解会去寻找我们项目中使用@QuartzDataSource配置的数据源,但是 我都已经在配置文件中自定义了数据源,再去手动配置一遍不是多此一举吗? 接着往下看

**SchedulerFactoryBean 的初始化方法部分源码▼**

```java
private void initSchedulerFactory(StdSchedulerFactory schedulerFactory) throws SchedulerException, IOException {
		Properties mergedProps = new Properties();

		if (this.dataSource != null) {
mergedProps.setProperty(StdSchedulerFactory.PROP_JOB_STORE_CLASS, LocalDataSourceJobStore.class.getName());
		}
}
```

我们在静态内部类设置过了数据源,初始化方法只要发现数据源不为空,那么就使用会使用 LocalDataSourceJobStore 覆盖我们quartz配置文件中设置的  org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX

而LocalDataSourceJobStore 中的初始化方法使用的是 SchedulerFactoryBean 中设置的数据源,所以我们quartz配置文件中的数据源才不会生效!!!

怎么解决呢?   我们上面提到了customize(SchedulerFactoryBean  schedulerFactoryBean) 方法,这个方法会寻找实现了 SchedulerFactoryBeanCustomizer 接口的配置类,在其实现方法 customize(SchedulerFactoryBean  schedulerFactoryBean)中 可对 SchedulerFactoryBean  使用代码自定义配置

所以 我们只要在SchedulerFactoryBean 创建后调用初始化方法之前,再将DataSource设置为null,那么SchedulerFactoryBean 初始化时,将会使用我们配置文件中的JobStoreTX去寻找我们配置的数据源了,至此,填坑完毕▼

```java
import org.springframework.boot.autoconfigure.quartz.SchedulerFactoryBeanCustomizer;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;

/**

 * @author Demo-Liu
 * @create 2020-06-12 11:20
 * @description 配置定时任务
 */

@Configuration
public class SchedulerConfig implements SchedulerFactoryBeanCustomizer {
    /**
     * @Author Demo-Liu
     * @Date 20200614 12:44
     * 自定义 quartz配置
     * @param schedulerFactoryBean
     */


    @Override
    public void customize(SchedulerFactoryBean schedulerFactoryBean) {
        schedulerFactoryBean.setDataSource(null);
    }
}
```

**以上**

**在文末附上我的GitHub小demo,其中包含了quartz的数据库建表sql,并提供了一种可以更加灵活便捷的通过yml文件配置定时任务的方式  地址: GitHub-BootQuartzYml**

**以下是yml配置文件配置定时任务的例子**

```java
#通过加载此配置文件实现动态创建Job 旨在通过一种更灵活便捷的方式来控制定时任务

#20200611 by Demo-Liu

#jobs:
#  jobList:
#    - jobConf:
#        name: 测试任务                             #任务名 可选
#        job: com.example.demo.quartz.DemoJob  #任务类包路径 必须
#        param:                                     #可为job类注入参数(可配置多项)   可选
#          jtbs: test
#        cron: 10 * * * * ?                         #任务执行频率 必须
#        active: true                               #任务激活状态 必须
jobs:
  jobList:
    - jobConf:
        name: 测试任务
        job: com.example.demo.quartz.DemoJob
        param:
          jtbs: test
          ss: test2
        cron: 0/10 * * * * ?
        active: true
    - jobConf:
        name: 测试任务2
        job: com.example.demo.quartz.DemoJob2
        param:
          jtbs: test
          ss: test2
        cron: 0/10 * * * * ?
        active: false
```

 