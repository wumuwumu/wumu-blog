---
title: Druid初始化
tags:
  - java
abbrlink: fdbf7476
date: 2019-03-25 18:17:33
---

```java
public void init() throws SQLException {
        if (inited) {
            return;
        }

        // bug fixed for dead lock, for issue #2980
        DruidDriver.getInstance();

        final ReentrantLock lock = this.lock;
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            throw new SQLException("interrupt", e);
        }

        boolean init = false;
        try {
            //双重检查
            if (inited) {
                return;
            }

            initStackTrace = Utils.toString(Thread.currentThread().getStackTrace());

            this.id = DruidDriver.createDataSourceId();
            if (this.id > 1) {
                long delta = (this.id - 1) * 100000;
                this.connectionIdSeedUpdater.addAndGet(this, delta);
                this.statementIdSeedUpdater.addAndGet(this, delta);
                this.resultSetIdSeedUpdater.addAndGet(this, delta);
                this.transactionIdSeedUpdater.addAndGet(this, delta);
            }

            if (this.jdbcUrl != null) {
                this.jdbcUrl = this.jdbcUrl.trim();
                initFromWrapDriverUrl();
            }

            for (Filter filter : filters) {
                filter.init(this);
            }

            if (this.dbType == null || this.dbType.length() == 0) {
                this.dbType = JdbcUtils.getDbType(jdbcUrl, null);
            }

            if (JdbcConstants.MYSQL.equals(this.dbType)
                    || JdbcConstants.MARIADB.equals(this.dbType)
                    || JdbcConstants.ALIYUN_ADS.equals(this.dbType)) {
                boolean cacheServerConfigurationSet = false;
                if (this.connectProperties.containsKey("cacheServerConfiguration")) {
                    cacheServerConfigurationSet = true;
                } else if (this.jdbcUrl.indexOf("cacheServerConfiguration") != -1) {
                    cacheServerConfigurationSet = true;
                }
                if (cacheServerConfigurationSet) {
                    this.connectProperties.put("cacheServerConfiguration", "true");
                }
            }

            if (maxActive <= 0) {
                throw new IllegalArgumentException("illegal maxActive " + maxActive);
            }

            if (maxActive < minIdle) {
                throw new IllegalArgumentException("illegal maxActive " + maxActive);
            }

            if (getInitialSize() > maxActive) {
                throw new IllegalArgumentException("illegal initialSize " + this.initialSize + ", maxActive " + maxActive);
            }

            if (timeBetweenLogStatsMillis > 0 && useGlobalDataSourceStat) {
                throw new IllegalArgumentException("timeBetweenLogStatsMillis not support useGlobalDataSourceStat=true");
            }

            if (maxEvictableIdleTimeMillis < minEvictableIdleTimeMillis) {
                throw new SQLException("maxEvictableIdleTimeMillis must be grater than minEvictableIdleTimeMillis");
            }

            if (this.driverClass != null) {
                this.driverClass = driverClass.trim();
            }

            initFromSPIServiceLoader();

            // 处理驱动
            if (this.driver == null) {
                if (this.driverClass == null || this.driverClass.isEmpty()) {
                    this.driverClass = JdbcUtils.getDriverClassName(this.jdbcUrl);
                }

                if (MockDriver.class.getName().equals(driverClass)) {
                    driver = MockDriver.instance;
                } else {
                    if (jdbcUrl == null && (driverClass == null || driverClass.length() == 0)) {
                        throw new SQLException("url not set");
                    }
                   
                    driver = JdbcUtils.createDriver(driverClassLoader, driverClass);
                }
            } else {
                if (this.driverClass == null) {
                    this.driverClass = driver.getClass().getName();
                }
            }
			// 进行参数的核对，没有什么逻辑
            initCheck();

            // 为不同的数据库处理异常，这个可以借鉴
            initExceptionSorter();
            initValidConnectionChecker();
            // 做了一些检查，不知道
            validationQueryCheck();

            // 创建数据统计对象
            if (isUseGlobalDataSourceStat()) {
                dataSourceStat = JdbcDataSourceStat.getGlobal();
                if (dataSourceStat == null) {
                    dataSourceStat = new JdbcDataSourceStat("Global", "Global", this.dbType);
                    JdbcDataSourceStat.setGlobal(dataSourceStat);
                }
                if (dataSourceStat.getDbType() == null) {
                    dataSourceStat.setDbType(this.dbType);
                }
            } else {
                dataSourceStat = new JdbcDataSourceStat(this.name, this.jdbcUrl, this.dbType, this.connectProperties);
            }
            dataSourceStat.setResetStatEnable(this.resetStatEnable);

            // 创建连接池
            connections = new DruidConnectionHolder[maxActive];
            evictConnections = new DruidConnectionHolder[maxActive];
            keepAliveConnections = new DruidConnectionHolder[maxActive];

            SQLException connectError = null;

            // 同步或者异步创建线程池
            if (createScheduler != null && asyncInit) {
                for (int i = 0; i < initialSize; ++i) {
                    createTaskCount++;
                    CreateConnectionTask task = new CreateConnectionTask(true);
                    this.createSchedulerFuture = createScheduler.submit(task);
                }
            } else if (!asyncInit) {
                // init connections
                while (poolingCount < initialSize) {
                    try {
                        PhysicalConnectionInfo pyConnectInfo = createPhysicalConnection();
                        DruidConnectionHolder holder = new DruidConnectionHolder(this, pyConnectInfo);
                        connections[poolingCount++] = holder;
                    } catch (SQLException ex) {
                        LOG.error("init datasource error, url: " + this.getUrl(), ex);
                        if (initExceptionThrow) {
                            connectError = ex;
                            break;
                        } else {
                            Thread.sleep(3000);
                        }
                    }
                }

                if (poolingCount > 0) {
                    poolingPeak = poolingCount;
                    poolingPeakTime = System.currentTimeMillis();
                }
            }
            
			// 用来打印线程池
            createAndLogThread();
            
            
            createAndStartCreatorThread();
            
            // 停止
            createAndStartDestroyThread();

            // 等待线程创建完成
            initedLatch.await();
            init = true;

            initedTime = new Date();
            
            // 注册mbean
            registerMbean();

            if (connectError != null && poolingCount == 0) {
                throw connectError;
            }

            // 检查连接池，防止连接池超出最大连接池
            if (keepAlive) {
                // async fill to minIdle
                if (createScheduler != null) {
                    for (int i = 0; i < minIdle; ++i) {
                        createTaskCount++;
                        CreateConnectionTask task = new CreateConnectionTask(true);
                        this.createSchedulerFuture = createScheduler.submit(task);
                    }
                } else {
                    this.emptySignal();
                }
            }

        } catch (SQLException e) {
            LOG.error("{dataSource-" + this.getID() + "} init error", e);
            throw e;
        } catch (InterruptedException e) {
            throw new SQLException(e.getMessage(), e);
        } catch (RuntimeException e){
            LOG.error("{dataSource-" + this.getID() + "} init error", e);
            throw e;
        } catch (Error e){
            LOG.error("{dataSource-" + this.getID() + "} init error", e);
            throw e;

        } finally {
            // 初始化成功
            inited = true;
            // 解锁
            lock.unlock();

            if (init && LOG.isInfoEnabled()) {
                String msg = "{dataSource-" + this.getID();

                if (this.name != null && !this.name.isEmpty()) {
                    msg += ",";
                    msg += this.name;
                }

                msg += "} inited";

                LOG.info(msg);
            }
        }
    }
```

