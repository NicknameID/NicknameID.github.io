---
title: Feign在实际项目中的应用实践总结
date: 2020-06-11 10:59:17
categories:
    - 应用总结
tags: 
    - Feign
    - HttpClient
    - 指标监控
    - Spring
cover: https://raw.githubusercontent.com/OpenFeign/feign/master/src/docs/overview.png
---

# Feign在实际项目中的应用实践总结

## Feign在是什么？

> 是一个声明式的HTTP请求处理库，可以将命令式的http请求的编程，更改为声明式的http请求编程。

下面是传统的命令式编程模式和Feign所代表的声明式编程模式的对比，可以清晰的看到声明式的代码逻辑比命令式更加的简洁，就像本地调用一样。

命令式:

```java
public class MyApp {
  public static void main(String... args) {
    CloseableHttpResponse httpResponse = 
      httpClientTool.doGet("https://api.github.com//repos/" + owner +"/"+ repo +"/contributors");
    String responseBody = EntityUtils.toString(httpResponse.getEntity());
    List<Contributor> contributors = JsonTool.parseArray(responseBody, Contributor.class);
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```



声明式:

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);

}


public class MyApp {
  public static void main(String... args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");

    // Fetch and print a list of the contributors to this library.
    List<Contributor> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```



从上面的对比中可以看到。通过命令式的http请求编程，过程复杂需要编码请求以及请求请求参数的定义，以及响应结果的字符串解析为Java对象的全过程。但声明式的http请求编程，只要事先定义好一个请求的模板（interface）剩下来的就直接交给Feign通过动态代理，将请求的细节托管不需要编码者处理具体的请求逻辑。



## Feign可以帮我解决什么问题，有什么优点？

Feign可以让我们书写http请求代码时更加容易，这也是他官方给Feign这个库的定义

> Feign makes writing java http clients easier

声明式编程和命令式编程相比的优点

1. 通过接口进行http请求的定义，可以做到集中管理请求，而且不用编写请求的细节。
2. 通过http请求的控制反转将http请求的细节交给Feign来完成，提高的请求编码的稳定性和可靠性。
3. 通过使用动态代理和基于注解的模板模式的设计，使请求编码的代码量大幅缩减，减少重复代码，提高开发效率。



### 什么情况下不适合使用Feign？

Feign也不是所有场景下都适用的，比如在请求地址不确定，参数不确定的编程的业务场景中，Feign这种需要预先定义好请求路径和目标主机地址的声明式模板反而会让代码灵活性的降低。这种场景下不适合Feign的使用，请选择命令式的http编程模式。



## Feign是如何实现声明式的HTTP请求定义的？

1. 通过动态代理技术根据用户定义的接口自动生成代理类来执行具体的请求操作
2. Feign只提供了HTTP客户端的接口抽象而并没有实现HTTP客户端的具体实现，所以说Feign并不是一个HTTP请求客户端，而需要和常用的HTTP请求客户端进行组合使用，其目的是简化HTTP客户端的使用
3. 提供序列化和反序列化的接口抽象，使用者可以按照自己的需求灵活的指定具体的序列化和反序列化实现类库
4. 支持指标监控和常用熔断器的集成
5. 支持多种Restful接口的定义规范，可以和已有微服务架构进行良好的集成





## Spring项目如何整合进Feign？

常用的基础依赖

```xml
<!-- https://mvnrepository.com/artifact/io.github.openfeign/feign-core -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
    <version>11.0</version>
</dependency>

<!-- https://mvnrepository.com/artifact/io.github.openfeign/feign-jackson -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-jackson</artifactId>
    <version>11.0</version>
</dependency>

<!-- https://mvnrepository.com/artifact/io.github.openfeign/feign-httpclient -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>11.0</version>
</dependency>

```

首先需要`feign-cor`引入核心代码，`feign-jackson`基于JSON的Restful接口的的序列化和反序列化工具库，`feign-httpclient`Feign集成Apach-Http-Client作为请求的具体客户端实现。

这里需要注意，如果项目不是基于Spring-Cloud的项目的微服务，不建议直接使用 `spring-cloud-starter-openfeign`，虽然他用起来很方便，可以实现自动配置，但不利于Feign和HTTP请求客户端的细节把控，而且会引入需要没必要的依赖。简单的项目直接使用核心jar包就可以了。



### 定义声明式接口

```java
// 接口定义
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);

}

// 数据类型定义
public static class Contributor {
  String login;
  int contributions;
}

// 数据类型定义
public static class Issue {
  String title;
  String body;
  List<String> assignees;
  int milestone;
  List<String> labels;
}
```



### 配置并生成一个代理类对象

```java
@Configuration
public class FeignClientConfig {
  // 系统中有默认jackson对象直接拿来用
  @Resource
  private ObjectMapper jacksonMapper;
  
  @Bean
  public ApacheHttpClient apacheHttpClient() {
    // 这里可以定义httpclient的具体配置
    CloseableHttpClient httpClinet = HttpClients.custom().build();
    return new ApacheHttpClient(httpClinet);
  }

  @Bean
  public JacksonDecoder jacksonDecoder() {
    // 这里可以直接使用SpringBoot自带的jackson作为反序列化器
    return new JacksonDecoder(jacksonMapper);
  }
  
  @Bean
  public JacksonEncoder jacksonEncoder() {
    // 这里可以直接使用SpringBoot自带的jackson作为序列化器
    return new JacksonEncoder(jacksonMapper);
  }
  
  
  // 生成一个实现GitHub这个接口的了代理对象并将其注入为一个Spring的Bean
  @Bean
  public GitHub gitHubFeignClient(
		ApacheHttpClient apacheHttpClient,
    JacksonEncoder jacksonEncoder,
    JacksonDecoder jacksonDecoder
  ) {
      return Feign.builder()
        // 自定义请求客户端
        .client(apacheHttpClient)
        // 自定义请求体序列化器
        .encoder(jacksonEncoder)
        // 自定义响应体反序列化器
        .decoder(jacksonDecoder)
        // 定义目标主机
        .target(GitHub.class, "https://api.github.com");
    }
}
```



### 发起请求

```java
public class MyApp {
  public static void main(String... args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");

    // Fetch and print a list of the contributors to this library.
    List<Contributor> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```



## 如何自定义底层HTTP客户端的实现？

要自定义HTTP客户端的实现，可以通过`Feign.builder().client()`的方法来指定

```java
public Feign.Builder client(Client client) {
  this.client = client;
  return this;
}
```

可以看到自定义的Client实例需要实现Client接口

所以我们需要使用依赖jar包

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>11.0</version>
</dependency>
```

查看类的内部，ApacheHttpClient实现了Client接口，并且有一个使用HttpClient类作为参数的构造函数
```java
public final class ApacheHttpClient implements Client {
  public ApacheHttpClient(HttpClient client) {
        this.client = client;
    }
}
```

所以我们只需要通过ApacheHttpClient的构造函数传入HttpClient的实例对象，就可以完成Feign和HttpClient的适配工作

```java
Feign.builder().client(new ApacheHttpClient(httpClientInstance));
```





## 如何实现全局HTTP请求的统一 tracer 日志？

在企业应用场景中，统一的HTTP的Tracer是非常有必要的。为什么？往小了说可以帮你定位问题，往大了说就能保住你饭碗。通常的一个Web系统，都会接入其他三方的资源和服务来扩充自己的服务能力，而常用的接入方案就是Rest接口。而三方的服务你是没办法保证的，只能保证自己的服务是没有问题的。在线上如果出现服务调用失败，统一的Tracer日志可进行错误的定位，是三方的服务接口出问题还是自己的代码写的有问题。在很多情况下，自己做为服务提供者，往往在出现问题后客户会要求查看接口请求日志。如果没有做HTTP的统一Tracer日志，只能空口评说了，搞不好饭碗都丢了。总之一句话，系统和系统交互的边界上一定要打日志，一定要打日志，一定要打日志（重要的事情说三遍！）



### 那如何给Feign接入统一的日志呢？

根据官方文档，Feign是支持Logger配置的，使用者可以选择适合自己的Logger进行请求日志的记录。但是Feign的日志打印有一个很严重的问题，他是分步打印的。也就是说一个HTTP请求，请求发起打印一行，请求结束打印一行。想想这会发生什么事情？

在系统打规模并发的情况下，一个请求的发起阶段和响应阶段的日志并不是在一起的，中间可能穿插着其他请求的日志记录。当我们需要去追溯日志的时候，发现根本没有办法定位到哪一个请求出了问题，只能找到请求的某一个阶段的日志。

现在有几个需求需要实现。

1. 将请求的请求阶段的数和响应阶段的数据放在同一行日志中
2. 每个请求都会带有唯一的请求ID，服务调用过程中都会携带这个请求ID



实现方式就是使用HttpClient的请求拦截器和响应拦截器进行对请求的拦截。HttpClient在请求过程中有一个请求级别的对象，RequestContext。这个对象在整个请求的生命周期中都可以访问和读写。那我们可以使用这个请求上下文对象用来记录请求过程中的参数



### 请求日志记录的代码实现



先定义我们的请求日志需要记录请求生命周期的哪些字段？

```java
// http请求日志字段枚举类
public enum HttpLogEnum {
        protocolVersion,
        method,
        url,
        requestHeaders,
        requestBody,
        responseHeaders,
        responseCode,
        responseBody
}
```



定义请求拦截器，将请求阶段需要记录的参数绑定到请求上下文对象上

```java
// 实现HttpRequestInterceptor请求拦截器接口
public static class RequestInterceptor implements HttpRequestInterceptor {
  // 实现process方法
  @Override
  public void process(HttpRequest httpRequest, HttpContext httpContext) throws HttpException, IOException {
    // 先将请求对象 httpRequest 进行包装操作方便获取请求阶段数据
    final HttpRequestWrapper httpRequestWrapper = HttpRequestWrapper.wrap(httpRequest);
    // 获取请求的协议版本
    httpContext.setAttribute(HttpLogEnum.protocolVersion.name(), httpRequestWrapper.getProtocolVersion().toString());
    // 获取请求的Method(GET、POST、PUT...)
    httpContext.setAttribute(HttpLogEnum.method.name(), httpRequestWrapper.getMethod());
    // 请求的url
    httpContext.setAttribute(HttpLogEnum.url.name(), httpRequestWrapper.getURI().toString());
    // 请求的请求头
    httpContext.setAttribute(HttpLogEnum.requestHeaders.name(), getHeaderMap(httpRequestWrapper.getAllHeaders()));

    // 请求的请求体
    if (httpRequestWrapper instanceof HttpEntityEnclosingRequest) {
      final HttpEntity entity = ((HttpEntityEnclosingRequest) httpRequestWrapper).getEntity();
      String content = EntityUtils.toString(entity, ENCODING);
      httpContext.setAttribute(HttpLogEnum.requestBody.name(), content);
      // 由于请求体前面已被读取，读取后需要重新写入请求体
      ((HttpEntityEnclosingRequest) httpRequest).setEntity(new StringEntity(content, ContentType.get(entity)));
    }else {
      // 没有携带请求体的请求
      httpContext.setAttribute(HttpLogEnum.requestBody.name(), null);
    }
  }
}
```



定义响应拦截器，获取响应的结果

```java
// 实现HttpResponseInterceptor响应拦截接口
public static class ResponseInterceptor implements HttpResponseInterceptor {
  @Override
  public void process(HttpResponse httpResponse, HttpContext httpContext) throws HttpException, IOException {
    // 获取响应头
    httpContext.setAttribute(HttpLogEnum.responseHeaders.name(), getHeaderMap(httpResponse.getAllHeaders()));
    // 获取HTTP响应码
    httpContext.setAttribute(HttpLogEnum.responseCode.name(), httpResponse.getStatusLine().getStatusCode());
    final HttpEntity entity = httpResponse.getEntity();
    String responseBody = EntityUtils.toString(entity, ENCODING);
    // 获取请求的响应体
    httpContext.setAttribute(HttpLogEnum.responseBody.name(), responseBody);
    // 重新写入响应体
    httpResponse.setEntity(new StringEntity(responseBody, ContentType.get(entity)));
    // 将httpContext对象保存的字段进行获取包装成一个Map对象
    Map<String, Object> log = new HashMap<>(HttpLogEnum.values().length);
    // 进行字段的读取
    for (HttpLogEnum item : HttpLogEnum.values()) {
      log.put(item.name(), httpContext.getAttribute(item.name()));
    }
    // 通过json工具对Map序列化成json文本，并通过标准输出进行打印
    System.out.println(JsonUtil.toJson(log));
  }
}
```



将请求/响应拦截器绑定到HttpClient对象上，并注册到Spring的容器中去

```java
@Configuration
public class HttpClinetConfig {
    @Bean
    public CloseableHttpClient httpClient() {
        return HttpClients.custom()
                  // 请求日志拦截器
                .addInterceptorFirst(new RequestInterceptor())
                  // 响应日志拦截器
                .addInterceptorFirst(new ResponseInterceptor())
                .build();
    }
}
```



将HttpClient指定为Feign的客户端实现

```java
@Configuration
public class FeignClientConfig {
	/*
	
	.....其他代码
	*/
  
  // 注入HttpClient的Bean
  @Resource
  private CloseableHttpClient httpClinet;
  
  @Bean
  public ApacheHttpClient apacheHttpClient() {
    // 将HttpClient包装成Feign的ApacheHttpClient
    return new ApacheHttpClient(httpClinet);
  }
  
  // 生成一个实现GitHub这个接口的了代理对象并将其注入为一个Spring的Bean
  @Bean
  public GitHub gitHubFeignClient(
		ApacheHttpClient apacheHttpClient,
    JacksonEncoder jacksonEncoder,
    JacksonDecoder jacksonDecoder
  ) {
      return Feign.builder()
        // 设置上面定义的请求客户端实现
        .client(apacheHttpClient)
        // 自定义请求体序列化器
        .encoder(jacksonEncoder)
        // 自定义响应体反序列化器
        .decoder(jacksonDecoder)
        // 定义目标主机
        .target(GitHub.class, "https://api.github.com");
    }
}
```





## 如何集成指标监控？以及数据的收集工作？

增加依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.github.openfeign</groupId>
  <artifactId>feign-micrometer</artifactId>
  <version>11.0</version>
</dependency>
```

配置指标收集器

```java
@Configuration
public class FeignClientBuilder {
  /* 其他代码  */
  
  // 注入actuator的指标注册器
    @Resource
    private MeterRegistry meterRegistry;

  // 将meterRegistry包装成feign的MicrometerCapability类
    @Bean
    public MicrometerCapability micrometerCapability() {
        return new MicrometerCapability(meterRegistry);
    }

    @Bean
    public BananaFeignClient bananaFeignClient(
            ApacheHttpClient apacheHttpClient,
            JacksonEncoder jacksonEncoder,
            JacksonDecoder jacksonDecoder,
      			MicrometerCapability micrometerCapability
    ) {
        return Feign.builder()
          // 将micrometerCapability绑定到Feign上
                .addCapability(micrometerCapability)
                .client(apacheHttpClient)
                .encoder(jacksonEncoder)
                .decoder(jacksonDecoder)
                .target(BananaFeignClient.class, bananaHost);
    }
}

```

增加actuator配置暴露metrics端点，不配置的话默认使用 health和info端点暴露

```yaml
management:
  endpoints:
    web:
      exposure:
        include:
          - httptrace
          - info
          - health
          - metrics
```

重启服务后，查看http://localhost:8080/actuator/metrics

```json
{
    "names": [
        "jvm.buffer.count",
        "jvm.buffer.memory.used",
        "jvm.buffer.total.capacity",
        "jvm.classes.loaded",
        "jvm.classes.unloaded",
        "jvm.gc.live.data.size",
        "jvm.gc.max.data.size",
        "jvm.gc.memory.allocated",
        "jvm.gc.memory.promoted",
        "jvm.gc.pause",
        "jvm.memory.committed",
        "jvm.memory.max",
        "jvm.memory.used",
        "jvm.threads.daemon",
        "jvm.threads.live",
        "jvm.threads.peak",
        "jvm.threads.states",
        "logback.events",
        "process.cpu.usage",
        "process.files.max",
        "process.files.open",
        "process.start.time",
        "process.uptime",
        "system.cpu.count",
        "system.cpu.usage",
        "system.load.average.1m",
        "tomcat.sessions.active.current",
        "tomcat.sessions.active.max",
        "tomcat.sessions.alive.max",
        "tomcat.sessions.created",
        "tomcat.sessions.expired",
        "tomcat.sessions.rejected"
    ]
}
```

发现并没有Feign相关的指标，因为没有发起过请求，所以没有数据。我们先发起一下请求后看到

```json
{
    "names": [
        "feign.Client",
        "feign.Feign",
        "feign.codec.Decoder",
        "feign.codec.Decoder.response_size",
        ......
    ]
}
```

多了Feign相关的指标

查看指标细节 http://localhost:8080/actuator/metrics/feign.Client

```json
{
    "name": "feign.Client",
    "description": null,
    "baseUnit": "seconds",
    "measurements": [
        {
            "statistic": "COUNT",
            "value": 1
        },
        {
            "statistic": "TOTAL_TIME",
            "value": 0.169881274
        },
        {
            "statistic": "MAX",
            "value": 0.043015818
        }
    ],
    "availableTags": [
        {
            "tag": "method",
            "values": [
                "validateUserToken"
            ]
        },
        {
            "tag": "host",
            "values": [
                "blog.mufeng.tech"
            ]
        },
        {
            "tag": "client",
            "values": [
                "tech.mufeng.feign.defined.ExampleFeignClient"
            ]
        }
    ]
}
```

接入prometheus监控后就能集成监控和报警的功能

## 参考资料

[Feign开源项目仓库地址](https://github.com/OpenFeign/feign)
