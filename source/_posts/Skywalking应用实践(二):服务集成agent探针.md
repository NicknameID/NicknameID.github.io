---
title: 'Skywalking应用实践(二):服务集成agent探针'
date: 2020-07-24 14:40:33
tags:
    - 监控
    - 分布式
    - 运维
categories:
    - 应用总结
---

# Skywalking应用实践(二):服务集成agent探针

SW支持多种语言的的探针，对于原有的服务代码没有侵入性



集成原理：

在在java的启动参数重添加javaagent参数指向skywalking-agent.jar。下面用一个SpringBoot项目为例子构建一个集成SW探针的Docker镜像



```dockerfile
# 打包SpringBoot服务
FROM maven:3-jdk-11 as builder
WORKDIR /data
COPY . .
RUN mvn clean package -B -DskipTests

# 使用jdk11-jre作为运行环境
FROM openjdk:11-jre

ENV TZ Asia/Shanghai
# 设置该服务的名字，到时候会是UI界面上的服务标识
ENV SW_AGENT_NAME example-app-service
# 设置需要投递数据目的地的oap服务的地址和端口
ENV SW_AGENT_COLLECTOR_BACKEND_SERVICES you-oap-server-address:11800

WORKDIR /
# 下载SkyWalking
ADD https://mirrors.tuna.tsinghua.edu.cn/apache/skywalking/${SKYWALKING_VERSION}/apache-skywalking-apm-${SKYWALKING_VERSION}.tar.gz /

# 解压并删除不需要的监控插件（这里删除了Kafka相关的监控）
RUN tar -zxvf /apache-skywalking-apm-${SKYWALKING_VERSION}.tar.gz && \
    mv apache-skywalking-apm-bin skywalking && \
    mv /skywalking/agent/optional-plugins/apm-trace-ignore-plugin* /skywalking/agent/plugins/ && \
    rm /skywalking/agent/plugins/*-kafka-plugin-*.jar

# 复制构建的应用jar包
WORKDIR /data
COPY --from=builder /data/target/APP.jar .

# 通过javaagent指定SW的探针jar包，并启动APP.jar
CMD java -javaagent:/skywalking/agent/skywalking-agent.jar -jar APP.jar
```



看下agent的配置文件

```yaml
# 服务的标识
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}

# 后端oap服务的地址
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

# 日志等级
logging.level=${SW_LOGGING_LEVEL:INFO}

# 日志文件的大小，超过后将写到新的日志文件中
# Logging max_file_size, default: 300 * 1024 * 1024 = 314572800
logging.max_file_size=${SW_LOGGING_MAX_FILE_SIZE:314572800}

# 保留最老的历史文件数量，-1表示删除立马删除最老的文件
logging.max_history_files=${SW_LOGGING_MAX_HISTORY_FILES:-1}
```



