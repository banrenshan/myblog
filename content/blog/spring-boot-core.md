---
title: spring-boot-core
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:42:06
---

# SpringApplication

SpringApplication类提供了一种从main方法启动的Spring应用程序方法。在许多情况下，您可以委托给静态的`SpringApplication.run`方法，如下例所示：

```java
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

当应用程序启动时，您应该会看到类似于以下输出的内容：

```shell
 .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.7.5)

2022-10-20 12:40:17.841  INFO 16284 --- [           main] o.s.b.d.f.s.MyApplication                : Starting MyApplication using Java 1.8.0_345 on myhost with PID 16284 (/opt/apps/myapp.jar started by myuser in /opt/apps/)
2022-10-20 12:40:17.849  INFO 16284 --- [           main] o.s.b.d.f.s.MyApplication                : No active profile set, falling back to 1 default profile: "default"
2022-10-20 12:40:20.443  INFO 16284 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-10-20 12:40:20.455  INFO 16284 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-10-20 12:40:20.455  INFO 16284 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.68]
2022-10-20 12:40:20.716  INFO 16284 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-10-20 12:40:20.716  INFO 16284 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2566 ms
2022-10-20 12:40:22.045  INFO 16284 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-10-20 12:40:22.073  INFO 16284 --- [           main] o.s.b.d.f.s.MyApplication                : Started MyApplication in 4.937 seconds (JVM running for 6.049)
```

默认情况下，会显示INFO日志消息，包括一些相关的启动详细信息，例如启动应用程序的用户。应用程序版本是使用主应用程序类包中的实现版本确定的。通过设置`spring.main.log-startup-info=false`可以关闭启动信息日志。这也将关闭 `active profiles` 的日志信息。

要在启动期间，添加更多的日志信息，可以重写SpringApplication类中的 logStartupInfo(boolean) 方法。

## 启动失败

如果您的应用程序无法启动，注册的FailureAnalyzers将提供错误消息和解决方案。例如，如果您在端口8080上启动web应用程序，并且该端口已经在使用中，您应该会看到类似于以下消息的内容：

```shell
***************************
APPLICATION FAILED TO START
***************************

Description:

Embedded servlet container failed to start. Port 8080 was already in use.

Action:

Identify and stop the process that is listening on port 8080 or configure this application to listen on another port.
```

spring boot 提供了多种 FailureAnalyzer 实现，你也可以实现自己的，参考 [howto ](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.application.failure-analyzer)页面。

如果没有故障分析器能够处理异常，您仍然可以显示完整的条件报告，以更好地了解出了什么问题。为此，需要为启用debug属性，或者设置`org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener`的日志级别为debug。

例如，如果您使用 `java -jar `运行应用程序，则可以按如下方式启用debug属性：

```shell
$ java -jar myproject-0.0.1-SNAPSHOT.jar --debug
```

## 惰性初始化

SpringApplication允许应用程序惰性初始化。当启用惰性初始化时，bean将根据需要而不是在应用程序启动期间创建。因此，启用惰性初始化可以减少应用程序启动所需的时间。在web应用程序中，启用延迟初始化将导致许多与web相关的bean在收到HTTP请求之前未被初始化。

惰性初始化的一个缺点是，它会延迟应用程序问题的发现。如果错误配置的bean被惰性初始化，那么在启动期间将不再发生故障，并且只有在bean被初始化时，问题才会变得明显。还必须注意确保JVM有足够的内存来容纳所有应用程序的bean，而不仅仅是那些在启动期间初始化的bean。出于这些原因，默认情况下不启用惰性初始化，建议在启用惰性初始化之前对JVM的堆大小进行微调。

可以使用`SpringApplicationBuilder`上的`lazyInitialization`方法或`SpringApplication`上的`setLazyIninitialization`方法以编程方式启用Lazy初始化。或者，可以使用`spring.main.lazy-initialization`启用它。如下例所示：

```properties
spring.main.lazy-initialization=true
```

如果希望在使用惰性初始化时禁用某些bean的惰性初始化，可以使用@lazy（false）注解。

## 自定义Banner

启动时打印的横幅可以通过在类路径添加 banner.txt 文件，或通过`spring.banner.location`属性设置为此banner.txt文件的位置。如果文件的编码不是UTF-8，则可以设置`spring.banner.charset`。

除了文本文件，您还可以在类路径添加 添加 banner.gif, banner.jpg, 或 banner.png 文件。或者通过`spring.banner.image.location` 显示指定图片位置。

在你的banner.txt文件中，您可以使用环境中可用的任何键以及以下任何占位符：

| Variable                                                     | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ${application.version}                                       | 应用程序的版本号，如MANIFEST.MF中声明的。例如，Implementation-Version: 1.0 打印为1.0。 |
| ${application.formatted-version}                             | 在MANIFEST.MF中声明的应用程序的版本号。并格式化以供显示（用括号包围并以v为前缀）。例如（v1.0）。 |
| ${spring-boot.version}                                       | spring boot的版本号                                          |
| ${spring-boot.formatted-version}                             | 您正在使用的Spring Boot版本，已格式化以供显示（用括号包围并以v为前缀）。例如（v2.7.5）。 |
| ${Ansi.NAME} (or ${AnsiColor.NAME}, ${AnsiBackground.NAME}, ${AnsiStyle.NAME}) | 其中，NAME是ANSI转义代码的名称。有关详细信息，请参见AnsiPropertySource。 |
| ${application.title}                                         | 应用程序的标题，如MANIFEST.MF中声明的。例如， Implementation-Title: MyApp 打印为MyApp。 |

如果你想通过编程的方式设置banner可以调用SpringApplication.setBanner(…) 方法。



你可以使用 `spring.main.banner-mode` 属性控制是否打印banner

- console: 使用 System.out 打印
- log： 
- off

banner 被注册为名为`springBootBanner `的单例bean。



只有在使用Spring Boot启动器时，${application.version}和${appliation.formattedversion}属性才可用。如果您正在运行一个未打包的jar，并使用 java-cp＜classpath＞＜mainclass＞启动它，则这些值将无法解析。这就是为什么我们建议您总是使用`java org.springframework.boot.loader.JarLauncher `启动未打包的jar。这将在构建classpath和启动应用程序之前 初始化application.* 变量 。

## 自定义SpringApplication

如果SpringApplication默认值不符合您的口味，您可以创建一个本地实例并对其进行自定义。例如，要关闭banner：

```java
import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(MyApplication.class);
        application.setBannerMode(Banner.Mode.OFF);
        application.run(args);
    }

}
```



传递给SpringApplication的构造函数参数是Springbean的配置类。在大多数情况下，这些是对`@Configuration`类的引用，但它们也可以是对`@Component`类的直接引用。

也可以使用`application.properties`配置SpringApplication。有关详细信息，请参阅外部化配置。

## Fluent Builder API

如果您需要构建ApplicationContext层次结构（具有父/子关系的多个上下文），或者如果您更喜欢使用“流畅”的构建器API，可以使用SpringApplicationBuilder。

SpringApplicationBuilder允许您将多个方法调用链接在一起，并包括允许您创建层次结构的父子上下文，如以下示例所示：

```java
new SpringApplicationBuilder()
        .sources(Parent.class)
        .child(Application.class)
        .bannerMode(Banner.Mode.OFF)
        .run(args);
```

创建ApplicationContext层次结构时有一些限制。例如，Web组件必须包含在子上下文中，并且父上下文和子上下文都使用相同的 Environment 。有关详细信息，请参阅SpringApplicationBuilder [Javadoc](https://docs.spring.io/spring-boot/docs/2.7.5/api/org/springframework/boot/SpringApplication.html)。

## 应用可用性

在平台上部署时，应用程序可以使用Kubernetes Probes等基础设施向平台提供有关其可用性的信息。Spring Boot提供了“活跃（liveness）”和“就绪（readiness）”可用性状态开箱即用的支持。如果您正在使用Spring Boot的“actuator”支持，那么这些状态将作为健康端点组公开。

此外，您还可以通过将ApplicationAvailability接口注入到自己的bean中来获得可用性状态。

### 活跃状态

应用程序的“活跃性”状态指示其内部状态是否允许其正常工作，或者在当前出现故障时能否自行恢复。中断的“活动性”状态意味着应用程序处于无法恢复的状态，基础结构应该重新启动应用程序。

通常，“活跃”状态不应基于外部检查，例如健康检查。如果是这样，失败的外部系统（数据库、Web API、外部缓存）将触发整个平台的大规模重启和级联故障。

Spring Boot应用程序的内部状态主要由Spring ApplicationContext表示。如果应用程序上下文已成功启动，Spring Boot将假定应用程序处于有效状态。上下文刷新后，应用程序即被视为活跃，请参阅Spring Boot应用程序生命周期和相关应用程序事件。

### 就绪状态

应用程序的“就绪”状态告诉应用程序是否准备好处理流量。失败的“就绪”状态告诉平台，目前不应将流量路由到应用程序。这通常发生在启动过程中，在处理`CommandLineRunner`和`ApplicationRunner`组件时，或者在应用程序确定其太忙而无法进行其他通信时。

一旦调用了应用程序和命令行运行程序，就认为应用程序已经就绪，请参阅Spring Boot应用程序生命周期和相关的应用程序事件。

预期在启动期间运行的任务应由CommandLineRunner和ApplicationRunner组件执行，而不是使用Spring组件生命周期回调（如@PostConstruct）。

### 管理应用程序可用性状态

应用程序组件可以通过注入ApplicationAvailability接口并对其调用方法，随时检索当前可用性状态。更常见的情况是，应用程序希望侦听状态更新或更新应用程序的状态。

例如，我们可以将应用程序的“就绪”状态导出到一个文件，以便Kubernetes“exec Probe”可以查看该文件：

```java
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class MyReadinessStateExporter {

    @EventListener
    public void onStateChange(AvailabilityChangeEvent<ReadinessState> event) {
        switch (event.getState()) {
            case ACCEPTING_TRAFFIC:
                // create file /tmp/healthy
                break;
            case REFUSING_TRAFFIC:
                // remove file /tmp/healthy
                break;
        }
    }

}
```

当应用程序中断且无法恢复时，我们还可以更新应用程序的状态：

```java
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.LivenessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

@Component
public class MyLocalCacheVerifier {

    private final ApplicationEventPublisher eventPublisher;

    public MyLocalCacheVerifier(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void checkLocalCache() {
        try {
            // ...
        }
        catch (CacheCompletelyBrokenException ex) {
            AvailabilityChangeEvent.publish(this.eventPublisher, ex, LivenessState.BROKEN);
        }
    }

}
```

## 应用程序事件和监听器

除了通常的Spring Framework事件（如ContextRefreshedEvent），SpringApplication还发送一些其他的应用程序事件。

有些事件实际上是在ApplicationContext创建之前触发的，因此您不能将侦听器注册为@Bean。您可以使用SpringApplication.addListeners（…) 方法或SpringApplicationBuilder.listers（…) 方法来注册他们



如果您希望自动注册这些侦听器，无论应用程序的创建方式如何，可以添加到 META-INF/spring.factories文件 ，并使用org.springframework.context.ApplicationListener 当作 key，如以下所示：org.springframework.context.ApplicationListener=com.example.project.MyListener



应用程序事件在应用程序运行时按以下顺序发送：

1. ApplicationStartingEvent在运行开始时发送，但在任何处理之前发送(listeners 和 initializers 注册除外)
2. 当上下文中要使用的环境已知但在创建上下文之前，将发送ApplicationEnvironmentPreparedEvent。
3. ApplicationContextInitializedEvent在ApplicationContext准备好并调用ApplicationContextInitializers时发送，但在加载任何bean定义之前。
4. ApplicationPreparedEvent在刷新开始之前但在加载bean定义之后发送。
5. ApplicationStartedEvent在上下文刷新后但在调用任何应用程序和命令行运行程序之前发送。
6. LivenessState.CORRECT表示应用程序被视为活动，随机发送AvailabilityChangeEvent
7. 在调用任何应用程序和命令行运行程序后，将发送ApplicationReadyEvent。
8. ReadinessState.ACCEPTING_TRAFFIC 状态后，发送 AvailabilityChangeEvent 
9. 如果启动时出现异常，将发送ApplicationFailedEvent。

以上列表仅包括绑定到SpringApplication的SpringApplicationEvents。除此之外，以下事件也在`ApplicationPreparedEvent`之后和`ApplicationStartedEvent`之前发布：

- `WebServerInitializedEvent`在WebServer就绪后发送。`ServletWebServerInitializedEvent`和`ReactiveWebServerIninitializedEvent`分别是servlet和反应式变体。
- 刷新`ApplicationContext`时会发送`ContextRefreshedEvent`。

您通常不需要使用应用程序事件，但知道它们的存在会很方便。在内部，Spring Boot使用事件来处理各种任务。

默认情况下，事件侦听器不应运行可能很长的任务，因为它们在同一线程中执行。请考虑改用应用程序和命令行运行程序。

应用程序事件通过使用Spring Framework的事件发布机制发送。此机制的一部分确保了在子上下文中发布给侦听器的事件也发布给任何祖先上下文中的侦听器。因此，如果您的应用程序使用SpringApplication实例的层次结构，侦听器可能会接收相同类型的应用程序事件的多个实例。

为了让侦听器区分其上下文的事件和后代上下文的事件，它应该在请求中注入其应用程序上下文，然后将注入的上下文与事件接受的上下文进行比较。上下文可以通过实现`ApplicationContextAware`来注入，如果侦听器是bean，则可以使用`@Autowired`来注入。

## web环境

`SpringApplication`尝试代表您创建正确类型的`ApplicationContext`。用于确定`WebApplicationType`的算法如下：

- 如果存在Spring MVC，则使用`AnnotationConfigServletWebServerApplicationContext`
- 如果Spring MVC不存在且Spring WebFlux存在，则使用`AnnotationConfigReactiveWebServerApplicationContext`
- 否则，将使用`AnnotationConfigApplicationContext`

这意味着，如果您在同一应用程序中使用`SpringMVC`和`SpringWebFlux`中的WebClient，则默认情况下将使用`Spring MVC`。您可以通过调用`setWebApplicationType（WebApplicationType）`轻松地覆盖它。

可以用setApplicationContextClass(…) 完全控制 ApplicationContext 的类型。

在JUnit测试中使用SpringApplication时，通常需要调用setWebApplicationType（WebApplicationType.NONE）。

## 访问应用程序参数

如果您需要访问传递给SpringApplication.run(…）的应用程序参数。 您可以注入`org.springframework.boot.ApplicationArguments `bean。ApplicationArguments接口提供对原始String[]参数的解析，如以下示例所示：

```java
import java.util.List;

import org.springframework.boot.ApplicationArguments;
import org.springframework.stereotype.Component;

@Component
public class MyBean {

        public MyBean(ApplicationArguments args) {
            boolean debug = args.containsOption("debug");
            List<String> files = args.getNonOptionArgs();
            if (debug) {
                System.out.println(files);
            }
            // if run with "--debug logfile.txt" prints ["logfile.txt"]
        }
}
```

Spring Boot还向Spring Environment注册`CommandLinePropertySource`。这还允许您使用`@Value`注释注入单个应用程序参数。

## ApplicationRunner 和 CommandLineRunner

如果需要在SpringApplication启动后运行某些特定代码，可以实现`ApplicationRunner`或`CommandLineRunner`接口。这两个接口的工作方式相同，并提供了单一的运行方法，该方法在SpringApplication.run(…) 调用完成前运行。

此契约非常适合于在应用程序启动后但在它开始接受流量之前运行的任务。



`CommandLineRunner`接口以字符串数组的形式提供对应用程序参数的访问，而`ApplicationRunner`使用前面讨论的`ApplicationArguments`接口。以下示例显示了带有run方法的`CommandLineRunner`：

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class MyCommandLineRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        // Do something...
    }

}
```



如果定义了几个必须按特定顺序调用的`CommandLineRunner`或`ApplicationRunner `bean，则可以另外实现`org.springframework.core.Ordered` 接口或使用`org.springframework.core.annotation.Order` 注释。

## 应用程序退出

每个`SpringApplication`向JVM注册一个关闭挂钩，以确保`ApplicationContext`在退出时正常关闭。保证所有标准Spring生命周期回调（例如DisposableBean接口或@PreDestroy注释）可以正常工作。

此外，如果他们希望在调用`SpringApplication.exit()`时返回特定的退出代码,可以实现`org.springframework.boot.ExitCodeGenerator`接口。退出代码传递给 `System.exit() `方法 返回，如以下示例所示：

```java
import org.springframework.boot.ExitCodeGenerator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class MyApplication {

    @Bean
    public ExitCodeGenerator exitCodeGenerator() {
        return () -> 42;
    }

    public static void main(String[] args) {
        System.exit(SpringApplication.exit(SpringApplication.run(MyApplication.class, args)));
    }

}
```

此外，`ExitCodeGenerator`接口可能由异常类实现。当遇到此类异常时，Spring Boot返回由`getExitCode()`方法提供的退出代码。

如果存在多个`ExitCodeGenerator`，则使用生成的第一个非零退出代码。要控制生成器的调用顺序，则可以另外实现`org.springframework.core.Ordered `接口或使用`org.springframework.core.annotation.Orde`r 注释。

## 管理功能

通过指定`spring.application.admin.enabled `属性 可以为应用程序启用与管理相关的功能。这将在MBeanServer平台上公开SpringApplicationAdminMXBean。您可以使用此功能远程管理Spring Boot应用程序。此特性对于任何服务包装器实现都可能有用。

如果您想知道应用程序在哪个HTTP端口上运行，请使用`local.server.port`键获取该属性。

## 应用程序启动跟踪

在应用程序启动期间，`SpringApplication`和`ApplicationContext`执行许多与应用程序生命周期、bean生命周期甚至处理应用程序事件相关的任务。使用`ApplicationStartup`，Spring Framework允许您使用`StartupStep`对象跟踪应用程序启动顺序。可以收集这些数据用于分析目的，或者只是为了更好地理解应用程序启动过程。

您可以在设置`SpringApplication`实例时选择`ApplicationStartup`实现。例如，要使用`BufferingApplicationStartup`，您可以编写：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.metrics.buffering.BufferingApplicationStartup;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(MyApplication.class);
        application.setApplicationStartup(new BufferingApplicationStartup(2048));
        application.run(args);
    }

}
```



第一个可用的实现`FlightRecorderApplicationStartup`由Spring Framework提供。它将特定于Spring的启动事件添加到`Java Flight Recorder`会话，用于分析应用程序并将其Spring上下文生命周期与JVM事件（例如分配、GC、类加载…). 配置后，您可以在启用飞行记录器的情况下运行应用程序来记录数据：

```shell
$ java -XX:StartFlightRecording:filename=recording.jfr,duration=10s -jar demo.jar
```

Spring Boot附带`BufferingApplicationStartup`变体；此实现用于缓冲启动步骤并将其排放到外部度量系统中。应用程序可以在任何组件中请求`BufferingApplicationStartup`类型的bean。

Spring Boot还可以配置为公开一个`startup`端点，该端点以JSON文档的形式提供此信息。下面是一个示例信息：

```json
{
    "springBootVersion": "2.7.4",
    "timeline": {
        "startTime": "2022-10-29T02:37:38.916369700Z",
        "events": [
            {
                "endTime": "2022-10-29T02:37:38.948011400Z",
                "duration": "PT0.0160168S",
                "startTime": "2022-10-29T02:37:38.931994600Z",
                "startupStep": {
                    "name": "spring.boot.application.starting",
                    "id": 0,
                    "tags": [
                        {
                            "key": "mainApplicationClass",
                            "value": "com.example.graphql.GraphqlApplication"
                        }
                    ],
                    "parentId": null
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.167809700Z",
                "duration": "PT0.1416612S",
                "startTime": "2022-10-29T02:37:39.026148500Z",
                "startupStep": {
                    "name": "spring.boot.application.environment-prepared",
                    "id": 1,
                    "tags": [],
                    "parentId": null
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.246328200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.246328200Z",
                "startupStep": {
                    "name": "spring.boot.application.context-prepared",
                    "id": 2,
                    "tags": [],
                    "parentId": null
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.277586500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.277586500Z",
                "startupStep": {
                    "name": "spring.boot.application.context-loaded",
                    "id": 3,
                    "tags": [],
                    "parentId": null
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.324487700Z",
                "duration": "PT0.0156507S",
                "startTime": "2022-10-29T02:37:39.308837Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 7,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.internalCachingMetadataReaderFactory"
                        }
                    ],
                    "parentId": 6
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.340124900Z",
                "duration": "PT0.0312879S",
                "startTime": "2022-10-29T02:37:39.308837Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 6,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.context.annotation.internalConfigurationAnnotationProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.795760100Z",
                "duration": "PT0.450084S",
                "startTime": "2022-10-29T02:37:39.345676100Z",
                "startupStep": {
                    "name": "spring.context.config-classes.parse",
                    "id": 9,
                    "tags": [
                        {
                            "key": "classCount",
                            "value": "113"
                        }
                    ],
                    "parentId": 8
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.811385800Z",
                "duration": "PT0.4712609S",
                "startTime": "2022-10-29T02:37:39.340124900Z",
                "startupStep": {
                    "name": "spring.context.beandef-registry.post-process",
                    "id": 8,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.context.annotation.ConfigurationClassPostProcessor@7c041b41"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.811385800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.811385800Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 10,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor@7a231dfd"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.811385800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.811385800Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 11,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor@30814f43"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.846643400Z",
                "duration": "PT0.0352576S",
                "startTime": "2022-10-29T02:37:39.811385800Z",
                "startupStep": {
                    "name": "spring.context.config-classes.enhance",
                    "id": 13,
                    "tags": [
                        {
                            "key": "classCount",
                            "value": "1"
                        }
                    ],
                    "parentId": 12
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.846643400Z",
                "duration": "PT0.0352576S",
                "startTime": "2022-10-29T02:37:39.811385800Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 12,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.context.annotation.ConfigurationClassPostProcessor@7c041b41"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.846643400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.846643400Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 14,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.boot.SpringApplication$PropertySourceOrderingBeanFactoryPostProcessor@45667d98"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.846643400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.846643400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 15,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "propertySourcesPlaceholderConfigurer"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanFactoryPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.846643400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.846643400Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 16,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.context.support.PropertySourcesPlaceholderConfigurer@3bbf9027"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.846643400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.846643400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 17,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.sql.init.dependency.DatabaseInitializationDependencyConfigurer$DependsOnDatabaseInitializationPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanFactoryPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0.0120072S",
                "startTime": "2022-10-29T02:37:39.846643400Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 18,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.boot.sql.init.dependency.DatabaseInitializationDependencyConfigurer$DependsOnDatabaseInitializationPostProcessor@15e0fe05"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 19,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.context.event.internalEventListenerProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanFactoryPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 20,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "preserveErrorControllerTargetClassPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanFactoryPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 21,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "forceAutoProxyCreatorToUseClassProxying"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanFactoryPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 23,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.context.event.internalEventListenerFactory"
                        }
                    ],
                    "parentId": 22
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 22,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.context.event.EventListenerMethodProcessor@6d2dc9d2"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 24,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration$PreserveErrorControllerTargetClassPostProcessor@b2f4ece"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.context.bean-factory.post-process",
                    "id": 25,
                    "tags": [
                        {
                            "key": "postProcessor",
                            "value": "org.springframework.boot.autoconfigure.aop.AopAutoConfiguration$ClassProxyingConfiguration$$Lambda$424/0x0000000800372c40@7e1f584d"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 26,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.context.annotation.internalAutowiredAnnotationProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 27,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.context.annotation.internalCommonAnnotationProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 30,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.context.internalConfigurationPropertiesBinderFactory"
                        },
                        {
                            "key": "beanType",
                            "value": "class org.springframework.boot.context.properties.ConfigurationPropertiesBinder$Factory"
                        }
                    ],
                    "parentId": 29
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 29,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.context.internalConfigurationPropertiesBinder"
                        },
                        {
                            "key": "beanType",
                            "value": "class org.springframework.boot.context.properties.ConfigurationPropertiesBinder"
                        }
                    ],
                    "parentId": 28
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.858650600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 28,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.context.properties.ConfigurationPropertiesBindingPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0.015634S",
                "startTime": "2022-10-29T02:37:39.858650600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 31,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.aop.config.internalAutoProxyCreator"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 32,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webServerFactoryCustomizerBeanPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 33,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "errorPageRegistrarBeanPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 34,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthEndpointGroupsBeanPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 35,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "meterRegistryPostProcessor"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.beans.factory.config.BeanPostProcessor"
                        }
                    ],
                    "parentId": 5
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0.5810356S",
                "startTime": "2022-10-29T02:37:39.293249Z",
                "startupStep": {
                    "name": "spring.context.beans.post-process",
                    "id": 5,
                    "tags": [],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.874284600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 38,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryConfiguration$EmbeddedTomcat"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.905535400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.905535400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 40,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration$TomcatWebSocketConfiguration"
                        }
                    ],
                    "parentId": 39
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.905535400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.905535400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 39,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "websocketServletWebServerCustomizer"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.905535400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.905535400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 42,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration"
                        }
                    ],
                    "parentId": 41
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.921162200Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 44,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.context.properties.BoundConfigurationProperties"
                        },
                        {
                            "key": "beanType",
                            "value": "class org.springframework.boot.context.properties.BoundConfigurationProperties"
                        }
                    ],
                    "parentId": 43
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0.0156268S",
                "startTime": "2022-10-29T02:37:39.905535400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 43,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "server-org.springframework.boot.autoconfigure.web.ServerProperties"
                        }
                    ],
                    "parentId": 41
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0.0156268S",
                "startTime": "2022-10-29T02:37:39.905535400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 41,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "servletWebServerFactoryCustomizer"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.921162200Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 45,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "tomcatServletWebServerFactoryCustomizer"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.921162200Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 47,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration$TomcatWebServerFactoryCustomizerConfiguration"
                        }
                    ],
                    "parentId": 46
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.921162200Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 46,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "tomcatWebServerFactoryCustomizer"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.921162200Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 49,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration"
                        }
                    ],
                    "parentId": 48
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.921162200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.921162200Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 48,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "localeCharsetMappingsCustomizer"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.946792800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 51,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration"
                        }
                    ],
                    "parentId": 50
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.946792800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 53,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration$DispatcherServletRegistrationConfiguration"
                        }
                    ],
                    "parentId": 52
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.946792800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 55,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration$DispatcherServletConfiguration"
                        }
                    ],
                    "parentId": 54
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.946792800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 56,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.mvc-org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties"
                        }
                    ],
                    "parentId": 54
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0.0060056S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 54,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "dispatcherServlet"
                        }
                    ],
                    "parentId": 52
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.952798400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 59,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.servlet.multipart-org.springframework.boot.autoconfigure.web.servlet.MultipartProperties"
                        }
                    ],
                    "parentId": 58
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.952798400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 58,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration"
                        }
                    ],
                    "parentId": 57
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:39.952798400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 57,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "multipartConfigElement"
                        }
                    ],
                    "parentId": 52
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0.0060056S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 52,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "dispatcherServletRegistration"
                        }
                    ],
                    "parentId": 50
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0.0060056S",
                "startTime": "2022-10-29T02:37:39.946792800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 50,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "errorPageCustomizer"
                        }
                    ],
                    "parentId": 37
                }
            },
            {
                "endTime": "2022-10-29T02:37:39.952798400Z",
                "duration": "PT0.0785138S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 37,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "tomcatServletWebServerFactory"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.boot.web.servlet.server.ServletWebServerFactory"
                        }
                    ],
                    "parentId": 36
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 62,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.metrics-org.springframework.boot.actuate.autoconfigure.metrics.MetricsProperties"
                        }
                    ],
                    "parentId": 61
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 61,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.web.servlet.WebMvcMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 60
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 64,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.export.simple.SimpleMetricsExportAutoConfiguration"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 66,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.metrics.export.simple-org.springframework.boot.actuate.autoconfigure.metrics.export.simple.SimpleProperties"
                        }
                    ],
                    "parentId": 65
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 65,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "simpleConfig"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 68,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.MetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 67
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.109472100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 67,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "micrometerClock"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.125084Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.125084Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 69,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "propertiesMeterFilter"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.125084Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.125084Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 71,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.web.client.HttpClientMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 70
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.125084Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.125084Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 70,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "metricsHttpClientUriTagFilter"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.125084Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.125084Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 72,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "metricsHttpServerUriTagFilter"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.140709100Z",
                "duration": "PT0.0156251S",
                "startTime": "2022-10-29T02:37:40.125084Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 74,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.JvmMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 73
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.140709100Z",
                "duration": "PT0.0156251S",
                "startTime": "2022-10-29T02:37:40.125084Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 73,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jvmGcMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0.0065076S",
                "startTime": "2022-10-29T02:37:40.140709100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 75,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jvmHeapPressureMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 76,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jvmMemoryMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 77,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jvmThreadMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 78,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "classLoaderMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 80,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.LogbackMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 79
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 79,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "logbackMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 82,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.SystemMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 81
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 81,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "uptimeMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 83,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "processorMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 84,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "fileDescriptorMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.147216700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.147216700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 85,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "diskSpaceMetrics"
                        }
                    ],
                    "parentId": 63
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.156725400Z",
                "duration": "PT0.0472533S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 63,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "simpleMeterRegistry"
                        }
                    ],
                    "parentId": 60
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.156725400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.156725400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 86,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webMvcTagsProvider"
                        }
                    ],
                    "parentId": 60
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.156725400Z",
                "duration": "PT0.0472533S",
                "startTime": "2022-10-29T02:37:40.109472100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 60,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webMvcMetricsFilter"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.boot.web.servlet.ServletContextInitializer"
                        }
                    ],
                    "parentId": 36
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.156725400Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.156725400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 88,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.web.ServletEndpointManagementContextConfiguration$WebMvcServletEndpointManagementContextConfiguration"
                        }
                    ],
                    "parentId": 87
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0.0156331S",
                "startTime": "2022-10-29T02:37:40.156725400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 89,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoints.web-org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointProperties"
                        }
                    ],
                    "parentId": 87
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.172358500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 91,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointAutoConfiguration$WebEndpointServletConfiguration"
                        }
                    ],
                    "parentId": 90
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.172358500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 93,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 92
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.172358500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 92,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webEndpointPathMapper"
                        }
                    ],
                    "parentId": 90
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.172358500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 95,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.web.ServletEndpointManagementContextConfiguration"
                        }
                    ],
                    "parentId": 94
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.172358500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 94,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "servletExposeExcludePropertyEndpointFilter"
                        }
                    ],
                    "parentId": 90
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.172358500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.172358500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 90,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "servletEndpointDiscoverer"
                        }
                    ],
                    "parentId": 87
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.187986100Z",
                "duration": "PT0.0312607S",
                "startTime": "2022-10-29T02:37:40.156725400Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 87,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "servletEndpointRegistrar"
                        },
                        {
                            "key": "beanType",
                            "value": "interface org.springframework.boot.web.servlet.ServletContextInitializer"
                        }
                    ],
                    "parentId": 36
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.187986100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.187986100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 96,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "requestContextFilter"
                        },
                        {
                            "key": "beanType",
                            "value": "interface javax.servlet.Filter"
                        }
                    ],
                    "parentId": 36
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.187986100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.187986100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 98,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration"
                        }
                    ],
                    "parentId": 97
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.187986100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.187986100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 97,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "formContentFilter"
                        },
                        {
                            "key": "beanType",
                            "value": "interface javax.servlet.Filter"
                        }
                    ],
                    "parentId": 36
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.187986100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.187986100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 99,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "characterEncodingFilter"
                        },
                        {
                            "key": "beanType",
                            "value": "interface javax.servlet.Filter"
                        }
                    ],
                    "parentId": 36
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0.3449495S",
                "startTime": "2022-10-29T02:37:39.874284600Z",
                "startupStep": {
                    "name": "spring.boot.webserver.create",
                    "id": 36,
                    "tags": [
                        {
                            "key": "factory",
                            "value": "class org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 100,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "graphqlApplication"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 101,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "bookController"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 102,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "testController"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 103,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.AutoConfigurationPackages"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 104,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 105,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 106,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.context.properties.EnableConfigurationPropertiesRegistrar.methodValidationExcludeFilter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 107,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 108,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 110,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.task.execution-org.springframework.boot.autoconfigure.task.TaskExecutionProperties"
                        }
                    ],
                    "parentId": 109
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 109,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "taskExecutorBuilder"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 111,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration$WhitelabelErrorViewConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 112,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "error"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.219234100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 113,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "beanNameViewResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.234859100Z",
                "duration": "PT0.015625S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 115,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.web-org.springframework.boot.autoconfigure.web.WebProperties"
                        }
                    ],
                    "parentId": 114
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.234859100Z",
                "duration": "PT0.015625S",
                "startTime": "2022-10-29T02:37:40.219234100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 114,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration$DefaultErrorViewResolverConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.234859100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.234859100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 116,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "conventionErrorViewResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.234859100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.234859100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 117,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "errorAttributes"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.234859100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.234859100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 118,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "basicErrorController"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.247370500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.247370500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 120,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration$WebMvcAutoConfigurationAdapter"
                        }
                    ],
                    "parentId": 119
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0.0035082S",
                "startTime": "2022-10-29T02:37:40.247370500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 121,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "metricsWebMvcConfigurer"
                        }
                    ],
                    "parentId": 119
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 123,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.graphql-org.springframework.boot.autoconfigure.graphql.GraphQlProperties"
                        }
                    ],
                    "parentId": 122
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 124,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.graphql.cors-org.springframework.boot.autoconfigure.graphql.GraphQlCorsProperties"
                        }
                    ],
                    "parentId": 122
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 122,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.graphql.servlet.GraphQlWebMvcAutoConfiguration$GraphQlEndpointCorsConfiguration"
                        }
                    ],
                    "parentId": 119
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0.0160196S",
                "startTime": "2022-10-29T02:37:40.234859100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 119,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration$EnableWebMvcConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 126,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcContentNegotiationManager"
                        }
                    ],
                    "parentId": 125
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 127,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcConversionService"
                        }
                    ],
                    "parentId": 125
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.250878700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 128,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcValidator"
                        }
                    ],
                    "parentId": 125
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 130,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration"
                        }
                    ],
                    "parentId": 129
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 132,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration$StringHttpMessageConverterConfiguration"
                        }
                    ],
                    "parentId": 131
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 131,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "stringHttpMessageConverter"
                        }
                    ],
                    "parentId": 129
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 134,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.http.JacksonHttpMessageConvertersConfiguration$MappingJackson2HttpMessageConverterConfiguration"
                        }
                    ],
                    "parentId": 133
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 136,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration$JacksonObjectMapperConfiguration"
                        }
                    ],
                    "parentId": 135
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 138,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration$JacksonObjectMapperBuilderConfiguration"
                        }
                    ],
                    "parentId": 137
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 140,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration$Jackson2ObjectMapperBuilderCustomizerConfiguration"
                        }
                    ],
                    "parentId": 139
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 141,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.jackson-org.springframework.boot.autoconfigure.jackson.JacksonProperties"
                        }
                    ],
                    "parentId": 139
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 139,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "standardJacksonObjectMapperBuilderCustomizer"
                        }
                    ],
                    "parentId": 137
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 143,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration$ParameterNamesModuleConfiguration"
                        }
                    ],
                    "parentId": 142
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.297765Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 142,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "parameterNamesModule"
                        }
                    ],
                    "parentId": 137
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.313387300Z",
                "duration": "PT0.0156223S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 145,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration"
                        }
                    ],
                    "parentId": 144
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.313387300Z",
                "duration": "PT0.0156223S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 144,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jsonComponentModule"
                        }
                    ],
                    "parentId": 137
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.313387300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.313387300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 146,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jsonMixinModule"
                        }
                    ],
                    "parentId": 137
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.313387300Z",
                "duration": "PT0.0156223S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 137,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jacksonObjectMapperBuilder"
                        }
                    ],
                    "parentId": 135
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.329013Z",
                "duration": "PT0.031248S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 135,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jacksonObjectMapper"
                        }
                    ],
                    "parentId": 133
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.329013Z",
                "duration": "PT0.031248S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 133,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mappingJackson2HttpMessageConverter"
                        }
                    ],
                    "parentId": 129
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.329013Z",
                "duration": "PT0.031248S",
                "startTime": "2022-10-29T02:37:40.297765Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 129,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "messageConverters"
                        }
                    ],
                    "parentId": 125
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.329013Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.329013Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 147,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "applicationTaskExecutor"
                        }
                    ],
                    "parentId": 125
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.354137600Z",
                "duration": "PT0.1032589S",
                "startTime": "2022-10-29T02:37:40.250878700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 125,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "requestMappingHandlerAdapter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.354137600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.354137600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 149,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcResourceUrlProvider"
                        }
                    ],
                    "parentId": 148
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.360679500Z",
                "duration": "PT0.0065419S",
                "startTime": "2022-10-29T02:37:40.354137600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 148,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "welcomePageHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.360679500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.360679500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 150,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "localeResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.360679500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.360679500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 151,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "themeResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.360679500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.360679500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 152,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "flashMapManager"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0.0156748S",
                "startTime": "2022-10-29T02:37:40.360679500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 153,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "requestMappingHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 154,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcPatternParser"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 155,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcUrlPathHelper"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 156,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcPathMatcher"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 157,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "viewControllerHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 158,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "beanNameHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 161,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.graphql.servlet.GraphQlWebMvcAutoConfiguration"
                        }
                    ],
                    "parentId": 160
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.376354300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 165,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.graphql.GraphQlAutoConfiguration"
                        }
                    ],
                    "parentId": 164
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.391938600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.391938600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 168,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.graphql.GraphQlMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 167
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.391938600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.391938600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 169,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "graphQlTagsProvider"
                        }
                    ],
                    "parentId": 167
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.391938600Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.391938600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 167,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "graphQlMetricsInstrumentation"
                        }
                    ],
                    "parentId": 166
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.407601600Z",
                "duration": "PT0.015663S",
                "startTime": "2022-10-29T02:37:40.391938600Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 170,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "annotatedControllerConfigurer"
                        }
                    ],
                    "parentId": 166
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.580240500Z",
                "duration": "PT0.2038862S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 166,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "graphQlSource"
                        }
                    ],
                    "parentId": 164
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.580240500Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.580240500Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 171,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "batchLoaderRegistry"
                        }
                    ],
                    "parentId": 164
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.580240500Z",
                "duration": "PT0.2038862S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 164,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "executionGraphQlService"
                        }
                    ],
                    "parentId": 163
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.580240500Z",
                "duration": "PT0.2038862S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 163,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webGraphQlHandler"
                        }
                    ],
                    "parentId": 162
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.580240500Z",
                "duration": "PT0.2038862S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 162,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "graphQlHttpHandler"
                        }
                    ],
                    "parentId": 160
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0.2195105S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 160,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "graphQlRouterFunction"
                        }
                    ],
                    "parentId": 159
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0.2195105S",
                "startTime": "2022-10-29T02:37:40.376354300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 159,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "routerFunctionMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 172,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "resourceHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 173,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "defaultServletHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 174,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "handlerFunctionAdapter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 175,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcUriComponentsContributor"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 176,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "httpRequestHandlerAdapter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 177,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "simpleControllerHandlerAdapter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 178,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "handlerExceptionResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.595864800Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 179,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mvcViewResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0.0156272S",
                "startTime": "2022-10-29T02:37:40.595864800Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 180,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "viewNameTranslator"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 181,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "defaultViewResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 183,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "viewResolver"
                        },
                        {
                            "key": "exception",
                            "value": "class org.springframework.beans.factory.BeanCurrentlyInCreationException"
                        },
                        {
                            "key": "message",
                            "value": "Error creating bean with name 'viewResolver': Requested bean is currently in creation: Is there an unresolvable circular reference?"
                        }
                    ],
                    "parentId": 182
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 182,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "viewResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 184,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.audit.AuditEventsEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 185,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.availability.ApplicationAvailabilityAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 186,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "applicationAvailability"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 187,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.availability.AvailabilityHealthContributorAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 188,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.beans.BeansEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 189,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "beansEndpoint"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 190,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.cache.CachesEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 191,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "cachesEndpoint"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 192,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "cachesEndpointWebExtension"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 193,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.servlet.ServletManagementContextAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 194,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "servletWebChildContextFactory"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 195,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "managementServletContext"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 196,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.health.HealthEndpointConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 198,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoint.health-org.springframework.boot.actuate.autoconfigure.health.HealthEndpointProperties"
                        }
                    ],
                    "parentId": 197
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 197,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthStatusAggregator"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.611492Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 199,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthHttpCodeStatusMapper"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0.0156237S",
                "startTime": "2022-10-29T02:37:40.611492Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 200,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthEndpointGroups"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 203,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.system.DiskSpaceHealthContributorAutoConfiguration"
                        }
                    ],
                    "parentId": 202
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 204,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.health.diskspace-org.springframework.boot.actuate.autoconfigure.system.DiskSpaceHealthIndicatorProperties"
                        }
                    ],
                    "parentId": 202
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 202,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "diskSpaceHealthIndicator"
                        }
                    ],
                    "parentId": 201
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 206,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.health.HealthContributorAutoConfiguration"
                        }
                    ],
                    "parentId": 205
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 205,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "pingHealthContributor"
                        }
                    ],
                    "parentId": 201
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 201,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthContributorRegistry"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 207,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthEndpoint"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 208,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.health.ReactiveHealthEndpointConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 209,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "reactiveHealthContributorRegistry"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 210,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.health.HealthEndpointWebExtensionConfiguration$MvcAdditionalHealthEndpointPathsConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 214,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.EndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 213
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 213,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "endpointOperationParameterMapper"
                        }
                    ],
                    "parentId": 212
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 215,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "endpointMediaTypes"
                        }
                    ],
                    "parentId": 212
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 216,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "endpointCachingOperationInvokerAdvisor"
                        }
                    ],
                    "parentId": 212
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.627115700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 217,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webExposeExcludePropertyEndpointFilter"
                        }
                    ],
                    "parentId": 212
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.642740Z",
                "duration": "PT0.0156243S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 212,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webEndpointDiscoverer"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.648254Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.648254Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 219,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.context.properties.ConfigurationPropertiesReportEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 218
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.648254Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.648254Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 220,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoint.configprops-org.springframework.boot.actuate.autoconfigure.context.properties.ConfigurationPropertiesReportEndpointProperties"
                        }
                    ],
                    "parentId": 218
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.648254Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.648254Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 218,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "configurationPropertiesReportEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.648254Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.648254Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 222,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.env.EnvironmentEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 221
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.648254Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.648254Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 223,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoint.env-org.springframework.boot.actuate.autoconfigure.env.EnvironmentEndpointProperties"
                        }
                    ],
                    "parentId": 221
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.648254Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.648254Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 221,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "environmentEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 225,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.health.HealthEndpointWebExtensionConfiguration"
                        }
                    ],
                    "parentId": 224
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 224,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthEndpointWebExtension"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 227,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.info.InfoEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 226
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 226,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "infoEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 229,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.condition.ConditionsReportEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 228
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 228,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "conditionsReportEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 230,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "configurationPropertiesReportEndpointWebExtension"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 231,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "environmentEndpointWebExtension"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 233,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.logging.LoggersEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 232
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 232,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "loggersEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 235,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.management.HeapDumpWebEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 234
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 234,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "heapDumpWebEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 237,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.management.ThreadDumpEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 236
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 236,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "dumpEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 239,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.MetricsEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 238
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.658761900Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.658761900Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 238,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "metricsEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 241,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.scheduling.ScheduledTasksEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 240
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 240,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "scheduledTasksEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 243,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.startup.StartupEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 242
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 242,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "startupEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 245,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.mappings.MappingsEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 244
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 247,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.mappings.MappingsEndpointAutoConfiguration$ServletWebConfiguration$SpringMvcConfiguration"
                        }
                    ],
                    "parentId": 246
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 246,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "dispatcherServletMappingDescriptionProvider"
                        }
                    ],
                    "parentId": 244
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 249,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.mappings.MappingsEndpointAutoConfiguration$ServletWebConfiguration"
                        }
                    ],
                    "parentId": 248
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 248,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "servletMappingDescriptionProvider"
                        }
                    ],
                    "parentId": 244
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 250,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "filterMappingDescriptionProvider"
                        }
                    ],
                    "parentId": 244
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 244,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mappingsEndpoint"
                        }
                    ],
                    "parentId": 211
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0.047281S",
                "startTime": "2022-10-29T02:37:40.627115700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 211,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "healthEndpointWebMvcHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 251,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.health.HealthEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 253,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.info-org.springframework.boot.autoconfigure.info.ProjectInfoProperties"
                        }
                    ],
                    "parentId": 252
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 252,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 254,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.info.InfoContributorAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.674396700Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 255,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.info-org.springframework.boot.actuate.autoconfigure.info.InfoContributorProperties"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0.0156254S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 257,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.jmx-org.springframework.boot.autoconfigure.jmx.JmxProperties"
                        }
                    ],
                    "parentId": 256
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0.0156254S",
                "startTime": "2022-10-29T02:37:40.674396700Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 256,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 259,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "objectNamingStrategy"
                        }
                    ],
                    "parentId": 258
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 260,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mbeanServer"
                        },
                        {
                            "key": "beanType",
                            "value": "interface javax.management.MBeanServer"
                        }
                    ],
                    "parentId": 258
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 258,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mbeanExporter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 262,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoints.jmx-org.springframework.boot.actuate.autoconfigure.endpoint.jmx.JmxEndpointProperties"
                        }
                    ],
                    "parentId": 261
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 261,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.jmx.JmxEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 264,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jmxIncludeExcludePropertyEndpointFilter"
                        }
                    ],
                    "parentId": 263
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 263,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jmxAnnotationEndpointDiscoverer"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.690022100Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 265,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "endpointObjectNameFactory"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0.0312489S",
                "startTime": "2022-10-29T02:37:40.690022100Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 266,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "jmxMBeanExporter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 267,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "eagerlyInitializeJmxEndpointExporter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 269,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "controllerExposeExcludePropertyEndpointFilter"
                        }
                    ],
                    "parentId": 268
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 268,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "controllerEndpointDiscoverer"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 270,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "pathMappedEndpoints"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 271,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.logging.LogFileWebEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 272,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoint.logfile-org.springframework.boot.actuate.autoconfigure.logging.LogFileWebEndpointProperties"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 273,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.CompositeMeterRegistryAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 274,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.integration.IntegrationMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 275,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.startup.StartupTimeMetricsListenerAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 276,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "startupTimeMetrics"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 277,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.task.TaskSchedulingAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 278,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "scheduledBeanLazyInitializationExcludeFilter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 280,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.task.scheduling-org.springframework.boot.autoconfigure.task.TaskSchedulingProperties"
                        }
                    ],
                    "parentId": 279
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 279,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "taskSchedulerBuilder"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 281,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.task.TaskExecutorMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 282,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.http.JacksonHttpMessageConvertersConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 283,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 284,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.web.client.RestTemplateMetricsConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 285,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "restTemplateExchangeTagsProvider"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 286,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "metricsRestTemplateCustomizer"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 287,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.metrics.web.tomcat.TomcatMetricsAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 288,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "tomcatMetricsBinder"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 289,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.trace.http.HttpTraceEndpointAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.721271Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 290,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0.015626S",
                "startTime": "2022-10-29T02:37:40.721271Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 291,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "springApplicationAdminRegistrar"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 292,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.aop.AopAutoConfiguration$ClassProxyingConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 293,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.aop.AopAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 294,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 295,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 297,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.lifecycle-org.springframework.boot.autoconfigure.context.LifecycleProperties"
                        }
                    ],
                    "parentId": 296
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 296,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "lifecycleProcessor"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 298,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.sql.init.SqlInitializationAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 299,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "spring.sql.init-org.springframework.boot.autoconfigure.sql.init.SqlInitializationProperties"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 300,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 301,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "multipartResolver"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 302,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.endpoint.web.servlet.WebMvcEndpointManagementContextConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.736897Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 304,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.endpoints.web.cors-org.springframework.boot.actuate.autoconfigure.endpoint.web.CorsEndpointProperties"
                        }
                    ],
                    "parentId": 303
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.748405300Z",
                "duration": "PT0.0115083S",
                "startTime": "2022-10-29T02:37:40.736897Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 303,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "webEndpointServletHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.748405300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.748405300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 305,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "controllerEndpointHandlerMapping"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.748405300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.748405300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 306,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.server.ManagementContextAutoConfiguration$SameManagementContextConfiguration$EnableSameManagementContextConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.748405300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.748405300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 307,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.server.ManagementContextAutoConfiguration$SameManagementContextConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.748405300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.748405300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 308,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.server.ManagementContextAutoConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.748405300Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.748405300Z",
                "startupStep": {
                    "name": "spring.beans.instantiate",
                    "id": 309,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "management.server-org.springframework.boot.actuate.autoconfigure.web.server.ManagementServerProperties"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.752912Z",
                "duration": "PT0.0045067S",
                "startTime": "2022-10-29T02:37:40.748405300Z",
                "startupStep": {
                    "name": "spring.beans.smart-initialize",
                    "id": 310,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.context.event.internalEventListenerProcessor"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.752912Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.752912Z",
                "startupStep": {
                    "name": "spring.beans.smart-initialize",
                    "id": 311,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "mbeanExporter"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.752912Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.752912Z",
                "startupStep": {
                    "name": "spring.beans.smart-initialize",
                    "id": 312,
                    "tags": [
                        {
                            "key": "beanName",
                            "value": "org.springframework.boot.actuate.autoconfigure.web.server.ManagementContextAutoConfiguration$SameManagementContextConfiguration"
                        }
                    ],
                    "parentId": 4
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.784170200Z",
                "duration": "PT1.5065837S",
                "startTime": "2022-10-29T02:37:39.277586500Z",
                "startupStep": {
                    "name": "spring.context.refresh",
                    "id": 4,
                    "tags": [],
                    "parentId": null
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.784170200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.784170200Z",
                "startupStep": {
                    "name": "spring.boot.application.started",
                    "id": 313,
                    "tags": [],
                    "parentId": null
                }
            },
            {
                "endTime": "2022-10-29T02:37:40.784170200Z",
                "duration": "PT0S",
                "startTime": "2022-10-29T02:37:40.784170200Z",
                "startupStep": {
                    "name": "spring.boot.application.ready",
                    "id": 314,
                    "tags": [],
                    "parentId": null
                }
            }
        ]
    }
}
```

# 外部配置化

Spring Boot允许外部化您的配置，以便您可以在不同的环境中使用相同的应用程序代码。您可以使用各种外部配置源，包括 Java 属性文件、YAML 文件、环境变量和命令行参数。

属性值可以使用`@Value`注释直接注入到 bean 中，或者通过`@ConfigurationProperties`绑定到结构化对象。

属性覆盖的顺序，优先级从从低到高：

1. 默认属性： `SpringApplication.setDefaultProperties `设置的属性
2. `@PropertySource`标注的`@Configuration`类。**请注意，在刷新应用程序上下文之前，此类属性源不会添加到环境中。配置某些属性可能为时已晚，如logging.\* 和 spring.main.\*，这些属性在刷新开始之前读取。**
3. 配置文件，例如 application.properties
4. RandomValuePropertySource，只针对random.*属性
5. 操作系统变量
6. Java 系统属性 System.getProperties(). -D 设置的参数
7. jndi属性 ， java:comp/env
8. ServletContext 初始化参数
9. ServletConfig 初始化参数
10. 来自SPRING_APPLICATION_JSON的属性（嵌入在环境变量或系统属性中的内联 JSON）。
11. 命令行参数
12. 测试属性，在@SpringBootTest和@Test上可用，用于测试应用程序的特定切片。
13. 在测试中@TestPropertySource注释。
14. 当开发工具处于活动状态时，$HOME/.config/spring-boot目录中的开发工具全局设置属性。



配置文件按以下顺序考虑：

1. 打包在 jar 中的应用程序属性（application.properties和 YAML 变体）。
2. 打包在 jar 中的特定于配置文件的应用程序属性（application-{profile}.properties和 YAML 变体）。
3. 打包的 jar 外部的应用程序属性（application.properties和 YAML 变体）。
4. 打包的 jar 外部的特定于配置文件的应用程序属性（application->{profile}.properties和 YAML 变体）。

建议为整个应用程序坚持使用一种格式。如果在同一位置具有同时具有 .properties 和 .yml 格式的配置文件，则 .properties 优先。

## 访问命令行属性

默认情况下，Spring 应用程序会将任何命令行选项参数（即以 --开头的参数，如 `--server.port=9000`）转换为属性，并将它们添加到 Spring 环境中。如前所述，命令行属性始终优先于基于文件的属性源。

如果不希望将命令行属性添加到环境中，可以使用SpringApplication.setAddCommandLineProperties(false）禁用它们。

## application json属性

环境变量和系统属性通常具有限制，这意味着某些属性名称不能使用。为了帮助解决这个问题，spring启动允许您将属性块编码为单个 JSON 结构。

当应用程序启动时，任何 `spring.application.json` 或`SPRING_APPLICATION_JSON`属性都将被解析并添加到环境中。

例如，可以在 UN*X shell 的命令行上将 `SPRING_APPLICATION_JSON `属性作为环境变量提供：

```shell
$ SPRING_APPLICATION_JSON='{"my":{"name":"test"}}' java -jar myapp.jar
```

在前面的示例中，您最终在sping环境中得到 my.name=test。

也可以将相同的 JSON 作为系统属性提供：

```shell
$ java -Dspring.application.json='{"my":{"name":"test"}}' -jar myapp.jar
```

或者，您可以使用命令行参数提供 JSON：

```shell
$ java -jar myapp.jar --spring.application.json='{"my":{"name":"test"}}'
```

尽管 JSON 中的空值将添加到生成的属性源中，但属性源属性解决程序将空属性视为缺失值。这意味着 JSON 无法用空值覆盖来自低阶属性源的属性。

## 外部属性

当您的应用程序启动时，spring boot将自动从以下位置查找并加载`application.properties`和`application.yaml` 文件(优先级从低到高)：

1. classpath

1. 1. classpath root
   2. classpath /config package

1. 当前目录

1. 1. 当前目录
   2. 当前目录下的config子目录
   3. 当前目录下的config子目录的子目录

如果不喜欢application作为配置文件名，则可以通过指定 `spring.config.name `环境属性切换到另一个文件名。例如，要查找`myproject.properties`和`myproject.yaml`文件，您可以按如下方式运行应用程序：

```shell
$ java -jar myproject.jar --spring.config.name=myproject
```

还可以使用`spring.config.location` 环境属性来引用显式位置。此属性接受一个或多个逗号分隔列表。下面的示例演示如何指定两个不同的文件：

```shell
$ java -jar myproject.jar --spring.config.location=\
    optional:classpath:/default.properties,\
    optional:classpath:/override.properties
```

如果`spring.config.location`包含目录（而不是文件），则它们应以` / `结尾。在运行时，它们将附加在从 `spring.config.name` 生成的名称之前。



在大多数情况下，每个`spring.config.location`添加的位置项将引用单个文件或目录。位置按其定义的顺序进行处理，稍后的位置可以覆盖先前位置的值。

如果您有一个复杂的位置设置，并且您使用了profile-specific的配置文件，那么您可能需要提供进一步的提示，以便Spring Boot知道应该如何对它们进行分组.位置组是在同一级别上考虑的所有位置的集合。例如，您可能希望对所有类路径位置进行分组，然后对所有外部位置进行分组。分组中的项目应以`;`分隔。

使用`spring.config.location`的位置替换默认位置。例如，如果`spring.config.location`= `optional:classpath:/custom-config/,optional:file:./custom-config/`，考虑的完整位置集为：

1. optional:classpath:custom-config/
2. optional:file:./custom-config/

如果您希望添加其他位置，而不是替换它们，可以使用`spring.config.additional-location`。从其他位置加载的特性可以覆盖默认位置中的特性。例如，如果`spring.config.additional-location=optional:classpath:/custom-config/,optional:file:./custom-config/`，考虑的完整位置集为：

1. `optional:classpath:/;optional:classpath:/config/`
2. `optional:file:./;optional:file:./config/;optional:file:./config/*/`
3. `optional:classpath:custom-config/`
4. `optional:file:./custom-config/`

此搜索顺序允许您在一个配置文件中指定默认值，然后在另一个配置中选择性地覆盖这些值。您可以在application.properties为应用程序提供默认值。然后，可以在运行时使用位于其中一个自定义位置的不同文件覆盖这些默认值。

如果使用环境变量而不是系统属性，大多数操作系统都不允许使用句点分隔的键名，但可以使用下划线（例如，SPRING_CONFIG_NAME而不是SPRING.CONFIG.NAME）。



默认情况下，当指定的配置数据位置不存在时，Spring Boot将抛出`ConfigDataLocationNotFoundException`，应用程序将不会启动。

如果您想指定一个位置，但不介意它不总是存在，可以使用`optional:`前缀。您可以将此前缀与`spring.config.location和spring.config.additional-location`属性以及`spring.config.import`声明一起使用。

如果要忽略所有`ConfigDataLocationNotFoundExceptions`并始终继续启动应用程序，可以使用`SpringApplication.setDefaultProperties(…) `或系统/环境变量设置 `spring.config.on-not-found=ignore` 。

#### Wildcard Locations

如果配置文件位置最后一个路径包含`*`字符，则它被视为通配符位置。加载配置时会展开通配符，以便也检查直接子目录。当存在多个配置属性源时，通配符位置在Kubernetes等环境中特别有用。

例如，如果您有一些Redis配置和一些MySQL配置，您可能希望将这两个配置分开，同时要求这两个都存在于`application.properties`文件。这可能导致两个单独的`application.properties`安装在不同位置，如`/config/redis/application.properties`和`/config/mysql/application.properties`。在这种情况下，通配符位置为`config/*/ `将导致两个文件都被处理。

默认情况下，Spring Boot在默认搜索位置中包含`config/*/`。这意味着将搜索jar之外的`/config`目录的所有子目录。



你可以在`spring.config.location` 和 `spring.config.additional-location` 属性上配置通配符位置。



通配符位置只能包含一个`*`，对于目录搜索位置，必须以`*/`结尾；对于文件搜索位置，则必须以` */<filename>` 结尾。带有通配符的位置根据文件名的绝对路径按字母顺序排序。

#### Profile Specific Files

除了application 属性文件，Spring Boot还将尝试使用命名约定`application-{profile}`加载特定profile的文件。例如，如果您的application 激活了一个名为prod的概要文件并使用了YAML文件，那么`application.yml` 和 `application-prod.yml `都会被激活



如果指定了多个profile，则采用最后获胜策略。 例如 `spring.profiles.active =  prod,live`，`application-prod.properties`文件中的属性会被 `application-live.properties` 覆盖。

继续上面的prod、live示例，我们可能会有以下文件：

```graphql
/cfg
  application-live.properties
/ext
  application-live.properties
  application-prod.properties
```

当`spring.config.location=classpath:/cfg/,classpath:/ext/`  我们在所有/ext文件之前处理所有/cfg文件：

1. /cfg/application-live.properties
2. /ext/application-prod.properties
3. /ext/application-live.properties

当`spring.config.location=classpath:/cfg/;classpath:/ext/ `:

1. /ext/application-prod.properties
2. /cfg/application-live.properties
3. /ext/application-live.properties


profiles 默认值是 default 

#### Importing Additional Data

`spring.config.import `可以指定其他的配置文件并导入。被导入的文件如果存在，则立即执行导入操作，并被视为在声明导入的文档的正下方插入的附加文档。

例如，application.properties文件包含下面的内容：

```graphql
spring.application.name=myapp
spring.config.import=optional:file:./dev.properties
```

这将触发当前目录中dev.properties文件的导入（如果存在这样的文件）。导入的dev.properties中的值将优先于触发导入的文件。在上面的例子中，dev.properties可以重新定义spring.application.name设置为不同的值。

导入将只导入一次，无论它声明了多少次。在properties/yaml文件中的单个文档中定义导入的顺序并不重要。例如，下面的两个示例产生了相同的结果：

```graphql
spring.config.import=my.properties
my.property=value
my.property=value
spring.config.import=my.properties
```

如果您想支持自己的位置，请参阅`org.springframework.boot.context.config`包中的`ConfigDataLocationResolver`和`ConfigDataLoader`类。

#### 导入无扩展名文件

某些云平台无法向 volume mounted files 添加文件扩展名。要导入这些无扩展名文件，您需要给Spring Boot一个提示，以便它知道如何加载它们。您可以通过将扩展提示放在方括号中来做到这一点。

例如，假设您有一个/etc/config/myconfig文件，希望将其导入为yaml。您可以从应用程序导入它。属性使用以下属性：

```graphql
spring.config.import=file:/etc/config/myconfig[.yaml]
```

#### 使用配置树

在云平台（如Kubernetes）上运行应用程序时，通常需要读取平台提供的配置值。为此目的使用环境变量并不少见，但这可能有缺点，特别是如果值应该保密的话。

作为环境变量的替代方案，许多云平台现在允许您将配置映射到装载的数据卷中。例如，Kubernetes可以同时装载`ConfigMaps`和`Secrets`。

可以使用两种常见的卷装载模式：

- 单个文件包含一组完整的属性（通常写为YAML）。
- 多个文件被写入目录树，文件名成为“键”，内容成为“值”。

对于第一种情况，可以使用 `pring.config.import `直接导入`YAML`或`Properties`文件。对于第二种情况，您需要使用` configtree:`前缀，以便Spring Boot知道它需要将所有文件作为属性公开。

举个例子，让我们假设Kubernetes已经挂载了以下卷：

```graphql
etc/
  config/
    myapp/
      username
      password
```

username文件的内容是配置值，password 文件的内容是secret

要导入这些属性，可以将以下内容添加到`application.properties` 或`application.yaml`文件：

```graphql
spring.config.import=optional:configtree:/etc/config/
```

然后你可以在Environment 中访问  `myapp.username `和 `myapp.password` 属性。

如果要从同一父文件夹导入多个配置树，可以使用通配符快捷方式。任何以`/*/`结尾的`configtree:`位置都会将所有直接子级作为配置树导入。

```graphql
etc/
  config/
    dbconfig/
      db/
        username
        password
    mqconfig/
      mq/
        username
        password
spring.config.import=optional:configtree:/etc/config/*/
```

这会添加 `db.username, db.password, mq.username `和 `mq.password` 属性。

配置树被docker secrets 使用。当 Docker swarm 被授权可以访问 secret 时，这个 secret  会被挂载到容器上。例如， 一个名为 db.password 的 secret  被挂载到 /run/secrets/ ，你可以用下面的配置：

```graphql
spring.config.import=optional:configtree:/run/secrets/
```

#### 属性占位符

语法：`${name:default}`

```graphql
app.name=MyApp
app.description=${app.name} is a Spring Boot application written by ${username:Unknown}
```

假设`username `属性未在别处设置，app.description 的实际值是：`MyApp is a Spring Boot application written by Unknown.`

您应该始终使用占位符中的规范形式(kebab-case的全小写)引用属性名称，这和`@ConfigurationProperties`的绑定逻辑相同

例如，`${demo.item-price} `会从`application.properties` 查找 `demo.item-price `和 `demo.itemPrice` 属性的值，以及从环境变量中查找`DEMO_ITEMPRICE` 。 如果你使用`${demo.itemPrice}` ，则 不会查找 `demo.item-price `和  `DEMO_ITEMPRICE` 

#### 使用多文档文件

Spring Boot允许您将单个物理文件拆分为多个逻辑文档，每个逻辑文档都独立添加。文档按从上到下的顺序处理。稍后的文档可以覆盖先前文档中定义的属性。

application.yml使用标准的YAML多文档语法。三个连续的连字符表示一个文档的结尾和下一个文档开始。

```yaml
spring:
  application:
    name: "MyApp"
---
spring:
  application:
    name: "MyCloudApp"
  config:
    activate:
      on-cloud-platform: "kubernetes"
```



application.properties 使用  `#---  `或 `!---` 注释 标记文档分割：

```properties
spring.application.name=MyApp
#---
spring.application.name=MyCloudApp
spring.config.activate.on-cloud-platform=kubernetes
```

属性文件分隔符不能有任何前导空格，并且必须正好有三个连字符。分隔符前后的行不能是相同的注释前缀。

多文档属性文件通常与激活属性（如`spring.config.activate.on-profile`）结合使用。

无法使用`@PropertySource`或`@TestProperty`源批注加载多文档属性文件。

#### Activation Properties

有时，仅在满足某些条件时激活给定的属性集是有用的。例如，您可能具有仅在特定配置文件处于活动状态时才使用相关的属性。

您可以使用`spring.config.activate.*`有条件地激活属性文档。

- `on-profile`:  表达式
- `on-cloud-platform`：要使文档处于活动状态，必须检测到的CloudPlatform。

例如，以下内容指定第二个文档仅在Kubernetes上运行时处于活动状态，并且仅在“prod”或“staging”配置文件处于活动状态时才处于活动状态：

```properties
myprop=always-set
#---
spring.config.activate.on-cloud-platform=kubernetes
spring.config.activate.on-profile=prod | staging
myotherprop=sometimes-set
```

## 加密属性

Spring Boot 不提供任何用于加密属性值的内置支持，但是，它确实提供了修改 Spring 环境中包含的值的挂钩点。环境后处理器接口允许您在应用程序启动之前操作环境。有关详细信息，请参阅[文档](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.application.customize-the-environment-or-application-context)。



如果您需要一种安全的方式来存储凭据和密码，参考 [Spring Cloud Vault](https://cloud.spring.io/spring-cloud-vault/)

## yaml

YAML 是 JSON 的超集，是指定分层配置数据的便捷格式。`SpringApplication `类会自动支持 YAML 作为属性的替代方法，只要您的类路径上有YAML库。



YAML文档需要从其分层格式转换为可以与Spring环境一起使用的平面结构。例如，请考虑以下 YAML 文档：

```yaml
environments:
  dev:
    url: "https://dev.example.com"
    name: "Developer Setup"
  prod:
    url: "https://another.example.com"
    name: "My Cool App"
```

为了从环境中访问这些属性，它们将被拼合，如下所示：

```properties
environments.dev.url=https://dev.example.com
environments.dev.name=Developer Setup
environments.prod.url=https://another.example.com
environments.prod.name=My Cool App
```

同样，YAML 列表也需要扁平化。它们表示为具有 [index] 的属性键。例如，请考虑以下 YAML：

```yaml
my:
 servers:
  - "dev.example.com"
  - "another.example.com"
```

前面的示例将转换为以下属性：

```properties
my.servers[0]=dev.example.com
my.servers[1]=another.example.com
```



使用 [索引] 表示法的属性可以使用spring boot的 Binder 类绑定到 Java List或Set对象。有关更多详细信息，请参阅下面的“类型安全的配置属性”部分。



**无法使用@PropertySource或@TestPropertySource注释加载 YAML 文件。因此，如果需要以这种方式加载值，则需要使用属性文件。**



spring框架提供了两个方便的类，可用于加载YAML文档。`YamlPropertiesFactoryBean `将 YAML 加载为“属性”，`YamlMapFactoryBean `将 YAML 作为Map 加载。如果要将 YAML 作为PropertySource加载，也可以使用 `YamlPropertySourceLoader`。

## 随机数

RandomValuePropertySource 帮助我们在配置中输入随机值，支持integers, longs, uuids, 或 strings。例如：

```properties
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number-less-than-ten=${random.int(10)}
my.number-in-range=${random.int[1024,65536]}
```

## 系统环境属性

spring启动支持为环境属性设置前缀。如果系统环境由具有不同配置要求的多个 Spring Boot 应用程序共享，这将非常有用。系统环境属性的前缀可以直接在`SpringApplication`上设置。

例如，如果将前缀设置为`input`，则在系统环境中，`remote.timeout`等属性也将解析为`input.remote.timeout`。

## 属性绑定

使用`@Value("${property}")`注解来注入配置属性有时会很麻烦，特别是如果您正在处理多个属性或数据本质上是分层的。SpringBoot提供了一种使用属性的替代方法，该方法允许强类型bean管理和验证应用程序的配置。

### JavaBean 属性绑定

可以将属性直接绑定到javabean上，例如：

```java
@ConfigurationProperties("my.service")
public class MyProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    public boolean isEnabled() {
        return this.enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public InetAddress getRemoteAddress() {
        return this.remoteAddress;
    }

    public void setRemoteAddress(InetAddress remoteAddress) {
        this.remoteAddress = remoteAddress;
    }

    public Security getSecurity() {
        return this.security;
    }

    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        public String getUsername() {
            return this.username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String getPassword() {
            return this.password;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        public List<String> getRoles() {
            return this.roles;
        }

        public void setRoles(List<String> roles) {
            this.roles = roles;
        }

    }

}
```

前面的POJO定义了以下属性：

- my.service.enabled : 默认值是false
- my.service.remote-address: 可以从spring转换的强制类型
- my.service.security.username
- my.service.security.password
- my.service.security.roles

这种安排依赖于默认的空构造函数，getter和setter通常是强制的，因为绑定是通过标准JavaBeans属性描述符进行的，就像SpringMVC中一样。在以下情况下，可省略setter：

- maps，只要被初始化，就需要一个getter，但不一定需要setter，因为它们可以被绑定器更改。
- 可以通过索引（通常使用YAML）的集合和数组或使用单个逗号分隔的值（属性）。在后一种情况下，setter是强制性的。我们建议始终为此类类型添加setter。如果您初始化一个集合，请确保它不是不可变的（如前面的示例所示）。
- 如果嵌套的POJO属性已初始化（如前面示例中的Security字段），则不需要setter。如果您希望绑定器使用其默认构造函数动态创建实例，则需要一个setter。

有些人使用Project Lombok自动添加getter和setter。确保Lombok不会为此类类型生成任何特定的构造函数，因为容器会自动使用它来实例化对象。

最后，只考虑标准JavaBean属性，不支持绑定静态属性。

### 构造函数绑定

上一节中的示例可以以不可变的方式重写，如以下示例所示：

```java
import java.net.InetAddress;
import java.util.List;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.ConstructorBinding;
import org.springframework.boot.context.properties.bind.DefaultValue;

@ConstructorBinding
@ConfigurationProperties("my.service")
public class MyProperties {

    private final boolean enabled;

    private final InetAddress remoteAddress;

    private final Security security;

    public MyProperties(boolean enabled, InetAddress remoteAddress, Security security) {
        this.enabled = enabled;
        this.remoteAddress = remoteAddress;
        this.security = security;
    }

    public boolean isEnabled() {
        return this.enabled;
    }

    public InetAddress getRemoteAddress() {
        return this.remoteAddress;
    }

    public Security getSecurity() {
        return this.security;
    }

    public static class Security {

        private final String username;

        private final String password;

        private final List<String> roles;

        public Security(String username, String password, @DefaultValue("USER") List<String> roles) {
            this.username = username;
            this.password = password;
            this.roles = roles;
        }

        public String getUsername() {
            return this.username;
        }

        public String getPassword() {
            return this.password;
        }

        public List<String> getRoles() {
            return this.roles;
        }

    }

}
```

在此设置中，`@ConstructorBinding`注释用于指示应使用构造函数绑定。如果您使用的是Java16或更高版本，那么构造函数绑定可以用于Record。一般情况下，除非您的记录有多个构造函数，否则不需要使用`@ConstructorBinding`。

`@ConstructorBinding`类的嵌套成员（例如上面示例中的Security）也将通过其构造函数绑定。

默认值可以在构造函数参数上使用`@DefaultValue`指定，或者在使用Java 16或更高版本时，使用Record组件指定。conversion service 将String值强制转换为缺少属性的目标类型。

参考前面的示例，如果没有属性绑定到Security，则MyProperties实例将包含一个空的security。要使它包含一个非空的Security实例，即使没有属性绑定到它，请使用空的`@DefaultValue`注释：

```java
public MyProperties(boolean enabled, InetAddress remoteAddress, @DefaultValue Security security) {
    this.enabled = enabled;
    this.remoteAddress = remoteAddress;
    this.security = security;
}
```

- 若要使用构造函数绑定，必须使用`@EnableConfigurationProperties`或配置属性扫描类。
- 不能将构造函数绑定用于由常规Spring机制创建的Bean（例如`@Component `Bean、使用@Bean方法创建的beans或使用`@Import`加载的beans）？？？因为对象不可变。
- 如果类有多个构造函数，也可以直接在应该绑定的构造函数上使用`@ConstructorBinding`。

### 启用@ConfigurationProperties注释类型

Spring Boot提供了绑定`@ConfigurationProperties`类型并将其注册为bean的基础结构。您可以逐个类启用配置属性，也可以启用与组件扫描类似的扫描方式。

有时，用`@ConfigurationProperties`注释的类可能不适合扫描，例如，如果您正在开发自己的自动配置，或者希望有条件地启用它们。在这些情况下，请使用`@EnableConfigurationProperties`注释指定要处理的类型列表。这可以在任何`@Configuration`类上完成，如以下示例所示：

```java
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(SomeProperties.class)
public class MyConfiguration {

}
```

要使用配置属性扫描，请将`@ConfigurationPropertiesScan`注释添加到应用程序中。通常，它被添加到带有`@SpringBootApplication`注释的主应用程序类中，但它可以添加到任何`@Configuration`类中。默认情况下，将从声明注释的类的包进行扫描。如果要定义要扫描的特定包，可以按以下示例所示进行操作：

```java
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan({ "com.example.app", "com.example.another" })
public class MyApplication {

}
```

当使用配置属性扫描或通过`@EnableConfigurationProperties`注册`@ConfigurationProperties` bean时，bean有一个常规名称：`＜prefix＞-＜fqn＞`，其中`＜prefix>`是`@configuration Properties`注释中指定的环境密钥前缀，`＜fqn>`是bean的完全限定名称。如果注释没有提供任何前缀，则只使用bean的完全限定名称。

上面的例子中bean的名称是：`com.example.app-com.example.app.SomeProperties`



我们建议`@ConfigurationProperties`只处理环境属性，特别是不要从上下文中注入其他bean。对于特殊情况，可以使用`setter`注入或框架提供的任何`*Aware`接口（例如，如果需要访问环境，则使用`EnvironmentAware`）。如果您仍然希望使用构造函数注入其他bean，则必须使用`@Component`注释配置属性bean，并使用基于JavaBean的属性绑定。



### 使用 @ConfigurationProperties-annotated 类型

这种类型的配置在`SpringApplication`外部YAML配置中尤其适用，如下例所示：

```yaml
my:
  service:
    remote-address: 192.168.1.1
    security:
      username: "admin"
      roles:
      - "USER"
      - "ADMIN"
```

要使用@ConfigurationProperties bean，您可以以与任何其他bean相同的方式注入它们，如下例所示：

```java
@Service
public class MyService {

    private final SomeProperties properties;

    public MyService(SomeProperties properties) {
        this.properties = properties;
    }

    public void openConnection() {
        Server server = new Server(this.properties.getRemoteAddress());
        server.start();
        // ...
    }

    // ...

}
```

### 第三方配置

除了使用`@ConfigurationProperties`注释类之外，您还可以在公共@Bean方法上使用它。当您希望将属性绑定到您无法控制的第三方组件时，这样做尤其有用。

要从`Environment`属性配置bean，请将`@ConfigurationProperties`添加到注册bean的方法上，如下例所示：

```java
@Configuration(proxyBeanMethods = false)
public class ThirdPartyConfiguration {

    @Bean
    @ConfigurationProperties(prefix = "another")
    public AnotherComponent anotherComponent() {
        return new AnotherComponent();
    }

}
```

### 灵活的绑定方式

Spring Boot使用一些宽松的规则将`Environment`属性绑定到`@ConfigurationProperties` bean，因此`Environmental`属性名称和bean属性名称之间不需要完全匹配。常用的示例包括破折号分隔的环境属性（例如，`context-path`绑定到`contextPath`）和大写的环境属性，例如，`PORT`绑定到 `port`。

```java
@ConfigurationProperties(prefix = "my.main-project.person")
public class MyPersonProperties {

    private String firstName;

    public String getFirstName() {
        return this.firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

}
```

对于前面的代码，以下属性名称都可以使用：

| Property                          | Note                                                        |
| --------------------------------- | ----------------------------------------------------------- |
| my.main-project.person.first-name | Kebab案例，建议在.properties和.yml文件中使用。              |
| my.main-project.person.firstName  | 标准驼峰大小写语法。                                        |
| my.main-project.person.first_name | 下划线表示法，这是一种用于.properties和.yml文件的替代格式。 |
| MY_MAINPROJECT_PERSON_FIRSTNAME   | 大写格式，建议在使用系统环境变量时使用。                    |

| Property Source       | Simple                                               | List                                       |
| --------------------- | ---------------------------------------------------- | ------------------------------------------ |
| Properties Files      | Camel大小写、kebab大小写或下划线符号                 | 使用[]或逗号分隔值的标准列表语法           |
| YAML Files            | Camel大小写、kebab大小写或下划线符号                 | 标准YAML列表语法或逗号分隔值               |
| Environment Variables | 以下划线为分隔符的大写格式（请参见从环境变量绑定）。 | 由下划线包围的数值（请参见从环境变量绑定） |
| System properties     | Camel大小写、kebab大小写或下划线符号                 | 使用[]或逗号分隔值的标准列表语法           |

我们建议在可能的情况下，属性以小写的kebab格式存储，例如`my.person.first-name=Rod`

#### Binding Maps

当绑定到Map属性时，可能需要使用特殊的括号符号，以便保留原始键值。如果键未被`[]`包围，则将删除所有非字母数字、`-`或`.`的字符。

例如，考虑将以下属性绑定到Map<String,String>：

```properties
my.map.[/key1]=value1
my.map.[/key2]=value2
my.map./key3=value3
```

对于YAML文件，括号需要用引号括起来，以便正确解析键。

上面的属性将绑定到一个Map，其中`/key1`、`/key2`和`key3`作为Map中的键。斜线已从`key3`中删除，因为它没有被方括号包围。

当绑定到标量值时，其中带有`.`的键不需要被`[]`包围。标量值包括枚举和java.lang包中所有的类型，Object除外。将`a.b=c`绑定到`Map＜String，String＞`将保留键中的`.`，并返回带有条目`｛"a.b"="c"｝`的Map。对于任何其他类型，如果您的键包含`.`，则需要使用括号表示法。例如，将`a.b=c`绑定到`Map＜String，Object＞`将返回带有条目`{"a"={"b"="c"}}`的Map，而`[a.b]=c`将返回带有条目`{"a.b"="c"}`的Map。

#### 从环境变量绑定

大多数操作系统对可用于环境变量的名称实施严格的规则。例如，Linux shell变量只能包含字母（a到z或A到Z）、数字（0到9）或下划线（_）。按照惯例，Unix Shell变量的名称也将为大写。

Spring Boot的宽松绑定规则尽可能与这些命名限制兼容。要将规范形式的属性名转换为环境变量名，可以遵循以下规则：

- 使用` .` 替换` _`
- 删除` -`
- 转换成大写

例如，配置`spring.main.log-startup-info` 是一个名为`SPRING_MAIN_LOGSTARTUPINFO`的环境变量。

绑定到对象列表时也可以使用环境变量。要绑定到列表，序号应在变量名中用下划线括起来。例如 `my.service[0].other` 对应 `MY_SERVICE_0_OTHER`

### 合并复杂类型

当在多个位置配置列表时，通过替换整个列表来覆盖列表。

例如，假设一个MyPojo对象的name 和 description属性默认为空。以下示例公开了MyProperties中的MyPojo对象列表：

```properties
@ConfigurationProperties("my")
public class MyProperties {

    private final List<MyPojo> list = new ArrayList<>();

    public List<MyPojo> getList() {
        return this.list;
    }

}
```

假如有如下配置：

```properties
my.list[0].name=my name
my.list[0].description=my description
#---
spring.config.activate.on-profile=dev
my.list[0].name=my another name
```

如果dev profile未处于活动状态，`MyProperties.list `包含一个`MyPojo`条目。但是，如果启用了dev profile，列表仍然只包含一个条目（name 为my another name，description 为空）。此配置不会向列表中添加第二个MyPojo实例，也不会合并项目。

当在多个 profiles 中指定列表时，将使用优先级最高的profiles（并且仅使用该配置文件）。考虑以下示例：

```properties
my.list[0].name=my name
my.list[0].description=my description
my.list[1].name=another name
my.list[1].description=another description
#---
spring.config.activate.on-profile=dev
my.list[0].name=my another name
```

在前面的示例中，如果dev profile处于活动状态，则为`MyProperties.list`包含一个MyPojo条目。对于YAML，逗号分隔列表和YAML列表都可以用于完全覆盖列表的内容。

对于Map属性，你可以从多个配置源绑定，但是，在多个配置源中有相同的属性，就会发生覆盖。例如：

```java
@ConfigurationProperties("my")
    public class MyProperties {

        private final Map<String, MyPojo> map = new LinkedHashMap<>();

        public Map<String, MyPojo> getMap() {
            return this.map;
        }

    }
my.map.key1.name=my name 1
my.map.key1.description=my description 1
#---
spring.config.activate.on-profile=dev
my.map.key1.name=dev name 1
my.map.key2.name=dev name 2
my.map.key2.description=dev description 2
```

dev profile 没有激活，MyProperties.map 只包括key1实体。 如果激活 dev, 则包括 key1和key2，其中key1的内容是  `name=dev name 1`

### 属性转换

当Spring Boot绑定到`@ConfigurationProperties` bean时，它试图强制转换数据类型。如果需要自定义类型转换，可以提供`ConversionService `bean（名为conversionService的bean）或自定义属性编辑器（通过`CustomEditorConfigurer `bean），或自定义Converter（带有注释为`@ConfigurationPropertiesBinding`的bean定义）。

由于这个bean是在应用程序生命周期的早期请求的，所以请确保限制您的ConversionService正在使用的依赖项。

#### 转换时间类型

Spring Boot支持持续时间。如果公开java.time.Duration属性，则应用程序属性中的以下格式可用：

- long 类型，使用毫秒作为默认单位
- java.time.Duration使用的标准ISO-8601格式
- 一种更可读的格式，其中值和单位是耦合的（10s表示10秒）

```java
@ConfigurationProperties("my")
public class MyProperties {

    @DurationUnit(ChronoUnit.SECONDS)
    private Duration sessionTimeout = Duration.ofSeconds(30);

    private Duration readTimeout = Duration.ofMillis(1000);

}
```

要指定30秒的会话超时，30、PT30S和30s都是等效的。500ms的读取超时可以以下列任何形式指定：500、PT0.5S和500ms。

默认单位为毫秒，可以使用@DurationUnit覆盖，如上面的示例所示。

如果您喜欢使用构造函数绑定，可以公开相同的属性，如以下示例所示：

```java
@ConfigurationProperties("my")
@ConstructorBinding
public class MyProperties {

    // fields...

    public MyProperties(@DurationUnit(ChronoUnit.SECONDS) @DefaultValue("30s") Duration sessionTimeout,
            @DefaultValue("1000ms") Duration readTimeout) {
        this.sessionTimeout = sessionTimeout;
        this.readTimeout = readTimeout;
    }

    // getters...

}
```

#### Converting Data Sizes

Spring Framework具有以字节表示大小的DataSize值类型。如果公开DataSize属性，则应用程序属性中的以下格式可用：

- long表示（除非指定了@DataSizeUnit，否则使用字节作为默认单位）
- 一种更可读的格式，其中值和单位是耦合的（10MB表示10 MB）

```java
@ConfigurationProperties("my")
public class MyProperties {

    @DataSizeUnit(DataUnit.MEGABYTES)
    private DataSize bufferSize = DataSize.ofMegabytes(2);

    private DataSize sizeThreshold = DataSize.ofBytes(512);

    // getters/setters...

}
```


要指定10 MB的缓冲区大小，10和10MB相当。256字节的大小阈值可以指定为256或256B。

### @ConfigurationProperties 校验

每当用Spring的@Validated注释@ConfigurationProperties类时，Spring Boot都会尝试验证它们。您可以使用JSR-303 javax.validation直接在配置类上添加验证约束。为此，请确保符合JSR-303的实现在您的类路径上，然后向字段添加约束注释，如下例所示：

```java
@ConfigurationProperties("my.service")
@Validated
public class MyProperties {

    @NotNull
    private InetAddress remoteAddress;

    // getters/setters...

}
```

您还可以通过在@Bean方法使用@Validated注释来触发验证。

为了确保始终为嵌套属性触发验证，即使找不到属性，关联字段也必须用@Valid进行注释。以下示例基于前面的MyProperties示例：

```java
@ConfigurationProperties("my.service")
@Validated
public class MyProperties {

    @NotNull
    private InetAddress remoteAddress;

    @Valid
    private final Security security = new Security();

    // getters/setters...

    public static class Security {

        @NotEmpty
        private String username;

        // getters/setters...

    }

}
```

您还可以通过创建名为`configurationPropertiesValidator`的bean定义来添加自定义SpringValidator。@Bean方法应声明为静态的。配置属性验证器是在应用程序生命周期的早期创建的，并且将@Bean方法声明为static可以创建Bean，而无需实例化`@configuration`类。这样做可以避免早期实例化可能导致的任何问题。

### @ConfigurationProperties vs. @Value

@Value注释是一个核心容器特性，它不提供与类型安全配置属性相同的特性。下表总结了`@ConfigurationProperties`和`@Value`支持的功能：

| Feature                                                      | @ConfigurationProperties | @Value                                                       |
| ------------------------------------------------------------ | ------------------------ | ------------------------------------------------------------ |
| [Relaxed binding](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding) | Yes                      | Limited (see [note below](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties.vs-value-annotation.note)) |
| [Meta-data support](https://docs.spring.io/spring-boot/docs/current/reference/html/configuration-metadata.html#appendix.configuration-metadata) | Yes                      | No                                                           |
| SpEL evaluation                                              | No                       | Yes                                                          |

如果您确实想使用`@Value`，我们建议您使用规范格式（仅使用小写字母的kebab大小写）来引用属性名称。这将允许Spring Boot使用与灵活绑定`@ConfigurationPropertie`s时相同的逻辑。

例如，@Value("${demo.item-price}") 将获取`demo.item-price` 和`demo.itemPrice `以及系统环境中的`DEMO_ITEMPRICE`。如果您使用@Value("${demo.itemPrice}")代替，`demo.item-price`和`DEMO_ITEMPRICE`将不予考虑。



如果您为自己的组件定义了一组配置键，我们建议您将它们分组到带有@ConfigurationProperties注释的POJO中。这样做将为您提供结构化的、类型安全的对象，您可以将其注入到自己的bean中。

在解析这些文件并填充环境时，不会处理应用程序属性文件中的SpEL表达式。但是，可以在`@Value`中编写SpEL表达式。如果应用程序属性文件中的属性值是SpEL表达式，则在使用`@value`时将对其求值。

# Profiles

SpringProfiles提供了一种方法来隔离应用程序配置的各个部分，并使其仅在特定环境中可用。加载时，任何`@Component`、`@Configuration`或`@ConfigurationProperties`都可以用`@Profile`标记以限制，如以下示例所示：

```shell
@Configuration(proxyBeanMethods = false)
@Profile("production")
public class ProductionConfiguration {

    // ...

}
```

如果`@ConfigurationProperties` bean是通过`@EnableConfigurationProperties`而不是自动扫描注册的，则需要在具有`@EnableConfigurationProperties`注释的`@Configuration`类上指定`@Profile`注释。在扫描`@ConfigurationProperties`的情况下，可以在`@ConfigurationProperties`类本身上指定`@Profile`。

您可以使用 `spring.profiles.active ` Environment属性指定哪些配置文件处于活动状态。您可以使用本章前面描述的任何方法指定属性。例如，您可以将其包含在`application.properties`中，如以下示例所示：

```properties
spring.profiles.active=dev,hsqldb
```

您还可以使用以下开关在命令行中指定它：`--spring.profiles.active=dev,hsqldb`。

如果没有profile 文件处于活动状态，则会启用`default `profile 文件。默认配置文件的名称为`default`，可以使用 `spring.profiles.default  `Environment  属性 对其进行修改。如以下示例所示：

```properties
spring.profiles.default=none
```

`spring.profiles.active `和 `spring.profiles.default `只能在non-profile 配置文件中使用 。例如：

```properties
# this document is valid
spring.profiles.active=prod
#---
# this document is invalid
spring.config.activate.on-profile=prod
spring.profiles.active=metrics
```

## 添加 Active Profiles

`spring.profiles.active`属性遵循与其他属性相同的排序规则：最高的PropertySource获胜。这意味着您可以在`application.properties`中指定活动配置文件，然后使用命令行开关替换它们。

有时，将属性添加到active profile文件而不是替换它们是有用的。`spring.profiles.include`属性可用于在`spring.profiles.active` 的配置文件之上添加活动配置文件。`SpringApplication`入口点还具有用于设置其他profiles文件的JavaAPI。请参见`SpringApplication`中的`setAdditionalProfiles`（）方法。

例如，当运行具有以下属性的应用程序时，即使使用 --spring.profiles.active 开关运行，common 和 local profile也将被激活:

```properties
spring.profiles.include[0]=common
spring.profiles.include[1]=local
```

类似于spring.profiles.active，spring.profiles.include只能用于非profile文件。

## Profile Groups

有时，您在应用程序中定义和使用的profile文件过于细粒度，难以使用。例如，您可能有proddb和prodmq profile文件，用于独立启用数据库和消息功能。

为了帮助实现这一点，Spring Boot允许您定义profile文件组。proflie文件组允许您为相关配置文件组定义逻辑名称。

例如，我们可以创建一个由proddb和prodmq配置文件组成的production 组:

```properties
spring.profiles.group.production[0]=proddb
spring.profiles.group.production[1]=prodmq
```

现在可以使用`--spring.profiles.active=production`启动我们的应用程序，以一次性激活proddb和prodmq配置文件。

## 编程的方式设置 proflies

您可以在应用程序运行之前 通过调用SpringApplication.setAdditionalProfiles(…) 以编程方式设置活动profile文件。还可以使用Spring的ConfigurableEnvironment接口激活配置文件。

# 日志

Spring Boot使用Commons Logging进行所有内部日志记录，但保留底层日志实现的开放性。为Java Util Logging、Log4J2和Logback提供了默认配置。在每种情况下，记录器都预先配置为使用控制台输出，也可以使用可选的文件输出。

默认情况下，如果使用“Starters”，则Logback用于日志记录。还包括适当的Logback路由，以确保使用Java Util Logging、Commons Logging，Log4J或SLF4J的依赖库都能正常工作。

当您将应用程序部署到servlet容器或应用程序服务器时，使用Java Util logging API执行的日志不会路由到应用程序的日志中。这将防止容器或已部署到容器的其他应用程序执行的日志记录出现在应用程序的日志中。

## 日志格式

默认的日志打印如下：

```plain
2022-10-20 12:40:11.311  INFO 16138 --- [           main] o.s.b.d.f.s.MyApplication                : Starting MyApplication using Java 1.8.0_345 on myhost with PID 16138 (/opt/apps/myapp.jar started by myuser in /opt/apps/)
2022-10-20 12:40:11.330  INFO 16138 --- [           main] o.s.b.d.f.s.MyApplication                : No active profile set, falling back to 1 default profile: "default"
2022-10-20 12:40:13.056  INFO 16138 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-10-20 12:40:13.070  INFO 16138 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-10-20 12:40:13.070  INFO 16138 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.68]
2022-10-20 12:40:13.178  INFO 16138 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-10-20 12:40:13.178  INFO 16138 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1762 ms
2022-10-20 12:40:13.840  INFO 16138 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-10-20 12:40:13.850  INFO 16138 --- [           main] o.s.b.d.f.s.MyApplication 
```

- Date and Time: 毫秒级精度，易于分拣。
- Log Level: ERROR, WARN, INFO, DEBUG, or TRACE.
- Process ID.
- 一个`---`分隔符，用于区分实际日志消息的开头。
- Thread name:
- Logger name:
- The log message:

## 控制台输出

默认的日志配置在写入消息时将消息回送到控制台。默认情况下，记录错误级别、警告级别和`INFO`级别的消息。您还可以通过使用`--debug`标志启动应用程序来启用“调试”模式。

```plain
$ java -jar myapp.jar --debug
```

启用调试模式后，会配置一系列核心记录器（嵌入式容器、Hibernate和Spring Boot）以输出更多信息。启用调试模式不会将应用程序配置为记录所有debug级别的消息。

或者，您可以通过使用--trace标志（或application.properties中的trace=true）启动应用程序来启用“跟踪”模式。这样做可以为一系列核心记录器（嵌入式容器、Hibernate模式生成和整个Spring组合）启用跟踪日志记录。

### 色彩输出

如果终端支持ANSI，则使用颜色输出来帮助可读性。您可以设置spring.output.ansi.enabled 为支持的值以覆盖自动检测。

使用`%clr`转换字配置颜色编码。在最简单的形式中，转换器根据日志级别对输出进行着色，如下例所示：

%clr(%5p)

下表描述了日志级别到颜色的映射：

| Level | Color  |
| ----- | ------ |
| FATAL | Red    |
| ERROR | Red    |
| WARN  | Yellow |
| INFO  | Green  |
| DEBUG | Green  |
| TRACE | Green  |

或者，您可以通过将其作为转换选项来指定应使用的颜色或样式。例如，要使文本变为黄色，请使用以下设置：

%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){yellow}

支持以下颜色和样式：

- blue
- cyan
- faint
- green
- magenta
- red
- yellow

## 文件输出

默认情况下，Spring Boot只记录到控制台，不写入日志文件。如果您想写入控制台输出之外的日志文件，则需要设置logging.file.name或logging.file.path属性（在application.properties中）

| logging.file.name | logging.file.path | Example  | Description                                                |
| ----------------- | ----------------- | -------- | ---------------------------------------------------------- |
| *(none)*          | *(none)*          |          | 输出到控制台                                               |
| 指定文件          | *(none)*          | my.log   | 写入指定的日志文件。名称可以是精确的位置或相对于当前目录。 |
| *(none)*          | 指定目录          | /var/log | 写入到指定的目录。名称可以是精确的位置或相对于当前目录。   |

当日志文件达到10MB时，它们会旋转，与控制台输出一样，默认情况下会记录ERROR级别、WARN级别和INFO级别的消息。

## 文件轮转

如果使用的是Logback，则可以使用应用程序微调日志旋转设置。对于所有其他日志记录系统，您需要自己直接配置旋转设置（例如，如果您使用Log4J2，则可以添加Log4J2.xml或Log4J2-spring.xml文件）。

支持以下轮换策略属性：

| Name                                                 | Description                                 |
| ---------------------------------------------------- | ------------------------------------------- |
| logging.logback.rollingpolicy.file-name-pattern      | 用于创建日志存档的文件名模式。              |
| logging.logback.rollingpolicy.clean-history-on-start | 如果应用程序启动时应进行日志归档清理。      |
| logging.logback.rollingpolicy.max-file-size          | 归档前日志文件的最大大小。                  |
| logging.logback.rollingpolicy.total-size-cap         | 日志存档在被删除之前可以占用的最大大小。    |
| logging.logback.rollingpolicy.max-history            | 要保留的存档日志文件的最大数量（默认为7）。 |

## 日志级别

通过使用`logging.level.<logger name>=<level>`设置日志记录级别，其中level是TRACE、DEBUG、INFO、WARN、ERROR、FATAL或OFF之一。可以使用`logging.level.root`配置根记录器。下面是参考示例：

```properties
logging.level.root=warn
logging.level.org.springframework.web=debug
logging.level.org.hibernate=error
```

还可以使用环境变量设置日志记录级别。例如，`LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_WEB=DEBUG`将设置org.springframework.web到debug级别。

上述方法仅适用于包级日志记录。由于轻松绑定总是将环境变量转换为小写，因此不可能以这种方式为单个类配置日志记录。如果需要为类配置日志记录，可以使用SPRING_APPLICATION_JSON变量。

## 日志分组

能够将相关的记录器分组在一起，以便它们可以同时配置，这通常很有用。例如，您可能通常会更改所有与Tomcat相关的记录器的日志级别，但您不容易记住顶级包。

为了帮助实现这一点，Spring Boot允许您在Spring环境中定义日志组。例如，下面是如何通过将“tomcat”组添加到application.properties来定义它的方法：

```properties
logging.group.tomcat=org.apache.catalina,org.apache.coyote,org.apache.tomcat
```

定义后，您可以通过单行更改组中所有记录器的级别：

logging.level.tomcat=trace

Spring Boot包括以下预定义的日志记录组，可以现成使用：

| web  | org.springframework.core.codec, org.springframework.http, org.springframework.web, org.springframework.boot.actuate.endpoint.web, org.springframework.boot.web.servlet.ServletContextInitializerBeans |
| ---- | ------------------------------------------------------------ |
| sql  | org.springframework.jdbc.core, org.hibernate.SQL, org.jooq.tools.LoggerListener |

## 使用日志的Shutdown 钩子

为了在应用程序终止时释放日志资源，提供了一个在JVM退出时触发日志系统清理的关闭挂钩。除非您的应用程序部署为war文件，否则此关闭挂钩将自动注册。如果您的应用程序具有复杂的上下文层次结构，则关闭挂钩可能无法满足您的需要。如果没有，请禁用关闭挂钩并调查底层日志记录系统直接提供的选项。例如，Logback提供了上下文选择器，允许在自己的上下文中创建每个Logger。您可以使用logging.register-shutdown-hook属性以禁用关机挂钩。

## 自定义日志

可以通过在类路径上包含适当的库来激活各种日志记录系统，并且可以通过在classpath的根目录中或在以下Spring Environment 配置 `logging.config `属性提供适当的配置文件来进一步定制。

您可以使用`org.springframework.Boot.logging.LoggingSystem`强制Spring Boot使用特定的日志系统。该值应为LoggingSystem实现的完全限定类名。您还可以使用none值完全禁用Spring Boot的日志配置。

由于日志是在创建ApplicationContext之前初始化的，因此无法从Spring`@Configuration`文件中的`@PropertySources`控制日志。更改或完全禁用日志记录系统的唯一方法是通过系统属性。

根据您的日志记录系统，将加载以下文件：

| Logging System          | Customization                                                |
| ----------------------- | ------------------------------------------------------------ |
| Logback                 | logback-spring.xml, logback-spring.groovy, logback.xml, or logback.groovy |
| Log4j2                  | log4j2-spring.xml or log4j2.xml                              |
| JDK (Java Util Logging) | logging.properties                                           |

如果可能，我们建议您在日志配置中使用`-spring`变量（例如，`logback-spring.xml`而不是`logback.xml`）。如果使用标准配置位置，Spring无法完全控制日志初始化。

Java Util Logging存在已知的类加载问题，这些问题会导致从“可执行jar”运行时出现问题。如果可能的话，我们建议您在从“可执行jar”运行时避免这种情况。

为了帮助定制，将一些其他属性从Spring Environment转移到System属性，如下表所述：

| Spring Environment                | System Property               | Comments                                                     |
| --------------------------------- | ----------------------------- | ------------------------------------------------------------ |
| logging.exception-conversion-word | LOG_EXCEPTION_CONVERSION_WORD | 记录异常时使用的转换词。                                     |
| logging.file.name                 | LOG_FILE                      | 如果已定义，它将在默认日志配置中使用。                       |
| logging.file.path                 | LOG_PATH                      | 如果已定义，它将在默认日志配置中使用。                       |
| logging.pattern.console           | CONSOLE_LOG_PATTERN           | 要在控制台上使用的日志模式（stdout）。                       |
| logging.pattern.dateformat        | LOG_DATEFORMAT_PATTERN        | 日志日期格式的附加模式。                                     |
| logging.charset.console           | CONSOLE_LOG_CHARSET           | 用于控制台日志记录的字符集。                                 |
| logging.pattern.file              | FILE_LOG_PATTERN              | 要在文件中使用的日志模式（如果启用了LOG_FILE ）。            |
| logging.charset.file              | FILE_LOG_CHARSET              | 用于文件日志记录的字符集（如果启用了LOG_FILE ）。            |
| logging.pattern.level             | LOG_LEVEL_PATTERN             | 渲染日志级别时使用的格式（默认为%5p）。                      |
| PID                               | PID                           | 当前进程ID（如果可能，并且尚未定义为操作系统环境变量时发现）。 |

如果使用Logback，还将传输以下属性：

| Spring Environment                                   | System Property                              | Comments                                                     |
| ---------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| logging.logback.rollingpolicy.file-name-pattern      | LOGBACK_ROLLINGPOLICY_FILE_NAME_PATTERN      | Pattern for rolled-over log file names (default ${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz). |
| logging.logback.rollingpolicy.clean-history-on-start | LOGBACK_ROLLINGPOLICY_CLEAN_HISTORY_ON_START | Whether to clean the archive log files on startup.           |
| logging.logback.rollingpolicy.max-file-size          | LOGBACK_ROLLINGPOLICY_MAX_FILE_SIZE          | Maximum log file size.                                       |
| logging.logback.rollingpolicy.total-size-cap         | LOGBACK_ROLLINGPOLICY_TOTAL_SIZE_CAP         | Total size of log backups to be kept.                        |
| logging.logback.rollingpolicy.max-history            | LOGBACK_ROLLINGPOLICY_MAX_HISTORY            | Maximum number of archive log files to keep.                 |

所有受支持的日志记录系统在解析其配置文件时都可以查阅系统属性。请参见spring-boot.jar中的默认配置：

- [Logback](https://github.com/spring-projects/spring-boot/tree/v2.7.5/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/logback/defaults.xml)
- [Log4j 2](https://github.com/spring-projects/spring-boot/tree/v2.7.5/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/log4j2/log4j2.xml)
- [Java Util logging](https://github.com/spring-projects/spring-boot/tree/v2.7.5/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/java/logging-file.properties)

如果你想在日志属性中使用占位符，你应该使用SpringBoot的语法，而不是底层框架的语法。值得注意的是，如果使用Logback，则应使用`:`作为属性名称和默认值之间的分隔符，而不要使用`:-`。

## Logback 扩展

Spring Boot包括许多Logback扩展，可以帮助进行高级配置。您可以在logback-spring.xml中使用这些扩展。

因为logback.xml文件过早被加载，你不能在其中使用扩展功能。

扩展不能用于Logback的配置扫描。如果您尝试这样做，对配置文件进行更改将导致类似以下记录之一的错误：

```properties
ERROR in ch.qos.logback.core.joran.spi.Interpreter@4:71 - no applicable action for [springProperty], current ElementPath is [[configuration][springProperty]]
ERROR in ch.qos.logback.core.joran.spi.Interpreter@4:71 - no applicable action for [springProfile], current ElementPath is [[configuration][springProfile]]
```

### Profile-specific配置

＜springProfile＞标签允许您根据活动的Spring配置文件选择性地包括或排除配置部分。配置文件部分在＜configuration＞元素中的任何位置都受支持。使用name属性指定接受配置的profile。<springProfile>标记可以包含profile文件名称（例如staging）或profile文件表达式。配置文件表达式允许表达更复杂的配置文件逻辑，例如`production&（eu central | eu west）`。有关详细信息，请参阅参考指南。以下列表显示了三个示例配置文件：

```xml
<springProfile name="staging">
  <!-- configuration to be enabled when the "staging" profile is active -->
</springProfile>

<springProfile name="dev | staging">
  <!-- configuration to be enabled when the "dev" or "staging" profiles are active -->
</springProfile>

<springProfile name="!production">
  <!-- configuration to be enabled when the "production" profile is not active -->
</springProfile>
```



### Environment  属性

＜springProperty＞标记允许您从Spring Environment中公开属性，以便在Logback中使用。如果您想从application.propertie访问值，那么这样做很有用该标签的工作方式与Logback的标准＜property＞标签类似。但是，不是指定直接值，而是指定属性的源（来自环境）。如果需要将属性存储在本地范围以外的其他地方，可以使用scope属性。如果需要回退值（如果未在环境中设置该属性），可以使用defaultValue属性。以下示例显示了如何公开属性以在Logback中使用：

```xml
<springProperty scope="context" name="fluentHost" source="myapp.fluentd.host"
        defaultValue="localhost"/>
<appender name="FLUENT" class="ch.qos.logback.more.appenders.DataFluentAppender">
    <remoteHost>${fluentHost}</remoteHost>
    ...
</appender>
```



# 国际化

Spring Boot支持本地化消息，以便您的应用程序能够满足不同语言偏好的用户。默认情况下，Spring Boot在类路径的根目录中查找消息资源包的存在。

当配置的资源包的默认属性文件可用（默认情况下为messages.properties）时，将应用自动配置。如果您的资源包仅包含特定于语言的属性文件，则需要添加默认值。如果未找到与任何已配置的基名称匹配的属性文件，则不会有自动配置的MessageSource。

可以使用spring.messages配置资源包的basename以及其他几个属性。如下例所示：

```xml
spring.messages.basename=messages,config.i18n.messages
spring.messages.fallback-to-system-locale=false
```

spring.messages.basename支持逗号分隔的位置列表，可以是包限定符，也可以是从类路径根解析的资源。

# JSON

Spring Boot提供了三个JSON映射库的集成：

- Gson
- Jackson
- JSON-B

Jackson是首选和默认库。

## Jackson

提供了Jackson的自动配置，Jackson是spring-boot-starter-json的一部分。当Jackson位于类路径上时，会自动配置`ObjectMapper `bean。提供了几个配置属性用于自定义`ObjectMapper`的配置。



### 自定义 `ObjectMapper`



### 自定义序列化和反序列化

如果您使用Jackson序列化和反序列化JSON数据，您可能需要编写自己的`JsonSerializer`和`JsonDeserializer`类。定制序列化程序通常通过一个模块向Jackson注册，但Spring Boot提供了另一个`@JsonComponent`注释，使直接注册Spring Beans更容易。

您可以直接在`JsonSerializer`、`JsonDeserializer`或`KeyDeserializer`实现上使用`@JsonComponent`注释。您还可以将其用于包含serializers/deserializers 作为内部类的类，如以下示例所示：

```java
@JsonComponent
public class MyJsonComponent {

    public static class Serializer extends JsonSerializer<MyObject> {

        @Override
        public void serialize(MyObject value, JsonGenerator jgen, SerializerProvider serializers) throws IOException {
            jgen.writeStartObject();
            jgen.writeStringField("name", value.getName());
            jgen.writeNumberField("age", value.getAge());
            jgen.writeEndObject();
        }

    }

    public static class Deserializer extends JsonDeserializer<MyObject> {

        @Override
        public MyObject deserialize(JsonParser jsonParser, DeserializationContext ctxt) throws IOException {
            ObjectCodec codec = jsonParser.getCodec();
            JsonNode tree = codec.readTree(jsonParser);
            String name = tree.get("name").textValue();
            int age = tree.get("age").intValue();
            return new MyObject(name, age);
        }

    }

}
```

ApplicationContext中的所有`@JsonComponent` bean都会自动向Jackson注册。因为`@JsonComponent`用`@Component`进行了元注释，所以通常的组件扫描规则适用。

Spring Boot还提供了`JsonObjectSerializer`和`JsonObjectDeserializer`基类，它们在序列化对象时提供了标准Jackson版本的有用替代方案。有关详细信息，请参见Javadoc中的[JsonObjectSerializer](https://docs.spring.io/spring-boot/docs/2.7.5/api/org/springframework/boot/jackson/JsonObjectSerializer.html)和[JsonObjectDeserializer](https://docs.spring.io/spring-boot/docs/2.7.5/api/org/springframework/boot/jackson/JsonObjectDeserializer.html)。

上面的示例可以重写为使用JsonObjectSerializer/JsonObjectDeserializer，如下所示：

```java
@JsonComponent
public class MyJsonComponent {

    public static class Serializer extends JsonObjectSerializer<MyObject> {

        @Override
        protected void serializeObject(MyObject value, JsonGenerator jgen, SerializerProvider provider)
                throws IOException {
            jgen.writeStringField("name", value.getName());
            jgen.writeNumberField("age", value.getAge());
        }

    }

    public static class Deserializer extends JsonObjectDeserializer<MyObject> {

        @Override
        protected MyObject deserializeObject(JsonParser jsonParser, DeserializationContext context, ObjectCodec codec,
                JsonNode tree) throws IOException {
            String name = nullSafeValue(tree.get("name"), String.class);
            int age = nullSafeValue(tree.get("age"), Integer.class);
            return new MyObject(name, age);
        }

    }

}
```

### Mixins

Jackson支持mixin，可用于将附加注释混合到目标类上已声明的注释中。Spring Boot的Jackson自动配置将扫描应用程序的包，查找用`@JsonMixin`注释的类，并将它们注册到自动配置的`ObjectMapper`中。注册由Spring Boot的`JsonMixinModule`执行。



# 任务执行和调度

`ThreadPoolTaskExecutor`??

`AsyncConfigurer`??



在上下文中没有`Executor `bean的情况下，Spring Boot会自动配置`ThreadPoolTaskExecuttor`，使其具有可自动关联到异步任务执行（@EnableAsync）和Spring MVC异步请求处理。

如果您在上下文中定义了自定义Executor，则常规任务执行（即@EnableAsync）将透明地使用它，但不会配置Spring MVC支持，因为它需要`AsyncTaskExecuttor`实现（命名为applicationTaskExecuter）。根据您的目标安排，您可以将执行器更改为`ThreadPoolTaskExecutor`，或者同时定义一个`ThreadPool TaskExecuter`和一个`AsyncConfigurer`来包装您的自定义执行器。

线程池使用8个核心线程，可以根据负载增长和收缩。这些默认设置可以使用spring.task.execution命名空间进行微调，如下例所示：

```properties
spring.task.execution.pool.max-size=16
spring.task.execution.pool.queue-capacity=100
spring.task.execution.pool.keep-alive=10s
```

这会将线程池更改为使用有界队列，这样当队列已满（100个任务）时，线程池将增加到最多16个线程。当线程空闲10秒（而不是默认的60秒）时，由于线程被回收，因此池的收缩更为严重。

如果需要与计划任务执行关联，ThreadPoolTaskScheduler也可以自动配置（例如使用@EnableSchedule）。线程池默认使用一个线程，其设置可以使用spring.task.scheduling进行微调。如下例所示：

```properties
spring.task.scheduling.thread-name-prefix=scheduling-
spring.task.scheduling.pool.size=2
```

如果需要创建自定义执行器或调度器，则可以在上下文中使用`TaskExecutorBuilder `bean和`TaskSchedulerBuilder `bean。



# 测试

Spring Boot提供了许多实用程序和注释，以在测试应用程序时提供帮助。测试支持由两个模块提供：spring-boot-test包含核心项目，spring-boot-test-autoconfigure 支持测试的自动配置。

大多数开发人员都使用spring-boot-starter-test“starter”，它导入了SpringBoot测试模块以及JUnit Jupiter、AssertJ、Hamcrest和许多其他有用的库

## 测试范围依赖项

spring-boot-starter-test 依赖下面的库：

- [JUnit 5](https://junit.org/junit5/): 单元测试Java应用程序的事实标准。
- [Spring Test](https://docs.spring.io/spring-framework/docs/5.3.23/reference/html/testing.html#integration-testing) & Spring Boot Test: Spring Boot应用程序的实用程序和集成测试支持。
- [AssertJ](https://assertj.github.io/doc/):  流畅的断言库。
- [Hamcrest](https://github.com/hamcrest/JavaHamcrest):  匹配器对象库（也称为约束或谓词）。
- [Mockito](https://site.mockito.org/):  Java模拟框架。
- [JSONassert](https://github.com/skyscreamer/JSONassert):  JSON的断言库。
- [JsonPath](https://github.com/jayway/JsonPath): XPath for JSON.

我们通常认为这些公共库在编写测试时很有用。如果这些库不适合您的需要，您可以添加自己的附加测试依赖项。

## 测试spring 程序

依赖注入的一个主要优点是它应该使代码更容易进行单元测试。您可以使用new操作符实例化对象，甚至不需要涉及Spring。您还可以使用模拟对象而不是实际的依赖关系。

通常，您需要跳过单元测试，开始集成测试（使用SpringApplicationContext）。能够在不需要部署应用程序或连接到其他基础设施的情况下执行集成测试是非常有用的。

Spring Framework包括用于此类集成测试的专用测试模块。您可以直接向org.springframework:spring-test声明一个依赖项，或者使用spring-boot-starter-test “starter”将其传递进来。



## 测试spring boot 程序

Spring Boot应用程序是一个Spring ApplicationContext，所以除了通常使用普通Spring上下文所做的测试之外，不需要做任何特别的事情来测试它。

SpringBoot的外部属性、日志记录和其他特性默认情况下仅在使用SpringApplication创建时才安装在上下文中。

SpringBoot提供了一个`@SpringBootTest`注释，当您需要Spring Boot特性时，它可以用作标准Spring test`@ContextConfiguration`注释的替代。注释的工作方式是通过`SpringApplication`创建测试中使用的`ApplicationContext`。除了`@SpringBootTest`之外，还提供了许多其他注释，用于测试应用程序的更具体部分。

默认情况下，`@SpringBootTest`不会启动服务器。您可以使用`@SpringBootTest`的`webEnvironment`属性进一步细化测试的运行方式：

- MOCK(Default) : 加载web ApplicationContext并提供模拟web环境。使用此注释时，不会启动嵌入式服务器。如果类路径上没有可用的web环境，则此模式将透明地返回到创建常规的非web `ApplicationContext`。它可以与`@AutoConfigureMockMvc`或`@AutoConfigureWebTestClient`结合使用，用于web应用程序的模拟测试。
- RANDOM_PORT: 加载`WebServerApplicationContext`并提供真实的web环境。嵌入式服务器启动并在随机端口上侦听。
- DEFINED_PORT： 加载`WebServerApplicationContext`并提供真实的web环境。嵌入式服务器启动并在定义的端口（来自application.properties）或默认端口8080上侦听。
- NONE: 使用SpringApplication加载ApplicationContext，但不提供任何web环境（模拟或其他）。



如果您的测试是`@Transactional`，默认情况下，它会在每个测试方法结束时回滚事务。然而，当与`RANDOM_PORT`或`DEFINED_PORT`一起使用时，隐含地提供了一个真实的servlet环境，HTTP客户端和服务器在单独的线程中运行，因此在单独的事务中运行。在这种情况下，在服务器上启动的任何事务都不会回滚。

### 探测程序的web类型

如果Spring MVC可用，则会配置常规的基于MVC的应用程序上下文。如果您只有SpringWebFlux，我们将检测到这一点并配置基于WebFlux的应用程序上下文。

如果两者都存在，则Spring MVC优先。如果要在此场景中测试反应式web应用程序，必须设置`spring.main.web-application-type`属性：

```properties
@SpringBootTest(properties = "spring.main.web-application-type=reactive")
class MyWebFluxTests {

    // ...

}
```



### 探测测试配置类

如果您熟悉Spring测试框架，您可能习惯使用`@ContextConfiguration（classes=…) `以便指定要加载的Spring`@Configuration`。或者，您可能经常在测试中使用嵌套的`@Configuration`类。

在测试Spring Boot应用程序时，这通常是不需要的。Spring Boot的`@*Test `注释会在您没有显式定义主配置时自动搜索主配置。

搜索算法从包含测试的包开始工作，直到找到一个用`@SpringBootApplication`或`@SpringBootConfiguration`注释的类。只要您以合理的方式构造代码，通常会找到您的主配置。

如果要自定义主配置，可以使用嵌套的@TestConfiguration类。与嵌套的@Configuration类（它将用于代替应用程序的主配置）不同，嵌套的@TestConfiguration类将用于应用程序的主要配置。

Spring的测试框架在测试之间缓存应用程序上下文。因此，只要您的测试共享相同的配置（无论如何发现），加载上下文的潜在耗时过程只会发生一次。

### 排除测试配置类

如果您的应用程序使用组件扫描（例如，如果您使用@SpringBootApplication或@ComponentScan），您可能会发现您仅为特定测试创建的顶级配置类会意外地到处出现。

正如我们前面所看到的，可以在测试的内部类上使用@TestConfiguration来定制主配置。当放置在顶级类上时，@TestConfiguration表示不应通过扫描来拾取src/test/java中的类。然后，您可以在需要的地方显式导入该类，如下例所示：

```java
@SpringBootTest
    @Import(MyTestsConfiguration.class)
    class MyTests {

        @Test
        void exampleTest() {
            // ...
        }

    }
```



### 使用程序的参数

如果应用程序需要参数，可以让@SpringBootTest使用args属性注入它们。

```java
@SpringBootTest(args = "--app.test=one")
class MyApplicationArgumentTests {

    @Test
    void applicationArgumentsPopulated(@Autowired ApplicationArguments args) {
        assertThat(args.getOptionNames()).containsOnly("app.test");
        assertThat(args.getOptionValues("app.test")).containsOnly("one");
    }

}
```

### 使用模拟环境进行测试

默认情况下，@SpringBootTest不会启动服务器，而是为测试web端点设置模拟环境。

使用Spring MVC，我们可以使用MockMvc或WebTestClient查询web端点，如下例所示：

```java
@SpringBootTest
@AutoConfigureMockMvc
class MyMockMvcTests {

    @Test
    void testWithMockMvc(@Autowired MockMvc mvc) throws Exception {
        mvc.perform(get("/")).andExpect(status().isOk()).andExpect(content().string("Hello World"));
    }

    // If Spring WebFlux is on the classpath, you can drive MVC tests with a WebTestClient
    @Test
    void testWithWebTestClient(@Autowired WebTestClient webClient) {
        webClient
                .get().uri("/")
                .exchange()
                .expectStatus().isOk()
                .expectBody(String.class).isEqualTo("Hello World");
    }

}
```

如果您只想关注web层而不想启动完整的ApplicationContext，请考虑改用@WebMvcTest。

对于Spring WebFlux端点，您可以使用WebTestClient，如以下示例所示：

```java
@SpringBootTest
@AutoConfigureWebTestClient
class MyMockWebTestClientTests {

    @Test
    void exampleTest(@Autowired WebTestClient webClient) {
        webClient
            .get().uri("/")
            .exchange()
            .expectStatus().isOk()
            .expectBody(String.class).isEqualTo("Hello World");
    }

}
```

























































































































# 创建自动配置

如果您在一家开发共享库的公司工作，您可能希望开发自己的自动配置。自动配置类可以捆绑在外部jar中，由SpringBoot拾取。我们先熟悉自动配置的原理，然后再学习如何创建starter。

## 自动配置类

实现自动配置的类用@AutoConfiguration注释。这个注释本身用@Configuration进行元注释，附加的@Conditional注释用于约束何时应用自动配置。通常，自动配置类使用@ConditionalOnClass和@ConditionalOnMissingBean注释。这可以确保只有在用户的@Configuration找不到声明的相关类时，自动配置才适用。下面是spring提供的RestTemplate的自动配置类

```java
@AutoConfiguration(after = HttpMessageConvertersAutoConfiguration.class)
@ConditionalOnClass(RestTemplate.class)
@Conditional(NotReactiveWebApplicationCondition.class)
public class RestTemplateAutoConfiguration {

	@Bean
	@Lazy
	@ConditionalOnMissingBean
	public RestTemplateBuilderConfigurer restTemplateBuilderConfigurer(
			ObjectProvider<HttpMessageConverters> messageConverters,
			ObjectProvider<RestTemplateCustomizer> restTemplateCustomizers,
			ObjectProvider<RestTemplateRequestCustomizer<?>> restTemplateRequestCustomizers) {
		RestTemplateBuilderConfigurer configurer = new RestTemplateBuilderConfigurer();
		configurer.setHttpMessageConverters(messageConverters.getIfUnique());
		configurer.setRestTemplateCustomizers(restTemplateCustomizers.orderedStream().collect(Collectors.toList()));
		configurer.setRestTemplateRequestCustomizers(
				restTemplateRequestCustomizers.orderedStream().collect(Collectors.toList()));
		return configurer;
	}

	@Bean
	@Lazy
	@ConditionalOnMissingBean
	public RestTemplateBuilder restTemplateBuilder(RestTemplateBuilderConfigurer restTemplateBuilderConfigurer) {
		RestTemplateBuilder builder = new RestTemplateBuilder();
		return restTemplateBuilderConfigurer.configure(builder);
	}

	static class NotReactiveWebApplicationCondition extends NoneNestedConditions {

		NotReactiveWebApplicationCondition() {
			super(ConfigurationPhase.PARSE_CONFIGURATION);
		}

		@ConditionalOnWebApplication(type = Type.REACTIVE)
		private static class ReactiveWebApplication {

		}

	}

}
```

## 定位自动配置类

Spring Boot检查所有的jar是否存在 **META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports** 文件并读取。该文件列出自动配置类，每行一个类名，如下例所示：

```shell
com.mycorp.libx.autoconfigure.LibXAutoConfiguration
com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration
```

可以使用#字符添加注释。

只能通过imports文件中来加载自动配置。确保它们是在特定的包空间中定义的，并且它们永远不是组件扫描的目标。此外，自动配置类不应启用组件扫描来查找其他组件。应改用特定的@Imports。

## 自动配置的顺序问题

如果您的配置需要按特定顺序应用，您可以使用@AutoConfiguration注释上的before、beforeName、after和afterName属性，或专用的@AutoConfigureBefore和@AutoConfigureAfter注释。例如上面的RestTemplateAutoConfiguration，需要在HttpMessageConvertersAutoConfiguration之后应用类。

如果您想排序某些不应该相互直接了解的自动配置，也可以使用@AutoConfigureOrder。该注释与常规@Order注释具有相同的语义，但为自动配置类提供了专用的顺序：

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {
    ....
}
```



与标准@Configuration类一样，自动配置类的应用顺序只影响其bean的定义顺序。随后创建这些bean的顺序不受影响，并由每个bean的依赖项和任何@DependsOn关系决定。

## 条件注解

您总是希望在自动配置类中包含一个或多个@Conditional注释。@ConditionalOnMissingBean注释是一个常见的示例，它允许开发人员在不满意您的默认值时重写自动配置。

SpringBoot包含许多@Conditional注释，注释@Configuration类或单个@Bean方法。

### Class Conditions

@ConditionalOnClass和@ConditionalOnMissingClass注释允许根据特定类的存在或不存在来包含@Configuration类。由于注释元数据是使用ASM解析的，因此您可以使用value属性来引用真实的类，即使该类实际上可能不会出现在运行的应用程序类路径上。如果希望使用String值指定类名，也可以使用name属性。

这种机制不适用于返回类型通常是条件目标的@Bean方法：在应用方法的条件之前，JVM将加载类并可能处理方法引用，如果类不存在，这些引用将失败。

为了处理这种情况，可以使用单独的@Configuration类来隔离条件，如以下示例所示：

```java
@AutoConfiguration
// Some conditions ...
public class MyAutoConfiguration {

    // Auto-configured beans ...

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(SomeService.class)
    public static class SomeServiceConfiguration {

        @Bean
        @ConditionalOnMissingBean
        public SomeService someService() {
            return new SomeService();
        }

    }

}
```



如果使用@ConditionalOnClass或@Conditional OnMissingClass作为元注释的一部分来组成自己的合成注释，则必须使用名称，因为在这种情况下，引用的类不会被处理。

### Bean Conditions

@ConditionalOnBean和@Conditional OnMissingBean注释允许根据特定bean的存在或不存在来包含bean。您可以使用value属性按类型指定bean，name属性按名称指定beans。search属性允许您限制搜索bean时应考虑的ApplicationContext层次结构。

当放置在@Bean方法上时，目标类型默认为该方法的返回类型，如下例所示：

```java
@AutoConfiguration
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SomeService someService() {
        return new SomeService();
    }

}
```

在前面的示例中，如果ApplicationContext中没有someService类型的bean，那么将创建someService bean。

您需要非常小心添加bean定义的顺序，因为这些条件是根据迄今为止处理的内容进行评估的。因此，我们建议在自动配置类上只使用@ConditionalOnBean和@ConditionalOnMissingBean注释（因为这些注释可以在添加任何用户定义的bean定义后加载）。



@ConditionalOnBean和@ConditionalOnMissingBean不会阻止创建被@Configuration标注的类。在类级别使用这些条件和用注释标记每个包含的@Bean方法之间的唯一区别是，如果条件不匹配，前者会阻止将@Configuration类注册为Bean。



在声明@Bean方法时，在方法的返回类型中提供尽可能多的类型信息。例如，如果bean的具体类实现了一个接口，那么bean方法的返回类型应该是具体类，而不是接口。当使用Bean条件时，在@Bean方法中提供尽可能多的类型信息尤其重要，因为它们的求值只能依赖于方法签名中可用的类型信息。

### Property Conditions

@ConditionalOnProperty注释允许基于Spring环境属性包含配置。使用prefix 和name属性指定应检查的属性。默认情况下，匹配任何存在且不等于false的属性。您还可以使用havingValue和matchIfMissing属性创建更高级的检查。

### Resource Conditions

@ConditionalOnResource注释仅允许在存在特定资源时包含配置。可以使用常见的Spring约定指定资源，如以下示例所示：file:/home/user/test.dat。

### Web Application Conditions

@ConditionalOnWebApplication和@ConditionalOnNotWebApplication注释允许根据应用程序是否为“web应用程序”来包含配置。基于servlet的web应用程序是使用Spring WebApplicationContext、定义会话范围或具有ConfigurableWebEnvironment的任何应用程序。反应式web应用程序是使用ReactiveWebApplicationContext或具有ConfigurableReactiveWeb环境的任何应用程序。

@ConditionalOnWarDeployment注释允许根据应用程序是否是部署到容器的传统WAR应用程序来包含配置。对于使用嵌入式服务器运行的应用程序，此条件将不匹配。

### SpEL Expression Conditions

@ConditionalOnExpression注释允许根据SpEL表达式的结果包含配置。

在表达式中引用bean将导致该bean在上下文刷新处理中很早就被初始化。因此，bean将无法进行后期处理（如配置属性绑定），其状态可能不完整。

## 测试自动配置

自动配置可能受到许多因素的影响：用户配置（@Bean定义和环境）、条件评估（特定库的存在）以及其他因素。具体来说，每个测试都应该创建一个定义良好的ApplicationContext，表示这些定制的组合。ApplicationContextRunner提供了一种很好的实现方法。

ApplicationContextRunner通常定义为测试类的一个字段，用于收集基本的通用配置。以下示例确保始终调用MyServiceAutoConfiguration：

```java
private final ApplicationContextRunner contextRunner = new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(MyServiceAutoConfiguration.class));
```

如果必须定义多个自动配置，则无需对其声明进行排序，因为它们的调用顺序与运行应用程序时的顺序完全相同。

每个测试都可以使用runner来表示特定的用例。例如，下面的示例调用用户配置（UserConfiguration）并检查自动配置是否正确回退。run方法提供了可与AssertJ一起使用的回调上下文。

```java
@Test
void defaultServiceBacksOff() {
    this.contextRunner.withUserConfiguration(UserConfiguration.class).run((context) -> {
        assertThat(context).hasSingleBean(MyService.class);
        assertThat(context).getBean("myCustomService").isSameAs(context.getBean(MyService.class));
    });
}

@Configuration(proxyBeanMethods = false)
static class UserConfiguration {

    @Bean
    MyService myCustomService() {
        return new MyService("mine");
    }

}
```

还可以轻松自定义环境变量，如以下所示：

```java
@Test
void serviceNameCanBeConfigured() {
    this.contextRunner.withPropertyValues("user.name=test123").run((context) -> {
        assertThat(context).hasSingleBean(MyService.class);
        assertThat(context.getBean(MyService.class).getName()).isEqualTo("test123");
    });
}
```



runner 还可用于显示ConditionEvaluationReport。报告在INFO或DEBUG级别打印。以下示例显示如何使用ConditionEvaluationReportLoggingListener在自动配置测试中打印报告。

```java
class MyConditionEvaluationReportingTests {

    @Test
    void autoConfigTest() {
        new ApplicationContextRunner()
            .withInitializer(new ConditionEvaluationReportLoggingListener(LogLevel.INFO))
            .run((context) -> {
                    // Test something...
            });
    }

}
```

### 模拟Web上下文

如果需要测试仅在servlet或反应式web应用程序上下文中运行的自动配置，请分别使用WebApplicationContextRunner或ReactiveWebApplicationContext Runner。

### 覆盖类路径

还可以测试运行时不存在特定类和/或包时发生的情况。Spring Boot附带了一个FilteredClassLoader，可供runner轻松使用。在下面的示例中，我们断言如果MyService不存在，自动配置将被正确禁用：

```java
@Test
void serviceIsIgnoredIfLibraryIsNotPresent() {
    this.contextRunner.withClassLoader(new FilteredClassLoader(MyService.class))
            .run((context) -> assertThat(context).doesNotHaveBean("myService"));
}
```

## 创建starter

典型的Spring Boot启动器包含用于自动配置和自定义给定技术的基础架构的代码，我们称之为“acme”。为了使其易于扩展，可以将专用命名空间中的许多配置键公开给环境。最后，提供单个“启动器”依赖项，以帮助用户尽可能轻松地入门。

具体而言，自定义启动器可以包含以下内容：

- 包含“acme”自动配置代码的autoconfigure模块。
- 依赖autoconfigure 模块和其他依赖关系的的starter模块。

如果自动配置相对简单并且没有可选功能，则在starter中合并两个模块绝对是一种选择。

### 命名规范

自定义的start格式：**acme-spring-boot-starter**

**官方的格式：****spring-boot-starter-web**

### 配置键

如果starter提供配置键，请为它们使用唯一的命名空间。特别是，不要将键包含在 Spring Boot 使用的命名空间中。

```java
import java.time.Duration;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("acme")
public class AcmeProperties {

    /**
     * Whether to check the location of acme resources.
     */
    private boolean checkLocation = true;

    /**
     * Timeout for establishing a connection to the acme server.
     */
    private Duration loginTimeout = Duration.ofSeconds(3);

    public boolean isCheckLocation() {
        return this.checkLocation;
    }

    public void setCheckLocation(boolean checkLocation) {
        this.checkLocation = checkLocation;
    }

    public Duration getLoginTimeout() {
        return this.loginTimeout;
    }

    public void setLoginTimeout(Duration loginTimeout) {
        this.loginTimeout = loginTimeout;
    }

}
```

确保触发元数据（META-INF/spring-configuration-metadata.json）生成，以便 IDE 开启自动提示功能。

### autoconfigure模块

自动配置模块包含库所需的所有内容。它还可能包含配置键定义（如@ConfigurationProperties）和任何可用于进一步自定义组件初始化方式的回调接口。

应将spring-boot-starter库的依赖项标记为可选，以便更轻松地在项目中包含自动配置模块。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-processor</artifactId>
    <optional>true</optional>
</dependency>
```

spring-boot-starter使用注释处理器在元数据文件中收集有关自动配置的条件（META-INF/spring-autoconfigure-metadata.properties）。如果该文件存在，则用于预先筛选不匹配的自动配置，这将缩短启动时间。

```properties
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration.AutoConfigureOrder=-2147483648
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration=
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration.AutoConfigureOrder=-2147483648
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration=
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration$JacksonConfiguration=
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration$JacksonConfiguration.ConditionalOnClass=com.fasterxml.jackson.databind.ObjectMapper
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration.ConditionalOnClass=com.couchbase.client.java.Cluster
```



```groovy
dependencies {
    annotationProcessor "org.springframework.boot:spring-boot-autoconfigure-processor"
}
```



### starter模块

starter是一个空jar。它的唯一目的是提供必要的依赖项来使用库。
