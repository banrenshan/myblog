---
title: spring-boot-admin
tags:
  - spring-boot
  - spring-boot-admin
categories:
  - java
  - spring-boot
date: 2022-12-02 11:58:00
---

# Spring Boot Admin
codecentric 的 Spring Boot Admin 是一个社区项目，用于管理和监控您的 Spring Boot ® 应用程序。 应用程序使用 Spring Boot Admin Client 向我们注册（通过 HTTP）或使用 Spring Cloud （例如 Eureka、Consul）被发现。

使用 Pyctuator 可以支持 Python 应用程序。

# 快速开始
### 安装Spring Boot Admin Server
首先，您需要设置您的服务器。 为此，只需设置一个简单的启动项目（使用 start.spring.io）。 Spring Boot Admin Server 能够作为 servlet 或 webflux 应用程序运行，你可以根据需要添加相应的 Spring Boot Starter。 在这个例子中，我们使用了 servlet web starter：

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>3.0.0-M3</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
然后，在配置类上添加@EnableAdminServer注解

```java
@Configuration
@EnableAutoConfiguration
@EnableAdminServer
public class SpringBootAdminApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdminApplication.class, args);
    }
}
```


## 注册客户端
要在 SBA 服务器上注册您的应用程序，您可以包含 SBA 客户端或使用 Spring Cloud Discovery（例如 Eureka、Consul ......）。 还有一个在 SBA 服务器端使用静态配置的简单选项。

### 客户端方式
1. 每个想要注册的应用程序都必须包含 Spring Boot Admin Client。 为了保护端点，还要添加 spring-boot-starter-security。

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>3.0.0-M3</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
2. 通过配置 Spring Boot Admin Server 的 URL 来启用 SBA Client：

```xml
spring.boot.admin.client.url=http://localhost:8080  
management.endpoints.web.exposure.include=*  
management.info.env.enabled=true 
```
* 要注册的 Spring Boot 管理服务器的 URL。
* 与 Spring Boot 2 一样，默认情况下大多数端点都不会通过 http 公开，我们将它们全部公开。 对于生产，您应该仔细选择要公开的端点。
* 从 Spring Boot 2.6 开始，默认情况下禁用 env info 。 因此，我们必须启用它。

3. 使执行器端点可访问：

```java
@Configuration
public static class SecurityPermitAllConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().permitAll()  
            .and().csrf().disable();
    }
}
```
为简洁起见，我们现在禁用安全性。更适合生产的配置请参考安全配置部分。

### 服务发现方式
如果您已经将 Spring Cloud Discovery 用于您的应用程序，则不需要 SBA 客户端。 只需在 Spring Boot Admin Server 中添加一个 DiscoveryClient，其余的由我们的 AutoConfiguration 完成。



以下步骤使用 Eureka，但也支持其他 Spring Cloud Discovery 实现。 

1. 添加依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>${eureka-client.version}</version>
</dependency>
```
2. 启用服务发现配置

```java
@Configuration
@EnableAutoConfiguration
@EnableDiscoveryClient
@EnableScheduling
@EnableAdminServer
public class SpringBootAdminApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdminApplication.class, args);
    }

    @Configuration
    public static class SecurityPermitAllConfig extends WebSecurityConfigurerAdapter {
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests().anyRequest().permitAll()  
                .and().csrf().disable();
        }
    }
}
```
3. 配置服务发现

```yaml
eureka:   
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health
    metadata-map:
      startup: ${random.int}    #needed to trigger info and endpoint update after restart
  client:
    registryFetchIntervalSeconds: 5
    serviceUrl:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/

management:
  endpoints:
    web:
      exposure:
        include: "*"  
  endpoint:
    health:
      show-details: ALWAYS
```
# 客户端
### 显示应用版本信息
对于 Spring Boot 应用程序，显示版本的最简单方法是使用 spring-boot-maven-plugin 中的 build-info 目标，该目标生成 META-INF/build-info.properties。 另请参阅 Spring Boot [参考指南](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-build-info)。

对于非 Spring Boot 应用程序，您可以将版本或 build.version 添加到注册元数据，版本将显示在应用程序列表中。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>build-info</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
```groovy
springBoot {
  buildInfo()
}
```


### JMX-beans 管理
要在管理 UI 中与 JMX-bean 交互，您必须在应用程序中包含 Jolokia。 由于 Jolokia 是基于 servlet 的，因此不支持反应式应用程序。 如果您使用的是 spring-boot-admin-starter-client ，它将为您提取，如果没有将 Jolokia 添加到您的依赖项中。 对于 Spring Boot 2.2.0，如果您想通过 JMX 公开 Spring bean，您可能需要设置 spring.jmx.enabled=true。

```xml
<dependency>
    <groupId>org.jolokia</groupId>
    <artifactId>jolokia-core</artifactId>
</dependency>
```
### 日志文件查看器
默认情况下，日志文件无法通过执行器端点访问，因此在 Spring Boot Admin 中不可见。 为了启用日志文件执行器端点，您需要通过设置 logging.file.path 或 logging.file.name 来配置 Spring Boot 以写入日志文件。



Spring Boot Admin 将检测所有看起来像 URL 的内容并将其呈现为超链接。

还支持 ANSI 颜色转义。 您需要设置自定义文件日志模式，因为 Spring Boot 的默认模式不使用颜色。

要强制使用 ANSI 彩色输出，请设置 spring.output.ansi.enabled=ALWAYS。 否则 Spring 会尝试检测 ANSI 彩色输出是否可用，并可能禁用它。

```yaml
logging.file.name=/var/log/sample-boot-application.log 
logging.pattern.file=%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(%5p) %clr(${PID}){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n%wEx 
```
### 显示标签
标签是为每个实例添加视觉标记的一种方式，它们将出现在应用程序列表以及实例视图中。 默认情况下，不会向实例添加标签，由客户端通过将信息添加到元数据或info端点来指定所需的标签。

```yaml
#using the metadata
spring.boot.admin.client.instance.metadata.tags.environment=test

#using the info endpoint
info.tags.environment=test
```
### Spring Boot Admin Client
Spring Boot Admin Client 在管理服务器上注册应用程序。 这是通过定期向 SBA 服务器发出 HTTP 发布请求来提供有关应用程序的信息来完成的。

| Property name                                                | Description                                                  | Default value                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| spring.boot.admin.client.enabled                             | 是否启用 Spring Boot Admin Client.                           | true                                                         |
| spring.boot.admin.client.url                                 | 逗号分隔的 要注册的 Spring Boot 管理服务器的 URL 的有序列表。 这会触发自动配置。 强制的。 |                                                              |
| spring.boot.admin.client.api-path                            | 管理服务器上注册端点的 Http 路径。                           | "instances"                                                  |
| spring.boot.admin.client.username<br>spring.boot.admin.client.password | 如果 SBA 服务器 api 受 HTTP 基本身份验证保护，则用户名和密码。 |                                                              |
| spring.boot.admin.client.period                              | 重复注册的时间间隔（以毫秒为单位）。                         | 10,000                                                       |
| spring.boot.admin.client.connect-timeout                     | 注册连接超时（以毫秒为单位）。                               | 5,000                                                        |
| spring.boot.admin.client.read-timeout                        | 注册的读取超时（以毫秒为单位）。                             | 5,000                                                        |
| spring.boot.admin.client.auto-registration                   | 如果设置为 true，则在应用程序准备好后自动安排注册应用程序的定期任务。 | true                                                         |
| spring.boot.admin.client.auto-deregistration                 | 当上下文关闭时，切换以在 Spring Boot 管理服务器上启用自动注销。 如果未设置该值，则如果检测到正在运行的 CloudPlatform，则该功能处于活动状态。 | null                                                         |
| spring.boot.admin.client.register-once                       | 如果设置为 true，客户端将只注册一个管理服务器（按 spring.boot.admin.instance.url 定义的顺序）； 如果该管理服务器出现故障，将自动注册到下一个管理服务器。 如果为 false，将对所有管理服务器进行注册。 | true                                                         |
| spring.boot.admin.client.instance.health-url                 | 注册的健康网址。 如果可访问的 URL 不同（例如 Docker），则可以覆盖。 在注册表中必须是唯一的。 | 根据 management-url 和 endpoints.health.id 猜测              |
| spring.boot.admin.client.instance.management-base-url        | 用于计算要注册的管理 URL 的基本 URL。 该路径在运行时推断，并附加到基本 url。 | 根据 management.port、service-url 和 server.servlet-path 猜测。 |
| spring.boot.admin.client.instance.management-url             | 要注册的管理网址。 如果可访问的 url 不同（例如 Docker），可以覆盖。 | 根据 management-base-url 和 management.context-path 猜测。   |
| spring.boot.admin.client.instance.service-base-url           | 用于计算要注册的服务 url 的基本 url。 该路径在运行时推断，并附加到基本 url。 在 Cloudfoundry 环境中，您可以像这样切换到 https：spring.boot.admin.client.instance.service-base-url=https://\${vcap.application.uris\[0\]} | 根据主机名 server.port 猜测。                                |
| spring.boot.admin.client.instance.service-url                | 要注册的服务网址。 如果可访问的 url 不同（例如 Docker），可以覆盖。 | 根据 service-base-url 和 server.context-path 猜测。          |
| spring.boot.admin.client.instance.service-path               | 注册的服务路径。 如果可达路径不同（例如，以编程方式设置的上下文路径），则可以覆盖。 | /                                                            |
| spring.boot.admin.client.instance.name                       | 要注册的名称。                                               | \${spring.application.name} 如果设置，否则为“spring-boot-application”。 |
| spring.boot.admin.client.instance.service-host-type          | 选择发送服务主机时应考虑哪些信息：<br>\* IP：使用 InetAddress.getHostAddress() 返回的 IP<br>\* HOST\_NAME：使用 InetAddress.getHostName() 返回的单台机器的主机名<br>\* CANONICAL\_HOST\_NAME：使用 InetAddress.geCanonicalHostName() 返回的 FQDN<br>如果在服务中设置了 server.address 或 management.address，则该值将覆盖该属性。 | CANONICAL_HOST_NAME                                          |
| spring.boot.admin.client.instance.metadata.\*                | 要与此实例关联的元数据键值对。                               |                                                              |
| spring.boot.admin.client.instance.metadata.tags.\*           | 要与此实例关联的标签键值对。                                 |                                                              |

# 服务端
## 在代理服务器后面运行
如果 Spring Boot Admin 服务器在反向代理后面运行，则可能需要配置 (spring.boot.admin.ui.public-url) 公共 url。 另外当反向代理终止https连接时，可能需要配置server.forward-headers-strategy=native（另见Spring Boot[参考指南](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-use-tomcat-behind-a-proxy-server)）。

## 配置项
| Property name                                                | Description                                                  | Default value                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| spring.boot.admin.server.enabled                             | 启用Spring Boot Admin Server.                                | true                                                         |
| spring.boot.admin.context-path                               | context-path 为应该提供 Admin Server 静态资产和 API 的路径添加前缀。 相对于 Dispatcher-Servlet。 |                                                              |
| spring.boot.admin.monitor.status-interval                    | 检查实例状态的时间间隔。                                     | 10,000ms                                                     |
| spring.boot.admin.monitor.status-lifetime                    | 状态的生命周期。 只要最后一个状态没有过期，状态就不会更新。  | 10,000ms                                                     |
| spring.boot.admin.monitor.info-interval                      | 检查实例信息的时间间隔。                                     | 1m                                                           |
| spring.boot.admin.monitor.info-lifetime                      | 信息的生命周期。 只要最后一个信息没有过期，信息就不会更新。  | 1m                                                           |
| spring.boot.admin.monitor.default-timeout                    | 发出请求时的默认超时。 可以使用 spring.boot.admin.monitor.timeout.\* 覆盖特定端点的各个值。 | 10,000                                                       |
| spring.boot.admin.monitor.timeout.\*                         | 每个 endpointId 具有超时的键值对。 默认为default-timeout。   |                                                              |
| spring.boot.admin.monitor.default-retries                    | 失败请求的默认重试次数。 永远不会重试修改请求（PUT、POST、PATCH、DELETE）。 可以使用 spring.boot.admin.monitor.retries.\* 覆盖特定端点的各个值。 | 0                                                            |
| spring.boot.admin.monitor.retries.\*                         |                                                              |                                                              |
| spring.boot.admin.metadata-keys-to-sanitize                  | 匹配这些正则表达式模式的键的元数据值将在所有 json 输出中进行清理。 | `".**password$", ".*secret$", ".*key$", ".*token$", ".*credentials.**", ".*vcap_services$"` |
| spring.boot.admin.probed-endpoints                           | 对于 Spring Boot 1.x 客户端应用程序，SBA 使用 OPTIONS 请求探测指定端点。 如果路径与 id 不同，您可以将其指定为 id:path (例如 health:ping).. | "health", "env", "metrics", "httptrace:trace", "threaddump:dump", "jolokia", "info", "logfile", "refresh", "flyway", "liquibase", "heapdump", "loggers", "auditevents" |
| spring.boot.admin.instance-auth.enabled                      | 启用从 spring 配置属性中提取凭据                             | true                                                         |
| spring.boot.admin.instance-auth.default-user-name            | 用于对注册服务进行身份验证的默认用户名。 spring.boot.admin.instance-auth.enabled 属性必须为 true。 | null                                                         |
| spring.boot.admin.instance-auth.default-password             | 用于对注册服务进行身份验证的默认用户密码。 spring.boot.admin.instance-auth.enabled 属性必须为 true。 | null                                                         |
| spring.boot.admin.instance-auth.service-map.\*.user-name     | 用于向具有指定名称的注册服务进行身份验证的用户名。 spring.boot.admin.instance-auth.enabled 属性必须为 true。 |                                                              |
| spring.boot.admin.instance-auth.service-map.\*.user-password | 用于向具有指定名称的注册服务进行身份验证的用户密码。 spring.boot.admin.instance-auth.enabled 属性必须为 true。 |                                                              |
| spring.boot.admin.instance-proxy.ignored-headers             | 向客户端发出请求时不转发标头。                               | "Cookie", "Set-Cookie", "Authorization"                      |
| spring.boot.admin.ui.public-url                              | 用于在 ui 中构建基本 href 的基本 url。                       | If running behind a reverse proxy (using path rewriting) this can be used to make correct self references. If the host/port is omitted it will be inferred from the request. |
| spring.boot.admin.ui.brand                                   | 要在导航栏中显示的品牌。                                     | "<img src="assets/img/icon-spring-boot-admin.svg"><span>Spring Boot Admin</span>" |
| spring.boot.admin.ui.title                                   | 要显示的页面标题。                                           | "Spring Boot Admin"                                          |
| spring.boot.admin.ui.login-icon                              | 图标用作登录页面上的图像。                                   | "assets/img/icon-spring-boot-admin.svg"                      |
| spring.boot.admin.ui.favicon                                 | 用作桌面通知的默认图标。                                     | "assets/img/favicon.png"                                     |
| spring.boot.admin.ui.favicon-danger                          | 当一项或多项服务关闭时用作图标图标并用于桌面通知。           | "assets/img/favicon-danger.png"                              |
| spring.boot.admin.ui.remember-me-enabled                     | 切换以显示/隐藏登录页面上的记住我复选框。                    | true                                                         |
| spring.boot.admin.ui.poll-timer.cache                        | 以毫秒为单位的轮询持续时间以获取新的缓存数据。               | 2500                                                         |
| spring.boot.admin.ui.poll-timer.datasource                   | 以毫秒为单位的轮询持续时间以获取新的数据源数据。             | 2500                                                         |
| spring.boot.admin.ui.poll-timer.gc                           | 以毫秒为单位的轮询持续时间以获取新的 gc 数据。               | 2500                                                         |
| spring.boot.admin.ui.poll-timer.process                      | 以毫秒为单位的轮询持续时间以获取新的流程数据。               | 2500                                                         |
| spring.boot.admin.ui.poll-timer.memory                       | 以毫秒为单位的轮询持续时间以获取新的内存数据。               | 2500                                                         |
| spring.boot.admin.ui.poll-timer.threads                      | 以毫秒为单位的轮询持续时间以获取新的线程数据。               | 2500                                                         |

## Spring Cloud Discovery
Spring Boot Admin Server 可以使用 Spring Clouds DiscoveryClient 来发现应用程序。 优点是客户端不必包含 spring-boot-admin-starter-client。 您只需将 DiscoveryClient 实现添加到您的管理服务器 - 其他一切都由 AutoConfiguration 完成。

### 使用 SimpleDiscoveryClient 的静态配置
Spring Cloud 提供了一个 SimpleDiscoveryClient。 它允许您通过静态配置指定客户端应用程序：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter</artifactId>
</dependency>
```
```yaml
spring:
  cloud:
    discovery:
      client:
        simple:
          instances:
            test:
              - uri: http://instance1.intern:8080
                metadata:
                  management.context-path: /actuator
              - uri: http://instance2.intern:8080
                metadata:
                  management.context-path: /actuator
```
## 通知
### 邮件通知
邮件通知将作为使用 Thymeleaf 模板呈现的 HTML 电子邮件发送。 要启用邮件通知，请使用 spring-boot-starter-mail 配置 JavaMailSender 并设置收件人。

![image](ZG00DOdyfbo1qIztR49RePSYdQk-101Gm7JsRpeVFNw.png)

1. 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```
2. 配置JavaMailSender

```yaml
spring.mail.host=smtp.example.com
spring.boot.admin.notify.mail.to=admin@example.com
```
| Property name                                       | Description                                                  | Default value                                                |
| --------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|                                                     |                                                              |                                                              |
| spring.boot.admin.notify.mail.enabled               | Enable mail notifications                                    | true                                                         |
| spring.boot.admin.notify.mail.ignore-changes        | 要忽略的状态更改的逗号分隔列表。 格式： ”：”。 允许使用通配符。 | "UNKNOWN:UP"                                                 |
| spring.boot.admin.notify.mail.template              | 用于渲染的 Thymeleaf 模板的资源路径。                        | "classpath:/META-INF/spring-boot-admin-server/mail/status-changed.html" |
| spring.boot.admin.notify.mail.to                    | 以逗号分隔的邮件收件人列表                                   | "root@localhost"                                             |
| spring.boot.admin.notify.mail.cc                    | 以逗号分隔的抄送收件人列表                                   |                                                              |
| spring.boot.admin.notify.mail.from                  | Mail sender                                                  | "Spring Boot Admin <noreply@localhost>"                      |
| spring.boot.admin.notify.mail.additional-properties | 可以从模板访问的其他属性                                     |                                                              |



# 安全
