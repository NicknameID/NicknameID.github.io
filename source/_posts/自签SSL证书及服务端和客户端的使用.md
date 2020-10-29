---
title: 自签SSL证书及服务端和客户端的使用
date: 2020-10-29 14:05:28
tags:
    - 证书
    - 网络
    - 加密
categories:
    - 应用总结
cover: https://www.symphonysoft.co.kr/wp-content/uploads/2016/09/14152514_l.jpg
---
# 自签SSL证书及服务端和客户端的使用
![](https://www.symphonysoft.co.kr/wp-content/uploads/2016/09/14152514_l.jpg)
## 基本生成步骤：
1. 生成CA根证书
2. 生成服务端证书
3. 生成客户端证书（如果需要做双向认证的话）

## 1.生成根证书
```shell
# 生成root私钥
openssl genrsa -out root.key 1024

# 根据私钥创建根证书请求文件，需要输入一些证书的元信息：邮箱、域名等
openssl req -new -out root.csr -key root.key

# 结合私钥和请求文件，创建根证书，有效期10年
openssl x509 -req -in root.csr -out root.crt -signkey root.key -CAcreateserial -days 3650
```

## 2.生成服务端证书
```shell
# 创建服务端私钥
openssl genrsa -out server.key 1024

# 根据私钥生成请求文件
openssl req -new -out server.csr -key server.key

# 结合私钥和请求文件创建服务端证书，有效期10年
openssl x509 -req -in server.csr -out server.crt -signkey server.key -CA ../root_ca/root.crt -CAkey ../root_ca/root.key -CAcreateserial -days 3650

```
如果需要只需要部署服务端证书端话，就可以结束了。拿着server.crt公钥和server.key私钥部署在服务器上，然后解析域名到改服务器指向到IP，证书就部署成功了。

## 3.生成客户端证书
如果需要做双向验证的，也就是服务端要验证客户端证书的情况。那么需要在同一个根证书下再生成一个客户端证书
```shell
# 生成私钥
openssl genrsa -out client.key 1024

# 申请请求文件
openssl req -new -out client.csr -key client.key

# 生成证书
openssl x509 -req -in client.csr -out client.crt -signkey client.key -CA ../root_ca/root.crt -CAkey ../root_ca/root.key -CAcreateserial -days 3650

# 生成客户端集成证书pkcs12格式的文件，方便浏览器或者http客户端访问（密码：123456）
openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
```

## 服务器配置
### 1.服务域名加证书
使用Caddy作为服务器来部署https服务
Caddyfile
```
{
  admin off # 关闭运维接口
  auto_https off # 自动配置https关闭
}

https://example.com {
    tls <证书文件路径> <私钥文件路径>
}

```


### 2.需要双向证书认证
```
{
  admin off # 关闭运维接口
  auto_https off # 自动配置https关闭
}

https://example.com {
    tls <证书文件路径> <私钥文件路径> {
        client_auth {
            mode require_and_verify
            trusted_ca_cert_file <根证书文件路径>
        }
    }
}
```

## 客户端配置
需要在客户端设置根证书，并信任根证书。因为服务端的证书是自签的证书，公共的CA是无法认证自签的证书的。所以需要在客户端添加并信任自己的根证书后才能通过https访问服务端。如果服务器上部署的是公共CA签发的证书，则不需要设置，因为系统中已经内置了大部分公共CA的证书

### 使用Postman
`Settings -> Certificates -> CA Certificates -> PEM file`中选择根证书文件。
配置好后就可以使用https访问服务了

### 使用HttpClient访问如何配置客户端
使用程序来调用https会比较复杂，大概总结起来可以分为一下几个步骤：
1. 读取根证书文件流
2. 获取KeyStore
3. 将根证书加载到KeyStore中
4. 获取SSLContexts实例
5. 将SSLContexts实例作为参数，设置协议http和https对应的处理socket链接工厂的对象
6. 将链接工厂作为作为参数，生成链接管理器
7. 将链接管理器作为参数，生成httpclient客户端
  
```java
public class SSLContextBuilder {
    public static String build(String keyStorePath, String keyStorepass) {
		SSLContext sc = null;
		FileInputStream instream = null;
		KeyStore trustStore = null;
		try {
			trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
			instream = new FileInputStream(new File(keyStorePath));
			trustStore.load(instream, keyStorepass.toCharArray());
			// 相信自己的CA和所有自签名的证书
			sc = SSLContexts.custom().loadTrustMaterial(trustStore, new TrustSelfSignedStrategy()).build();
		} catch (KeyStoreException | NoSuchAlgorithmException| CertificateException | IOException | KeyManagementException e) {
			e.printStackTrace();
		} finally {
			try {
				instream.close();
			} catch (IOException e) {
			}
		}
		return sc;
    }
}
```

```java
public class HttpClientInstanceBuilder {
    public static CloseableHttpClient builde() {
        // 如果密码为空，则用"nopassword"代替
        SSLContext sslcontext = SSLContextBuilder.build("<根证书文件路径>", "<证书密码>");
         // 设置协议http和https对应的处理socket链接工厂的对象
        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
            .register("http", PlainConnectionSocketFactory.INSTANCE)
            .register("https", new SSLConnectionSocketFactory(sslcontext))
            .build();

        PoolingHttpClientConnectionManager connManager = 
        new PoolingHttpClientConnectionManager(socketFactoryRegistry);
        
        CloseableHttpClient clientInstance = HttpClients
            .custom()
            .setConnectionManager(connManager)
            .build();
        return clientInstance;
    }
}
```
这样就能拿到支持访问自签证书的httpClient客户端了

### https双向认证
如果服务器开启了双向认证，那么客户端还要提供同和服务器证书同属于一个根证书的客户端证书。
#### 给客户端添加客户端证书和私钥
客户端添加证书可以采用PKCS格式证书，它会将证书和私钥绑定到一个文件中



##### 那如何加载`.p12`或`.pfx`文件到客户端？

和加载根证书类似

```java
public class SSLContextBuilder {
    // 如果密码为空，则用"nopassword"代替
    public static String build(String keyStorePath, String keyStorepass) {
		SSLContext sc = null;
		FileInputStream instream = null;
		KeyStore trustStore = null;
		try {
			KeyStore trustStore = KeyStore.getInstance("PKCS12");
			instream = new FileInputStream(new File(keyStorePath));
			trustStore.load(instream, keyStorepass.toCharArray());
            // 和根证书加载到区别就在这里，加载证书的方式不同
            sc = SSLContexts.custom().loadKeyMaterial(trustStore, keyStorepass.toCharArray()).build();
		} catch (KeyStoreException | NoSuchAlgorithmException| CertificateException | IOException | KeyManagementException e) {
			e.printStackTrace();
		} finally {
			try {
				instream.close();
			} catch (IOException e) {
			}
		}
		return sc;
    }
}
```
