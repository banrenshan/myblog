---
title: spring-cloud-discovery
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:53:04
---

# @EnableDiscoveryClient

Spring Cloud Commons 提供了 @EnableDiscoveryClient 注解。 这会在META-INF/spring.factories（org.springframework.cloud.client.discovery.EnableDiscoveryClient是key，value是具体的实现） 文件中搜索 DiscoveryClient 和 ReactiveDiscoveryClient 接口的实现。 DiscoveryClient 实现的示例包括 Spring Cloud Netflix Eureka、Spring Cloud Consul Discovery 和 Spring Cloud Zookeeper Discovery。



默认情况下，Spring Cloud 将提供阻塞和反应式服务发现客户端。 您可以通过设置 spring.cloud.discovery.blocking.enabled=false 或 spring.cloud.discovery.reactive.enabled=false 轻松禁用阻塞和/或反应客户端。 要完全禁用服务发现，您只需设置 spring.cloud.discovery.enabled=false。



默认情况下， DiscoveryClient 的实现会自动向远程发现服务器注册本地 Spring Boot 应用。 可以通过在 @EnableDiscoveryClient 中设置 autoRegister=false 来禁用此行为。



@EnableDiscoveryClient不需要显示声明。 只需要在类路径上放置一个 DiscoveryClient 实现。



# 健康检查



- `spring.cloud.discovery.client.health-indicator.enabled=false`.禁用健康检查
- 要禁用描述字段，请设置 spring.cloud.discovery.client.health-indicator.include-description=false。 否则，它可能会冒泡作为汇总的 HealthIndicator 的描述。
- 要禁用服务检索，请设置 spring.cloud.discovery.client.health-indicator.use-services-query=false。 默认情况下，指标调用客户端的 getServices 方法。 在具有许多注册服务的部署中，在每次检查期间检索所有服务的成本可能太高。 这将跳过服务检索，而是使用客户端的探测方法。

##### [DiscoveryCompositeHealthContributor](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#discoverycompositehealthcontributor)复合健康指标基于所有已注册的 DiscoveryHealthIndicator bean。 要禁用，请设置 spring.cloud.discovery.client.composite-indicator.enabled=false。



DiscoveryClientHealthIndicator里面实现：

```java
	public Health health() {
		Health.Builder builder = new Health.Builder();

		if (this.discoveryInitialized.get()) {
			try {
				DiscoveryClient client = this.discoveryClient.getIfAvailable();
				String description = (this.properties.isIncludeDescription()) ? client.description() : "";

				if (properties.isUseServicesQuery()) {
					List<String> services = client.getServices();
					builder.status(new Status("UP", description)).withDetail("services", services);
				}
				else {
					client.probe(); // ==client.getServices();
					builder.status(new Status("UP", description));
				}
			}
			catch (Exception e) {
				this.log.error("Error", e);
				builder.down(e);
			}
		}
		else {
			builder.status(new Status(Status.UNKNOWN.getCode(), "Discovery Client not initialized"));
		}
		return builder.build();
	}
```



## 发现客户端的顺序

DiscoveryClient 接口扩展了 Ordered。 这在使用多个发现客户端时很有用，因为它允许您定义返回的发现客户端的顺序，类似于如何对 Spring 应用程序加载的 bean 进行排序。 默认情况下，任何 DiscoveryClient 的顺序设置为 0。如果您想为自定义 DiscoveryClient 实现设置不同的顺序，您只需覆盖 getOrder() 方法，以便它返回适合您设置的值。 除此之外，您可以使用属性来设置 Spring Cloud 提供的 DiscoveryClient 实现的顺序，其中包括 ConsulDiscoveryClient、EurekaDiscoveryClient 和 ZookeeperDiscoveryClient。 为此，您只需将 spring.cloud.{clientIdentifier}.discovery.order （或 Eureka 的 eureka.client.order）属性设置为所需的值。



## [ SimpleDiscoveryClient](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#simplediscoveryclient)

如果类路径中没有 Service-Registry-backed DiscoveryClient，将使用 SimpleDiscoveryClient 实例，它使用属性来获取有关服务和实例的信息。



有关可用实例的信息应按以下格式通过属性传递：

spring.cloud.discovery.client.simple.instances.service1[0].uri=http://s11:8080，其中spring.cloud.discovery.client.simple.instances为公共前缀，那么service1代表服务ID，而 [0] 表示实例的索引号（如示例中可见，索引从 0 开始），然后 uri 的值是实例可用的实际 URI。





# 注册

Commons 现在提供了一个 ServiceRegistry 接口，该接口提供 register(Registration) 和 deregister(Registration) 等方法，让您可以提供自定义注册服务。

```java
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguration {
    private ServiceRegistry registry;

    public MyConfiguration(ServiceRegistry registry) {
        this.registry = registry;
    }

    // called through some external process, such as an event or a custom actuator endpoint
    public void register() {
        Registration registration = constructRegistration();
        this.registry.register(registration);
    }
}
```

- `ZookeeperRegistration` used with `ZookeeperServiceRegistry`
- `EurekaRegistration` used with `EurekaServiceRegistry`
- `ConsulRegistration` used with `ConsulServiceRegistry`



默认情况下，ServiceRegistry 实现会自动注册正在运行的服务。 要禁用该行为，您可以设置： @EnableDiscoveryClient(autoRegister=false) 以永久禁用自动注册。  spring.cloud.service-registry.auto-registration.enabled=false 禁用行为。



服务自动注册时将触发两个事件。 第一个事件称为 InstancePreRegisteredEvent，在注册服务之前触发。 第二个事件称为 InstanceRegisteredEvent，在注册服务后触发。 您可以注册一个 ApplicationListener(s) 来监听和响应这些事件。





Spring Cloud Commons 提供了一个 /service-registry 执行器端点。 此端点依赖于 Spring 应用程序上下文中的注册 bean。 使用 GET 调用 /service-registry 会返回注册的状态。 对具有 JSON 正文的同一端点使用 POST 会将当前注册的状态更改为新值。 JSON 正文必须包含具有首选值的状态字段。 请参阅更新状态和状态返回值时用于允许值的 ServiceRegistry 实现的文档。 例如，Eureka 支持的状态是 UP、DOWN、OUT_OF_SERVICE 和 UNKNOWN。



## 忽略网络接口

有时，忽略某些命名的网络接口很有用，以便它们可以从服务发现注册中排除（例如，在 Docker 容器中运行时）。 可以设置正则表达式列表以导致所需的网络接口被忽略。 以下配置忽略了 docker0 接口和所有以 veth 开头的接口：

```yaml
spring:
  cloud:
    inetutils:
      ignoredInterfaces:
        - docker0
        - veth.*
```

您还可以通过使用正则表达式列表强制仅使用指定的网络地址，如以下示例所示：

```yaml
spring:
  cloud:
    inetutils:
      preferredNetworks:
        - 192.168
        - 10.0
```

您还可以强制仅使用站点本地地址，如以下示例所示：

```yaml
spring:
  cloud:
    inetutils:
      useOnlySiteLocalInterfaces: true
```



# 创建http client的工厂



Spring Cloud Commons 提供了用于创建 Apache HTTP 客户端 (ApacheHttpClientFactory) 和 OK HTTP 客户端 (OkHttpClientFactory) 的 bean。 只有当 OK HTTP jar 位于类路径上时，才会创建 OkHttpClientFactory bean。 此外，Spring Cloud Commons 提供了用于创建两个客户端使用的连接管理器的 bean：ApacheHttpClientConnectionManagerFactory 用于 Apache HTTP 客户端，OkHttpClientConnectionPoolFactory 用于 OK HTTP 客户端。 如果您想自定义如何在下游项目中创建 HTTP 客户端，您可以提供您自己的这些 bean 的实现。 此外，如果您提供类型为 HttpClientBuilder 或 OkHttpClient.Builder 的 bean，则默认工厂使用这些构建器作为返回到下游项目的构建器的基础。 您还可以通过将 spring.cloud.httpclientfactories.apache.enabled 或 spring.cloud.httpclientfactories.ok.enabled 设置为 false 来禁用这些 bean 的创建。
