---
title: gradle-java
tags:
  - java
  - gradle
categories:
  - 中间件
---

# 插件的生命周期

Base插件提供了大多数构建通用的一些任务和约定，并为构建添加了一个结构，以提高它们运行方式的一致性。它最重要的贡献是一组生命周期任务，充当其他插件和具体任务的保护伞。

主要任务和声明周期:

* clean — Delete: 删除build目录及其子目录下的所有内容，即Project.getBuildDir（）项目属性指定的路径。
* check — lifecycle task：插件和构建作者应使用`check.dependsOn(*task*)`将验证任务（例如运行测试的任务）附加到此生命周期任务。
* assemble — lifecycle task：插件和构建作者应该将生成发行版和其他可消费工件的任务附加到此生命周期任务。例如，jar为Java库生成可消费的工件。使用`assemble.dependsOn(*task*)`将任务附加到此生命周期任务
* build — lifecycle task：依赖`check`, `assemble`。旨在构建一切，包括运行所有测试、生成生产工件和生成文档。您可能很少直接将具体任务附加到构建中，因为assemble和check通常更合适。
* buildConfiguration — task rule： 组装附加到命名配置的那些工件。例如，buildArchives将执行任务，将所有工件绑定到archives 配置。
* cleanTask — task rule： 删除任务的输出，例如cleanJar将删除Java插件的JAR任务生成的JAR文件。

base插件没有为依赖项添加配置，但它添加了以下配置：

* **default**: 消费者项目使用的回退配置。假设您的项目B依赖于项目A。Gradle使用一些内部逻辑来确定项目A的哪些工件和依赖项添加到项目B的指定配置中。如果没有其他因素适用-您不必担心这些因素是什么-那么Gradle会回到使用项目A的默认配置中的所有内容。新版本和插件不应使用默认配置！由于向后兼容的原因，它仍然存在。
* **archives**: 项目生产工件的标准配置。

base插件将base扩展添加到项目中。这允许在专用DSL块内配置以下属性:

```groovy
base {
    archivesName = "gradle"
    distsDirectory = layout.buildDirectory.dir('custom-dist')
    libsDirectory = layout.buildDirectory.dir('custom-libs')
}
```

* archivesName : 默认**$project.name**
* distsDirectory：默认**$buildDir/distributions** ：创建分发存档（即非JAR）的目录的默认名称。
* libsDirectory： 默认**$buildDir/libs**： 创建库存档（即JAR）的目录的默认名称。

该插件还为任何扩展AbstractArchiveTask的任务提供以下属性的默认值：

* **destinationDirectory**：对于非JAR归档文件，默认为distsDirectory；对于JAR及其派生文件，例如WAR，默认为libsDirectory。
* archiveVersion： 默认为$project.version或**unspecified**（如果项目没有版本）。
* archiveBaseName： 默认值为$archivesBaseName。







# 构建java项目

Gradle使用约定优于配置的方法来构建基于JVM的项目，该方法借鉴了Apache Maven的一些约定。特别是，它对源文件和资源使用相同的默认目录结构，并与Maven兼容的存储库一起工作。



## 入门项目

Java项目最简单的构建脚本 先从应用Java *Library* 插件开始，设置项目版本并选择要使用的Java工具链： 

**build.gradle:**

```groovy
plugins {
    id 'java-library'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(11)
    }
}

version = '1.2.1'
```

通过应用Java Library插件，您可以获得一整套功能：

* compileJava任务，编译src/main/java下的所有Java源文件
* compileTestJava任务，编译src/test/java下的所有Java源文件
* test 任务， 运行src/test/java下的测试用例
* 一个jar任务，将主要编译类（src/main/java下的类）和src/main/resources中的资源打包到一个名为<project>-<version>.jar的jar中
* 为主要类生成javadoc的javadoc任务

> 尽管示例中的属性是可选的，但我们建议您在项目中指定它们。工具链选项可防止使用不同Java版本构建的项目出现问题。版本字符串对于跟踪项目进度非常重要。默认情况下，项目版本也用于存档名称中。

Java Library插件还将上述任务集成到标准Base插件生命周期任务中：

* jar 绑定到 assemble 生命周期
* test 绑定到 check 生命周期



## 定义sourceSet

源代码集主要思想是源文件和资源按类型进行逻辑分组，例如应用程序代码、单元测试和集成测试。每个逻辑组通常都有自己的文件依赖项集、类路径等。值得注意的是，形成源集的文件不必位于同一目录中！

源集是一个强大的概念，它将编译的几个方面联系在一起：

* 源文件及其位置
* 编译类路径，包括任何必需的依赖项（通过Gradle配置）
* 编译的类文件所在的位置

您可以在这个图表中看到它们是如何相互关联的：

![java sourcesets compilation](gradle-java/java-sourcesets-compilation.png)

着色框表示源集本身的属性。除此之外，Java Library 插件会自动为您或插件定义的每个源集和几个依赖项配置创建一个编译任务（名为compileSourceSetJava）。

Java项目通常包括源文件以外的资源，例如属性文件，这些资源可能需要处理（例如，通过替换文件中的标记），并在最终JAR中打包。Java Library 插件通过为每个定义的源集自动创建一个专用任务来处理此问题，该任务称为processSourceSetResources（或main源集的processResources）。下图显示了源集如何适应此任务：

![java sourcesets process resources](gradle-java/java-sourcesets-process-resources.png)

如前所述，阴影框表示源集的属性，在本例中，源集包括资源文件的位置及其复制到的位置。

除了主源代码集之外，Java Library插件还定义了一个表示项目测试的测试源代码集。此源集由运行测试的测试任务使用。您可以在Java测试一章中了解有关此任务和相关主题的更多信息。

项目通常将此源集用于单元测试，但如果您愿意，也可以将其用于集成、验收和其他类型的测试。另一种方法是为每个其他测试类型定义一个新的源集，这通常是出于以下一个或两个原因：

* 为了美观性和可管理性，您希望将测试彼此分开
* 不同的测试类型需要不同的编译或运行时类路径或设置中的一些其他差异

### 自定义sourceSet的位置

假设您有一个遗留项目，它为生产代码使用src目录，为测试代码使用test。传统的目录结构不起作用，因此您需要告诉Gradle在哪里可以找到源文件。您可以通过源代码集配置来实现。

```groovy
sourceSets {
    main {
         java {
            srcDirs = ['src']
         }
    }

    test {
        java {
            srcDirs = ['test']
        }
    }
}
```

现在Gradle**只会**直接在src中搜索并测试相应的源代码。如果您不想重写约定，而只是想添加一个额外的源目录，也许其中包含一些您希望保持独立的第三方源代码，该怎么办？语法类似：

```groovy
sourceSets {
    main {
        java {
            srcDir 'thirdParty/src/main/java'
        }
    }
}
```

重要的是，我们在这里使用srcDir（）方法来附加目录路径，而设置srcDirs属性将替换任何现有值。这是Gradle中的一个常见约定：设置属性替换值，而相应的方法附加值。

### 更改编译配置

大多数编译器选项都可以通过相应的任务访问，例如compileJava和compileTestJava。这些任务属于JavaCompile类型，因此请阅读任务参考以获取最新和全面的选项列表。

例如，如果您希望为编译器使用单独的JVM进程，并防止编译失败导致生成失败，则可以使用以下配置：

```groovy
compileJava {
    options.incremental = true
    options.fork = true
    options.failOnError = false
}
```

### 更改java的版本

默认情况下，Gradle会将Java代码编译到运行Gradle的JVM的语言级别。通过使用Java工具链，您可以通过确保构建定义的给定Java版本用于编译、执行和文档来打破这一联系。然而，可以在任务级别重写一些编译器和执行选项。

从版本9开始，Java编译器可以被配置为为较旧的Java版本生成字节码，同时确保代码不使用任何较新版本的API。Gradle现在直接在CompileOptions上支持此[release](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.compile.CompileOptions.html#org.gradle.api.tasks.compile.CompileOptions:release)标志，用于Java编译:

```
compileJava {
    options.release = 7
}
```

此选项优先于下面描述的属性。

Java编译器的历史选项仍然可用：

* **sourceCompatibility**: 定义源文件应被视为哪种语言版本的Java。
* **targetCompatibility**: 定义代码应该运行的最小JVM版本，即它决定编译器生成的字节代码的版本。

这些选项可以为每个JavaCompile任务设置，也可以在所有编译任务的`java｛｝`扩展上设置，使用具有相同名称的属性。

然而，这些选项并不能防止使用在以后的Java版本中引入的API。

## 处理资源文件

许多Java项目使用源文件以外的资源，如图像、配置文件和本地化数据。有时，这些文件打包时不变，有时需要作为模板文件或以其他方式处理。无论哪种方式，Java库插件都会为每个源集添加一个特定的复制任务，以处理其相关资源的处理。

任务的名称遵循processSourceSetResources（或主源集的processResources）的约定，它将自动将src/[sourceSet]/resources中的任何文件复制到将包含在生产JAR中的目录中。该目标目录也将包含在测试的运行时类路径中。


您可以通过WriteProperties任务轻松创建Java属性文件。每次都会生成一个唯一的文件，即使使用相同的属性和值，因为它在注释中包含时间戳。Gradle的WriteProperties任务在所有属性都未更改的情况下生成完全相同的输出字节。这是通过对属性文件的生成方式进行一些调整来实现的：

* 没有时间戳注释添加到输出
* 行分隔符与系统无关，但可以显式配置（默认为“\n”）
* 属性按字母顺序排序



# 构建JVM组件

所有特定的JVM插件都构建在Java插件之上。上面的示例仅说明了这个基本插件提供的概念，并与所有JVM插件共享。
继续阅读以了解哪些插件适合哪个项目类型，因为建议选择特定的插件，而不是直接应用Java插件。

## 构建java库

库项目的独特之处在于它们被其他Java项目使用。这意味着与JAR文件一起发布的依赖元数据（通常以Maven POM的形式）至关重要。特别是，库的使用者应该能够区分两种不同类型的依赖关系：仅编译库所需的依赖关系和编译被消费者所需的依存关系。

Gradle通过Java Library插件管理这一区别，该插件除了本章介绍的实现之外，还引入了api配置。如果依赖项的类型出现在库的公共类的公共字段或方法中，则依赖项通过库的公共API公开，因此应添加到API配置中。否则，依赖项是一个内部实现细节，应该添加到*implementation*。

## 构建java应用

打包为JAR的Java应用程序不适合从命令行或桌面环境轻松启动。application插件通过创建一个包含生产JAR、其依赖项和类似Unix和Windows系统的启动脚本来启动应用。

## 构建java web应用

Java web应用程序可以以多种方式打包和部署，具体取决于您使用的技术。例如，您可以将Spring Boot与一个胖JAR或一个运行在Netty上的基于Reactive的系统一起使用。无论您使用什么技术，Gradle及其庞大的插件社区都将满足您的需求。然而，CoreGradle仅直接支持部署为WAR文件的传统基于Servlet的web应用程序。

该支持通过War插件提供，该插件自动应用Java插件并添加额外的打包步骤，该步骤执行以下操作：

* 将src/main/webapp中的静态资源复制到WAR的根目录中
* 将编译的生产类复制到WAR的WEB-INF/classes子目录中
* 将库依赖项复制到WAR的WEB-INF/lib子目录中

这是由war任务完成的，它有效地替换了jar任务（尽管该任务仍然存在），并附加到assemble生命周期任务。

## 构建JAVA [Platforms](https://docs.gradle.org/current/userguide/building_java_projects.html#sec:building_java_platform)

Java平台表示一组依赖性声明和约束，这些声明和约束形成了一个用于消费项目的内聚单元。该平台没有自己的来源和人工制品。它在Maven世界中映射到BOM。

该支持通过Java平台插件提供，该插件设置了不同的配置和发布组件。
