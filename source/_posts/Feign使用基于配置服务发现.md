---
title: Feign使用基于配置服务发现
date: 2020-11-01 14:16:33
categories:
    - 应用总结
tags: 
    - Feign
    - HttpClient
    - Spring
cover: https://i.loli.net/2020/06/11/oTC3KN8FzuxpkHq.png
---

# Feign使用基于配置服务发现

之前写了篇[《Feign在实际项目中的应用实践总结》](/public/2020/06/11/Feign在实际项目中的应用实践总结/)总结了在一般项目中如何使用Feign这个提升开发效率的利器。最近在看Feign的文档的时候发现了之前遗漏的一些点，所以写了这篇文章进行补充。


## pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```



## JavaConfig

```java
@Slf4j
@Configuration
@EnableFeignClients
public class FeignConfig {
    @Resource
    private SimpleDiscoveryProperties simpleDiscoveryProperties;

  // 在K8s环境中不需要服务注册中心可以自定义一个DiscoveClient的Bean，作为简单的一个服务列表
    @Bean
    public DiscoveryClient discoveryClient() {
        log.info(JSON.toJSONString(simpleDiscoveryProperties)); // 这里打印了加载的配置参数
        return new SimpleDiscoveryClient(simpleDiscoveryProperties);
    }
}
```



## 配置（在里面写服务的路由配置）：

```yaml
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false
    discovery:
      client:
        simple:
          instances:
            feign-attempt-client:
              - uri: http://localhost:8080
```



## 为什么要使用本地的服务列表？

服务部署在K8s环境中，可以使用k8s提供的DNS解析和负载均衡功能。所以不需要再引入一个服务注册中心组建导致复杂性的增加。



### 使用

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "feign-attempt-client")
public interface FeignServerClient {
    @GetMapping("/feign/server/hello-world/hello")
    String hello();
}
```



### 运行单元测试

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import javax.annotation.Resource;
import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class FeignServiceClientTest {
    @Resource
    private FeignServiceClient feignServiceClient;

    @Test
    void hello() {
        final String res = feignServiceClient.hello(); // HTTP请求返回字符串“Hi!”
        assertEquals(res, "Hi!");
    }
}
```



### 运行细节

加载的服务路由配置，可以看到instances的对象加载了我们定义的feign-attempt-client服务，它包含了一个服务实例。为什么只要定义个服务实例呢？因为我们服务是部署在K8s上的。k8s会帮我们做多实例的负载均衡的工作

```json
{
    "instances": {
        "feign-attempt-client": [{
            "host": "localhost",
            "metadata": {},
            "port": 8080,
            "secure": false,
            "serviceId": "feign-attempt-client",
            "uri": "http://localhost:8080"
        }]
    },
    "local": {
        "host": "172.16.8.55",
        "metadata": {},
        "port": 8080,
        "secure": false,
        "serviceId": "application",
        "uri": "http://172.16.8.55:8080"
    },
    "order": 0
}
```



