---
title: spring-boot-how-to
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:42:53
---

## 构建时替换属性

您可以使用构建工具中设定的属性来替换应用属性，而不是对项目的生成配置中指定的某些属性进行硬编码。这在Maven和Gradle都是可能的。

### maven

您可以通过使用资源过滤从Maven项目自动展开属性。如果使用spring-boot-starter-parent，则可以使用`@..@`占位符引用Maven的“project properties，如下例所示：

```properties
app.encoding=@project.build.sourceEncoding@
app.java.version=@java.version@
```

如果启用addResources标志，springboot:run目标可以将src/main/resources直接添加到类路径（用于热重新加载）。这样做可以避免资源筛选和此功能。相反，您可以使用exec:java目标或自定义插件的配置。有关详细信息，请参见插件使用页面。????



如果不使用starter父级，则需要在pom.xml的＜build/＞元素中包含以下元素：

```xml
<resources>
    <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering>
    </resource>
</resources>
```

您还需要在＜plugins/＞中包含以下元素：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>2.7</version>
    <configuration>
        <delimiters>
            <delimiter>@</delimiter>
        </delimiters>
        <useDefaultDelimiters>false</useDefaultDelimiters>
    </configuration>
</plugin>
```

### Gradle

您可以通过配置Java插件的processResources任务来自动扩展Gradle项目中的属性，如下例所示：

```groovy
tasks.named('processResources') {
    expand(project.properties)
}
```

然后，可以使用占位符引用Gradle项目的属性，如以下示例所示：

```properties
app.name=${name}
app.description=${description}
```

Gradle的expand方法使用Groovy的SimpleTemplateEngine，它转换${..}标记。${..}样式与Spring自己的属性占位符机制冲突。要将Spring属性占位符与自动扩展一起使用，请按如下方式转义Spring属性占位符：\${..}。

## 生成build信息

Maven插件和Gradle插件都允许生成包含项目坐标、名称和版本的构建信息。插件还可以通过配置文件添加附加属性。当存在这样的文件时，Spring Boot会自动配置BuildProperties bean。

要使用Maven生成构建信息，请为执行添加build-info 目标，如下例所示：

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <version>2.7.5</version>
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

以下示例对Gradle执行相同操作：

```groovy
springBoot {
    buildInfo()
}
```

## 生成git信息

Maven和Gradle都允许生成git.properties文件，其中包含有关项目生成时git源代码存储库状态的信息。

对于Maven用户，spring-boot-starter-parent POM包括一个预配置的插件，用于生成 git.properties 文件。要使用它，请将以下声明添加到POM中：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>pl.project13.maven</groupId>
            <artifactId>git-commit-id-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

gradle 如下：

```xml
plugins {
    id "com.gorylenko.gradle-git-properties" version "2.3.2"
}
```

## 覆盖spring starter指定的版本

应用依赖项管理插件时，导入的spring-boot-dependencies bom来控制它管理的依赖项的版本。浏览Spring Boot参考中的[Dependency版本附录](https://docs.spring.io/spring-boot/docs/2.7.5/reference/htmlsingle/#dependency-versions-properties)，以获得这些属性的完整列表。

要自定义托管版本，请设置其相应的属性。例如：

```xml
ext['slf4j.version'] = '1.7.20'
```
