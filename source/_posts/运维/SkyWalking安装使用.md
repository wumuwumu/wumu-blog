---
title: SkyWalking安装使用
date: 2020-10-24 12:00:00
---

#### 1、SkyWalking

SkyWalking是国内开源的基于字节码注入的调用链分析以及应用监控分析工具。特点是支持多种插件，UI功能较强，接入端无代码侵入。目前使用厂商最多，版本更新较快，已成为 Apache 基金会顶级项目。
 官网：[http://skywalking.apache.org/zh/downloads/](https://links.jianshu.com/go?to=http%3A%2F%2Fskywalking.apache.org%2Fzh%2Fdownloads%2F)
 GitHub: [https://github.com/apache/skywalking](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapache%2Fskywalking)

#### 2、Windows使用教程

###### 1、下载SkyWalking7.0.0



![img](https:////upload-images.jianshu.io/upload_images/11997591-49ade3bef0b63d56.png?imageMogr2/auto-orient/strip|imageView2/2/w/930/format/webp)

image.png

###### 2、安装

下载加压后目录如下



![img](https:////upload-images.jianshu.io/upload_images/11997591-f59aedf9f82cdf77.png?imageMogr2/auto-orient/strip|imageView2/2/w/1184/format/webp)

image.png

###### 3、启动

在bin目录下执行startup.bat即可启动服务



![img](https:////upload-images.jianshu.io/upload_images/11997591-82facb797138f2b8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

说明：执行startup.bat之后会启动如下两个服务：
 （1）Skywalking-Collector：追踪信息收集器，通过 gRPC/Http 收集客户端的采集信息 ，Http默认端口 12800，gRPC默认端口 11800。
 （2）Skywalking-Webapp：管理平台页面 默认端口 8080
 （3）登录信息 admin/admin

###### 4、配置信息

a、application.yml
 主要配置SkyWakling集群方式、数据存储，配置文件内容如下

```ruby
core:
  default:
  restHost: ${SW_CORE_REST_HOST:0.0.0.0}
  restPort: ${SW_CORE_REST_PORT:12800}
  restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
  gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
  gRPCPort: ${SW_CORE_GRPC_PORT:11800}
  downsampling:
    - Hour
    - Day
    - Month
  # Set a timeout on metric data. After the timeout has expired, the metric data will automatically be deleted.
  recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:90} # Unit is minute
  minuteMetricsDataTTL: ${SW_CORE_MINUTE_METRIC_DATA_TTL:90} # Unit is minute
  hourMetricsDataTTL: ${SW_CORE_HOUR_METRIC_DATA_TTL:36} # Unit is hour
  dayMetricsDataTTL: ${SW_CORE_DAY_METRIC_DATA_TTL:45} # Unit is day
  monthMetricsDataTTL: ${SW_CORE_MONTH_METRIC_DATA_TTL:18} # Unit is month
storage:
  h2:
    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
    user: ${SW_STORAGE_H2_USER:sa}
    #  elasticsearch:
    #    # nameSpace: ${SW_NAMESPACE:""}
    #    clusterNodes:     ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    #    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:2}
    #    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    #    # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
    #    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:2000} # Execute the bulk every 2000 requests
    #    bulkSize: ${SW_STORAGE_ES_BULK_SIZE:20} # flush the bulk every 20mb
    #    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    #    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
receiver-register:
    default:
receiver-trace:
    default:
bufferPath: ${SW_RECEIVER_BUFFER_PATH:../trace-buffer/}  # Path to trace buffer files, suggest to use absolute path
bufferOffsetMaxFileSize: ${SW_RECEIVER_BUFFER_OFFSET_MAX_FILE_SIZE:100} # Unit is MB
bufferDataMaxFileSize: ${SW_RECEIVER_BUFFER_DATA_MAX_FILE_SIZE:500} # Unit is MB
bufferFileCleanWhenRestart: ${SW_RECEIVER_BUFFER_FILE_CLEAN_WHEN_RESTART:false}
sampleRate: ${SW_TRACE_SAMPLE_RATE:10000} # The sample rate precision is 1/10000. 10000 means 100% sample in default.
receiver-jvm:
    default:
#service-mesh:
    #  default:
#    bufferPath: ${SW_SERVICE_MESH_BUFFER_PATH:../mesh-buffer/}  # Path to trace buffer files, suggest to use absolute path
#    bufferOffsetMaxFileSize: ${SW_SERVICE_MESH_OFFSET_MAX_FILE_SIZE:100} # Unit is MB
#    bufferDataMaxFileSize: ${SW_SERVICE_MESH_BUFFER_DATA_MAX_FILE_SIZE:500} # Unit is MB
#    bufferFileCleanWhenRestart: ${SW_SERVICE_MESH_BUFFER_FILE_CLEAN_WHEN_RESTART:false}
#istio-telemetry:
  #  default:
#receiver_zipkin:
  #  default:
  #    host: ${SW_RECEIVER_ZIPKIN_HOST:0.0.0.0}
  #    port: ${SW_RECEIVER_ZIPKIN_PORT:9411}
  #    contextPath: ${SW_RECEIVER_ZIPKIN_CONTEXT_PATH:/}
```

#### 3、IDEA 部署探针

修改项目启动的 VM 运行参数

###### 1、点击菜单栏中的 Run -> EditConfigurations...



![img](https:////upload-images.jianshu.io/upload_images/11997591-8eca2d4c662aa27a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

###### 2、增加如下参数到VM options中：

```undefined
-javaagent:C:\Users\ke\Desktop\apache-skywalking-apm-6.6.0\apache-  skywalking-apm-bin\agent\skywalking-agent.jar 
-Dskywalking.agent.service_name=service-eureka 
-Dskywalking.collector.backend_service=localhost:8761
```

-javaagent：用于指定探针路径
 -Dskywalking.agent.service_name：用于重写 agent/config/agent.config 配置文件中的服务名
 -Dskywalking.collector.backend_service：用于重写 agent/config/agent.config 配置文件中的服务地址

启动后看到如下启动日志：

```css
    INFO 2020-02-13 14:57:36:310 main SnifferConfigInitializer : Config file found in C:\Users\ke\Desktop\apache-skywalking-apm-6.6.0\apache-skywalking-apm-bin\agent\config\agent.config.
```

#### 4、Java 命令行启动方式

```undefined
java -javaagent:C:\Users\ke\Desktop\apache-skywalking-apm-6.6.0\apache-skywalking-apm-bin\agent/skywalking-agent.jar=-Dskywalking.agent.service_name=ijep-eureka-provider,-Dskywalking.collector.backend_service=localhost:11800 -jar ijep-registry-eureka.jar
```

注：-javaagent配置参数一定要在 -jar 之前。

##### 附: 配置文件详解

```objectivec
# 当前的应用编码，最终会显示在webui上。
# 建议一个应用的多个实例，使用有相同的application_code。请使用英文  agent.application_code=Your_ApplicationName

 # 每三秒采样的Trace数量
# 默认为负数，代表在保证不超过内存Buffer区的前提下，采集所有的Trace
# agent.sample_n_per_3_secs=-1

# 设置需要忽略的请求地址
# 默认配置如下
# agent.ignore_suffix=.jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.mp3,.mp4,.html,.svg

# 探针调试开关，如果设置为true，探针会将所有操作字节码的类输出  到/debugging目录下
# skywalking团队可能在调试，需要此文件
# agent.is_open_debugging_class = true

# 对应Collector的config/application.yml配置文件中 agent_server/jetty/port 配置内容
# 例如：
# 单节点配置：SERVERS="127.0.0.1:8080" 
# 集群配置：SERVERS="10.2.45.126:8080,10.2.45.127:7600" 
collector.servers=127.0.0.1:10800

# 日志文件名称前缀
logging.file_name=skywalking-agent.log

# 日志文件最大大小
# 如果超过此大小，则会生成新文件。
# 默认为300M
logging.max_file_size=314572800

# 日志级别，默认为DEBUG。
logging.level=DEBUG
```

#### 3、Centos 7使用教程

###### 1、环境准备

● 系统版本：Centos 7
 ● 内存配置：2G
 ● JDK版本：1.8.0_51
 ● IP为：192.168.1.180

###### 2、下载解压

```bash
# 下载SkyWalking
wget http://mirrors.tuna.tsinghua.edu.cn/apache/skywalking/6.5.0/apache-skywalking-apm-6.5.0.tar.gz
# 解压
tar -zxf apache-skywalking-apm-6.5.0.tar.gz
mkdir /usr/local/skywalking
mv apache-skywalking-apm-bin /usr/local/skywalking
```

###### 3、配置

SkyWakling共有三个配置文件，分别为：

● agent/config/agent.config
 ● conf/application.yml
 ● webapp/webapp.yml

其中application.yml和webapp.yml配置是服务器运行相关，agent.config是在代理节点上进行配置的。

###### a、application.yml

主要配置SkyWakling集群方式、数据存储，配置文件内容如下；

```ruby
 # 配置skywalking集群
cluster:
# 这里采用zk，需要注意的是，在skywalking6.5版本中要求zk版本也必须大于3.5
zookeeper:
  namespace: skywalking_brief
  # zk集群以","进行分割
  hostPort: 192.168.1.180:2283
  # 配置重试次数以及重试间隔
  baseSleepTimeMs: 3000
  maxRetries: 5
# 核心配置
core:
  default:
    role: ${SW_CORE_ROLE:Mixed}
    # backend配置，如果是Mixed或者Receiver类型，那么下面的配置将会生效，服务启动之后将会作为collector收集agent发送过来的链路数据
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_CONTEXT_PATH: /}
    # gRPC配置
    gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
    downsampling:
      - Hour
      - Day
      - Month
    # 是否允许删除度量数据
    enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR: true}
    # datakeeper删除数据执行间隔，单位为分钟
    dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD: 5}
    # 单位是分钟
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:90}
    minuteMetricsDataTTL: ${SW_CORE_MINUTE_METRIC_DATA_TTL:90}
    hourMetricsDataTTL: ${SW_CORE_HOUR_METRIC_DATA_TTL:36}
    dayMetricsDataTTL: ${SW_CORE_DAY_METRIC_DATA_TTL:45}
    monthMetricsDataTTL: ${SW_CORE_MONTH_METRIC_DATA_TTL:18}
    enableDatabaseSession: ${SW_CORE_ENABLE_DATABASE_SESSION: true}
    # 配置数据存储方式，默认为H2，这里修改为宿主机上的ES
storage:
  elasticsearch:
     namespace: elasticsearch
     #es地址，多master之间以","分割
     clusterNodes: 192.168.1.151:9800
     protocol: http
     # 配置index
     # index分片，es中默认为5，在skywalking中默认为2
     indexShardsNumber: 2
     # 分片备份数量，默认为0
     indexReplicasNumber: 0
     recordDataTTL: 7
     otherMetricsDataTTL: 45
     monthMetricsDataTTL: 18
     # 批量操作配置
     bulkActions: 2000          # 每2000个请求执行一次bulk操作
     flushsInterval: 5          # 每5s执行一次bulk操作，skywalking将会在至少一个条件满足的时候执行bulk操作
     concurrentRequests: 2
     resultWindowMaxSize: 10000
     metadataQueryMaxSize: 5000
     segmentQueryMaxSize: 200
receiver-sharing-server:
  default:
receiver-register:
  default: 
receiver-trace:
  default:
    bufferPath: ${SW_RECEIVER_BUFFER_PATH:../trace-buffer/}
    # 单位为MB
    bufferOffsetMaxFileSize: ${SW_RECEIVER_BUFFER_OFFSET_MAX_FILE_SIZE: 100}
bufferDataMaxFileSize: ${SW_RECEIVER_BUFFER_DATA_MAX_FILE_SIZE: 500}
   bufferFileCleanWhenRestart: ${SW_RECEIVER_BUFFER_FILE_CLEAN_WHEN_RESTART:false}
   # 实际采样率=1/10000 * sampleRate，默认为全采样
   sampleRate: ${SW_TRACE_SAMPLE_RATE: 10000}
receiver-jvm:
  default: 
receiver-clr:
  default: 
# service-mesh是6.0.0之后添加的新功能
service-mesh:
  default:
    bufferPath: ${SW_SERVICE_MESH_BUFFER_PATH:../mesh-buffer/}
    bufferOffsetMaxFileSize: ${SW_SERVICE_MESH_OFFSET_MAX_FILE_SIZE: 100}
    bufferDataMaxFileSize: ${SW_SERVICE_MESH_BUFFER_DATA_MAX_FILE_SIZE: 500}
    bufferFileCleanWhenRestart: ${SW_SERVICE_MESH_BUFFER_FILE_CLEAN_WHEN_RESTART: false}
istio-telemetry:
  default:
envoy-metric:
  default:
query:
  graphql:
  path: ${SW_QUERY_GRAPHQL_PATH:/graphql}
# 告警配置的具体配置在同级目录下的alarm-settings.xml中
alarm:
  default:
telemetry:
  none:
# 配置webapp服务的注册中心
configuration:
  none:
```

注：一般情况下，只需要配置cluster和storage部分，其它部分保持默认即可。

###### b、webapp.yml

主要是配置SkyWalking Webapp的。

```objectivec
# 配置服务端口
server:
  port: 8888
# 配置collector地址以及路径
collector: 
  # 和前面的config/application.yml中的query.path相对应
  path: /graphql
  ribbon: 
    ReadTimeout: 10000
    # 配置collector服务集群地址，以","进行分割
    listOfServers: 127.0.0.1:12800
```

注：服务运行后，SkyWalking UI监控端口为 8888，访问地址: [http://localhost:8888](https://links.jianshu.com/go?to=http%3A%2F%2Flocalhost%3A8888)

###### c、agent.conf

agent需要从SkyWalking 的安装包中拷贝出来，拷贝至需要监控的服务所在的服务器中(例如：我的服务启动在宿主机上，就需要将agent目录拷贝F:\tools\skywalking-agent中，最好一个服务一个目录一个配置)。

```csharp
# 代理的命名空间，默认为default-namespace
agent.namespace=skywalking_brief
# 服务名，推荐一个服务一个名称，这个将会在界面的拓补图中显示
agent.service_name=ijep-service-sys
# 每三秒追踪链采样数量，负数表示尽可能的多采集，默认为-1
agent.sample_n_per_3_secs=-1
# 配置认证，需要和backend服务中的认证配置相符
# agent.authentication=${SW_AGENT_AUTHENTICATION:XXX}

# 配置在单个segment中出现的span的最大数量，skywalking将用这个配置来估计应用的内存开销
# agent.span_limit_per_segment=${SW_AGENT_SPAN_LIMIT:300}

# 配置哪些资源不会被skywalking所捕获
# agent.ignore_suffix=${SW_AGENT_IGNORE_SUFFIX:.jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.mp3,.mp4,.html,.svg}

# 操作名称最大长度
# agent.operation_name_threshold=${SW_AGENT_OPERATION_NAME_THRESHOLD:500}

# 配置collector的地址，多个地址之间以“,”隔开
collector.backend_service=192.168.1.180:11800

### agent相关配置
logging.file_name=skywalking_luke.log
# 日志级别，默认为DEBUG
logging.level=INFO
logging.dir=F:\\logs\\skywalking
# 日志文件最大为300M一个
logging.max_file_size=${SW_LOGGING_MAX_FILE_SIZE:314572800}
# 历史日志文件的最大数量，当数量超过配置上限的时候，旧的日志文件将会被删除。负数或者0意味着该功能将关闭，默认为-1
# logging.max_history_files=${SW_LOGGING_MAX_HISTORY_FILES:-1}

# mysql插件配置
# 配置追踪sql参数，这样可以在sql错误的时候查看是否是参数所引起的
plugin.mysql.trace_sql_parameters=true
```

##### 4、运行SkyWalking服务端

```csharp
# 首先打开collector的监听端口
sudo firewall-cmd --zone=public --add-port=11800/tcp --permanent
# 然后打开webapp的服务端口
sudo firewall-cmd --zone=public --add-port=8888/tcp --permanent
# 然后重启防火墙
sudo systemctl restart firewalld
# 运行服务
sh /usr/local/skywalking/bin/startup.sh
# 查看日志
tail -f /usr/local/skywalking/logs/webapp.log
```

##### 5、运行SkyWalking代理端

```css
java -javaagent:F:\\skywalking-agent\\agent-sys\\skywalking-agent.jar -jar F:\\services\\ijep-service-sys.jar
```

注：-javaagent配置参数一定要在 -jar 之前。

#### 4、访问SkyWalking

● 地址：http://192.168.1.180:8888/
 ● 功能：仪表盘、拓扑图、追踪、告警以及指标对比



![img](https:////upload-images.jianshu.io/upload_images/11997591-849d325f14ca0fbe.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

作者：天生小包

链接：https://www.jianshu.com/p/5524b4545421

来源：简书

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。