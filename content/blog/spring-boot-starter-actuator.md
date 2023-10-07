---
title: spring-boot-starter-actuator
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:59:36
---

# Spring Boot性能监控
Spring Boot 包含许多附加功能，可帮助您在将应用程序推送到生产环境时对其进行监控和管理。 您可以选择使用 HTTP 端点或 JMX 来管理和监视您的应用程序。 审计、健康和指标收集也可以自动应用于您的应用程序。

这些功能是通过 `spring-boot-actuator` 模块实现的，你只需要在类路径添加`spring-boot-starter-actuator` 依赖即可。

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```
# 端点
执行器端点使您可以监视应用程序并与之交互。Spring Boot 包含许多内置端点，并允许您添加自己的端点。例如，`health`端点提供基本的应用程序健康信息。

您可以[启用或禁用](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.enabling)每个单独的端点并[通过 HTTP 或 JMX 公开它们（使它们可以远程访问）](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.exposing)。当端点被启用和公开时，它被认为是可用的。内置端点仅在可用时才会自动配置。大多数应用程序选择通过 HTTP 公开，其中端点的 ID 和前缀`/actuator`映射到 URL。例如，默认情况下，`health`端点映射到`/actuator/health`.

以下与技术无关的端点：

| ID               | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| auditevents      | 公开当前应用程序的审计事件信息。<br>需要一个`AuditEventRepository bean`。 |
| beans            | 显示应用程序中所有 Spring bean 的完整列表。                  |
| caches           | 公开可用的缓存。                                             |
| conditions       | 显示在配置和自动配置类上评估的条件<br>以及它们匹配或不匹配的原因。 |
| configprops      | 显示所有`@ConfigurationProperties`.                          |
| env              | 公开 Spring 的`ConfigurableEnvironment`.                     |
| flyway           | 显示已应用的任何 Flyway 数据库迁移。<br>需要一个或多个`Flyway bean`。 |
| health           | 显示应用程序运行状况信息。                                   |
| httptrace        | 显示 HTTP 跟踪信息（默认情况下，<br>最近 100 个 HTTP 请求-响应交换）。<br>需要一个`HttpTraceRepository bean`。 |
| info             | 显示任意应用程序信息。                                       |
| integrationgraph | 显示 Spring 集成图。需要依赖`spring-integration-core`.       |
| loggers          | 显示和修改应用程序中记录器的配置。                           |
| liquibase        | 显示已应用的任何 Liquibase 数据库迁移。<br>需要一个或多个`Liquibase bean`。 |
| metrics          | 显示当前应用程序的“指标”信息。                               |
| mappings         | 显示所有`@RequestMapping`路径的整理列表。                    |
| quartz           | 显示有关 Quartz 调度程序作业的信息。                         |
| scheduledtasks   | 显示应用程序中的计划任务。                                   |
| sessions         | 允许从 Spring Session 支持的会话存储中检索<br>和删除用户会话。需要使用 Spring Session 的<br>基于 servlet 的 Web 应用程序。 |
| shutdown         | 让应用程序正常关闭。默认禁用。                               |
| startup          | 显示由`ApplicationStartup` 收集的启动过程数据<br>需要`SpringApplication`配置`BufferingApplicationStartup`. |
| threaddump       | 执行线程转储。                                               |

如果您的应用程序是 Web 应用程序（Spring MVC、Spring WebFlux 或 Jersey），您可以使用以下附加端点：

| ID         | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| heapdump   | 返回一个堆转储文件。在 HotSpot JVM 上，<br>返回一个`HPROF-format` 文件。在 OpenJ9 JVM 上，<br>返回一个 `PHD-format` 文件。 |
| jolokia    | 当 Jolokia 在类路径上时，通过 HTTP 公开 JMX bean<br>（不适用于 WebFlux）。需要依赖`jolokia-core`. |
| logfile    | 返回日志文件的内容（如果已设置`logging.file.name`）。<br>`logging.file.path`支持使用 HTTP`Range`标头检索部分日志文件内容。 |
| prometheus | 以 Prometheus 服务器可以抓取的格式公开指标。<br>需要依赖`micrometer-registry-prometheus`. |

## 启动端点
默认，除了 `shutdown` 端点，其他端点都是启用状态。开启端点的配置： `management.endpoint.<id>.enabled` ，例如：

```yaml
management:
  endpoint:
    shutdown:
      enabled: true
```
如果您希望端点启用是选择加入而不是选择退出，请将 `management.endpoints.enabled-by-default` 属性设置为 false 并使用单个启用端点的属性来选择重新加入。以下示例启用 info 端点并禁用 所有其他端点：

```yaml
management:
  endpoints:
    enabled-by-default: false
  endpoint:
    info:
      enabled: true
```
> 禁用的端点完全从应用程序上下文中删除。 如果您只想更改暴露端点的技术，请改用 include 和 exclude 属性。

## 公开端点
由于端点可能包含敏感信息，您应该仔细考虑何时公开它们。下标列出了默认公开的端点：

| ID               | JMX  | Web  |
| ---------------- | ---- | ---- |
| auditevents      | Yes  | No   |
| beans            | Yes  | No   |
| caches           | Yes  | No   |
| conditions       | Yes  | No   |
| configprops      | Yes  | No   |
| env              | Yes  | No   |
| flyway           | Yes  | No   |
| health           | Yes  | Yes  |
| heapdump         | N/A  | No   |
| httptrace        | Yes  | No   |
| info             | Yes  | No   |
| integrationgraph | Yes  | No   |
| jolokia          | N/A  | No   |
| logfile          | N/A  | No   |
| loggers          | Yes  | No   |
| liquibase        | Yes  | No   |
| metrics          | Yes  | No   |
| mappings         | Yes  | No   |
| prometheus       | N/A  | No   |
| quartz           | Yes  | No   |
| scheduledtasks   | Yes  | No   |
| sessions         | Yes  | No   |
| shutdown         | Yes  | No   |
| startup          | Yes  | No   |
| threaddump       | Yes  | No   |

要更改公开的端点，请使用以下特定于技术的include和exclude属性：

| Property                                  | Default |
| ----------------------------------------- | ------- |
| management.endpoints.jmx.exposure.exclude |         |
| management.endpoints.jmx.exposure.include | *       |
| management.endpoints.web.exposure.exclude |         |
| management.endpoints.web.exposure.include | health  |

`include`属性列出了公开的端点的 ID。`exclude`属性列出不应公开的端点的 ID。`exclude`优先于`include`。您可以使用端点 ID 列表来配置`include`和`exclude`属性。

```yaml
management:
  endpoints:
    jmx:
      exposure:
        include: "health,info"
```
`*`可用于选择所有端点。例如，要通过 HTTP 公开除`env`和`beans`端点之外的所有内容，请使用以下属性：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
        exclude: "env,beans"
```
`*`在 YAML 中具有特殊含义，因此如果要包含（或排除）所有端点，请务必添加引号。

## 安全
出于安全目的，默认情况下只有`/health`端点通过 HTTP 公开。

如果 Spring Security 在类路径上并且不存在其他`WebSecurityConfigurerAdapter`或`SecurityFilterChain`bean，则除`/health`之外的所有端点都受保护。

如果您希望为 HTTP 端点配置自定义安全性（例如，只允许具有特定角色的用户访问它们），Spring Boot 提供了一些方便`RequestMatcher`的对象，您可以将它们与 Spring Security 结合使用：

```java
import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration(proxyBeanMethods = false)
public class MySecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.requestMatcher(EndpointRequest.toAnyEndpoint())
                .authorizeRequests((requests) -> requests.anyRequest().hasRole("ENDPOINT_ADMIN"));
        http.httpBasic();
        return http.build();
    }

}
```
#### 跨站点请求伪造保护
由于 Spring Boot 依赖于 Spring Security 的默认设置，因此默认开启 CSRF 保护。这意味着在使用默认安全配置时，所有的POST（shutdown端点）和PUT、DELETE请求会收到403失败。

>   我们建议仅在创建非浏览器客户端使用的服务时完全禁用 CSRF 保护。



不带任何参数的读取操作,端点会自动缓存响应。要配置端点缓存的时间，请使用`cache.time-to-live`属性。以下示例将`beans`端点缓存的生存时间设置为 10 秒：

```yaml
management:
  endpoint:
    beans:
      cache:
        time-to-live: "10s"
```


CORS 支持默认禁用，只有在您设置了`management.endpoints.web.cors.allowed-origins`属性后才会启用

```yaml
management:
  endpoints:
    web:
      cors:
        allowed-origins: "https://example.com"
        allowed-methods: "GET,POST"
```
## 自定义端点
如果您添加一个带有@Endpoint 注释的@Bean，那么任何带有@ReadOperation、@WriteOperation 或@DeleteOperation 的方法都会自动通过JMX 公开，并且在Web 应用程序中，也可以通过HTTP 公开。 可以使用 Jersey、Spring MVC 或 Spring WebFlux 通过 HTTP 公开端点。 如果 Jersey 和 Spring MVC 都可用，则使用 Spring MVC。



TDB



# 健康信息
您可以使用健康信息来检查正在运行的应用程序的状态。当生产系统出现故障时，监控软件经常使用它来提醒某人。端点公开的`health`信息取决于`management.endpoint.health.show-details`和`management.endpoint.health.show-components`属性，可以使用以下值之一进行配置：

| Name            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| never           | 从不显示细节。默认                                           |
| when-authorized | 详细信息仅向授权用户显示。可以使用 <br>`management.endpoint.health.roles`配置授权角色。 |
| always          | 向所有用户显示详细信息。                                     |





# 指标监控
Spring Boot 自动配置一个复合 MeterRegistry， 并自动添加类路径上的所有registry实现。 在运行时类路径中依赖 micrometer-registry-{system} 就足以让 Spring Boot 配置注册表。

即使 Micrometer registry 实现在类路径上，您也可以禁用特定registry。 以下示例禁用 Datadog：

```yaml
management:
  metrics:
    export:
      datadog:
        enabled: false
```


#### Prometheus
Spring Boot 提供了一个执行器端点`/actuator/prometheus` 暴漏指标数据。该端点默认不公开，你需要暴漏。

prometheus.yml配置如下：

```yaml
scrape_configs:
  - job_name: "spring"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["HOST:PORT"]
```
prometheus间隔访问`/actuator/prometheus` 来获取指标数据。

对于可能存在时间不够长而无法被抓取的临时或批处理作业，您可以使用[Prometheus Pushgateway](https://github.com/prometheus/pushgateway)支持，将指标推送给 Prometheus。要启用 Prometheus Pushgateway 支持，请将以下依赖项添加到您的项目中：

```Plain Text
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_pushgateway</artifactId>
</dependency>
```
当类路径上存在 Prometheus Pushgateway 依赖项并且`management.metrics.export.prometheus.pushgateway.enabled`属性设置为 `true`时，`PrometheusPushGatewayManager`会自动配置 bean。`PrometheusPushGatewayManager`管理将指标推送到 Prometheus Pushgateway。

对于高级配置，您还可以提供自己的`PrometheusPushGatewayManager`bean。
