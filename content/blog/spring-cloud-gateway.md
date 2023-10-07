---
title: spring-cloud-gateway
tags:
  - spring-cloud-gateway
categories:
  - java
  - spring
  - spring-cloud
date: 2022-12-02 11:46:10
---

# spring cloud gateway
# 概述
**基于版本3.1.1**

该项目提供了一个建立在Spring生态系统之上的API网关，包括：Spring 5，Spring Boot 2和Project Reactor。Spring Cloud Gateway旨在提供一种简单而有效的方法来路由到API，并为它们提供跨领域的关注点，例如：安全性，监控/指标和可伸缩性。

`org.springframework.cloud:spring-cloud-starter-gateway`

* **路由**：网关的基本构建基块。它由 ID、目标 URI、谓词集合和筛选器集合定义。如果聚合谓词为 true，则匹配路由。
* **谓词**：这是一个 [Java 8 函数谓词](https://docs.oracle.com/javase/8/docs/api/java/util/function/Predicate.html)。输入类型是[Spring Framework ](https://docs.spring.io/spring/docs/5.0.x/javadoc-api/org/springframework/web/server/ServerWebExchange.html)`ServerWebExchange`。这使您可以匹配 HTTP 请求中的任何内容，例如标头或参数。
* **过滤器**：这些是使用特定工厂构建的`网关过滤器`实例。在这里，您可以修改发送下游请求之前或之后的请求和响应。

## 如何工作
![image](aH9g6w44iVseRM1zOGCsk15nxQklubk1_n5qI0oeN1Y.png)

客户端向 Spring Cloud Gateway 发出请求。 如果`Gateway Handler Mapping`确定请求与路由匹配，则将其发送到`Gateway Web Handler`。 此处理程序通过特定于请求的过滤器链运行请求。 过滤器被虚线分隔的原因是过滤器可以在发送代理请求之前和之后运行逻辑。 执行所有“pre”过滤器逻辑。 然后进行代理请求。 发出代理请求后，将运行“post”过滤器逻辑。

## 如何配置
有两种方法可以配置谓词和过滤器：简洁方式和完全模式。

简洁方式

```java
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - Cookie=mycookie,mycookievalue
```
定义了 路由谓词工厂（Cookie ），cookie 名称(mycookie) 和匹配值(mycookievalue)。

完全模式

```java
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - name: Cookie
          args:
            name: mycookie
            regexp: mycookievalue
```
可以看出，完全模式，需要指定 name 和 args 参数。



# 路由谓词工厂
Spring Cloud Gateway 匹配路由作为 Spring WebFlux HandlerMapping 基础结构的一部分。 Spring Cloud Gateway 包含许多内置的路由谓词工厂。 所有这些谓词都匹配 HTTP 请求的不同属性。 您可以将多个路由谓词工厂组合在一起。



## After 路由谓词
After 路由谓词工厂接受一个参数（`datetime ,`这是一个 java ZonedDateTime）。 此谓词匹配在指定日期时间之后发生的请求。 以下示例配置了一个 after 路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```
## Before 路由谓词
After 路由谓词工厂接受一个参数（`datetime ,`这是一个 java ZonedDateTime）。 此谓词匹配在指定日期时间之前发生的请求。 以下示例配置了一个 before 路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: https://example.org
        predicates:
        - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```
## `Between` 路由谓词
路由谓词工厂之间有两个参数，datetime1 和 datetime2，它们是 java ZonedDateTime 对象。 此谓词匹配发生在 datetime1 之后和 datetime2 之前的请求。 datetime2 参数必须在 datetime1 之后。 以下示例配置了一个 between 路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: between_route
        uri: https://example.org
        predicates:
        - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
```
## `Cookie` 路由谓词
Cookie 路由谓词工厂有两个参数，即 cookie 名称和一个 regexp（这是一个 Java 正则表达式）。 此谓词匹配具有给定名称且其值与正则表达式匹配的 cookie。 以下示例配置 cookie 路由谓词工厂：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: https://example.org
        predicates:
        - Cookie=chocolate, ch.p
```
## `Header` 路由谓词
Header 路由谓词工厂接受两个参数，标题名称和一个 regexp（这是一个 Java 正则表达式）。 此谓词与具有给定名称的标头匹配，其值与正则表达式匹配。 以下示例配置标头路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: https://example.org
        predicates:
        - Header=X-Request-Id, \d+
```
## `Host` 路由谓词
主机路由谓词工厂采用一个参数：主机名模式列表。 该模式是 Ant 风格的模式。 此谓词匹配的 Host 标头。 以下示例配置主机路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
```
还支持 URI 模板变量（例如 {sub}.myhost.org）。



此谓词提取 URI 模板变量（例如 sub，在前面的示例中定义）作为名称和值的映射，并将其放置在 ServerWebExchange.getAttributes() 中，并使用 ServerWebExchangeUtils.URI\_TEMPLATE\_VARIABLES\_ATTRIBUTE 中定义的键。 然后这些值可供 GatewayFilter 工厂使用

## `Method` 路由谓词
方法路由谓词工厂采用一个方法参数：要匹配的 HTTP 方法。 以下示例配置方法路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: https://example.org
        predicates:
        - Method=GET,POST
```
## `Path` 路由谓词
Path Route Predicate Factory 接受两个参数：一个 Spring PathMatcher 模式列表和一个名为 matchTrailingSlash 的可选标志（默认为 true）。 以下示例配置路径路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment},/blue/{segment}
```
如果 matchTrailingSlash 设置为 false，则不会匹配请求路径 /red/1/。



此谓词提取 URI 模板变量（例如在前面的示例中定义的segment）作为名称和值的映射，并将其放置在 ServerWebExchange.getAttributes() 中，并使用在 ServerWebExchangeUtils.URI\_TEMPLATE\_VARIABLES\_ATTRIBUTE 中定义的键。 然后这些值可供 GatewayFilter 工厂使用



可以使用实用方法（称为 get）来更轻松地访问这些变量。 以下示例显示了如何使用 get 方法：

```java
Map<String, String> uriVariables = ServerWebExchangeUtils.getPathPredicateVariables(exchange);

String segment = uriVariables.get("segment");
```
## `Query` 路由谓词
查询路由谓词工厂有两个参数：一个必需的参数和一个可选的正则表达式（这是一个 Java 正则表达式）。 以下示例配置查询路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=red, gree.
```
## `RemoteAddr` 路由谓词
RemoteAddr 路由谓词工厂采用源列表（最小大小 1），这些源是 CIDR 表示法（IPv4 或 IPv6）字符串，例如 192.168.0.1/16（其中 192.168.0.1 是 IP 地址，16 是子网掩码 ）。 以下示例配置 RemoteAddr 路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: https://example.org
        predicates:
        - RemoteAddr=192.168.1.1/24
```
默认情况下，RemoteAddr 路由谓词工厂使用来自传入请求的远程地址。 如果 Spring Cloud Gateway 位于代理层之后，这可能与实际客户端 IP 地址不匹配。



您可以通过设置自定义的RemoteAddressResolver 来解析远程地址。 Spring Cloud Gateway 带有一个基于 X-Forwarded-For 标头 的XForwardedRemoteAddressResolver (非默认)RemoteAddr 解析器。



XForwardedRemoteAddressResolver 有两个静态构造函数方法，它们采取不同的安全方法：

* XForwardedRemoteAddressResolver::trustAll 返回一个 RemoteAddressResolver，它总是采用在 X-Forwarded-For 标头中找到的第一个 IP 地址。 这种方法容易受到欺骗，因为恶意客户端可以为 X-Forwarded-For 设置一个初始值，该值将被解析器接受。
* XForwardedRemoteAddressResolver::maxTrustedIndex 采用一个索引，该索引与运行在 Spring Cloud Gateway 前面的受信任基础设施的数量相关。 例如，如果 Spring Cloud Gateway 只能通过 HAProxy 访问，则应使用值 1。 如果在访问 Spring Cloud Gateway 之前需要两跳可信基础设施，则应使用值 2。



假如有如下：

```java
X-Forwarded-For: 0.0.0.1, 0.0.0.2, 0.0.0.3
```
根据设置的maxTrustedIndex，具体的取值如下表：

| maxTrustedIndex            | result                                       |
| -------------------------- | -------------------------------------------- |
| \[`Integer.MIN_VALUE`,0\]  | 无效, 初始化时抛出`IllegalArgumentException` |
| 1                          | 0.0.0.3                                      |
| 2                          | 0.0.0.2                                      |
| 3                          | 0.0.0.1                                      |
| \[4, `Integer.MAX_VALUE`\] | 0.0.0.1                                      |

使用java配置XForwardedRemoteAddressResolver ：

```java
RemoteAddressResolver resolver = XForwardedRemoteAddressResolver
    .maxTrustedIndex(1);

...

.route("direct-route",
    r -> r.remoteAddr("10.1.1.1", "10.10.1.1/24")
        .uri("https://downstream1")
.route("proxied-route",
    r -> r.remoteAddr(resolver, "10.10.1.1", "10.10.1.1/24")
        .uri("https://downstream2")
)
```
## `Weight` 路由谓词
Weight 路由谓词工厂采用两个参数：group 和 weight（一个 int）。 权重是按组计算的。 以下示例配置权重路由谓词：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: https://weighthigh.org
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: https://weightlow.org
        predicates:
        - Weight=group1, 2
```
该路由会将约 80% 的流量转发到 weighthigh.org，将约 20% 的流量转发到 weightlow.org

XForear????

# filter工厂
路由过滤器允许以某种方式修改传入的 HTTP 请求或传出的 HTTP 响应。 Spring Cloud Gateway 包括许多内置的 GatewayFilter 工厂。

## 添加类型
### `AddRequestHeader`
AddRequestHeader 有两个参数：name和value。 以下示例配置 AddRequestHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-red, blue
```
将 X-Request-red:blue 标头添加到所有匹配请求的下游请求标头中。



AddRequestHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用并在运行时扩展。 以下示例配置使用变量的 AddRequestHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - AddRequestHeader=X-Request-Red, Blue-{segment}
```
### `AddRequestParameter`
`AddRequestParameter` 两个参数：name和value:

```java
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        filters:
        - AddRequestParameter=red, blue
```
同理，也可以使用URI变量：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - AddRequestParameter=foo, bar-{segment}
```
### `AddResponseHeader`
`AddResponseHeader` 两个参数：name和value:

```java
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        filters:
        - AddResponseHeader=X-Response-Red, Blue
```
同理，也可以使用URI变量：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - AddResponseHeader=foo, bar-{segment}
```
## 删除类型
### `RemoveRequestHeader`
RemoveRequestHeader GatewayFilter 工厂采用 name 参数。 它是要删除的标题的名称。 以下清单配置了 RemoveRequestHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestheader_route
        uri: https://example.org
        filters:
        - RemoveRequestHeader=X-Request-Foo
```
这会在向下游发送之前删除 X-Request-Foo 标头。

### `RemoveResponseHeader`
RemoveResponseHeader GatewayFilter 工厂采用 name 参数。 它是要删除的标题的名称。 以下清单配置了 RemoveResponseHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: removeresponseheader_route
        uri: https://example.org
        filters:
        - RemoveResponseHeader=X-Response-Foo
```
要删除任何类型的敏感标头，您应该为您可能想要这样做的任何路由配置此过滤器。 此外，您可以使用 spring.cloud.gateway.default-filters 配置一次此过滤器，并将其应用于所有路由。

### `RemoveRequestParameter`
RemoveRequestParameter 传入name参数。 它是要删除的查询参数的名称。 以下示例配置 RemoveRequestParameter GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestparameter_route
        uri: https://example.org
        filters:
        - RemoveRequestParameter=red
```
## 修改请求头类型
### `SetRequestHeader`
传入name和value参数：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        filters:
        - SetRequestHeader=X-Request-Red, Blue
```
此 GatewayFilter 替换（而不是添加）具有给定名称的所有标头。 因此，如果下游服务器以 X-Request-Red:1234 响应，这将替换为 X-Request-Red:Blue，这是下游服务将收到的(确定不是网关收到的？？？)。



SetRequestHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用并在运行时扩展。 以下示例配置使用变量的 SetRequestHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetRequestHeader=foo, bar-{segment}
```
### SetResponseHeader
传入name和value参数：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        filters:
        - SetResponseHeader=X-Response-Red, Blue
```
此 GatewayFilter 替换（而不是添加）具有给定名称的所有标头。 因此，如果下游服务器以 X-Response-Red:1234 响应，这将替换为 X-Response-Red:Blue，这是网关客户端将收到的。

SetResponseHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用，并将在运行时扩展。 以下示例配置使用变量的 SetResponseHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetResponseHeader=foo, bar-{segment}
```
### `SetStatus`
SetStatus GatewayFilter 工厂采用单个参数 status。 它必须是有效的 Spring HttpStatus。 它可能是整数值 404 或枚举的字符串表示形式：NOT\_FOUND。 以下清单配置了 SetStatus GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setstatusstring_route
        uri: https://example.org
        filters:
        - SetStatus=BAD_REQUEST
      - id: setstatusint_route
        uri: https://example.org
        filters:
        - SetStatus=401
```
无论哪种情况，响应的 HTTP 状态都设置为 401。



您可以配置 SetStatus GatewayFilter 以在响应的标头中返回来自代理请求的原始 HTTP 状态代码。 如果配置了以下属性，则将标头添加到响应中：

```java
spring:
  cloud:
    gateway:
      set-status:
        original-status-header-name: original-http-status
```
### `RewriteResponseHeader`
RewriteResponseHeader GatewayFilter 工厂采用name、regexp和replacement 。 它使用 Java 正则表达式来灵活地重写响应头值。 以下示例配置 RewriteResponseHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: rewriteresponseheader_route
        uri: https://example.org
        filters:
        - RewriteResponseHeader=X-Response-Red, , password=[^&]+, password=***
```
对于 /42?user=ford&password=omg!what&flag=true 的 header 值，在发出下游请求后设置为 /42?user=ford&password=\*\*\*&flag=true。 由于 YAML 规范，您必须使用 \$\\ 来表示 \$。

### `SetRequestHostHeader`
在某些情况下，可能需要覆盖主机标头。 在这种情况下， SetRequestHostHeader GatewayFilter 工厂可以用指定的值替换现有的主机头。 过滤器采用主机参数。 以下清单配置了 SetRequestHostHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: set_request_host_header_route
        uri: http://localhost:8080/headers
        predicates:
        - Path=/headers
        filters:
        - name: SetRequestHostHeader
          args:
            host: example.org
```
### DedupeResponseHeader
DedupeResponseHeader GatewayFilter 工厂采用name参数和可选的strategy 参数。 name 可以包含以空格分隔的标题名称列表。 以下示例配置 DedupeResponseHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: dedupe_response_header_route
        uri: https://example.org
        filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```
在网关 CORS 逻辑和下游逻辑都添加它们的情况下，删除 Access-Control-Allow-Credentials 和 Access-Control-Allow-Origin 响应标头的重复值。



DedupeResponseHeader 过滤器还接受一个可选的`strategy` 参数。 接受的值为 RETAIN\_FIRST（默认）、RETAIN\_LAST 和 RETAIN\_UNIQUE。

### `PreserveHostHeader`
PreserveHostHeader GatewayFilter 工厂没有参数。 此过滤器设置在请求的attribute设置值，然后 路由过滤器查看该请求属性是否为true，以确定是否应发送原始host标头，而不是由 HTTP 客户端确定的主机标头。 以下示例配置 PreserveHostHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: preserve_host_route
        uri: https://example.org
        filters:
        - PreserveHostHeader
```
### 
```java
MapRequestHeader
```
MapRequestHeader GatewayFilter 工厂采用 fromHeader 和 toHeader 参数。 它创建一个新的命名标头 (toHeader)，并从传入的 http 请求提取命名标头 (fromHeader) 中的值。 如果输入标头不存在，则过滤器没有影响。 如果新命名的标头已存在，则其值将使用新值进行扩充。 以下示例配置 MapRequestHeader：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: map_request_header_route
        uri: https://example.org
        filters:
        - MapRequestHeader=Blue, X-Request-Red
```
传入的http请求携带Blue请求头，该过滤器将该请求头的值添加到X-Request-Red请求头中，然后传递给下游

## 修改请求体或响应体
### `ModifyRequestBody`
您可以使用 ModifyRequestBody 过滤器,在请求正文被网关发送到下游之前对其进行修改。



此过滤器只能通过使用 Java DSL 进行配置。



```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("rewrite_request_obj", r -> r.host("*.rewriterequestobj.org")
            .filters(f -> f.prefixPath("/httpbin")
                .modifyRequestBody(String.class, Hello.class, MediaType.APPLICATION_JSON_VALUE,
                    (exchange, s) -> return Mono.just(new Hello(s.toUpperCase())))).uri(uri))
        .build();
}

static class Hello {
    String message;

    public Hello() { }

    public Hello(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```
### `ModifyResponseBody`
您可以使用 ModifyResponseBody 过滤器在响应正文发送回客户端之前对其进行修改。

此过滤器只能通过使用 Java DSL 进行配置。

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("rewrite_response_upper", r -> r.host("*.rewriteresponseupper.org")
            .filters(f -> f.prefixPath("/httpbin")
                .modifyResponseBody(String.class, String.class,
                    (exchange, s) -> Mono.just(s.toUpperCase()))).uri(uri))
        .build();
}
```
如果响应没有正文，则 RewriteFilter 将传递 null。 应返回 Mono.empty() 以在响应中分配缺失的主体。

## 修改路径
### `PrefixPath`
PrefixPath GatewayFilter 工厂采用单个prefix 参数。 以下示例配置 PrefixPath GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - PrefixPath=/mypath
```
这会将 /mypath 前缀添加到所有匹配请求的路径之前。 因此，对 /hello 的请求将被发送到 /mypath/hello。

### `RewritePath`
RewritePath GatewayFilter 传入regexp 参数和replacement参数。 这使用 Java 正则表达式来灵活地重写请求路径。 以下清单配置了 RewritePath GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/red/**
        filters:
        - RewritePath=/red/?(?<segment>.*), /$\{segment}
```
对于 /red/blue 的请求路径，这会在发出下游请求之前将路径设置为 /blue。 请注意，由于 YAML 规范，\$ 应替换为 \$\\。

### `SetPath`
SetPath GatewayFilter 工厂采用路径模板参数。 它提供了一种通过允许路径的模板化段来操作请求路径的简单方法。 这使用了 Spring Framework 中的 URI 模板。 允许多个匹配段。 以下示例配置 SetPath GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setpath_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - SetPath=/{segment}
```
对于 /red/blue 的请求路径，这会在发出下游请求之前将路径设置为 /blue。

### `StripPrefix`
StripPrefix GatewayFilter 工厂采用一个参数，parts。 该参数指示在将请求发送到下游之前要从请求中剥离的路径中的部分数。 以下清单配置了 StripPrefix GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: nameRoot
        uri: https://nameservice
        predicates:
        - Path=/name/**
        filters:
        - StripPrefix=2
```
当通过网关向 /name/blue/red 发出请求时，对 nameservice 发出的请求看起来像 nameservice/red。

## 重定向
### `RedirectTo`
RedirectTo GatewayFilter 工厂接受两个参数，status 和 url。 status 参数应该是一个 300 系列的重定向 HTTP 代码，比如 301。url 参数应该是一个有效的 URL。 这是 Location 标头的值。 对于相对重定向，您应该使用 uri: no://op 作为路由定义的 uri。 以下清单配置了一个 RedirectTo GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - RedirectTo=302, https://acme.org
```
这将发送带有 Location:https://acme.org 标头的状态 302 以执行重定向。

### `RewriteLocationResponseHeader`
RewriteLocationResponseHeader GatewayFilter 工厂修改 Location 响应头的值，通常是为了摆脱后端特定的细节。 它采用 stripVersionMode、locationHeaderName、hostValue 和 protocolsRegex 参数。 以下清单配置了 RewriteLocationResponseHeader GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: rewritelocationresponseheader_route
        uri: http://example.org
        filters:
        - RewriteLocationResponseHeader=AS_IN_REQUEST, Location, ,
```
例如，对于POST api.example.com/some/object/name的请求，将Location响应头的值 `o``bject-service.prod.example.net/v2/some/object/id` 改写为api.example.com/some/object/id。



stripVersionMode 参数具有以下可能的值：NEVER\_STRIP、AS\_IN\_REQUEST（默认）和 ALWAYS\_STRIP。

* NEVER\_STRIP: 即使原始请求路径不包含版本，也不会剥离版本。
* `AS_IN_REQUEST` : 仅当原始请求路径不包含版本时才会剥离版本。
* `ALWAYS_STRIP` : 版本总是被剥离，即使原始请求路径包含版本。



hostValue 参数（如果提供）用于替换响应 Location 标头的 host:port 部分。 如果未提供，则使用 Host 请求标头的值。



protocolRegex 参数必须是有效的正则表达式字符串，与协议名称匹配。 如果不匹配，则过滤器不执行任何操作。 默认为 http|https|ftp|ftps。

## 对请求进行限制
### `RequestSize`
当请求大小大于允许的限制时，RequestSize GatewayFilter 工厂可以限制请求到达下游服务。 过滤器采用 maxSize 参数。 maxSize 是一个 \`DataSize 类型，因此值可以定义为一个数字，后跟一个可选的 DataUnit 后缀，例如 'KB' 或 'MB'。 字节的默认值为“B”。 它是以字节为单位定义的请求的允许大小限制。 以下清单配置了 RequestSize GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: request_size_route
        uri: http://localhost:8080/upload
        predicates:
        - Path=/upload
        filters:
        - name: RequestSize
          args:
            maxSize: 5000000
```
当请求因大小而被拒绝时，RequestSize GatewayFilter 工厂将响应状态设置为 413 Payload Too Large 并带有一个额外的标头 errorMessage。 以下示例显示了这样的错误消息：

```java
errorMessage` : `Request size is larger than permissible limit. Request size is 6.0 MB where permissible limit is 5.0 MB
```
如果未在路由定义中作为过滤器参数提供，则默认请求大小设置为 5 MB。

### `RequestRateLimiter`
RequestRateLimiter GatewayFilter 工厂使用 RateLimiter 实现来确定是否允许继续处理当前请求。 如果不是，则返回 HTTP 429 - Too Many Requests（默认情况下）状态。

此过滤器采用可选的 keyResolver 参数和特定于速率限制器的参数（本节稍后介绍）。



keyResolver 是一个实现 KeyResolver 接口的 bean。 在配置中，使用 SpEL 按名称引用 bean。 #{@myKeyResolver} 是一个 SpEL 表达式，它引用名为 myKeyResolver 的 bean。 以下清单显示了 KeyResolver 接口：

```java
public interface KeyResolver {
    Mono<String> resolve(ServerWebExchange exchange);
}
```
KeyResolver 接口让可插拔策略派生出限制请求的key。 在未来的里程碑版本中，将有一些 KeyResolver 实现。

KeyResolver 的默认实现是 PrincipalNameKeyResolver，它从 ServerWebExchange 检索 Principal 并调用 Principal.getName()。

默认情况下，如果 KeyResolver 未找到key，则拒绝请求。 您可以通过设置 spring.cloud.gateway.filter.request-rate-limiter.deny-empty-key（true 或 false）和 spring.cloud.gateway.filter.request-rate-limiter.empty-key-status-code来调整此行为



Redis 实现基于在 [Stripe](https://stripe.com/blog/rate-limiters) 完成的工作。 它需要使用 spring-boot-starter-data-redis-reactive Spring Boot starter。

使用的算法是令牌桶算法。

redis-rate-limiter.replenishRate 属性是您希望允许用户每秒执行多少请求。 这是令牌桶填充的速率。

redis-rate-limiter.burstCapacity 属性是允许用户在一秒内执行的最大请求数。 这是令牌桶可以容纳的令牌数量。 将此值设置为零会阻止所有请求。

redis-rate-limiter.requestedTokens 属性是请求花费多少令牌。 这是每个请求从存储桶中获取的令牌数量，默认为 1。

稳定的速率是通过在replyRate 和burstCapacity 中设置相同的值来实现的。 通过将burstCapacity 设置为高于replyRate，可以允许临时突发。 在这种情况下，需要允许速率限制器在突发之间有一段时间（根据replyRate），因为两个连续的突发将导致请求丢失（HTTP 429 - Too Many Requests）。 以下清单配置了 redis-rate-limiter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
            redis-rate-limiter.requestedTokens: 1
```
java代码

```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
}
```
这将每个用户的请求速率限制定义为 10。 允许突发 20 个，但在下一秒，只有 10 个请求可用。 KeyResolver 是一个简单的获取用户请求参数的方法（注意，不推荐用于生产）。



您还可以将速率限制器定义为实现 RateLimiter 接口的 bean。 在配置中，您可以使用 SpEL 按名称引用 bean。 #{@myRateLimiter} 是一个 SpEL 表达式，它引用名为 myRateLimiter 的 bean。 下面的清单定义了一个速率限制器，它使用在前面的清单中定义的 KeyResolver：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            rate-limiter: "#{@myRateLimiter}"
            key-resolver: "#{@userKeyResolver}"
```
## 安全相关
### 
```java
 Token Relay

```
令牌中继是 OAuth2 消费者充当客户端并将传入令牌转发到传出资源请求的地方。 消费者可以是纯客户端（如 SSO 应用程序）或资源服务器。

Spring Cloud Gateway 可以将 OAuth2 访问令牌下游转发到它正在代理的服务。 要将此功能添加到网关，您需要像这样添加 TokenRelayGatewayFilterFactory：

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route("resource", r -> r.path("/resource")
                    .filters(f -> f.tokenRelay())
                    .uri("http://localhost:9000"))
            .build();
}
```
```java
spring:
  cloud:
    gateway:
      routes:
      - id: resource
        uri: http://localhost:9000
        predicates:
        - Path=/resource
        filters:
        - TokenRelay=
```
并且它将（除了登录用户并获取令牌之外）将身份验证令牌下游传递给服务（在本例中为 /resource）。

要为 Spring Cloud Gateway 启用此功能，请添加以下依赖项:

```java
org.springframework.boot:spring-boot-starter-oauth2-client
```
{githubmaster}/src/main/java/org/springframework/cloud/gateway/security/TokenRelayGatewayFilterFactory.java\[filter\] 从当前已验证的用户中提取访问令牌，并将其放入下游请求的请求头中。



只有在设置了正确的 spring.security.oauth2.client.\* 属性时才会创建 TokenRelayGatewayFilterFactory bean，这将触发 ReactiveClientRegistrationRepository bean 的创建。



TokenRelayGatewayFilterFactory 使用的 ReactiveOAuth2AuthorizedClientService 的默认实现使用内存数据存储。 如果您需要更强大的解决方案，您将需要提供自己的实现 ReactiveOAuth2AuthorizedClientService。

### `SecureHeaders`
根据[本博客文章中](https://blog.appcanary.com/2017/http-security-headers.html)提出的建议，SecureHeaders GatewayFilter 工厂向响应添加了许多标头。

* `X-Xss-Protection:1 (mode=block`)
* `Strict-Transport-Security (max-age=631138519`)

```java
X-Frame-Options (DENY)
```
```java
X-Content-Type-Options (nosniff)
```
```java
Referrer-Policy (no-referrer)
```
```java
Content-Security-Policy (default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline)'
```
```java
X-Download-Options (noopen)
```
```java
X-Permitted-Cross-Domain-Policies (none)
```
要更改默认值，请在 spring.cloud.gateway.filter.secure-headers 命名空间中设置适当的属性。 以下属性可用：



```java
xss-protection-header
```
```java
strict-transport-security
```
```java
x-frame-options
```
```java
x-content-type-options
```
```java
referrer-policy
```
```java
content-security-policy
```
```java
x-download-options
```
```java
x-permitted-cross-domain-policies
```
要禁用默认值，请使用逗号分隔值设置 spring.cloud.gateway.filter.secure-headers.disable 属性。 以下示例显示了如何执行此操作：

```java
spring.cloud.gateway.filter.secure-headers.disable=x-frame-options,strict-transport-security
```
## 服务降级相关
### CircuitBreaker
Spring Cloud CircuitBreaker GatewayFilter 工厂使用 Spring Cloud CircuitBreaker API 将网关路由包装在断路器中。 Spring Cloud CircuitBreaker 可与 Spring Cloud Gateway 一起使用。 Spring Cloud 支持开箱即用的 Resilience4J。



要启用 Spring Cloud CircuitBreaker 过滤器，您需要将 spring-cloud-starter-circuitbreaker-reactor-resilience4j 放在类路径上。 以下示例配置 Spring Cloud CircuitBreaker GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: https://example.org
        filters:
        - CircuitBreaker=myCircuitBreaker
```
Spring Cloud CircuitBreaker 过滤器还可以接受可选的 fallbackUri 参数。 目前，仅支持转发。 如果调用回退，则请求将转发到与 URI 匹配的控制器。 以下示例配置了这样的回退：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: lb://backing-service:8088
        predicates:
        - Path=/consumingServiceEndpoint
        filters:
        - name: CircuitBreaker
          args:
            name: myCircuitBreaker
            fallbackUri: forward:/inCaseOfFailureUseThis
        - RewritePath=/consumingServiceEndpoint, /backingServiceEndpoint
```
同样的java配置如下：

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("circuitbreaker_route", r -> r.path("/consumingServiceEndpoint")
            .filters(f -> f.circuitBreaker(c -> c.name("myCircuitBreaker").fallbackUri("forward:/inCaseOfFailureUseThis"))
                .rewritePath("/consumingServiceEndpoint", "/backingServiceEndpoint")).uri("lb://backing-service:8088")
        .build();
}
```
当调用断路器回退时，此示例转发到 /inCaseofFailureUseThis URI。 请注意，此示例还演示了（可选）Spring Cloud LoadBalancer 负载平衡（由目标 URI 上的 lb 前缀定义）。



主要场景是使用 fallbackUri 在网关应用程序中定义内部controller 或handler。 但是，您也可以将请求重新路由到外部应用程序中的controller 或handler，如下所示：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: ingredients
        uri: lb://ingredients
        predicates:
        - Path=//ingredients/**
        filters:
        - name: CircuitBreaker
          args:
            name: fetchIngredients
            fallbackUri: forward:/fallback
      - id: ingredients-fallback
        uri: http://localhost:9994
        predicates:
        - Path=/fallback
```
在此示例中，网关应用程序中没有回退端点或处理程序。 但是，在另一个应用程序中有一个，在 localhost:9994 下注册。



如果请求被转发到回退，Spring Cloud CircuitBreaker Gateway 过滤器还提供导致它的 Throwable。 它作为 ServerWebExchangeUtils.CIRCUITBREAKER\_EXECUTION\_EXCEPTION\_ATTR 属性添加到 ServerWebExchange 中，可在网关应用程序中处理回退时使用。



对于外部控制器/处理程序场景，可以添加带有异常详细信息的标头。 您可以在 FallbackHeaders GatewayFilter Factory 部分找到有关这样做的更[多信息](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#fallback-headers)。



在某些情况下，您可能希望根据从其环绕的路由返回的状态代码来触发断路器。 断路器配置对象采用状态代码列表，如果返回这些状态代码，将导致断路器跳闸。 在设置要使断路器跳闸的状态代码时，您可以使用带有状态代码值的整数或 HttpStatus 枚举的字符串表示形式。

```java
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: lb://backing-service:8088
        predicates:
        - Path=/consumingServiceEndpoint
        filters:
        - name: CircuitBreaker
          args:
            name: myCircuitBreaker
            fallbackUri: forward:/inCaseOfFailureUseThis
            statusCodes:
              - 500
              - "NOT_FOUND"
```
等效的java配置

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("circuitbreaker_route", r -> r.path("/consumingServiceEndpoint")
            .filters(f -> f.circuitBreaker(c -> c.name("myCircuitBreaker").fallbackUri("forward:/inCaseOfFailureUseThis").addStatusCode("INTERNAL_SERVER_ERROR"))
                .rewritePath("/consumingServiceEndpoint", "/backingServiceEndpoint")).uri("lb://backing-service:8088")
        .build();
}
```
### `FallbackHeaders`
FallbackHeaders 工厂允许您在转发到外部应用程序中的 fallbackUri 的请求的标头中添加 Spring Cloud CircuitBreaker 执行异常详细信息，如下面的场景：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: ingredients
        uri: lb://ingredients
        predicates:
        - Path=//ingredients/**
        filters:
        - name: CircuitBreaker
          args:
            name: fetchIngredients
            fallbackUri: forward:/fallback
      - id: ingredients-fallback
        uri: http://localhost:9994
        predicates:
        - Path=/fallback
        filters:
        - name: FallbackHeaders
          args:
            executionExceptionTypeHeaderName: Test-Header
```
在此示例中，在运行断路器时发生执行异常后，请求将转发到在 localhost:9994 上运行的应用程序中的回退端点或处理程序。 FallbackHeaders 过滤器将带有异常类型、消息和（如果可用）根本原因异常类型和消息的标头添加到该请求中。



您可以通过设置以下参数的值（显示为它们的默认值）来覆盖配置中标头的名称：

* `executionExceptionTypeHeaderName` (`"Execution-Exception-Type"`)
* `executionExceptionMessageHeaderName` (`"Execution-Exception-Message"`)
* `rootCauseExceptionTypeHeaderName` (`"Root-Cause-Exception-Type"`)
* `rootCauseExceptionMessageHeaderName` (`"Root-Cause-Exception-Message"`)

### `Retry`
支持以下参数:

* retries : 重试的次数，默认3
* statuses： 应该重试的 HTTP 状态码，用 org.springframework.http.HttpStatus 表示。
* methods：应该重试的 HTTP 方法，使用 org.springframework.http.HttpMethod 表示。默认GET 
* series：要重试的状态码系列，用org.springframework.http.HttpStatus.Series表示。默认5XX 
* exceptions： 应该重试的抛出异常的列表。默认`IOException` and `TimeoutException`
* backoff： 为重试配置的指数退避。 在 firstBackoff \* (factor ^ n) 的退避间隔后执行重试，其中 n 是迭代。 如果配置了 maxBackoff，则应用的最大退避限制为 maxBackoff。 如果 basedOnPreviousValue 为真，则使用 prevBackoff \* 因子计算回退。默认disabled

```java
spring:
  cloud:
    gateway:
      routes:
      - id: retry_test
        uri: http://localhost:8080/flakey
        predicates:
        - Host=*.retry.com
        filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY
            methods: GET,POST
            backoff:
              firstBackoff: 10ms
              maxBackoff: 50ms
              factor: 2
              basedOnPreviousValue: false
```
使用带有 forward: 前缀的重试过滤器时，应仔细编写目标端点，以便在出现错误时不会执行任何可能导致响应被发送到客户端并提交的操作。 例如，如果目标端点是带注释的控制器，则目标控制器方法不应返回带有错误状态代码的 ResponseEntity。 相反，它应该抛出异常或发出错误信号（例如，通过 Mono.error(ex) 返回值），重试过滤器可以配置为通过重试来处理。



将重试过滤器与任何带有正文的 HTTP 方法一起使用时，正文将被缓存，网关将受到内存限制。 正文缓存在由 ServerWebExchangeUtils.CACHED\_REQUEST\_BODY\_ATTR 定义的请求属性中。 对象的类型是 org.springframework.core.io.buffer.DataBuffer。

## 其他
### `SaveSession`
SaveSession GatewayFilter 工厂在向下游转发调用之前强制执行 WebSession::save 操作。 这在将 Spring Session 之类的东西与惰性数据存储一起使用时特别有用，并且您需要确保在进行转发调用之前已保存会话状态。 以下示例配置 SaveSession GatewayFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: save_session
        uri: https://example.org
        predicates:
        - Path=/foo/**
        filters:
        - SaveSession
```
如果您将 Spring Security 与 Spring Session 集成并希望确保安全详细信息已转发到远程进程，那么这很关键。



# 全局过滤器
GlobalFilter 接口与 GatewayFilter 具有相同的签名。 这些是有条件地应用于所有路由的特殊过滤器。



## 组合全局过滤器以及`GatewayFilter` 排序
当请求与路由匹配时， filtering web handler 会将 GlobalFilter 的所有实例和 GatewayFilter 的所有特定于路由的实例添加到过滤器链中。 这个组合过滤器链由 org.springframework.core.Ordered 接口排序，您可以通过实现 getOrder() 方法设置该接口。



由于 Spring Cloud Gateway 区分过滤器逻辑执行的“pre”和“post”阶段（参见 How it Works），具有最高优先级的过滤器是“pre”阶段的第一个，“post”阶段的最后一个阶段。



```java
@Bean
public GlobalFilter customFilter() {
    return new CustomGlobalFilter();
}

public class CustomGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("custom global filter");
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```
## `ForwardRoutingFilter` 
ForwardRoutingFilter 在交换属性 ServerWebExchangeUtils.GATEWAY\_REQUEST\_URL\_ATTR 中查找 URI。 如果 URL 有转发方案（例如 forward:///localendpoint），它会使用 Spring DispatcherHandler 来处理请求。 请求 URL 的路径部分被转发 URL 中的路径覆盖。 未修改的原始 URL 将附加到 ServerWebExchangeUtils.GATEWAY\_ORIGINAL\_REQUEST\_URL\_ATTR 属性中的列表中。

## `ReactiveLoadBalancerClientFilter` 
ReactiveLoadBalancerClientFilter 在名为 ServerWebExchangeUtils.GATEWAY\_REQUEST\_URL\_ATTR 的交换属性中查找 URI。 如果 URL 具有 lb 方案（例如 lb://myservice），则它使用 Spring Cloud ReactorLoadBalancer 将名称（在此示例中为 myservice）解析为实际主机和端口，并替换同一属性中的 URI。 未修改的原始 URL 将附加到 ServerWebExchangeUtils.GATEWAY\_ORIGINAL\_REQUEST\_URL\_ATTR 属性中的列表中。 过滤器还会查看 ServerWebExchangeUtils.GATEWAY\_SCHEME\_PREFIX\_ATTR 属性以查看它是否等于 lb。如果是，则应用相同的规则。 以下清单配置了一个 ReactiveLoadBalancerClientFilter：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: myRoute
        uri: lb://service
        predicates:
        - Path=/service/**
```
默认情况下，当 ReactorLoadBalancer 找不到服务实例时，会返回 503。 您可以通过设置 spring.cloud.gateway.loadbalancer.use404=true 将网关配置为返回 404。



从 ReactiveLoadBalancerClientFilter 返回的 ServiceInstance 的 isSecure 值会覆盖向网关发出的请求中指定的方案。 例如，如果请求通过 HTTPS 进入网关，但 ServiceInstance 指示它不安全，则通过 HTTP 发出下游请求。 相反的情况也可以适用。 但是，如果在网关配置中为路由指定了 GATEWAY\_SCHEME\_PREFIX\_ATTR，则前缀将被剥离，并且来自路由 URL 的结果方案将覆盖 ServiceInstance 配置。



## Netty 路由过滤器
如果位于 ServerWebExchangeUtils.GATEWAY\_REQUEST\_URL\_ATTR 交换属性中的 URL 具有 http 或 https 方案，则 Netty 路由过滤器运行。 它使用 Netty HttpClient 发出下游代理请求。 响应放在 ServerWebExchangeUtils.CLIENT\_RESPONSE\_ATTR 交换属性中，以供稍后过滤器使用。 （还有一个实验性的 WebClientHttpRoutingFilter 执行相同的功能但不需要 Netty。）



## `NettyWriteResponseFilter` 
如果 ServerWebExchangeUtils.CLIENT\_RESPONSE\_ATTR 交换属性中存在 Netty HttpClientResponse，则 NettyWriteResponseFilter 运行。 它在所有其他过滤器完成后运行，并将代理响应写回网关客户端响应。 （还有一个实验性的 WebClientWriteResponseFilter 可以执行相同的功能，但不需要 Netty。）

## `RouteToRequestUrlFilter` 
如果 ServerWebExchangeUtils.GATEWAY\_ROUTE\_ATTR 交换属性中有 Route 对象，则 RouteToRequestUrlFilter 运行。 它基于请求 URI 创建一个新的 URI，但使用 Route 对象的 URI 属性进行更新。 新 URI 放置在 ServerWebExchangeUtils.GATEWAY\_REQUEST\_URL\_ATTR 交换属性中。

如果 URI 具有方案前缀，例如 lb:ws://serviceid，则 lb 方案将从 URI 中剥离并放置在 ServerWebExchangeUtils.GATEWAY\_SCHEME\_PREFIX\_ATTR 中，以便稍后在过滤器链中使用。



## websocket 路由过滤器
如果位于 ServerWebExchangeUtils.GATEWAY\_REQUEST\_URL\_ATTR 交换属性中的 URL 具有 ws 或 wss 方案，则 websocket 路由过滤器运行。 它使用 Spring WebSocket 基础结构向下游转发 websocket 请求。

您可以通过在 URI 前加上 lb 来对 websockets 进行负载平衡，例如 lb:ws://serviceid。

```java
spring:
  cloud:
    gateway:
      routes:
      # SockJS route
      - id: websocket_sockjs_route
        uri: http://localhost:3001
        predicates:
        - Path=/websocket/info/**
      # Normal Websocket route
      - id: websocket_route
        uri: ws://localhost:3001
        predicates:
        - Path=/websocket/**
```
## metrics过滤器
要启用网关指标，请将 spring-boot-starter-actuator 添加为项目依赖项。 然后，默认情况下，只要属性 spring.cloud.gateway.metrics.enabled 未设置为 false，网关指标过滤器就会运行。 此过滤器添加了一个名为 gateway.requests 的计时器指标，并带有以下标签：

* routeId：路由ID
* routeUri：API 路由到的 URI。
* outcome：结果，由 HttpStatus.Series 分类。
* status：返回给客户端的请求的 HTTP 状态
* httpStatusCode：返回给客户端的请求的 HTTP 状态。
* httpMethod：用于请求的 HTTP 方法。

然后可以从 /actuator/metrics/gateway.requests 抓取这些指标，并且可以轻松地与 Prometheus 集成以创建 Grafana 仪表板。

要启用 prometheus 端点，请将 micrometer-registry-prometheus 添加为项目依赖项。



## 将交换标记为已路由
网关路由 ServerWebExchange 后，它通过将 gatewayAlreadyRouted 添加到交换属性来将该交换标记为“已路由”。 一旦请求被标记为路由，其他路由过滤器将不会再次路由该请求，实质上是跳过过滤器。 有一些方便的方法可用于将交换标记为已路由或检查交换是否已被路由。



ServerWebExchangeUtils.isAlreadyRouted 接受一个 ServerWebExchange 对象并检查它是否已被“路由”。

ServerWebExchangeUtils.setAlreadyRouted 接受一个 ServerWebExchange 对象并将其标记为“已路由”。



# [HttpHeadersFilters](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#httpheadersfilters)
HttpHeadersFilters 在向下游发送请求之前应用于请求，例如在 NettyRoutingFilter 中。

## 转发headers的过滤器
转发头过滤器创建转发头以发送到下游服务。 它将当前请求的 Host 标头、方案和端口添加到任何现有的 Forwarded 标头中。

## `RemoveHopByHop` 
RemoveHopByHop 标头过滤器从转发的请求中删除标头。 删除的默认标头列表来自 IETF。



默认一出的标头：

* Connection
* Keep-Alive
* Proxy-Authenticate
* Proxy-Authorization
* TE
* Trailer
* Transfer-Encoding
* Upgrade



要更改此设置，请将 spring.cloud.gateway.filter.remove-hop-by-hop.headers 属性设置为要删除的标头名称列表。



## `XForwarded` 
XForwarded 标头过滤器创建各种 X-Forwarded-\* 标头以发送到下游服务。 它使用当前请求的主机头、方案、端口和路径来创建各种头。

可以通过以下布尔属性（默认为 true）控制单个标题的创建：

```java
spring.cloud.gateway.x-forwarded.for-enabled
```
```java
spring.cloud.gateway.x-forwarded.host-enabled
```
```java
spring.cloud.gateway.x-forwarded.port-enabled
```
```java
spring.cloud.gateway.x-forwarded.proto-enabled
```
```java
spring.cloud.gateway.x-forwarded.prefix-enabled
```
附加多个标题可以由以下布尔属性控制（默认为 true）：

```java
spring.cloud.gateway.x-forwarded.for-append
```
```java
spring.cloud.gateway.x-forwarded.host-append
```
```java
spring.cloud.gateway.x-forwarded.port-append
```
```java
spring.cloud.gateway.x-forwarded.proto-append
```
```java
spring.cloud.gateway.x-forwarded.prefix-append
```
# TLS和SSL
网关可以通过遵循通常的 Spring 服务器配置来侦听 HTTPS 上的请求。 以下示例显示了如何执行此操作：

```java
server:
  ssl:
    enabled: true
    key-alias: scg
    key-store-password: scg1234
    key-store: classpath:scg-keystore.p12
    key-store-type: PKCS12
```
您可以将网关路由路由到 HTTP 和 HTTPS 后端。 如果您要路由到 HTTPS 后端，则可以使用以下配置将网关配置为信任所有下游证书：

```java
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          useInsecureTrustManager: true
```
使用不安全的信任管理器不适合生产。 对于生产部署，您可以使用一组可以信任的已知证书配置网关，并使用以下配置：

```java
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          trustedX509Certificates:
          - cert1.pem
          - cert2.pem
```
如果 Spring Cloud Gateway 未提供受信任的证书，则使用默认信任存储（您可以通过设置 javax.net.ssl.trustStore 系统属性来覆盖）。



## TLS握手
网关维护一个客户端池，用于路由到后端。 通过 HTTPS 通信时，客户端会发起 TLS 握手。 许多超时与此握手相关联。 您可以配置这些超时可以配置（默认显示）如下：

```java
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          handshake-timeout-millis: 10000
          close-notify-flush-timeout-millis: 3000
          close-notify-read-timeout-millis: 0
```
# 配置
Spring Cloud Gateway 的配置由一组 RouteDefinitionLocator 实例驱动。 以下清单显示了 RouteDefinitionLocator 接口的定义：

```java
public interface RouteDefinitionLocator {
    Flux<RouteDefinition> getRouteDefinitions();
}
```
默认情况下，PropertiesRouteDefinitionLocator 使用 Spring Boot 的 @ConfigurationProperties 机制加载属性。

较早的配置示例都使用使用位置参数而不是命名参数的快捷表示法。 下面两个例子是等价的：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: setstatus_route
        uri: https://example.org
        filters:
        - name: SetStatus
          args:
            status: 401
      - id: setstatusshortcut_route
        uri: https://example.org
        filters:
        - SetStatus=401
```
对于网关的某些用途，属性就足够了，但某些生产用例受益于从外部源（例如数据库）加载配置。 未来的里程碑版本将具有基于 Spring Data Repositories 的 RouteDefinitionLocator 实现，例如 Redis、MongoDB 和 Cassandra。



# 路由元数据配置
您可以使用元数据为每个路由配置附加参数，如下所示：

```java
spring:
  cloud:
    gateway:
      routes:
      - id: route_with_metadata
        uri: https://example.org
        metadata:
          optionName: "OptionValue"
          compositeObject:
            name: "value"
          iAmNumber: 1
```
您可以从交换中获取所有元数据属性，如下所示：

```java
Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
// get all metadata properties
route.getMetadata();
// get a single metadata property
route.getMetadata(someKey);
```
# http超时配置
可以为所有路由配置 Http 超时（响应和连接），并为每个特定路由覆盖。

## 全局配置
要配置全局 http 超时：

连接超时必须以毫秒为单位指定。

响应超时必须指定为 java.time.Duration

```java
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 1000
        response-timeout: 5s
```
## 单路由配置
要配置每条路由超时：

连接超时必须以毫秒为单位指定。

响应超时必须以毫秒为单位指定。

```java
      - id: per_route_timeouts
        uri: https://example.org
        predicates:
          - name: Path
            args:
              pattern: /delay/{timeout}
        metadata:
          response-timeout: 200
          connect-timeout: 200
```
java配置

```java
import static org.springframework.cloud.gateway.support.RouteMetadataUtils.CONNECT_TIMEOUT_ATTR;
import static org.springframework.cloud.gateway.support.RouteMetadataUtils.RESPONSE_TIMEOUT_ATTR;

      @Bean
      public RouteLocator customRouteLocator(RouteLocatorBuilder routeBuilder){
         return routeBuilder.routes()
               .route("test1", r -> {
                  return r.host("*.somehost.org").and().path("/somepath")
                        .filters(f -> f.addRequestHeader("header1", "header-value-1"))
                        .uri("http://someuri")
                        .metadata(RESPONSE_TIMEOUT_ATTR, 200)
                        .metadata(CONNECT_TIMEOUT_ATTR, 200);
               })
               .build();
      }
```
## java路由API
为了允许在 Java 中进行简单的配置，RouteLocatorBuilder bean 包含一个流畅的 API。 以下清单显示了它的工作原理：

```java
// static imports from GatewayFilters and RoutePredicates
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder, ThrottleGatewayFilterFactory throttle) {
    return builder.routes()
            .route(r -> r.host("**.abc.org").and().path("/image/png")
                .filters(f ->
                        f.addResponseHeader("X-TestHeader", "foobar"))
                .uri("http://httpbin.org:80")
            )
            .route(r -> r.path("/image/webp")
                .filters(f ->
                        f.addResponseHeader("X-AnotherHeader", "baz"))
                .uri("http://httpbin.org:80")
                .metadata("key", "value")
            )
            .route(r -> r.order(-1)
                .host("**.throttle.org").and().path("/get")
                .filters(f -> f.filter(throttle.apply(1,
                        1,
                        10,
                        TimeUnit.SECONDS)))
                .uri("http://httpbin.org:80")
                .metadata("key", "value")
            )
            .build();
}
```
这种风格还允许更多的自定义谓词断言。 RouteDefinitionLocator bean 定义的谓词使用逻辑和组合。 通过使用 fluent Java API，您可以在 Predicate 类上使用 and()、or() 和 negate() 运算符。



## `DiscoveryClient` 路由配置
您可以将网关配置为基于向 DiscoveryClient 兼容服务注册表注册的服务创建路由。

要启用此功能，请设置 spring.cloud.gateway.discovery.locator.enabled=true 并确保 DiscoveryClient 实现（例如 Netflix Eureka、Consul 或 Zookeeper）在类路径上并已启用。

### 为 DiscoveryClient 路由配置谓词和过滤器
默认情况下，网关为使用 DiscoveryClient 创建的路由定义单个谓词和过滤器。

默认谓词是使用模式 /serviceId/\*\* 定义的路径谓词，其中 serviceId 是来自 DiscoveryClient 的服务的 ID。



默认过滤器是带有正则表达式 /serviceId/?(?.\*) 和替换 /\${remaining} 的重写路径过滤器。 这会在向下游发送请求之前从路径中剥离服务 ID。



如果要自定义 DiscoveryClient 路由使用的谓词或过滤器，请设置 spring.cloud.gateway.discovery.locator.predicates\[x\] 和 spring.cloud.gateway.discovery.locator.filters\[y\]。 这样做时，如果您想保留该功能，您需要确保包含前面显示的默认谓词和过滤器。 以下示例显示了它的样子：



```java
spring.cloud.gateway.discovery.locator.predicates[0].name: Path
spring.cloud.gateway.discovery.locator.predicates[0].args[pattern]: "'/'+serviceId+'/**'"
spring.cloud.gateway.discovery.locator.predicates[1].name: Host
spring.cloud.gateway.discovery.locator.predicates[1].args[pattern]: "'**.foo.com'"
spring.cloud.gateway.discovery.locator.filters[0].name: CircuitBreaker
spring.cloud.gateway.discovery.locator.filters[0].args[name]: serviceId
spring.cloud.gateway.discovery.locator.filters[1].name: RewritePath
spring.cloud.gateway.discovery.locator.filters[1].args[regexp]: "'/' + serviceId + '/?(?<remaining>.*)'"
spring.cloud.gateway.discovery.locator.filters[1].args[replacement]: "'/${remaining}'"
```
# Reactor Netty 访问日志
要启用 Reactor Netty 访问日志，请设置 -Dreactor.netty.http.server.accessLogEnabled=true。它必须是 Java 系统属性，而不是 Spring Boot 属性。

您可以将日志记录系统配置为具有单独的访问日志文件。 以下示例创建一个 Logback 配置：

```java
    <appender name="accessLog" class="ch.qos.logback.core.FileAppender">
        <file>access_log.log</file>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    <appender name="async" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="accessLog" />
    </appender>

    <logger name="reactor.netty.http.server.AccessLog" level="INFO" additivity="false">
        <appender-ref ref="async"/>
    </logger>
```
# CORS 配置
您可以配置网关以控制 CORS 行为。 “全局” CORS 配置是 URL 模式到 Spring Framework CorsConfiguration 的映射。 以下示例配置 CORS：

```java
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://docs.spring.io"
            allowedMethods:
            - GET
```
在前面的示例中，对于所有 GET 请求路径，都允许来自 docs.spring.io 的请求发出 CORS 请求。

要为某些网关路由谓词未处理的请求提供相同的 CORS 配置，请将 spring.cloud.gateway.globalcors.add-to-simple-url-handler-mapping 属性设置为 true。 当您尝试支持 CORS 预检请求并且您的路由谓词由于 HTTP 方法是选项而未评估为 true 时，这很有用。



# Actuator API
/gateway 执行器端点允许您监视 Spring Cloud Gateway 应用程序并与之交互。 要远程访问，必须在应用程序属性中启用并通过 HTTP 或 JMX 公开端点。 以下清单显示了如何执行此操作：

```java
management.endpoint.gateway.enabled=true # default value
management.endpoints.web.exposure.include=gateway
```
Spring Cloud Gateway 中添加了一种新的、更详细的格式。 它为每个路由添加了更多详细信息，让您可以查看与每个路由关联的谓词和过滤器以及任何可用的配置。 以下示例配置 /actuator/gateway/routes：

```java
[
  {
    "predicate": "(Hosts: [**.addrequestheader.org] && Paths: [/headers], match trailing slash: true)",
    "route_id": "add_request_header_test",
    "filters": [
      "[[AddResponseHeader X-Response-Default-Foo = 'Default-Bar'], order = 1]",
      "[[AddRequestHeader X-Request-Foo = 'Bar'], order = 1]",
      "[[PrefixPath prefix = '/httpbin'], order = 2]"
    ],
    "uri": "lb://testservice",
    "order": 0
  }
]
```
默认情况下启用此功能。 要禁用它，请设置以下属性：

```java
spring.cloud.gateway.actuator.verbose.enabled=false
```
## 检索路由过滤器
要检索应用于所有路由的全局过滤器，请向 /actuator/gateway/globalfilters 发出 GET 请求。 得到的响应类似于以下内容：

```java
{
  "org.springframework.cloud.gateway.filter.ReactiveLoadBalancerClientFilter@77856cc5": 10100,
  "org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter@4f6fd101": 10000,
  "org.springframework.cloud.gateway.filter.NettyWriteResponseFilter@32d22650": -1,
  "org.springframework.cloud.gateway.filter.ForwardRoutingFilter@106459d9": 2147483647,
  "org.springframework.cloud.gateway.filter.NettyRoutingFilter@1fbd5e0": 2147483647,
  "org.springframework.cloud.gateway.filter.ForwardPathFilter@33a71d23": 0,
  "org.springframework.cloud.gateway.filter.AdaptCachedBodyGlobalFilter@135064ea": 2147483637,
  "org.springframework.cloud.gateway.filter.WebsocketRoutingFilter@23c05889": 2147483646
}
```
响应包含现有全局过滤器的详细信息。 对于每个全局过滤器，都有过滤器对象的字符串表示（例如，org.springframework.cloud.gateway.filter.ReactiveLoadBalancerClientFilter@77856cc5）和过滤器链中的对应顺序。}



要检索应用于路由的 GatewayFilter 工厂，请向 /actuator/gateway/routefilters 发出 GET 请求。 得到的响应类似于以下内容：

```java
{
  "[AddRequestHeaderGatewayFilterFactory@570ed9c configClass = AbstractNameValueGatewayFilterFactory.NameValueConfig]": null,
  "[SecureHeadersGatewayFilterFactory@fceab5d configClass = Object]": null,
  "[SaveSessionGatewayFilterFactory@4449b273 configClass = Object]": null
}
```
响应包含应用于任何特定路由的 GatewayFilter 工厂的详细信息。 对于每个工厂，都有对应对象的字符串表示（例如，\[SecureHeadersGatewayFilterFactory@fceab5d configClass = Object\]）。 请注意，空值是由于端点控制器的实现不完整，因为它尝试设置过滤器链中对象的顺序，这不适用于 GatewayFilter 工厂对象。



## 刷新路由缓存
要清除路由缓存，请向 /actuator/gateway/refresh 发出 POST 请求。 请求返回 200 没有响应正文。



## 检索网关定义的路由
要检索网关中定义的路由，请向 /actuator/gateway/routes 发出 GET 请求。 得到的响应类似于以下内容：

```java
[{
  "route_id": "first_route",
  "route_object": {
    "predicate": "org.springframework.cloud.gateway.handler.predicate.PathRoutePredicateFactory$$Lambda$432/1736826640@1e9d7e7d",
    "filters": [
      "OrderedGatewayFilter{delegate=org.springframework.cloud.gateway.filter.factory.PreserveHostHeaderGatewayFilterFactory$$Lambda$436/674480275@6631ef72, order=0}"
    ]
  },
  "order": 0
},
{
  "route_id": "second_route",
  "route_object": {
    "predicate": "org.springframework.cloud.gateway.handler.predicate.PathRoutePredicateFactory$$Lambda$432/1736826640@cd8d298",
    "filters": []
  },
  "order": 0
}]
```
响应包含网关中定义的所有路由的详细信息。 下表描述了响应的每个元素（每个元素都是一个路由）的结构：

| Path                   | Type   | Description                                                  |
| ---------------------- | ------ | ------------------------------------------------------------ |
| route_id               | String | The route ID.                                                |
| route_object.predicate | Object | The route predicate.                                         |
| route_object.filters   | Array  | The `[GatewayFilter](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gatewayfilter-factories)` [factories](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gatewayfilter-factories) applied to the route. |
| order                  | Number | The route order.                                             |

要检索有关单个路由的信息，请向 /actuator/gateway/routes/{id}（例如，/actuator/gateway/routes/first\_route）发出 GET 请求。 得到的响应类似于以下内容：

```java
{
  "id": "first_route",
  "predicates": [{
    "name": "Path",
    "args": {"_genkey_0":"/first"}
  }],
  "filters": [],
  "uri": "https://www.uri-destination.org",
  "order": 0
}]
```
下表描述了响应的结构：

| Path       | Type   | Description                                                  |
| ---------- | ------ | ------------------------------------------------------------ |
| id         | String | The route ID.                                                |
| predicates | Array  | The collection of route predicates. Each item defines the name and the arguments of a given predicate. |
| filters    | Array  | The collection of filters applied to the route.              |
| uri        | String | The destination URI of the route.                            |
| order      | Number | The route order.                                             |

要创建路由，请使用指定路由字段的 JSON 正文向 /gateway/routes/{id\_route\_to\_create} 发出 POST 请求（请参阅检索有关特定路由的信息）。

要删除路由，请向 /gateway/routes/{id\_route\_to\_delete} 发出 DELETE 请求。



回顾：所有端点的列表

| ID            | HTTP Method | Description                                                  |
| ------------- | ----------- | ------------------------------------------------------------ |
| globalfilters | GET         | Displays the list of global filters applied to the routes.   |
| routefilters  | GET         | Displays the list of `GatewayFilter` factories applied to a particular route. |
| refresh       | POST        | Clears the routes cache.                                     |
| routes        | GET         | Displays the list of routes defined in the gateway.          |
| routes/{id}   | GET         | Displays information about a particular route.               |
| routes/{id}   | POST        | Adds a new route to the gateway.                             |
| routes/{id}   | DELETE      | Removes an existing route from the gateway.                  |

# 日志
以下记录器可能包含 DEBUG 和 TRACE 级别的有价值的故障排除信息：

```java
org.springframework.cloud.gateway
```
```java
org.springframework.http.server.reactive
```
```java
org.springframework.web.reactive
```
```java
org.springframework.boot.autoconfigure.web
```
```java
reactor.netty
```
```java
redisratelimiter
```
Reactor Netty HttpClient 和 HttpServer 可以启用窃听。 当与将 reactor.netty 日志级别设置为 DEBUG 或 TRACE 结合使用时，它可以记录信息，例如通过线路发送和接收的标头和正文。 要启用窃听，请分别为 HttpServer 和 HttpClient 设置 spring.cloud.gateway.httpserver.wiretap=true 或 spring.cloud.gateway.httpclient.wiretap=true。



# 自定义过滤器
## 自己编写谓词
为了编写路由谓词，您需要实现 RoutePredicateFactory。 有一个名为 AbstractRoutePredicateFactory 的抽象类，您可以对其进行扩展。

```java
public class MyRoutePredicateFactory extends AbstractRoutePredicateFactory<HeaderRoutePredicateFactory.Config> {

    public MyRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        // grab configuration from Config object
        return exchange -> {
            //grab the request
            ServerHttpRequest request = exchange.getRequest();
            //take information from the request to see if it
            //matches configuration.
            return matches(config, request);
        };
    }

    public static class Config {
        //Put the configuration properties for your filter here
    }

}
```
## 自己编写GatewayFilter
要编写 GatewayFilter，您必须实现 GatewayFilterFactory。 您可以扩展名为 AbstractGatewayFilterFactory 的抽象类。 以下示例显示了如何执行此操作：

```java
public class PreGatewayFilterFactory extends AbstractGatewayFilterFactory<PreGatewayFilterFactory.Config> {

    public PreGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        // grab configuration from Config object
        return (exchange, chain) -> {
            //If you want to build a "pre" filter you need to manipulate the
            //request before calling chain.filter
            ServerHttpRequest.Builder builder = exchange.getRequest().mutate();
            //use builder to manipulate the request
            return chain.filter(exchange.mutate().request(builder.build()).build());
        };
    }

    public static class Config {
        //Put the configuration properties for your filter here
    }

}
```
```java
public class PostGatewayFilterFactory extends AbstractGatewayFilterFactory<PostGatewayFilterFactory.Config> {

    public PostGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        // grab configuration from Config object
        return (exchange, chain) -> {
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                ServerHttpResponse response = exchange.getResponse();
                //Manipulate the response in some way
            }));
        };
    }

    public static class Config {
        //Put the configuration properties for your filter here
    }

}
```
## 在配置中命名自定义过滤器和引用
自定义过滤器类名称应以 GatewayFilterFactory 结尾。

例如，要在配置文件中引用名为Something 的过滤器，该过滤器必须位于名为SomethingGatewayFilterFactory 的类中。

可以创建一个没有 GatewayFilterFactory 后缀命名的网关过滤器，例如类 AnotherThing。 此过滤器可以在配置文件中作为 AnotherThing 引用。 这不是受支持的命名约定，并且可能会在未来版本中删除此语法。 请更新过滤器名称以符合要求。



## 自定义全局过滤器
要编写自定义全局过滤器，您必须实现 GlobalFilter 接口。 这将过滤器应用于所有请求。

以下示例分别显示了如何设置全局前置过滤器和后置过滤器：

```java
@Bean
public GlobalFilter customGlobalFilter() {
    return (exchange, chain) -> exchange.getPrincipal()
        .map(Principal::getName)
        .defaultIfEmpty("Default User")
        .map(userName -> {
          //adds header to proxied request
          exchange.getRequest().mutate().header("CUSTOM-REQUEST-HEADER", userName).build();
          return exchange;
        })
        .flatMap(chain::filter);
}

@Bean
public GlobalFilter customGlobalPostFilter() {
    return (exchange, chain) -> chain.filter(exchange)
        .then(Mono.just(exchange))
        .map(serverWebExchange -> {
          //adds header to response
          serverWebExchange.getResponse().getHeaders().set("CUSTOM-RESPONSE-HEADER",
              HttpStatus.OK.equals(serverWebExchange.getResponse().getStatusCode()) ? "It worked": "It did not work");
          return serverWebExchange;
        })
        .then();
}
```
# 使用 Spring MVC 或 Webflux 构建简单网关
Spring Cloud Gateway 提供了一个名为 ProxyExchange 的实用程序对象。 您可以在常规 Spring Web 处理程序中使用它作为方法参数。 它通过反映 HTTP 动词的方法支持基本的下游 HTTP 交换。 对于 MVC，它还支持通过 forward() 方法转发到本地处理程序。 要使用 ProxyExchange，请在类路径中包含正确的模块（spring-cloud-gateway-mvc 或 spring-cloud-gateway-webflux）。

以下 MVC 示例将向 /test 下游的请求代理到远程服务器：

```java
@RestController
@SpringBootApplication
public class GatewaySampleApplication {

    @Value("${remote.home}")
    private URI home;

    @GetMapping("/test")
    public ResponseEntity<?> proxy(ProxyExchange<byte[]> proxy) throws Exception {
        return proxy.uri(home.toString() + "/image/png").get();
    }

}
```
以下示例对 Webflux 执行相同的操作：

```java
@RestController
@SpringBootApplication
public class GatewaySampleApplication {

    @Value("${remote.home}")
    private URI home;

    @GetMapping("/test")
    public Mono<ResponseEntity<?>> proxy(ProxyExchange<byte[]> proxy) throws Exception {
        return proxy.uri(home.toString() + "/image/png").get();
    }

}
```
ProxyExchange 上的便捷方法使处理程序方法能够发现和增强传入请求的 URI 路径。 例如，您可能希望提取路径的尾随元素以将它们传递到下游：

```java
@GetMapping("/proxy/path/**")
public ResponseEntity<?> proxyPath(ProxyExchange<byte[]> proxy) throws Exception {
  String path = proxy.path("/proxy/path/");
  return proxy.uri(home.toString() + "/foos/" + path).get();
}
```
Spring MVC 和 Webflux 的所有功能都可用于网关处理程序方法。 因此，您可以注入请求标头和查询参数，例如，您可以使用映射注释中的声明来约束传入的请求。 有关这些功能的更多详细信息，请参阅 Spring MVC 中 @RequestMapping 的文档。

您可以使用 ProxyExchange 上的 header() 方法向下游响应添加标头。

您还可以通过向 get() 方法（和其他方法）添加映射器来操作响应标头（以及响应中您喜欢的任何其他内容）。 映射器是一个函数，它接受传入的 ResponseEntity 并将其转换为传出的。

为不向下游传递的“敏感”标头（默认情况下，cookie 和授权）和“代理”（x-forwarded-\*）标头提供一流的支持。



# 属性列表
[https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/appendix.html](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/appendix.html)



# 源码
ServerWebExchange: HTTP 请求-响应交互的契约。 提供对 HTTP 请求和响应的访问，还公开其他与服务器端处理相关的属性和功能，例如请求属性。

```java
public interface ServerWebExchange {
    String LOG_ID_ATTRIBUTE = ServerWebExchange.class.getName() + ".LOG_ID";

    ServerHttpRequest getRequest();

    ServerHttpResponse getResponse();

    Map<String, Object> getAttributes();

    @Nullable
    default <T> T getAttribute(String name) {
        return this.getAttributes().get(name);
    }

    default <T> T getRequiredAttribute(String name) {
        T value = this.getAttribute(name);
        Assert.notNull(value, () -> {
            return "Required attribute '" + name + "' is missing";
        });
        return value;
    }

    default <T> T getAttributeOrDefault(String name, T defaultValue) {
        return this.getAttributes().getOrDefault(name, defaultValue);
    }

    Mono<WebSession> getSession();

    <T extends Principal> Mono<T> getPrincipal();

    Mono<MultiValueMap<String, String>> getFormData();

    Mono<MultiValueMap<String, Part>> getMultipartData();

    LocaleContext getLocaleContext();

    @Nullable
    ApplicationContext getApplicationContext();

    boolean isNotModified();

    boolean checkNotModified(Instant var1);

    boolean checkNotModified(String var1);

    boolean checkNotModified(@Nullable String var1, Instant var2);

    String transformUrl(String var1);

    void addUrlTransformer(Function<String, String> var1);

    String getLogPrefix();

    default ServerWebExchange.Builder mutate() {
        return new DefaultServerWebExchangeBuilder(this);
    }

    public interface Builder {
        ServerWebExchange.Builder request(Consumer<org.springframework.http.server.reactive.ServerHttpRequest.Builder> var1);

        ServerWebExchange.Builder request(ServerHttpRequest var1);

        ServerWebExchange.Builder response(ServerHttpResponse var1);

        ServerWebExchange.Builder principal(Mono<Principal> var1);

        ServerWebExchange build();
    }
}

```
