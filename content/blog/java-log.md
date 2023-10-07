---
title: java-log
tags:
  - java
categories:
  - 技术
date: 2022-12-02 13:02:32
---

# JAVA日志体系
# 日志门面
SLF4J(Simple Logging Facade for Java)是一套日志门面，或者说规范。其他的日志组件都是基于这个门面进行实际实现，比如` java.util.logging`,` logback` 和 `reload4j`. 

用户只需要在代码中使用SLF4J的API，而不需要关系具体的实现，是不是与JDBC很像。

## 快速入门
1. 在类路径添加依赖：`slf4j-api-xx.jar`
2. 编写下面示例代码：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HelloWorld {
  public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger(HelloWorld.class);
    logger.info("Hello World");
  }
}
```
运行此代码，控制台会打印以下警告信息：

```bash
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```
打印此警告是因为在类路径上找不到 slf4j 绑定实现。此时我们将 `slf4j-simple-xxx.jar` 添加到类路径，就会打印出下面的正常日志信息：

```bash
0 [main] INFO HelloWorld - Hello World
```
## 典型的使用方式
**占位符**

```java
logger.debug("your name {}.",name );
logger.trace("hello world");
logger.info("hello world");
logger.warn("hello world");
logger.error("hello world");
```
`SELF4J2.0`之后引入了`fluent API`，并且向后兼容，这意味着你不需要修改旧版本的API。

`fluent API` 想法是使用 `LoggingEventBuilder `逐个构建日志事件，并在事件完全构建后进行日志记录。 atTrace()、atDebug()、atInfo()、atWarn() 和 atError() 方法都是 `org.slf4j.Logger `接口中的新方法，它们返回 `LoggingEventBuilder` 的实例。 对于禁用的日志级别，返回的 `LoggingEventBuilder `实例什么都不做，从而保留了传统日志接口的纳秒级性能。下面是示例：

```java
logger.atInfo().log("Hello world.");
// 等同于 logger.info("Hello world.");
```
以下日志语句在其输出中是等效的（对于默认实现）：

```java
int newT = 15;
int oldT = 16;
logger.debug("set to {}. Old was {}.", newT, oldT);
logger.atDebug().addArgument(newT).addArgument(oldT).log("set to {}. Old was {}.");
logger.atDebug().log("set to {}. Old was {}.", newT, oldT);
logger.atDebug().addArgument(newT).log("set to {}. Old was {}.", oldT);
logger.atDebug().addArgument(() -> t16()).log(msg, "set to {}. Old  was {}.", oldT);
```
API还支持k-v形式：

```java
int newT = 15;
int oldT = 16;
logger.debug("oldT={} newT={} Temperature changed.", newT, oldT);
logger.atDebug().addKeyValue("oldT", oldT).addKeyValue("newT", newT).log("Temperature changed."); 
```
## 日志绑定
![image](xCIE7Y7vEIfK_mUfcCBr5A4IA2wItTe3Qfp25Vf3sQQ.png)

如前所述，SLF4J 支持各种日志框架。 SLF4J 发行版附带了几个称为“SLF4J 绑定”的 jar 文件，每个绑定对应一个受支持的框架。

* *slf4j-log4j12-1.7.36.jar：*log4j 1.2 版的绑定。 鉴于 log4j 1.x 已在 2015 年和 2022 年宣布 EOL，从 SLF4J 1.7.35 开始，slf4j-log4j 模块在构建时自动重定向到 slf4j-reload4j 模块。 假设您希望继续使用 log4j 1.x 框架，我们强烈建议您改用 slf4j-reload4j。* *
* *slf4j-reload4j-1.7.36.jar：*Reload4j 是 log4j 版本 1.2.7 的直接替代品。 您还需要将 reload4j.jar 放在您的类路径上。
* *slf4j-jdk14-1.7.36.jar：*java.util.logging
* *slf4j-nop-1.7.36.jar： NOP 的绑定，默默地丢弃所有日志记录。*
* *slf4j-simple-1.7.36.jar： *简单实现，它将所有事件输出到 System.err。 只打印级别 INFO 和更高级别的消息。 此绑定在小型应用程序的上下文中可能很有用。
* *slf4j-jcl-1.7.36.jar：*Jakarta Commons Logging 的绑定/提供者。 此绑定会将所有 SLF4J 日志记录委托给 JCL。
* *logback-classic-1.2.10.jar (requires logback-core-1.2.10.jar)：* SLF4J 的原生实现： logback。 Logback 的 ch.qos.logback.classic.Logger 类是 SLF4J 的 org.slf4j.Logger 接口的直接实现。

要切换日志框架，只需替换类路径上的 slf4j 绑定。 例如，要从 java.util.logging 切换到 log4j，只需将 slf4j-jdk14-1.7.36.jar 替换为 slf4j-log4j12-1.7.36.jar。

SLF4J 不依赖任何特殊的类加载器机制。 事实上，每个 SLF4J 绑定在编译时都是硬连线的，以使用一个且只有一个特定的日志框架。 例如，slf4j-log4j12-1.7.36.jar 绑定在编译时绑定为使用 log4j。 在您的代码中，除了 slf4j-api-1.7.36.jar 之外，您只需将一个且只有一个您选择的绑定拖放到适当的类路径位置。 不要在类路径上放置多个绑定。

从 2.0.0 版开始，SLF4J 绑定被称为提供者。 尽管如此，总体思路还是一样的。 SLF4J API 版本 2.0.0 依赖 ServiceLoader 机制来查找其日志记录后端。

## 日志桥接
通常，您依赖的某些组件可能依赖于 SLF4J 以外的日志记录 API。为了处理这种情况，SLF4J 附带了几个桥接模块，这些模块将调用重定向到 log4j、JCL 和 java.util.logging API，使其表现得像是对 SLF4J API 进行的调用一样。下图说明了这个想法。

![image](ksLsrComdIIuTr90fTkVcWTN7hBsRT_JN2mHNrkIDk4.png)

请注意，对于您控制的源代码，您应该使用[slf4j-migrator](https://www.slf4j.org/migrator.html)。

**jul-to-slf4j.jar 和 slf4j-jdk14.jar 不能同时存在**

slf4j-jdk14.jar 的存在，即 SLF4J 的 jul 绑定，将强制 SLF4J 调用委托给 jul。 另一方面，jul-to-slf4j.jar 的存在，加上 SLF4JBridgeHandler 的安装，通过调用“SLF4JBridgeHandler.install()”将 jul 日志路由到 SLF4J。 因此，如果两个 jar 同时存在（并且安装了 SLF4JBridgeHandler），slf4j 调用将被委托给 jul，而 jul 记录将被路由到 SLF4J，从而导致无限循环。

## SLF4J 迁移器
SLF4J 迁移器是一个小型 Java 工具，用于将 Java 源文件从 Jakarta Commons Logging (JCL) API 迁移到 SLF4J。 它还可以从 log4j API 迁移到 SLF4J，或从 java.util.logging API 迁移到 SLF4J。



SLF4J 迁移器由一个 jar 文件组成，可以作为独立的 java 应用程序启动。 这是命令：

```bash
java -jar slf4j-migrator-2.0.0-alpha7.jar
```
启动应用程序后，应出现类似于以下内容的窗口:

![image](1ZHoft9SGID9g028uPDvf9MUG51V1i6KlKGm9VnLGWE.gif)

SLF4J 迁移器旨在作为一个简单的工具来帮助您将使用 JCL、log4j 或 JUL 的项目源迁移到 SLF4J。 它只能执行基本的转换步骤。 本质上，它将替换适当的导入行和记录器声明。

迁移之前：

```java
package some.package;
      
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
      
public MyClass {    
            
  Log logger = LogFactory.getLog(MyClass.class);
      
  public void someMethod() { 
    logger.info("Hello world");
  }
}
```
迁移之后：

```java
package some.package;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public MyClass {    

  Logger logger = LoggerFactory.getLogger(MyClass.class);

  public void someMethod() { 
    logger.info("Hello world");
  }
}
```
# logback
logback分为三个模块:

* logback-core: 为其他两个模块奠定了基础
* logback-classic: 原生实现了[SLF4J API](http://www.slf4j.org/)
* logback-access: 与 Servlet 容器集成以提供HTTP 访问日志功能

Logback 建立在三个主要类之上:

* Logger: logback-classic 模块的一部分
* Appender: logback-core模块的一部分
* Layout / Encoder: logback-core模块的一部分 ,Encoder 负责将事件转换为字节数组，并将该字节数组写入 OutputStream。  在以前的版本中，大多数Appender依赖于Layout 将事件转换为字符串并使用 java.io.Writer 将其写出。现在，只需要在 FileAppender 使用 Encoder 即可。

这三种类型的组件协同工作，使开发人员能够根据消息类型和级别记录消息，并在运行时控制这些消息的格式和存储位置。



## Logger
每个logger都会绑定到一个`LoggerContext` 。`LoggerContext` 制造logger并将他们按照树状排列。

记录器是命名实体。它们的名称区分大小写，并遵循分层命名规则。例如名为`com.foo` logger 是 名为 `com.foo.Bar` logger的父级。

root logger位于层次结构的顶部，它的特殊之处在于它从一开始就是每个层次结构的一部分。像每个logger一样，它可以通过名称被检索：

```java
Logger rootLogger = LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME);
```
为了确保所有logger最终都可以继承一个级别，根logger始终具有分配的级别。默认情况下，此级别为 DEBUG。

下面是几个级别继承的示例

| Logger name | Assigned level | Effective level |
| ----------- | -------------- | --------------- |
| root        | DEBUG          | DEBUG           |
| X           | none           | DEBUG           |
| X.Y         | none           | DEBUG           |
| X.Y.Z       | none           | DEBUG           |



| Logger name | Assigned level | Effective level |
| ----------- | -------------- | --------------- |
| root        | ERROR          | ERROR           |
| X           | INFO           | INFO            |
| X.Y         | DEBUG          | DEBUG           |
| X.Y.Z       | WARN           | WARN            |



| Logger name | Assigned level | Effective level |
| ----------- | -------------- | --------------- |
| root        | DEBUG          | DEBUG           |
| X           | INFO           | INFO            |
| X.Y         | none           | INFO            |
| X.Y.Z       | ERROR          | ERROR           |



| Logger name | Assigned level | Effective level |
| ----------- | -------------- | --------------- |
| root        | DEBUG          | DEBUG           |
| X           | INFO           | INFO            |
| X.Y         | none           | INFO            |
| X.Y.Z       | none           | INFO            |



根据定义，打印方法决定了日志请求的级别。 例如，如果 L 是记录器实例，则语句 L.info("..") 是级别 INFO 的记录语句。



如果日志请求的级别高于或等于其记录器的有效级别，则称该日志请求已启用。 否则，该请求被称为被禁用。级别排序如下：

`TRACE < DEBUG < INFO <  WARN < ERROR`



## Appender
Logback 允许将日志记录请求打印到多个目的地。在 logback 中，输出目的地称为 appender。目前，存在控制台、文件、远程套接字服务器、MySQL、PostgreSQL、Oracle 和其他数据库、JMS 和远程 UNIX Syslog 守护程序的附加程序。



### 控制台
```xml
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n</pattern>
    </encoder>
  </appender>
```
### FileAppender
FileAppender 是 OutputStreamAppender 的子类，将日志事件附加到文件中。 目标文件由 File 选项指定。 如果文件已经存在，则根据 append 属性的值将其追加或截断。

```xml
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>testFile.log</file>
    <append>true</append>
    <immediateFlush>true</immediateFlush>
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
    </encoder>
  </appender>
```
* append: 如果为 true，则将事件附加到现有文件的末尾。否则任何现有文件都将被截断。append选项默认设置为 true 。
* immediateFlush: 默认情况下，每个日志事件都会立即刷新到底层输出流。 这种默认方法更安全，因为如果您的应用程序在没有正确关闭`appender `的情况下退出，日志事件不会丢失。 但是，为了显着提高日志记录吞吐量，您可能需要将 immediateFlush 属性设置为 false。

在应用程序开发阶段或Job应用程序的情况下，例如 批处理应用程序，最好在每次新应用程序启动时创建一个新日志文件。 在 <timestamp> 元素的帮助下，这很容易做到：

```xml
  <timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss"/>

  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>log-${bySecond}.txt</file>
    <encoder>
      <pattern>%logger{35} - %msg%n</pattern>
    </encoder>
  </appender>
```
### RollingFileAppender
RollingFileAppender 扩展了 FileAppender 具有翻转日志文件的能力。 例如，RollingFileAppender 可以记录到一个名为 log.txt 的文件，一旦满足某个条件，就可以将其记录目标更改为另一个文件。

有两个重要的子组件与 RollingFileAppender 交互

* RollingPolicy：负责执行翻转所需的操作。 
* TriggeringPolicy：确定是否以及何时发生翻转。 

RollingFileAppender 必须同时设置 RollingPolicy 和 TriggeringPolicy。 但是，如果它的 RollingPolicy 也实现了 TriggeringPolicy 接口，那么只需要显式指定前者即可。

#### TimeBasedRollingPolicy
TimeBasedRollingPolicy 可能是最流行的滚动策略。 它定义了基于时间的翻转策略，例如按天或按月。 TimeBasedRollingPolicy 承担翻转以及触发所述翻转的责任。 实际上，TimeBasedTriggeringPolicy 实现了 RollingPolicy 和 TriggeringPolicy 接口。

```xml
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logFile.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>logFile.%d{yyyy-MM-dd}.log</fileNamePattern>
      <maxHistory>30</maxHistory>
      <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
    </encoder>
  </appender> 
```
* fileNamePattern: 必需的，定义滚动（归档）日志文件的名称。 它的值应该由文件名加上一个适当放置的 %d 转换说明符组成。 %d 转换说明符可以包含由 java.text.SimpleDateFormat 类指定的日期和时间模式。 如果省略日期和时间模式，则假定默认模式 yyyy-MM-dd。 翻转周期是从 fileNamePattern 的值推断出来的。
* **maxHistory: **可选, 控制要保留的存档文件的最大数量，异步删除旧文件。 例如，如果您指定每月翻转，并将 maxHistory 设置为 6，则将保留 6 个月的存档文件，而 6 个月之前的文件将被删除。 请注意，由于旧的归档日志文件已被删除，因此为归档日志文件而创建的任何文件夹都将被适当地删除。
* totalSizeCap: 可选的 , 控制所有存档文件的总大小。 当超过总大小上限时，最旧的档案将被异步删除。 totalSizeCap 属性也需要设置 maxHistory 属性。 此外，始终首先应用“最大历史记录”限制，然后应用“总大小上限”限制。

#### SizeAndTimeBasedRollingPolicy
有时您可能希望基本上按日期归档文件，但同时限制每个日志文件的大小。为了满足这个要求，logback 附带了 SizeAndTimeBasedRollingPolicy。

这是一个示例配置文件，展示了基于时间和大小的日志文件归档。

```xml
  <appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>mylog.txt</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>mylog-%d{yyyy-MM-dd}.%i.txt</fileNamePattern>
       <maxFileSize>100MB</maxFileSize>    
       <maxHistory>60</maxHistory>
       <totalSizeCap>20GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>%msg%n</pattern>
    </encoder>
  </appender>
```
#### FixedWindowRollingPolicy
翻转时，FixedWindowRollingPolicy 根据固定窗口算法重命名文件，如下所述。

fileNamePattern 选项表示归档（翻转）日志文件的文件名模式。 此选项是必需的，并且必须在模式中的某处包含整数标记 %i。

```xml
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>test.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
      <fileNamePattern>tests.%i.log.zip</fileNamePattern>
      <minIndex>1</minIndex>
      <maxIndex>3</maxIndex>
    </rollingPolicy>

    <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
      <maxFileSize>5MB</maxFileSize>
    </triggeringPolicy>
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
    </encoder>
  </appender>
```
### `additivity `
`addAppender` 方法将 `appender `添加到给定的记录器。给定记录器的每个启用的记录请求都将转发到该记录器中的所有`appender`。换句话说，`appender `是从 logger 层次结构中继承的。例如，如果将控制台`appender`添加到根记录器，则所有启用的日志记录请求至少将打印在控制台上。如果另外一个文件`appender`被添加到一个记录器中，比如L ，那么为L和L的孩子启用的记录请求将打印在一个文件上 和 控制台上。可以通过将记录器的 `additivity `标志设置为 false 来覆盖此默认行为，以便不再累加`appender`。

下表是`additivity ` 示例

| Logger Name     | Attached Appenders | Additivity Flag | Output Targets         |
| --------------- | ------------------ | --------------- | ---------------------- |
| root            | A1                 | not applicable  | A1                     |
| x               | A-x1, A-x2         | true            | A1, A-x1, A-x2         |
| x.y             | none               | true            | A1, A-x1, A-x2         |
| x.y.z           | A-xyz1             | true            | A1, A-x1, A-x2, A-xyz1 |
| security        | A-sec              | false           | A-sec                  |
| security.access | none               | true            | A-sec                  |

通常，用户不仅希望自定义输出目标，还希望自定义输出格式。 这是通过将Layout与Appenders相关联来实现的。 Layout负责根据用户的意愿格式化日志记录请求，而Appenders负责将格式化的输出发送到其目的地。 PatternLayout 是标准 logback 分发的一部分，允许用户根据类似于 C 语言 printf 函数的转换模式指定输出格式。例如，具有转换模式 "%-4relative \[%thread\] %-5level %logger{32} - %msg%n" 的 PatternLayout 将输出类似于：

```Plain Text
176  [main] DEBUG manual.architecture.HelloWorld2 - Hello world.
```


## 运行过程
![image](iGNzsObBRzO-cHdTveSjYOGiykLXuUxA5YnTNFudpOE.png)



现在让我们分析当用户调用名为com.wombat的记录器的 `info()`方法时 logback 所采取的步骤：

#### 1.获取过滤器链决策
如果存在，则调用 TurboFilter 链。 Turbo 过滤器可以设置上下文范围的阈值，或者根据与每个日志记录请求相关联的标记、级别、记录器、消息或 Throwable 等信息过滤掉某些事件。 如果过滤器链的回复是 FilterReply.DENY，则记录请求被丢弃。 如果是FilterReply.NEUTRAL，那么我们继续下一步，即步骤2。如果回复是FilterReply.ACCEPT，我们跳过下一步，直接跳到步骤3。

#### 2.应用[基本选择规则](https://logback.qos.ch/manual/architecture.html#basic_selection)
这一步，logback会比较logger的有效级别和请求的级别。如果根据此测试禁用了日志记录请求，则 logback 将丢弃该请求而无需进一步处理。否则，它进入下一步。

#### 3.创建一个`LoggingEvent`对象
如果请求通过了之前的过滤器，logback 将创建一个`ch.qos.logback.classic.LoggingEvent`对象，其中包含请求的所有相关参数，例如请求的记录器、请求级别、消息本身、可能与请求一起传递的异常、当前时间、当前线程、有关发出日志记录请求的类的各种数据以及`MDC`. 请注意，其中一些字段是延迟初始化的，即仅在实际需要时才初始化。`MDC`用于用额外的上下文信息装饰日志记录请求。MDC 将在[后续章节](https://logback.qos.ch/manual/mdc.html)中讨论。

#### 4.调用appender
创建`LoggingEvent`对象后，logback 会调用`doAppend()`所有适用的 appender 的方法，即继承自 logger 上下文的 appender。

logback 发行版附带的所有 appender 都扩展了 在同步块`AppenderBase`中实现方法的抽象类， `doAppend`以确保线程安全。如果存在任何此类过滤器，该`doAppend()`方法 还调用附加到`appender`的自定义过滤器。`AppenderBase`可以动态附加到任何 appender 的自定义过滤器在[单独的章节](https://logback.qos.ch/manual/filters.html)中介绍。

#### 5\. 格式化输出
被调用的`appender`负责格式化日志事件。但是，一些（但不是全部）`appender`将格式化日志事件的任务委托给`Layout`。`Layout`格式化`LoggingEvent`实例并将结果作为字符串返回。请注意，某些附加程序（例如 ） `SocketAppender`不会将日志记录事件转换为字符串，而是将其序列化。因此，它们没有也不需要布局。

#### 6\. 发出`LoggingEvent`
在日志事件完全格式化后，每个`appender`将其发送到其目的地。



## 配置
Logback 可以通过编程方式进行配置，也可以使用以 XML 或 Groovy 格式表示的配置脚本进行配置。

[https://logback.qos.ch/translator/](https://logback.qos.ch/translator/) ：这个网址可以将log4j的配置文件转成logback。

logback初始化的流程：

1. [Logback 尝试在 classpath](https://logback.qos.ch/faq.html#configFileLocation)中找到一个名为 *logback-test.xml* 的文件。
2. 如果没有找到这样的文件，它会检查 [类路径中的](https://logback.qos.ch/faq.html#configFileLocation)*logback.xml* [文件](https://logback.qos.ch/faq.html#configFileLocation).
3. 如果没有找到这样的文件，则 [使用服务提供者加载工具](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html)（在 JDK 1.6 中引入） 通过查找文件 META-INF\\services\\ch.qos.logback.classic.spi.Configurator来解析接口 。其内容应指定所需 实现的完全限定类名： `com.qos.logback.classic.spi.Configurator`
4. 如果以上都没有成功，logback 会自动配置自己：`BasicConfigurator` 将日志输出到控制台。



配置文件主要有下面几个元素组成：

![image](BFhLJpsXKa-Ksq7TFZ0axkq4vrkASwu2rXgpFOzBx_E.bin)
![image](mDW7_COdeMXOQ25OUsr0B8WEhw9_oO6AEWaWZSm84sU.png)



### 调试
```xml
<configuration debug="true">

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <!‐- encoders are  by default assigned the type
         ch.qos.logback.classic.encoder.PatternLayoutEncoder ‐‐>
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```


### 定义变量
```xml
<configuration>

  <property name="USER_HOME" value="/home/sebastien" />

  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>${USER_HOME}/myApp.log</file>
    <encoder>
      <pattern>%msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="FILE" />
  </root>
</configuration>
```
变量也可以通过命令行传入，例如：

```bash
java -DUSER_HOME="/home/sebastien" MyApp2
```
变量也可以存在配置文件中

```xml
<configuration>

  <property resource="resource1.properties" />

  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
     <file>${USER_HOME}/myApp.log</file>
     <encoder>
       <pattern>%msg%n</pattern>
     </encoder>
   </appender>

   <root level="debug">
     <appender-ref ref="FILE" />
   </root>
</configuration>
```
### 条件处理
```xml
<configuration debug="true">

  <if condition='property("HOSTNAME").contains("torino")'>
    <then>
      <appender name="CON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
          <pattern>%d %-5level %logger{35} - %msg %n</pattern>
        </encoder>
      </appender>
      <root>
        <appender-ref ref="CON" />
      </root>
    </then>
  </if>

  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>${randomOutputDir}/conditional.log</file>
    <encoder>
      <pattern>%d %-5level %logger{35} - %msg %n</pattern>
   </encoder>
  </appender>

  <root level="ERROR">
     <appender-ref ref="FILE" />
  </root>
</configuration>
```
