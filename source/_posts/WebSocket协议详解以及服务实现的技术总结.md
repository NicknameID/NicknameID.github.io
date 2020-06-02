---
title: WebSocket协议详解以及服务实现的技术总结
date: 2020-06-02 17:34:33
categories:
    - 应用总结
tags: 
    - WebSocket
    - Spring
cover: https://i.loli.net/2020/06/02/AU7RhwT5BvkrPqd.png
---

# WebSocket协议详解以及WebSocket服务实现的技术总结



## WebSocket是什么？（下面简称ws）

> WebSocket是一种在**单个TCP**连接上进行**全双工**通信的网络传输协议。客户端与服务端完成一次握手后，两者之间可以创建持久性的连接，并进行双向数据传输。



## ws技术可以解决什么样的业务场景问题？



### 业务场景

客户端需要持续监测服务器数据变动的业务场景下，如股票交易、抢单、即时通讯、多人协作服务等。这些业务场景对于消息的即时性有很高的要求，而且还要保证消息的可靠传递



## WS的技术背景

在ws协议出现之前，对于实时性要求高的业务场景往往采用的是基于**HTPP**协议的的轮询技术。也就是每隔一段时间向服务器发送请求，如果有最新的数据就返回给客户端。这种传统的模式有很明显的缺点，即客户端向服务器不断的发送请求，然而HTTP请求与回复的过程中包含很多较长的**头部**（因为HTTP协议是基于文本的协议），其中真正有效的数据可能只是很小的一部分，会导致消耗很多无效的带宽资源。而且消息的实时性的指标与轮询的频率成正相关，但是频率越高所消耗的带宽资源就越高，随着互联网用户规模的庞大，显然遇到了瓶颈。

对于HTTP轮询协议的改进，就出现了Comet技术。Comet中普遍采用的[HTTP长连接](https://zh.wikipedia.org/wiki/HTTP持久链接)也会消耗服务器资源。本质上是将HTTP连接的超时时间强制延长，来减少带宽的浪费，但本质上还是HTTP技术，需要反复的发出请求。

ws技术的出现解决了上面的问题。ws通过兼容HTTP协议常用的80和443端口来实现客户端和服务器的双向通信，可以绕过大多数防火墙的限制。



## ws技术的优点

- 减少控制开销，消息头部只有2至10字节，相比于HTTP协议的头部明显减少
- 更强的实时性，由于是全双工的通道，在客户端和服务可以同时向对方发送消息，相对于HTTP请求需要等待客户端发起请求服务端才能响应，延迟明显更少
- 保持连接状态（有状态的连接），ws和http协议的不同点就是，http协议是无状态的协议，业务上为了保证连接的状态，每次进行通信都会携带状态消息（如身份认证等）。而ws协议是有状态协议，彼此之间通信无需重复传递状态消息
- 可以传输二进制数据，相对HTTP，可以更轻松地处理二进制内容。



## ws的的协议细节

ws连接的创建需要客户端首先发起请求连接，而握手请求使用的是HTTP协议。在客户端的请求头上的**Upgrade**字段是**websocket**，表明需要升级为ws协议进行通讯。服务器在收到请求后，返回101状态码表示服务理解了客户端的请求，并将通过Upgrade消息头通知客户端采用不同的协议来完成这个请求。



![WebSocket握手时序图](https://i.loli.net/2020/06/02/AU7RhwT5BvkrPqd.png)

### 典型的握手请求

客户端HTTP请求

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

服务端响应

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### 字段说明

- Connection必须设置Upgrade，表示客户端希望连接升级
- Upgrade字段必须设置Websocket，表示希望升级到Websocket协议。
- Sec-WebSocket-Key是随机的字符串，服务器端会用这些数据来构造出一个SHA-1的信息摘要。把“Sec-WebSocket-Key”加上一个特殊字符串“258EAFA5-E914-47DA-95CA-C5AB0DC85B11”，然后计算[SHA-1](https://zh.wikipedia.org/wiki/SHA-1)摘要，之后进行[Base64](https://zh.wikipedia.org/wiki/Base64)编码，将结果做为“Sec-WebSocket-Accept”头的值，返回给客户端。如此操作，可以尽量避免普通HTTP请求被误认为Websocket协议。
- Sec-WebSocket-Version 表示支持的Websocket版本。RFC6455要求使用的版本是13，之前草案的版本均应当弃用。
- Origin字段是可选的，通常用来表示在浏览器中发起此Websocket连接所在的页面，类似于[Referer](https://zh.wikipedia.org/wiki/HTTP来源地址)。但是，与Referer不同的是，Origin只包含了协议和主机名称。
- 其他一些定义在HTTP协议中的字段，如[Cookie](https://zh.wikipedia.org/wiki/Cookie)等，也可以在Websocket中使用。

### 数据帧格式

从左到右，单位是Bit，[RFC6455参考，5.2章节](https://tools.ietf.org/html/rfc6455#section-5.2)

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

##### FIN：1bit

如果是1代表这是单条消息，没有后续分片了。而如果是0代表，代表此数据帧是不是一个完整的消息，而是一个消息的分片，并且不是最后一个分片后面还有其他分片



##### RSV1, RSV2, RSV3:  1 bit each

必须是0，除非客户端和服务端使用WS扩展时，可以为非0。



##### Opcode: 4bit

这个为操作码，表示对后面的有效数据荷载的具体操作，如果未知接收端需要断开连接

- ％x0：表示连续帧

- ％x1：表示文本帧

- ％x2：表示二进制帧

- ％x3-7：保留用于其他非控制帧

- ％x8：表示连接关闭

- ％x9：表示ping操作

- ％xA：表示pong操作

- ％xB-F：保留用于其他控制帧



##### Mask:  1bit

是否进行过掩码，比如客户端给服务端发送消息，需要进行掩码操作。而服务端到客户端不需要



##### Payload Length:  7 bits, 7+16 bits, or 7+64 bits

“有效载荷数据”的长度（以字节为单位）：如果为0-125，则为有效载荷长度。 如果为126，则以下2个字节解释为16位无符号整数是有效载荷长度。 如果是127，以下8个字节解释为64位无符号整数（最高有效位必须为0）是有效载荷长度。 多字节长度数量以网络字节顺序表示。 注意在所有情况下，必须使用最小字节数进行编码长度，例如124字节长的字符串的长度不能编码为序列126、0、124。有效载荷长度是“扩展数据”的长度+“应用程序数据”。 “扩展数据”的长度可以是零，在这种情况下，有效负载长度是 “应用程序数据”。



##### Masking-key:  0 or 4 bytes (32bit)

所有从客户端传送到服务端的数据帧，数据载荷都进行了掩码操作，Mask为1，且携带了4字节的Masking-key。如果Mask为0，则没有Masking-key。



##### Payload data:  (x+y) bytes

“有效载荷数据”定义为串联的“Extension data”与“Application data”。

-  Extension data:  x bytes

如果没有协商使用扩展的话，扩展数据数据为0字节。所有的扩展都必须声明扩展数据的长度，或者可以如何计算出扩展数据的长度。此外，扩展如何使用必须在握手阶段就协商好。如果扩展数据存在，那么载荷数据长度必须将扩展数据的长度包含在内。

- Application data:  y bytes

任意的应用数据，在扩展数据之后（如果存在扩展数据），占据了数据帧剩余的位置。载荷数据长度 减去 扩展数据长度，就得到应用数据的长度。




## spring体系下如何搭建ws服务？

采用SpringBoot搭建WS服务



### 项目依赖

POM.xml

```xml
<!-- 需要的依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```
![](https://i.loli.net/2020/06/02/lHDMrcfYRLVNhQI.png)

可以看到`srping-boot-starter-websocket`包含了`spring-boot-starter-web`，所以我们不需要再引入服务器依赖。其中还用到了，`spring-message`和`spring-websocket`

![](https://i.loli.net/2020/06/02/ZTq4EOUHWPIz2h5.png)

可以看到`srping-boot-starter-websocket`默认使用tomcat作为web服务器，所以我们后续可以进行HTTP接口的开发

依赖搞定之后需要，进行ws服务的实现。要实现ws服务器，需要实现一下几个部分，与上面的ws通讯过程对应

1. 握手拦截处理器
2. 消息处理器
3. Session管理器（可选，简单应用可以不用session管理器）
4. WS配置Bean



### 握手拦截处理器

```java
/*
握手处理器，用于客户端的握手请求
需要实现HandshakeInterceptor接口并注册成spring的一个Bean
*/
@Component
@Slf4j
public class CustomHandshakeInterceptor implements HandshakeInterceptor {
    public final static String TOKEN = "token";
    public final static String CHANNEL_ID = "channelId";

  // 握手前的调用，可在这里进行请求的校验工作（如权限的校验）
    @Override
    public boolean beforeHandshake(
      ServerHttpRequest serverHttpRequest, 
      ServerHttpResponse serverHttpResponse, 
      WebSocketHandler webSocketHandler, 
      Map<String, Object> attributes) throws Exception 
    {
        final String queryString = serverHttpRequest.getURI().getQuery();
        final Map<String, String> queryMap = mapQueryString(queryString);
        if (queryMap.containsKey(TOKEN) && queryMap.containsKey(CHANNEL_ID)) {
          // attributes是可以用来绑定一些自定义的数据到当前session上，在session的整个生命周期内都可以获取到
            attributes.put(TOKEN, queryMap.get(TOKEN));
            attributes.put(CHANNEL_ID, queryMap.get(CHANNEL_ID));
          // 校验成功返回true，失败返回false，拒绝连接
            return true;
        }
        return false;
    }

    private Map<String, String> mapQueryString(String queryString) {
        Map<String, String> paramMap = new HashMap<>();
        if (StringUtils.isEmpty(queryString)) {
            return paramMap;
        }
        final String[] kvArray = queryString.split("&");
        for (String kv : kvArray) {
            final String[] kvPair = kv.split("=");
            if (kvPair.length != 2) {
                continue;
            }
            String key = kvPair[0];
            String value = kvPair[1];
            if (StringUtils.isNotEmpty(key) && StringUtils.isNotEmpty(value)) {
                paramMap.put(key, value);
            }
        }
        return paramMap;
    }
  
  
    @Override
    public void afterHandshake(
      ServerHttpRequest serverHttpRequest, 
      ServerHttpResponse serverHttpResponse, 
      WebSocketHandler webSocketHandler, 
      Exception e) 
    {
      // 握手之后调用
    }
}
```



### 消息处理器

消息处理器是ws服务的核心处理器，客户端发送来的消息都会进来进行处理

```java
/*
消息处理器，用处接收来自客户端的请求
需要继承AbstractWebSocketHandler这个抽象类来实现自己的自定义消息处理器
TextWebSocketHandler是用于处理文本消息处理器，也是AbstractWebSocketHandler的派生类
将CustomWebSocketHandler注册成spring的一个Bean
*/
@Component
@Slf4j
public class CustomWebSocketHandler extends TextWebSocketHandler {
    @Resource
    private WsSessionManager wsSessionManager;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        final Object channelId = session.getAttributes().get(CustomHandshakeInterceptor.CHANNEL_ID);
        if (Objects.isNull(channelId)) {
            throw new RuntimeException("ID获取异常");
        }
        final Object wsTmpToken = session.getAttributes().get(CustomHandshakeInterceptor.TOKEN);
        if (Objects.isNull(wsTmpToken)) {
            throw new RuntimeException("认证token丢失");
        }
        // 托管session
        final boolean addSuccess = wsSessionManager.add(channelId.toString(), session);
        if (!addSuccess) {
            session.close(CloseStatus.NORMAL.withReason("频道被占用，请更换频道"));
        }
    }
    
    // 来自客户端的消息在此处理
    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) throws IOException {
        log.debug("客户端消息: {}", message.getPayload());
    }

    // 在连接断后后会调用该方法进行回收处理
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        final Object channelId = session.getAttributes().get(CustomHandshakeInterceptor.CHANNEL_ID);
        log.info("断开客户端连接:channelId={}", channelId);
        // 移除session
        wsSessionManager.remove(channelId.toString());
        log.info("Session移除成功:channelId={}", channelId);
    }
}
```



### Session管理器

```java
/*
Session管理器，用于管理session，并注册成spring的一个Bean
*/
@Component
@Slf4j
public class WsSessionManager {
  // 使用ConcurrentHashMap在多线程修改时保证线程安全
    private static final ConcurrentHashMap<String, WebSocketSession> 
      SESSION_POOL = new ConcurrentHashMap<>();;

  // 托管连接
    public boolean add(String id, WebSocketSession session) {
        final WebSocketSession oldSession = get(id);
        if (oldSession != null) {
            if (oldSession.isOpen()) {
              // 同频道的连接存在并且活跃的状态的话，托管失败
                return false;
            }
            //  移除失效的的老连接
            remove(id);
        }
      // 加入Map容器进行托管
        SESSION_POOL.put(id, session);
        return true;
    }

  // 获取连接
    public WebSocketSession get(String id) {
        return SESSION_POOL.get(id);
    }

  // 移除连接
    public WebSocketSession remove(String id) {
      // 从Map容器中移除
        final WebSocketSession session = SESSION_POOL.remove(id);
        if (session != null && session.isOpen()) {
            try {
              // 移除后关闭连接
                session.close();
            }catch (IOException e) {
                log.error(String.format("Session关闭异常, channelId=%s", id), e);
            }
        }
        return session;
    }
}
```



### 配置WS

```java
/*
这是一个配置Bean，实现了WebSocketConfigurer接口
使用@Configuration注解将该类注册成一个配置类
使用@EnableWebSocket开启WebSocket自动配置
*/
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
  // 注入握手拦截器
    @Resource
    private CustomHandshakeInterceptor customHandshakeInterceptor;
  
  // 注入消息处理器
    @Resource
    private CustomWebSocketHandler customWebSocketHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
      // 注册消息处理器，并使用 "/channel"作为处理器的标识，客户端连接路径使用"/channel"就把请求发送给指定的处理器
        registry.addHandler(customWebSocketHandler, "/channel") 
                .addInterceptors(customHandshakeInterceptor) // 注册拦截器
                .setAllowedOrigins("*"); // 允许跨域
    }
}
```

至此基本的ws服务搭建完成。启动服务，使用客户端连接使用



提供一个简单的前端代码，通过控制台简单使用

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<input type="text" placeholder="token" id="token">
<input type="text" placeholder="channelId" id="channelId">
<input type="button" value="连接" onclick="link()">
<input type="button" value="关闭" onclick="close()">

<body>

</body>
<script>
    var ws
    var timer
    function link() {
        // 打开一个 web socket
        var token = document.getElementById("token").value
        var channelId = document.getElementById("channelId").value
        ws = new WebSocket(`ws://localhost:8080/channel?token=${token}&channelId=${channelId}`);

        ws.onopen = function () {
            console.log("连接完成，可以发送数据");
          // 固定频率发送消息保持连接在线
            timer = setInterval(() => {
                ws.send(Date.now())
            }, 10000)
        };

        ws.onmessage = function (evt) {
            var received_msg = evt.data;
            console.log("数据已接收...");
            console.log(received_msg)
        };

        ws.onclose = function () {
            // 关闭 websocket
            console.log("连接已关闭...");
        };

        ws.onerror = function (err) {
            console.error(err)
        }
    }

    function close() {
        if (ws === null || ws === undefined) {
            alert("请先连接")
            return
        }
        ws.close()
        if (!timer) {
            alert("请先连接")
            return
        }
        timer.clear()
    }
</script>

</html>
```





## 实际开发中ws服务设计需要注意的方面

### 业务层消息的确认机制

虽然WS是基于TCP连接的通讯机制，TCP协议特性能保证传输层一定能成功发送数据。但是对于复杂业务和可靠性要求高的业务，最佳的实践是在业务层进行消息的确认。服务端对每一条消息映射一个唯一的标识，客户端在收到消息后，需要将该消息的唯一标识返回给服务端。否则服务端进行一定次数的重试推送，从而来保证消息的可靠推送。



### 心跳机制

对于服务侧重在服务端消息推送的业务。那客户端需要随时保持对连接的监听，而长时间没有数据来往的情况下，不同的客户端和服务端实现会尝试关闭连接。所以为了保证服务端的及时推送，客户端需要和服务端保持一定频率的心跳连接。

- 可以定时的往服务端发送消息，如果发现连接断开，则重新发起连接请求
- 可以服务端对客户端发送ping操作，客户端响应pong操作来实现心跳



### 分布式多实例部署的ws服务session的共享问题

由于ws连接的特殊性，即连接是有状态的。所以一旦连接断开后状态就消失了，下次再进行连接时和上一次的连接并不能对应上。所以平常Web开发中常用的基于序列化和反序列化机制的外部缓存对于面向长连接的ws来说是无法实现的。所以需要通过下面的几种机制来实现



#### 定向分配机制

配备服务的连接注册中心，将用户和真实连接的节点进行映射。在需要向某一个指定用户推送消息时，通过连接注册中心找到当前处理当前用户连接的真实节点，然后将消息推送给处理节点，处理节点转发消息给用户。

**优点**：推送精准，避免无效的广播开销，架构可以根据业务压力的增加进行水平的扩展，适用于大型服务架构

**缺点**：实现复杂，需要独立开发一个分布式的连接管理中心。生产上需要高可用架构的设计



#### MQ或总线的广播机制

通过MQ或Redis的消息订阅和发布机制，进行消息的广播。将ws服务节点接入到统一的MQ或者Redis中，订阅同一个主题或频道。当需要发送消息的时候，将消息广播给所有节点，节点收到广播后会去匹配当前消息的目标连接是否在本节点上。如果在本节点就进行消息推送。不在本节点就自动忽略。

**优点**：实现简单，维护方便。架构上只需引入一个MQ或Redis中间件。适合ws服务节点规模不大的场景

**缺点**：需要良好的代码实现，搞不好容易发生广播风暴，拖垮集群。而且一般一个用户只会连接在集群中的某一个节点，而将消息广播给每一个节点，其实是没必要的。当集群规模扩张到一定程度，当发送一个广播后，所有节点开始计算，导致集群的计算负载短时间内出现峰值。

#### 总结

实际开发中，如果预测到集群规模不大的情况。可以优先考虑使用广播机制进行消息广播。但集群规模很大的情况下，考虑定向分配的架构设计。



## 参考

- https://zh.wikipedia.org/wiki/WebSocket
- https://cloud.tencent.com/developer/article/1530872
- https://tools.ietf.org/html/rfc6455#section-5.2
