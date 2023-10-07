---
title: flyway
tags:
  - java
categories:
  - 技术
date: 2022-12-02 13:01:55
---

# Flyway
## 工作原理
1. 项目启动，应用程序完成数据库连接池的建立后，Flyway自动运行。
2. 初次使用时，Flyway会创建一个`flyway_schema_history`表，用于记录sql执行记录。
3. Flyway会扫描项目指定路径下(默认是`classpath:db/migration`)的所有sql脚本，与`flyway_schema_history`表脚本记录进行比对。**根据脚本的名称提取版本号来比对**
4. 如果脚本没有执行过，则执行脚本。如果脚本执行过，则比对文件是否发生变更，如果发生了变更，则抛出异常，终止迁移

## 在spring boot中使用
1. 初始化一个SpringBoot项目，引入MySQL数据库驱动依赖等，并且需要引入Flyway依赖：

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>6.1.0</version>
</dependency>
```


2. 添加Flyway配置：

```yaml
spring:
  # 数据库连接配置
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ssm-demo?characterEncoding=utf-8&useSSL=false&serverTimezone=GMT%2B8
    username: xxx
    password: xxx
  flyway:
    # 是否启用flyway
    enabled: true
    # 编码格式，默认UTF-8
    encoding: UTF-8
    # 迁移sql脚本文件存放路径，默认db/migration
    locations: classpath:db/migration
    # 迁移sql脚本文件名称的前缀，默认V
    sql-migration-prefix: V
    # 迁移sql脚本文件名称的分隔符，默认2个下划线__.前面的用作版本号，后面的用作描述信息
    sql-migration-separator: __
    # 迁移sql脚本文件名称的后缀
    sql-migration-suffixes: .sql
    # 迁移时是否进行校验，默认true
    validate-on-migrate: true
    # 当迁移发现数据库非空且存在没有元数据的表时，自动执行基准迁移，新建schema_version表
    baseline-on-migrate: 
```
3. 根据在配置文件的脚本存放路径的配置，在resource目录下建立文件夹`db/migration`。
4. 添加需要运行的sql脚本。sql脚本的命名规范为：V+版本号(版本号的数字间以”.“或”\_“分隔开)+双下划线(用来分隔版本号和描述)+文件描述+后缀名，例如：V20201100\_\_create\_user.sql。如图所示：

![image](XonVWry8sSqFeEu_jjkmj_OSwC07YCaCrMmyixXO8xw.png)

5. 启动项目。启动成功后，在数据库中可以看到已按照定义好的脚本，完成数据库变更，并在`flyway_schema_history`表插入了sql执行记录：

![image](bVJ78_N5cntefo3HAoI5lIkFwRtD_yz3DLRnYxCb_X0.png)



## 主要配置项
* flyway.baseline-on-migrate： 当迁移时发现目标schema非空，而且带有没有元数据的表时，是否自动执行基准迁移（创建元数据表，然后执行sql脚本），默认false.
* flyway.baseline-version开始执行基准迁移时对现有的schema的版本打标签，默认值为1.



* flyway.validate-on-migrate迁移时是否校验，默认为true. 校验机制检查本地迁移是否仍与数据库中已执行的迁移具有相同的校验和。主要防止已迁移的本地文件发生了变动，数据库却没有更新这种变化。这是一种预警机制。
* flyway.clean-on-validation-error当发现校验错误时是否自动调用clean，这是开发环境中的方便机制。默认false.  警告！ 不要在生产中启用！
* flyway.ignore-failed-future-migration当读取元数据表时是否忽略错误的迁移，默认false.
* flyway.out-of-order是否允许无序的迁移，默认false.



* flyway.enabled是否开启flywary，默认true.
* flyway.password：目标数据库的密码.
* flyway.url：迁移时使用的JDBC URL，如果没有指定的话，将使用配置的主数据源
* flyway.user：迁移数据库的用户名
* flyway.schemas设定需要flywary迁移的schema，大小写敏感，默认为连接默认的schema.
* flyway.tableflyway使用的元数据表名，默认为schema\_version


* flyway.placeholder-prefix设置每个placeholder的前缀，默认\${.
* flyway.placeholder-suffix设置每个placeholder的后缀，默认}.
* flyway.placeholder-replacementplaceholders是否要被替换，默认true.
* flyway.placeholders.\[placeholder name\]设置placeholder的value



* flyway.sql-migration-prefix迁移文件的前缀，默认为V.
* flyway.sql-migration-separator迁移脚本的文件名分隔符，默认`__`
* flyway.sql-migration-suffix迁移脚本的后缀，默认为.sql
* flyway.init-sqls当初始化好连接时要执行的SQL.
* flyway.locations迁移脚本的位置，默认db/migration.
* flyway.encoding设置迁移时的编码，默认UTF-8.



## 版本迁移的问题
当对flyway升级的时候，会出现兼容性问题，例如checksum不一致：

```Plain Text
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'flywayInitializer' defined in class path resource [org/springframework/boot/autoconfigure/flyway/FlywayAutoConfiguration$FlywayConfiguration.class]: Invocation of init method failed; nested exception is org.flywaydb.core.api.FlywayException: Validate failed: Migration checksum mismatch for migration version 1.0.0.01
-> Applied to database : 1062144176
-> Resolved locally : 1432425380
```
可以通过执行修复方法来解决。将已应用迁移的校验和、描述和类型与可用迁移的校验和、描述和类型重新对齐

```java
flyway.repair();
```
