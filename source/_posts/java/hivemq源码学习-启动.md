---
title: hivemq源码学习-启动
abbrlink: 2e9ab0f6
date: 2020-01-14 14:00:00
---

# hivemq技术选型

- **使用Guice做DI**
- **使用Netty 4做网络框架**
- **使用JGroups做Cluster Node之间的集群通讯**
- **使用Exodus做Broker信息文件持久化存储**
- **使用Dropwizard Metrics做Broker的统计、监控**
- **使用Kryo做序列化/反序列化**
- **使用Jetty做Broker端servlet容器**
- **使用Resteasy做Broker端restfull框架**
- **使用Quartz/做Broker端任务的调度**

# 程序启动main函数

```java

final long startTime = System.nanoTime();

final MetricRegistry metricRegistry = new MetricRegistry();
metricRegistry.addListener(new MetricRegistryLogger());

final SystemInformationImpl systemInformation;
LoggingBootstrap.prepareLogging();

log.info("Starting HiveMQ Community Edition Server");

log.trace("Initializing HiveMQ home directory");
//Create SystemInformation this early because logging depends on it
systemInformation = new SystemInformationImpl(true);

log.trace("Initializing Logging");
LoggingBootstrap.initLogging(systemInformation.getConfigFolder());

log.trace("Initializing Exception handlers");
HiveMQExceptionHandlerBootstrap.addUnrecoverableExceptionHandler();

log.trace("Initializing configuration");
final FullConfigurationService configService = ConfigurationBootstrap.bootstrapConfig(systemInformation);

final HivemqId hiveMQId = new HivemqId();
log.info("This HiveMQ ID is {}", hiveMQId.get());

//ungraceful shutdown does not delete tmp folders, so we clean them up on broker start
log.trace("Cleaning up temporary folders");
TemporaryFileUtils.deleteTmpFolder(systemInformation.getDataFolder());

//must happen before persistence injector bootstrap as it creates the persistence folder.
log.trace("Checking for migrations");
final Map<MigrationUnit, PersistenceType> migrations = Migrations.checkForTypeMigration(systemInformation);
final Set<MigrationUnit> valueMigrations = Migrations.checkForValueMigration(systemInformation);

log.trace("Initializing persistences");
final Injector persistenceInjector =
    GuiceBootstrap.persistenceInjector(systemInformation, metricRegistry, hiveMQId, configService);
//blocks until all persistences started
persistenceInjector.getInstance(PersistenceStartup.class).finish();

if (ShutdownHooks.SHUTTING_DOWN.get()) {
    return;
}
if (configService.persistenceConfigurationService().getMode() != PersistenceMode.IN_MEMORY) {

    if (migrations.size() + valueMigrations.size() > 0) {
        if(migrations.size() > 0) {
            log.info("Persistence types has been changed, migrating persistent data.");
        } else {
            log.info("Persistence values has been changed, migrating persistent data.");
        }
        for (final MigrationUnit migrationUnit : migrations.keySet()) {
            log.debug("{} needs to be migrated.", StringUtils.capitalize(migrationUnit.toString()));
        }
        for (final MigrationUnit migrationUnit : valueMigrations) {
            log.debug("{} needs to be migrated.", StringUtils.capitalize(migrationUnit.toString()));
        }
        Migrations.migrate(persistenceInjector, migrations, valueMigrations);
    }

    Migrations.afterMigration(systemInformation);

} else {
    log.info("Starting with in memory persistences");
}

log.trace("Initializing Guice");
final Injector injector = GuiceBootstrap.bootstrapInjector(systemInformation,
                                                           metricRegistry,
                                                           hiveMQId,
                                                           configService,
                                                           persistenceInjector);
if (injector == null) {
    return;
}

if (ShutdownHooks.SHUTTING_DOWN.get()) {
    return;
}
// 创建hivemq服务器进行相关逻辑操作
final HiveMQServer instance = injector.getInstance(HiveMQServer.class);

if (InternalConfigurations.GC_AFTER_STARTUP) {
    log.trace("Starting initial garbage collection after startup");
    final long start = System.currentTimeMillis();
    //Start garbage collection of objects we don't need anymore after starting up
    System.gc();
    log.trace("Finished initial garbage collection after startup in {}ms", System.currentTimeMillis() - start);
}

if (ShutdownHooks.SHUTTING_DOWN.get()) {
    return;
}

/* It's important that we are modifying the log levels after Guice is initialized,
        otherwise this somehow interferes with Singleton creation */
LoggingBootstrap.addLoglevelModifiers();
instance.start(null);

if (ShutdownHooks.SHUTTING_DOWN.get()) {
    return;
}

log.info("Started HiveMQ in {}ms", TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime));

if (ShutdownHooks.SHUTTING_DOWN.get()) {
    return;
}

// 发送hivemq官方使用数据，不用管
final UsageStatistics usageStatistics = injector.getInstance(UsageStatistics.class);
usageStatistics.start();
```

