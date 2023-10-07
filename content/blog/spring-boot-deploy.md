---
title: spring-boot-deploy
tags:
  - java
categories:
  - 技术
date: 2022-12-02 13:00:20
---

# Spring Boot应用部署
# 传统服务器部署
除了使用 `java -jar` 命令运行程序之外，你还可以将jar包制作成可执行的，这样你只需要使用 `./xx.jar` 的方式就可以启动程序。

完全可执行的 jar 文件通过在文件前面嵌入一个额外的脚本来工作。 包含 jar 的目录用作应用程序的工作目录。

可以通过构建工具制作完全可执行jar：

**maven方式：**

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
    </configuration>
</plugin>
```
**gradle方式：**

```groovy
tasks.named('bootJar') {
    launchScript()
}
```
## 安装成Systemd服务
我们可以将可执行jar与系统服务结合使用，例如systemd。

假设您在`/var/myapp`中安装了 Spring Boot 应用程序，要将 Spring Boot 应用程序安装为`systemd`服务，请创建一个名为`myapp.service`的脚本并将其放置在`/etc/systemd/system`目录中：

```bash
[Unit] 
Description=myapp 
After=syslog.target 

[Service] 
User=myapp 
ExecStart=/var/myapp/myapp.jar 
SuccessExitStatus=143 

[Install] 
WantedBy=multi-user.target
```
请注意，与作为`init.d`服务运行时不同，运行应用程序的用户、PID 文件和控制台日志文件由`systemd`自身管理，因此必须使用“service”脚本中的适当字段进行配置。

## 自定启动脚本
有两种方式可以更改启动脚本，一种是在写入之前，另一种是在启动之前。

### 写入之前
所谓写入之前，即在脚本写入到jar包的时候。你可以使用maven插件的`embeddedLaunchScriptProperties或` gradle的`launchScript`更改脚本的内容。

默认脚本支持以下属性替换：

| 参数                     | 描述                                                         | Gradle default                                               | Maven default                                                |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| mode                     | 脚本的模式                                                   | auto                                                         | auto                                                         |
| initInfoProvides         | 服务的名称                                                   | ${task.baseName}|${project.artifactId}                       |                                                              |
| initInfoRequiredStart    | 服务启动前依赖的其他服务                                     | $remote_fs $syslog $network|$remote_fs $syslog $network      |                                                              |
| initInfoRequiredStop     | 服务关闭前的，需要关闭的其他的服务                           | $remote_fs $syslog $network|$remote_fs $syslog $network      |                                                              |
| initInfoDefaultStart     | 服务的启动级别                                               | 2 3 4 5                                                      | 2 3 4 5                                                      |
| initInfoDefaultStop      | 服务的关闭级别                                               | 0 1 6                                                        | 0 1 6                                                        |
| initInfoShortDescription | 服务的简短描述                                               | Single-line version of `${project.description}` (falling back to `${task.baseName}`) | ${project.name}                                              |
| initInfoDescription      | 服务的详细描述                                               | `${project.description}` (falling back to `${task.baseName}`) | `${project.description}` (falling back to `${project.name}`) |
| initInfoChkconfig        | 检查启动状态                                                 | 2345 99 01                                                   | 2345 99 01                                                   |
| confFolder               | 配置文件的目录                                               | 包含jar包的目录                                              | 包含jar包的目录                                              |
| inlinedConfScript        | 对应在默认启动脚本中内联的文件脚本的引用。 这可用于在加载任何外部配置文件之前设置环境变量，例如 JAVA\_OPTS |                                                              |                                                              |
| logFolder                | LOG\_FOLDER 的默认值。 仅对 init.d 服务有效                  |                                                              |                                                              |
| logFilename              | LOG\_FILENAME 的默认值。 仅对 init.d 服务有效                |                                                              |                                                              |
| pidFolder                | PID\_FOLDER 的默认值。 仅对 init.d 服务有效                  |                                                              |                                                              |
| pidFilename              | PID\_FOLDER 中 PID 文件名称的默认值。 仅对 init.d 服务有效   |                                                              |                                                              |
| useStartStopDaemon       | 是否应该使用 start-stop-daemon 命令（当它可用时）来控制进程  | true                                                         | true                                                         |
| stopWaitTime             | STOP\_WAIT\_TIME 的默认值（以秒为单位）。 仅对 init.d 服务有效 | 60                                                           | 60                                                           |

### 启动之前
在脚本启动之前，可以通过修改环境变量的方式或配置文件的方式，影响启动脚本的运行。

默认脚本支持以下环境属性：

| 环境变量              | 描述                                                         |
| --------------------- | ------------------------------------------------------------ |
| MODE                  | 操作的“模式”。默认值取决于 jar 的构建方式，但通常是`auto`<br>（这意味着它会尝试通过检查它是否是名为`init.d` 的目录中的符号链接来猜测它是否是一个 init 脚本）。<br>如果您想在前台运行脚本，您可以将其显式设置为`service`以便`run stop|start|status|restart`命令 |
| RUN_AS_USER           | 将用于运行应用程序的用户。未设置时，将使用拥有 jar 文件的用户。 |
| USE_START_STOP_DAEMON | `start-stop-daemon`当命令可用时，是否应该使用该命令来控制进程。默认为`true`. |
| PID_FOLDER            | pid 文件夹名称（`/var/run`默认情况下）。                     |
| LOG_FOLDER            | 放置日志文件的文件夹的名称（默认`/var/log`）。               |
| CONF_FOLDER           | 从中读取 .conf 文件的文件夹的名称（默认情况下与 jar 文件相同的文件夹）。 |
| LOG_FILENAME          | `LOG_FOLDER`（`<appname>.log`默认情况下）中的日志文件的名称。 |
| APP_NAME              | 应用程序的名称。如果 jar 是从符号链接运行的，则脚本会猜测应用程序名称。如果它不是符号链接或者您想显式设置应用程序名称，这可能很有用。 |
| RUN_ARGS              | 要传递给程序（Spring Boot 应用程序）的参数。                 |
| JAVA_HOME             | `java`默认情况下使用 发现可执行文件的位置`PATH`，但如果在`$JAVA_HOME/bin/java`. |
| JAVA_OPTS             | 启动时传递给 JVM 的选项。                                    |
| JARFILE               | jar 文件的显式位置，以防脚本用于启动实际上未嵌入的 jar。     |
| DEBUG                 | 如果不为空，则`-x`在 shell 进程上设置标志，允许您查看脚本中的逻辑。 |
| STOP_WAIT_TIME        | 在强制关闭之前停止应用程序时等待的时间（`60`默认情况下）。   |

除了`JARFILE`和之外`APP_NAME`，上一节中列出的设置都可以通过`.conf`文件进行配置。该文件应位于 jar 文件旁边，并具有相同的名称但后缀为`.conf`. 例如，一个名为的 jar`/var/myapp/myapp.jar`使用名为 `/var/myapp/myapp.conf`的配置文件，如下例所示：

```bash
JAVA_OPTS=-Xmx1024M
LOG_FOLDER=/custom/log/folder
```


# Docker 部署
很容易将 Spring Boot fat jar 打包为 docker 镜像。 然而，在 docker 镜像中复制和运行 fat jar 有很多缺点。 在不解压的情况下运行 fat jar 总是有一定的开销，在容器化环境中这可能很明显。 另一个问题是，将应用程序的代码及其所有依赖项放在 Docker 映像中的同一层是次优的。 由于您可能更频繁地重新编译您的代码，而spring boot依赖却不变化，因此最好分开一些东西。 如果你把jar文件放在你的应用程序类之前的层，Docker通常只需要改变最底层，就可以从它的缓存中提取其他的。



如果您从容器运行应用程序，则可以使用可执行 jar，但分解它并以不同的方式运行它通常有很多好处。 某些 PaaS 实现也可能会选择在运行前解压缩。 例如，Cloud Foundry 就是这样运作的。 运行解压的一种方法是启动适当的启动器，如下所示：

```bash
$ jar -xf myapp.jar
$ java org.springframework.boot.loader.JarLauncher
```
这实际上在启动时（取决于 jar 的大小）比从未分解的存档中运行要快一些。 在运行时，您不应期望有任何差异。



解压缩 jar 文件后，您还可以通过使用其main方法，（ 比JarLauncher稍微慢点）。 例如：

```bash
$ jar -xf myapp.jar
$ java -cp BOOT-INF/classes:BOOT-INF/lib/* com.example.MyApplication
```
> main方式需要扫描类路径下的文件， JarLauncher直接读取classpath.idx 文件，所以耗时更少。



为了更容易创建优化的 Docker 镜像，Spring Boot 支持向 jar 中添加层索引文件。 它提供了层列表和应包含在其中的 jar 的部分。 索引中的层列表是根据应将层添加到 Docker/OCI 映像的顺序进行排序的。 开箱即用，支持以下层：

* `dependencies` (for regular released dependencies)
* `spring-boot-loader` (for everything under `org/springframework/boot/loader`)
* `snapshot-dependencies` (for snapshot dependencies)
* `application` (for application classes and resources)

`layers.idx `文件示例如下：

```bash
- "dependencies":
  - BOOT-INF/lib/library1.jar
  - BOOT-INF/lib/library2.jar
- "spring-boot-loader":
  - org/springframework/boot/loader/JarLauncher.class
  - org/springframework/boot/loader/jar/JarEntry.class
- "snapshot-dependencies":
  - BOOT-INF/lib/library3-SNAPSHOT.jar
- "application":
  - META-INF/MANIFEST.MF
  - BOOT-INF/classes/a/b/C.class
```
此分层旨在根据应用程序构建之间更改的可能性来分离代码。 库代码在构建之间不太可能发生变化，因此它被放置在自己的层中，以允许工具重新使用缓存中的层。 应用程序代码更有可能在构建之间发生变化，因此它被隔离在一个单独的层中。

Spring Boot 还支持在 layers.idx 的帮助下对 war 文件进行分层。

虽然只需 Dockerfile 中的几行代码就可以将 Spring Boot fat jar 转换为 docker 镜像，但我们将使用分层功能来创建优化的 docker 镜像。 当您创建一个包含层索引文件的 jar 时，spring-boot-jarmode-layertools jar 将作为依赖项添加到您的 jar 中。 使用类路径中的这个 jar，您可以在特殊模式下启动应用程序，该模式允许引导代码运行与您的应用程序完全不同的东西，例如，提取层的东西。

> layertools 模式不能与包含启动脚本的完全可执行的 Spring Boot 存档一起使用。 在构建旨在与 layertools 一起使用的 jar 文件时禁用启动脚本配置。

以下是使用 layertools jar 模式启动 jar 的方法：

```bash
$ java -Djarmode=layertools -jar my-app.jar
```
这将提供以下输出：

```bash
Usage:
  java -Djarmode=layertools -jar my-app.jar

Available commands:
  list     List layers from the jar that can be extracted
  extract  Extracts layers from the jar for image creation
  help     Help about any command
```
extract 命令可用于轻松地将应用程序拆分为要添加到 dockerfile 的层。 这是使用 jarmode 的 Dockerfile 示例:

```docker
FROM adoptopenjdk:11-jre-hotspot as builder
WORKDIR application
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM adoptopenjdk:11-jre-hotspot
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```
假设上面的 Dockerfile 在当前目录下，你的 docker 镜像可以用 `docker build . `构建，或者可选地指定你的应用程序 jar 的路径，如下例所示：

```bash
$ docker build --build-arg JAR_FILE=path/to/myapp.jar .
```
这是一个多阶段的 dockerfile。 构建器阶段提取稍后需要的目录。 每个 COPY 命令都与 jarmode 提取的层相关。



当然，不使用 jarmode 也可以编写 Dockerfile。 您可以使用 unzip 和 mv 的某种组合将内容移动到正确的层，但 jarmode 简化了这一点。



Dockerfiles 只是构建 docker 镜像的一种方式。 构建 docker 镜像的另一种方法是直接从您的 Maven 或 Gradle 插件中使用 buildpacks。 如果您曾经使用过 Cloud Foundry 或 Heroku 等应用程序平台，那么您可能使用过 buildpack。 Buildpacks 是平台的一部分，它将您的应用程序转换为平台可以实际运行的东西。 例如，Cloud Foundry 的 Java buildpack 会注意到您正在推送 .jar 文件并自动添加相关的 JRE。



使用 Cloud Native Buildpacks，您可以创建可以在任何地方运行的 Docker 兼容镜像。 Spring Boot 直接包含对 Maven 和 Gradle 的 buildpack 支持。 这意味着您只需键入一个命令，就可以快速将合理的映像放入本地运行的 Docker 守护程序中。



请参阅单独的插件文档，了解如何将 buildpacks 与[Maven](https://docs.spring.io/spring-boot/docs/2.6.6/maven-plugin/reference/htmlsingle/#build-image)和[Gradle](https://docs.spring.io/spring-boot/docs/2.6.6/gradle-plugin/reference/htmlsingle/#build-image)一起使用。



# k8s部署
Spring Boot 通过检测 `*_SERVICE_HOST` 和 `*_SERVICE_PORT` 环境变量，来确定当前环境是否是k8s。你可以设置 `spring.main.cloud-platform `来覆盖此行为

当 Kubernetes 删除一个应用程序实例时，关闭过程会同时涉及多个子系统：关闭钩子、注销服务、从负载均衡器中删除实例……因为这个关闭过程是并行发生的（并且由于分布式系统的性质）， 存在请求被路由到正在关闭的pod上。

您可以在 preStop 处理程序中配置睡眠执行，以避免将请求路由到已经开始关闭的 pod。 这种休眠应该足够长，以使新请求停止路由到 pod，并且其持续时间因部署而异。 preStop 处理程序可以使用 Pod 配置文件中的 PodSpec 进行配置，如下所示：

```yaml
spec:
  containers:
  - name: "example-container"
    image: "example-image"
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```
一旦 pre-stop 钩子完成，SIGTERM 将被发送到容器并开始正常关闭，允许任何剩余的正在进行的请求完成。

当 Kubernetes 向 pod 发送 SIGTERM 信号时，它会等待一个称为终止宽限期的指定时间（默认为 30 秒）。 如果容器在宽限期后仍在运行，则会向它们发送 SIGKILL 信号并强制删除。 如果 pod 的关闭时间超过 30 秒，这可能是因为您增加了 spring.lifecycle.timeout-per-shutdown-phase，请确保通过在 Pod YAML 中设置 terminateGracePeriodSeconds 选项来增加终止宽限期。

# OpenShift部署
TBD
