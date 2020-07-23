---
title: 'Skywalking应用实践(一): oap服务的搭建'
date: 2020-07-23 19:34:33
categories:
    - 应用总结
tags: 
    - 监控
    - 分布式
    - 运维
---

# Skywalking应用实践(一): oap服务的搭建

## SkyWalking是什么？

SkyWalking（下面建成SW）是一个APM（应用程序性能监视器）系统。可以监控分布式服务的性能、分布式调用链路追踪、监控结果的可视化。



## SW的组成

SW由三部分组成：

1. skywalking-agent：需要和具体服务集成的探针包，有各种语言的版本
2. Skywalking-oap-server：用于收集和存储各个服务推送过来的指标数据，并会给UI系统提供查询接口
3. Skywalking-UI：可视化服务系统，以Web的方式提供可视化的数据展示





## 以Java项目为例子，SW版本是8.0.1为例子开始搭建整个监控系统

先下载SkyWalking的预编译包，下载地址是：https://skywalking.apache.org/zh/downloads/

选择需要的版本进行下载，下载完成后进行解压就能得到目录。实际搭建的服务的时候不会使用这样的方式，这里主要是说明涉及到的几个重要的配置文件可以在下载好的目录里面找到。真实部署的时候，一般都用Docker镜像进行部署。通过搜索DockerHub中的镜像，发现Apache官方给出了SW的的官方镜像的。

下载的包中几个比较重要的目录和文件

- agent目录：这个目录是探针的目录，里面包含了skywalking-agent.jar这个探针jar包
- agent/config/agent.config文件：这个是探针的配置文件
- config/application.yml文件：skywalking-oap系统的配置文件





### 搭建Skywalking-oap-server集群

数据收集服务的搭建有很多种方式。这里选择用Docker镜像的方式进行部署

镜像地址：https://hub.docker.com/r/apache/skywalking-oap-server

镜像的具体使用方法，请看镜像介绍页面的介绍。

先看配置文件：config/application.yml。这个是skywalking-oap-server的配置文件。相关的启动配置都在这里面

每个分类下面的selecor标识用于表示服务启用时选择读取哪一块的配置

```yaml
cluster:
  selector: ${SW_CLUSTER:standalone}
  standalone:
  # Please check your ZooKeeper is 3.5+, However, it is also compatible with ZooKeeper 3.4.x. Replace the ZooKeeper 3.5+
  # library the oap-libs folder with your ZooKeeper 3.4.x library.
  zookeeper:
    nameSpace: ${SW_NAMESPACE:""}
    hostPort: ${SW_CLUSTER_ZK_HOST_PORT:localhost:2181}
    # Retry Policy
    baseSleepTimeMs: ${SW_CLUSTER_ZK_SLEEP_TIME:1000} # initial amount of time to wait between retries
    maxRetries: ${SW_CLUSTER_ZK_MAX_RETRIES:3} # max number of times to retry
    # Enable ACL
    enableACL: ${SW_ZK_ENABLE_ACL:false} # disable ACL in default
    schema: ${SW_ZK_SCHEMA:digest} # only support digest schema
    expression: ${SW_ZK_EXPRESSION:skywalking:skywalking}
  kubernetes:
    watchTimeoutSeconds: ${SW_CLUSTER_K8S_WATCH_TIMEOUT:60}
    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector,release=skywalking}
    uidEnvName: ${SW_CLUSTER_K8S_UID:SKYWALKING_COLLECTOR_UID}
  consul:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    # Consul cluster nodes, example: 10.0.0.1:8500,10.0.0.2:8500,10.0.0.3:8500
    hostPort: ${SW_CLUSTER_CONSUL_HOST_PORT:localhost:8500}
    aclToken: ${SW_CLUSTER_CONSUL_ACLTOKEN:""}
  etcd:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    # etcd cluster nodes, example: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379
    hostPort: ${SW_CLUSTER_ETCD_HOST_PORT:localhost:2379}
```

通过配置文件我们可以看到，采集服务的搭建支持单例模式和集群模式，生产环境下为了高可用肯定要选择集群模式

集群模式中可以选择不同的集群协调中间件：

- zookeeper
- kubernetes
- consul
- etcd

选择自己合适的中间件来搭建集群



下面是核心的配置

```yaml
core:
  selector: ${SW_CORE:default}
  default:
    # Mixed: Receive agent data, Level 1 aggregate, Level 2 aggregate
    # Receiver: Receive agent data, Level 1 aggregate
    # Aggregator: Level 2 aggregate
    role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
    gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
    gRPCSslEnabled: ${SW_CORE_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_CORE_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_CORE_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAPath: ${SW_CORE_GRPC_SSL_TRUSTED_CA_PATH:""}
    downsampling:
      - Hour
      - Day
      - Month
    enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true} 
    dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5} 
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
    metricsDataTTL: ${SW_CORE_RECORD_DATA_TTL:7} # Unit is day
```

SW_CORE_REST_PORT：该环境变量可以指定暴力给UI服务查询的端口

SW_CORE_GRPC_PORT：该环境变量可以指定agent服务上传数据的端口

SW_CORE_RECORD_DATA_TTL：该环境变量可以指定数据的存放时间



下面部分是收集服务的数据存储类型的选择

```yaml
storage:
  selector: ${SW_STORAGE:elasticsearch}
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # Secrets management file in the properties format includes the username, password, which are managed by 3rd party tool.
    dayStep: ${SW_STORAGE_DAY_STEP:1} # Represent the number of days in the one minute/hour/day index.
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # The index shards number is for store metrics data rather than basic segment record
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} # Super data set has been defined in the codes, such as trace segments. This factor provides more shards for the super data set, shards number = indexShardsNumber * superDatasetIndexShardsFactor. Also, this factor effects Zipkin and Jaeger traces.
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000} # Execute the bulk every 1000 requests
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
    advanced: ${SW_STORAGE_ES_ADVANCED:""}
  elasticsearch7:
    nameSpace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
    dayStep: ${SW_STORAGE_DAY_STEP:1} # Represent the number of days in the one minute/hour/day index.
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # Secrets management file in the properties format includes the username, password, which are managed by 3rd party tool.
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # The index shards number is for store metrics data rather than basic segment record
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} # Super data set has been defined in the codes, such as trace segments. This factor provides more shards for the super data set, shards number = indexShardsNumber * superDatasetIndexShardsFactor. Also, this factor effects Zipkin and Jaeger traces.
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000} # Execute the bulk every 1000 requests
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
    advanced: ${SW_STORAGE_ES_ADVANCED:""}
  h2:
    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
    user: ${SW_STORAGE_H2_USER:sa}
    metadataQueryMaxSize: ${SW_STORAGE_H2_QUERY_MAX_SIZE:5000}
  mysql:
    properties:
      jdbcUrl: ${SW_JDBC_URL:"jdbc:mysql://localhost:3306/swtest"}
      dataSource.user: ${SW_DATA_SOURCE_USER:root}
      dataSource.password: ${SW_DATA_SOURCE_PASSWORD:root@1234}
      dataSource.cachePrepStmts: ${SW_DATA_SOURCE_CACHE_PREP_STMTS:true}
      dataSource.prepStmtCacheSize: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_SIZE:250}
      dataSource.prepStmtCacheSqlLimit: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_LIMIT:2048}
      dataSource.useServerPrepStmts: ${SW_DATA_SOURCE_USE_SERVER_PREP_STMTS:true}
    metadataQueryMaxSize: ${SW_STORAGE_MYSQL_QUERY_MAX_SIZE:5000}
  influxdb:
    # InfluxDB configuration
    url: ${SW_STORAGE_INFLUXDB_URL:http://localhost:8086}
    user: ${SW_STORAGE_INFLUXDB_USER:root}
    password: ${SW_STORAGE_INFLUXDB_PASSWORD:}
    database: ${SW_STORAGE_INFLUXDB_DATABASE:skywalking}
    actions: ${SW_STORAGE_INFLUXDB_ACTIONS:1000} # the number of actions to collect
    duration: ${SW_STORAGE_INFLUXDB_DURATION:1000} # the time to wait at most (milliseconds)
    fetchTaskLogMaxSize: ${SW_STORAGE_INFLUXDB_FETCH_TASK_LOG_MAX_SIZE:5000} # the max number of fetch task log in a request
```

可以看到服务支持的后端输出存储有：

- elasticsearch
- h2
- mysql
- influxdb

官方推荐使用elasticsearch作为后端存储

- SW_STORAGE_ES_CLUSTER_NODES：指向elasticsearch服务的地址
- SW_ES_USER：elasticsearch的用户名
- SW_ES_PASSWORD：elasticsearch的密码





启动Docker镜像

可以根据配置文件的中的配置，可以传入环境变量进行配置的覆盖

```shell
$ docker run --name oap --restart always -d \
-e SW_STORAGE=elasticsearch \
-e SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200 \
apache/skywalking-oap-server

```



## 参考

- 项目地址：https://github.com/apache/skywalking

- 文档：https://github.com/apache/skywalking/tree/master/docs