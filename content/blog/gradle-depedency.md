---
title: gradle 依赖解析
tags:
  - java
  - gradle
categories:
  - 中间件
---

# 依赖解析

Gradle项目声明的每个依赖项都适用于特定范围。例如，一些依赖项应该用于编译源代码，而其他依赖项只需要在运行时可用。Gradle在Configuration的帮助下表示依赖项的范围。每个Configuration都可以用唯一的名称标识。

下面是java 插件的一个示例：

![dependency management configurations](gradle-depedency/dependency-management-configurations.png)

## Configuration

配置可以扩展其他配置以形成继承层次结构。子配置拥有父配置声明的整个依赖项集。配置继承被Gradle核心插件（如Java插件）大量使用。例如，testImplementation配置扩展了 implementation 配置。

假设你想写一套烟雾测试。每个烟雾测试都会发出一个HTTP调用来验证web服务端点。作为底层测试框架，项目已经使用了JUnit。您可以定义一个名为smokeTest的新配置，该配置从testImplementation配置扩展，以重用现有的测试框架依赖项。

```groovy
configurations {
    smokeTest.extendsFrom testImplementation
}

dependencies {
    testImplementation 'junit:junit:4.13'
    smokeTest 'org.apache.httpcomponents:httpclient:4.5.5'
}
```

配置是Gradle中依赖关系解决的基本部分。在依赖解析的上下文中，区分消费者和生产者是很有用的。按照这些原则，配置至少具有3种不同的角色：

* 声明依赖项
* 作为消费者，解析文件的一组依赖关系
* 作为一个生产者，暴露工件及其依赖关系，供其他项目使用



例如，为了表示app应用程序依赖于lib库，至少需要一种配置：

```groovy
configurations {
    // declare a "configuration" named "someConfiguration"
    someConfiguration
}
dependencies {
    // add a project dependency to the "someConfiguration" configuration
    someConfiguration project(":lib")
}
```

配置可以通过从其他配置扩展来继承依赖关系。现在，请注意，上面的代码没有告诉我们任何有关此配置的预期消费者的信息。特别是，它没有告诉我们如何使用配置。假设lib是一个Java库：它可能会暴露不同的东西，例如它的API、实现或测试装置。根据我们正在执行的任务（根据lib的API编译、执行应用程序、编译测试等），可能需要更改我们如何解析app的依赖关系。为了解决这个问题，您通常会发现伴随配置，它们旨在明确地声明用法：

```groovy
configurations {
    // declare a configuration that is going to resolve the compile classpath of the application
    compileClasspath.extendsFrom(someConfiguration)

    // declare a configuration that is going to resolve the runtime classpath of the application
    runtimeClasspath.extendsFrom(someConfiguration)
}
```

此时，我们有3种不同的配置，具有不同的角色：

* someConfiguration声明应用程序的依赖关系。它只是一个可以保存依赖项列表的桶。
* compileClasspath和runtimeClasspath是要解析的配置：解析后，它们应该分别包含编译类路径和应用程序的运行时类路径。

这种区别由配置类型中的canBeResolved标志表示。可以解析的配置是我们可以计算依赖关系图的配置，因为它包含解析所需的所有信息. 也就是说，我们将计算一个依赖关系图，解析图中的组件，最终得到工件。canBeResolved设置为false的配置不会被解析。这样的配置只用于声明依赖项。原因是，根据用法（编译类路径、运行时类路径），它可以解析为不同的图。尝试解析canBeResolved设置为false的配置是错误的。在某种程度上，这类似于不应该实例化的抽象类（canBeResolved=false）和扩展抽象类的具体类（canBeResolved=true）。可解析配置将扩展至少一个不可解析配置（并且可以扩展多个）。

另一方面，在库项目端（生产者），我们也使用配置来表示可以使用的内容。例如，库可以公开API或运行时，我们可以将工件附加到其中一个、另一个或两者。通常，要针对lib进行编译，我们需要lib的API，但不需要它的运行时依赖项。因此，lib项目将公开一个apiElements配置，该配置面向寻找其API的用户。这种配置是可消费的，但并不意味着可解析。这通过配置的canBeConsumed标志表示：

```
configurations {
    // A configuration meant for consumers that need the API of this component
    exposedApi {
        // This configuration is an "outgoing" configuration, it's not meant to be resolved
        canBeResolved = false
        // As an outgoing configuration, explain that consumers may want to consume it
        canBeConsumed = true
    }
    // A configuration meant for consumers that need the implementation of this component
    exposedRuntime {
        canBeResolved = false
        canBeConsumed = true
    }
}
```

简而言之，配置的角色由canBeResolved和canBeConsumed标志组合决定：

| Configuration role        | can be resolved | can be consumed |
| ------------------------- | --------------- | --------------- |
| Bucket of dependencies    | false           | false           |
| Resolve for certain usage | true            | false           |
| Exposed to consumers      | false           | true            |
| Legacy, don’t use         | true            | true            |

为了向后兼容，这两个标志的默认值都为true，但作为插件作者，您应该始终为这些标志确定正确的值，否则可能会意外引入解析错误。





项目有时不依赖二进制存储库产品，例如JFrog Artifactory或Sonatype Nexus来托管和解决外部依赖关系。通常的做法是将这些依赖项托管在共享驱动器上，或者将它们与项目源代码一起检入版本控制。这些依赖关系称为文件依赖关系，原因是它们表示的文件没有附加任何元数据（如有关传递依赖关系、来源或作者的信息）。

![dependency management file dependencies](https://docs.gradle.org/current/userguide/img/dependency-management-file-dependencies.png)

```groovy
configurations {
    antContrib
    externalLibs
    deploymentTools
}

dependencies {
    runtimeOnly files('libs/a.jar', 'libs/b.jar')
    runtimeOnly fileTree('libs') { include '*.jar' }
    antContrib files('ant/antcontrib.jar')
    externalLibs files('libs/commons-lang.jar', 'libs/log4j.jar')
    deploymentTools(fileTree('tools') { include '*.exe' })
}
```

正如您在代码示例中看到的，每个依赖项都必须定义其在文件系统中的确切位置。创建文件引用的最主要方法是Project.files（java.lang.Object…), ProjectLayout.files（java.lang.Object…) 和Project.fileTree（java.lang.Object）或者，也可以以平面目录存储库的形式定义一个或多个文件依赖项的源目录。

> FileTree中文件的顺序不稳定，即使在一台计算机上也是如此。这意味着以这种构造为种子的依赖配置可能会产生具有不同顺序的解析结果，可能会影响使用结果作为输入的任务的可缓存性。建议尽可能使用更简单的文件。



软件项目通常将软件组件分解为模块，以提高可维护性并防止强耦合。模块可以定义彼此之间的依赖关系，以便在同一项目中重用代码。

![dependency management project dependencies](gradle-depedency/dependency-management-project-dependencies.png)

Gradle可以建模模块之间的依赖关系。这些依赖关系称为项目依赖关系，因为每个模块都由Gradle项目表示。

```
dependencies {
    implementation project(':shared')
}
```

在运行时，构建会自动确保以正确的顺序构建项目依赖项，并将其添加到类路径中进行编译。

自Gradle 7以来，Gradle为项目依赖项提供了一个实验性的类型安全API。与上述相同的示例现在可以改写为：

```
dependencies {
    implementation projects.utils
    implementation projects.api
}
```

项目访问器是从项目路径映射的。例如，如果项目路径是：commons:utils:some:lib，那么项目访问器将是projects.commons.utils.some.lib（这是projects.getCommons().getUtils().getSome().get-lib（）的缩写）。
带有kebab case（someLib）或snake case（some_lib）的项目名称将在accessors中转换为camel-case：projects.someLib。



## 流程

Gradle需要依赖关系图中包含的模块的元数据。需要这些信息：

* 当声明的版本是动态的时，确定模块的现有版本。
* 确定给定版本的模块依赖关系。

面对动态版本，Gradle需要确定具体的匹配版本：

* 每个存储库都会被检查，Gradle不会在第一个返回一些元数据时停止。定义多个时，将按添加顺序检查它们。
* 对于Maven存储库，Gradle将使用maven-metadata.xml（提供有关可用版本的信息）。
* 对于Ivy存储库，Gradle将使用目录列表。

此过程生成一个候选版本列表，然后将其与所表达的动态版本相匹配。注意Gradle缓存了版本信息，更多信息可以在控制动态版本缓存一节中找到。



给定一个所需的依赖项和一个版本，Gradle试图通过搜索依赖项指向的模块来解决依赖项。

* 按顺序检查每个存储库。
  * 根据存储库的类型，Gradle查找描述模块的元数据文件（.module、.pom或ivy.xml文件）或直接查找工件文件。
  * 具有模块元数据文件（.module、.pom或ivy.xml文件）的模块优于仅具有工件文件的模块。
  * 一旦存储库返回元数据结果，将忽略后面存储库。
* 如果找到依赖项的元数据，将检索并分析该元数据
  * 如果模块元数据是声明了父POM的POM文件，Gradle将递归地尝试解析POM的每个父模块。
* 然后从上述过程中选择的同一存储库中请求模块的所有工件。
* 然后，所有这些数据（包括存储库源和潜在未命中）都存储在依赖缓存中。



当Gradle无法从存储库中检索信息时，它将在构建期间禁用它，并使所有依赖项解析失败。最后一点对于再现性很重要。如果允许继续构建，而忽略错误的存储库，那么一旦存储库恢复联机，后续的构建可能会产生不同的结果。

Gradle将多次尝试连接到给定的存储库，然后将其禁用。如果连接失败，Gradle将对某些可能是暂时性的错误进行重试，从而增加每次重试之间等待的时间。

当由于永久错误或已达到最大重试次数而无法联系存储库时，就会出现黑名单。



### 依赖缓存

Gradle包含一个高度复杂的依赖缓存机制，该机制旨在最大限度地减少依赖解析中发出的远程请求数量，同时努力确保依赖解析的结果正确且可重复。

Gradle依存关系缓存由位于Gradle_USER_HOME/caches下的两种存储类型组成：

* 基于文件的下载工件存储，包括二进制文件（如jar）以及原始下载元数据（如POM文件和Ivy文件）。下载的工件的存储路径包括SHA1校验和，这意味着可以轻松缓存具有相同名称但不同内容的两个工件。
* 已解析模块元数据的二进制存储，包括解析动态版本、模块描述符和工件的结果。



Gradle 会跟踪访问依赖项缓存中的工件。 定期（最多每 24 小时）扫描缓存以查找超过 30 天未使用的项目。 然后删除过时的项目，以确保缓存不会无限增长。



# 版本声明

## 声明依赖版本

1. 声明具体的版本号，例如：1.3`, `1.3.0-beta3`, `1.0-20150201.131010-1
2. maven 风格的版本号范围。例如`[1.0,)`, `[1.1, 2.0)`, `(1.2, 1.5]` 。
   1. [\]表示包含边界， （）表示不包含。默认都是取范围内的最大版本。
   2. 符号`]`可以代替`（`表示排他下限，`[`代替`）`表示排他上限。例如]1.0、2.0[
   3. 上限排除充当前缀排除。这意味着[1.0，2.0]也将排除所有以2.0开头且小于2.0的版本。例如，2.0-dev1或2.0-SNAPSHOT等版本不再包含在范围内。

3. 前缀范围： `1.+`, `1.3.+ ，下载当前前缀下的最大可用版本`
4. latest-status： 例如 `latest.integration`, `latest.release`



基本的版本声明：


```groovy
dependencies {
    implementation('org.slf4j:slf4j-api:1.7.15')
}
```

然后，该版本被认为是必需的版本，这意味着它应该最低为1.7.15，但可以由引擎进行升级（乐观升级）。然而，有一种严格版本的速记符号，使用`!!`符号：

```groovy
dependencies {
    // short-hand notation with !!
    implementation('org.slf4j:slf4j-api:1.7.15!!')
    // is equivalent to
    implementation("org.slf4j:slf4j-api") {
        version {
           strictly '1.7.15'
        }
    }

    // or...
    implementation('org.slf4j:slf4j-api:[1.7, 1.8[!!1.7.25')
    // is equivalent to
    implementation('org.slf4j:slf4j-api') {
        version {
           strictly '[1.7, 1.8['
           prefer '1.7.25'
        }
    }
}
```

一个严格的版本不能被升级并覆盖任何源于此依赖关系的可传递依赖关系。建议对严格版本使用范围。



对于较大的项目，推荐的做法是声明没有版本的依赖项，并在版本声明中使用依赖项约束。其优点是依赖项约束允许您在一个地方管理所有依赖项的版本，包括可传递的依赖项。例如：

```groovy
dependencies {
    implementation 'org.springframework:spring-web'
}

dependencies {
    constraints {
        implementation 'org.springframework:spring-web:5.0.2.RELEASE'
    }
}
```

### 版本策略

声明版本的时候，可以指定以下策略：

* strictly：任何与此版本符号不匹配的版本都将被排除在外。这是最强的版本声明。在声明的依赖项上，strictly可以降级版本。在传递依赖项上时，如果无法选择此子句可接受的版本，则会导致依赖项解析失败。有关详细信息，请参阅重写依赖项版本。此术语支持动态版本。
* require：意味着所选版本不能低于所需接受的版本，但通过冲突解决可以更高，即使更高版本具有独占上限。这就是直接依赖关系的含义。这个术语支持动态版本。
* prefer：这是一个非常软的版本声明。只有在对模块的版本没有更强烈的非动态意见时，它才适用。此术语不支持动态版本。
* reject：声明模块不接受特定版本。如果唯一可选择的版本也被拒绝，这将导致依赖性解析失败。此术语支持动态版本。



```groovy
ependencies {
    implementation('org.slf4j:slf4j-api') {
        version {
            strictly '[1.7, 1.8['
            prefer '1.7.25'
        }
    }

    constraints {
        implementation('org.springframework:spring-core') {
            version {
                require '4.2.9.RELEASE'
                reject '4.3.16.RELEASE'
            }
        }
    }
}
```


Gradle 通过在依赖关系图中找到的最新版本来解决冲突,但有的时候我们需要旧的版本，该怎么办呢？

```groovy
    implementation 'org.apache.httpcomponents:httpclient:4.5.4'
    implementation('commons-codec:commons-codec:1.9')
```

根据gradle的依赖解析规则，commons-codec的版本会是1.10，并不是我们想要的1.9。可以下面的设置更改

```groovy
    implementation 'org.apache.httpcomponents:httpclient:4.5.4'
    implementation('commons-codec:commons-codec') {
        version {
            strictly '1.9'
        }
    }
```

> 这种方式是有风险的，因为强制的指定了版本，依赖此库的app无法覆盖传递依赖，造成无法提升版本的问题。





在许多情况下，您需要使用特定模块依赖项的最新版本，或一系列版本中的最新。这可能是开发过程中的一项要求，或者您正在开发一个旨在与一系列依赖关系版本一起使用的库。通过使用动态版本，您可以轻松地依赖这些不断变化的依赖关系。动态版本可以是版本范围（例如2.+），也可以是最新版本的占位符，例如latest.integration。

或者，您请求的模块可以随时间变化，即使是同一版本，也就是所谓的变化版本。这种类型的更改模块的一个例子是Maven SNAPSHOT模块，它总是指向最新发布的工件。换句话说，标准Maven快照是一个不断发展的模块，它是一个“不断变化的模块”。



> 使用动态版本和更改模块可能导致无法生产的构建。随着特定模块的新版本发布，其API可能会与源代码不兼容。小心使用此功能！



项目可能会采用更积极的方法来消费对模块的依赖。例如，您可能希望始终集成最新版本的依赖项，以便在任何给定时间使用尖端功能。动态版本允许解析给定模块的版本范围的最新版本或最新版本。

```groovy
plugins {
    id 'java-library'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework:spring-web:5.+'
}
```

构建扫描可以有效地可视化动态依赖关系版本及其各自的选定版本。

![dependency management dynamic dependency build scan](gradle-depedency/dependency-management-dynamic-dependency-build-scan.png)

默认情况下，Gradle缓存依赖项的动态版本24小时。在此时间框架内，Gradle不会尝试从声明的存储库中解析较新的版本。可以根据需要配置阈值，例如，如果您希望更早地解析新版本。



团队可能会决定在发布应用程序或库的新版本之前实现一系列功能。一种常见的策略是允许消费者尽早集成他们的工件的未完成版本，通常是发布一个具有所谓更改版本的模块。不断变化的版本表明该功能集仍在积极开发中，尚未发布稳定的版本以供全面使用。

在Maven存储库中，更改的版本通常称为快照版本。快照版本包含后缀-Snapshot。以下示例演示如何在Spring依赖项上声明快照版本。

```
plugins {
    id 'java-library'
}

repositories {
    mavenCentral()
    maven {
        url 'https://repo.spring.io/snapshot/'
    }
}

dependencies {
    implementation 'org.springframework:spring-web:5.0.3.BUILD-SNAPSHOT'
}
```

默认情况下，Gradle会在24小时内缓存依赖项的更改版本。在此时间框架内，Gradle不会尝试从声明的存储库中解析较新的版本。可以根据需要配置阈值，例如，如果要更早地解析新的快照版本。

Gradle足够灵活，可以将任何版本视为正在更改的版本，例如，如果您想为Ivy模块建模快照行为。您只需设置属性ExternalModuleDependency.setChanging（布尔值）设置为true。



可以使用命令行选项覆盖默认缓存模式。您还可以使用解析策略以编程方式更改构建中的缓存过期时间。

```
configurations.all {
    resolutionStrategy.cacheDynamicVersionsFor 10, 'minutes'
    resolutionStrategy.cacheChangingModulesFor 4, 'hours'

}
```





--offline命令行开关告诉Gradle始终使用缓存中的依赖模块，无论它们是否需要再次检查。在脱机运行时，Gradle不会尝试访问网络以执行依赖关系解析。如果依赖缓存中不存在所需的模块，则build执行将失败。



要刷新依赖项缓存中的所有依赖项，请在命令行上使用--refresh-dependencies选项。Gradle忽略已解析模块和工件的所有缓存条目。将对所有配置的存储库执行新的解析，重新计算动态版本、刷新模块并下载工件。然而，如果可能，Gradle将在再次下载之前检查先前下载的工件是否有效。这是通过将存储库中发布的SHA1值与现有下载工件的SHA1数值进行比较来完成的。



当多个版本与版本选择器匹配时，组件选择规则可能会影响应选择哪个组件实例。规则针对每个可用版本应用，并允许规则明确拒绝该版本。这允许Gradle忽略任何不满足规则设置的条件的组件实例。示例包括：

* 对于像1.+这样的动态版本，某些版本可能会被明确拒绝选择。
* 对于像1.4这样的静态版本，可能会基于额外的组件元数据（如Ivy分支属性）拒绝实例，从而允许使用后续存储库中的实例。

通过ComponentSelectionRules对象配置规则。将使用ComponentSelection对象作为参数调用配置的每个规则，该对象包含有关正在考虑的候选版本的信息。调用ComponentSelection.reject（java.lang.String）会导致给定的候选版本被显式拒绝，在这种情况下，选择器不会考虑候选版本。

以下示例显示了一个规则，该规则不允许模块的特定版本，但允许动态版本选择下一个最佳候选:

```groovy
configurations {
    rejectConfig {
        resolutionStrategy {
            componentSelection {
                // Accept the highest version matching the requested version that isn't '1.5'
                all { ComponentSelection selection ->
                    if (selection.candidate.group == 'org.sample' && selection.candidate.module == 'api' && selection.candidate.version == '1.5') {
                        selection.reject("version 1.5 is broken for 'org.sample:api'")
                    }
                }
            }
        }
    }
}

dependencies {
    rejectConfig "org.sample:api:1.+"
}
```

请注意，版本选择首先从最高版本开始应用。所选版本将是所有组件选择规则接受的第一个版本。如果没有规则明确拒绝某个版本，则认为该版本被接受。

类似地，规则可以针对特定模块。必须以group:module的形式指定模块。

```
configurations {
    targetConfig {
        resolutionStrategy {
            componentSelection {
                withModule("org.sample:api") { ComponentSelection selection ->
                    if (selection.candidate.version == "1.5") {
                        selection.reject("version 1.5 is broken for 'org.sample:api'")
                    }
                }
            }
        }
    }
}
```

组件选择规则还可以在选择版本时考虑组件元数据。可以考虑的其他元数据包括ComponentMetadata和IvyModuleDescriptor。请注意，这些额外信息可能并不总是可用的，因此应检查空值。

```groovy
configurations {
    metadataRulesConfig {
        resolutionStrategy {
            componentSelection {
                // Reject any versions with a status of 'experimental'
                all { ComponentSelection selection ->
                    if (selection.candidate.group == 'org.sample' && selection.metadata?.status == 'experimental') {
                        selection.reject("don't use experimental candidates from 'org.sample'")
                    }
                }
                // Accept the highest version with either a "release" branch or a status of 'milestone'
                withModule('org.sample:api') { ComponentSelection selection ->
                    if (selection.getDescriptor(IvyModuleDescriptor)?.branch != "release" && selection.metadata?.status != 'milestone') {
                        selection.reject("'org.sample:api' must be a release branch or have milestone status")
                    }
                }
            }
        }
    }
}
```



## 升级传递依赖的版本

组件可能有两种不同的依赖关系：

* 直接依赖关系。直接依赖性也称为第一级依赖性。例如，如果您的项目源代码需要Guava，则应将Guava声明为直接依赖项。
* 传递依赖是您的组件需要的依赖，但这只是因为另一个依赖需要它们。

依赖关系管理的问题是关于可传递的依赖关系，这是很常见的。开发人员通常通过添加直接依赖项来错误地解决可传递依赖项问题。为了避免这种情况，Gradle提供了依赖约束的概念。



依赖关系约束允许您定义生成脚本中声明的依赖关系和可传递依赖关系的版本或版本范围。它是表示应 应用于配置的所有依赖项的约束的首选方法。

当Gradle试图解析模块版本的依赖关系时，将考虑该模块的所有版本依赖关系声明、所有可传递依赖关系和所有依赖约束。选择符合所有条件的最高版本。如果找不到这样的版本，Gradle将失败，并显示一个错误，显示冲突的声明。

如果发生这种情况，您可以调整依赖关系或依赖约束声明，或者根据需要对传递依赖关系进行其他调整。与依赖关系声明类似，依赖关系约束声明由配置确定范围，因此可以为构建的部分选择性地定义。

```groovy
dependencies {
    implementation 'org.apache.httpcomponents:httpclient'
    constraints {
        implementation('org.apache.httpcomponents:httpclient:4.5.3') {
            because 'previous versions have a bug impacting this application'
        }
        implementation('commons-codec:commons-codec:1.11') {
            because 'version 1.9 pulled from httpclient has bugs affecting this application'
        }
    }
}
```

在本例中，依赖关系声明中省略了所有版本。而是在约束块中定义版本。commons-codec:1.11 仅在作为传递依赖项引入时考虑，因为其在项目中未定义为依赖项。否则，约束无效。依赖性约束也可以定义富版本约束，并支持严格版本来强制执行版本，即使它与传递依赖性定义的版本相冲突（例如，如果版本需要降级）。



Gradle通过选择依赖关系图中找到的最新版本来解决任何依赖关系版本冲突。某些项目可能需要偏离默认行为，并强制实施早期版本的依赖关系。

假设一个项目使用HttpClient库来执行HTTP调用。HttpClient将Commons Codec1.10版本作为可传递依赖项引入。然而，项目的生产源代码需要Commons Codec1.9中的API，而1.10版本中不再提供该API。依赖版本可以通过在构建脚本中将其声明为严格版本来强制执行：

```groovy
dependencies {
    implementation 'org.apache.httpcomponents:httpclient:4.5.4'
    implementation('commons-codec:commons-codec') {
        version {
            strictly '1.9'
        }
    }
}
```

然而，对于消费者来说，在图形解析过程中，严格的版本仍然是全局考虑的，如果消费者不同意，可能会触发错误。
例如，假设您的项目B严格依赖于C:1.0。现在，消费者A同时依赖于B和C:1.1。然后，这将触发解析错误，因为A表示它需要C:1.1，而B在其子图中严格需要1.0。这意味着，如果您在严格约束中选择了一个版本，则该版本将无法再升级，除非用户也对同一模块设置了严格的版本约束。

在上面的例子中，A必须说它严格依赖于1.1。

因此，一个好的做法是，如果您使用严格的版本，则应使用范围和在此范围内的首选版本来表示它们。例如，B可能会说，它严格依赖于[1.0，2.0]范围，而不是严格依赖于1.0。然后，如果消费者选择1.1（或范围内的任何其他版本），则构建将不再失败（约束已解决）。

如果出于某种原因，您不能使用严格的版本，则可以强制依赖项执行以下操作：

```
dependencies {
    implementation 'org.apache.httpcomponents:httpclient:4.5.4'
    implementation('commons-codec:commons-codec:1.9') {
        force = true
    }
}
```

如果项目需要特定版本的配置级别依赖项，则可以通过调用ResolutionStrategy.force（java.lang.Object[]）方法来实现。

```
configurations {
    compileClasspath {
        resolutionStrategy.force 'commons-codec:commons-codec:1.9'
    }
}

dependencies {
    implementation 'org.apache.httpcomponents:httpclient:4.5.4'
}
```



# 文件操作

## 复制文件

通过创建Gradle的内置Copy任务的实例，可以复制文件。此示例模拟将生成的报告复制到打包目录中：

```groovy
tasks.register('copyReport', Copy) {
    from layout.buildDirectory.file("reports/my-report.pdf")
    into layout.buildDirectory.dir("toArchive")
}
```

Project.file（java.lang.Object）方法用于创建与当前项目相关的文件或目录路径。您甚至可以直接使用路径而不使用file（）方法：

```groovy
tasks.register('copyReport2', Copy) {
    from "$buildDir/reports/my-report.pdf"
    into "$buildDir/toArchive"
}
```

通过为from（）提供多个参数，可以很容易地将前面的示例扩展到多个文件：

```groovy
tasks.register('copyReportsForArchiving', Copy) {
    from layout.buildDirectory.file("reports/my-report.pdf"), layout.projectDirectory.file("src/docs/manual.pdf")
    into layout.buildDirectory.dir("toArchive")
}
```

复制指定模式的文件：

```groovy
tasks.register('copyPdfReportsForArchiving', Copy) {
    from layout.buildDirectory.dir("reports")
    include "*.pdf"
    into layout.buildDirectory.dir("toArchive")
}
```

您可以使用Ant样式的glob模式（`**/*`）将文件包含在子目录中，如下示例所示：

```groovy
tasks.register('copyAllPdfReportsForArchiving', Copy) {
    from layout.buildDirectory.dir("reports")
    include "**/*.pdf"
    into layout.buildDirectory.dir("toArchive")
}
```

需要记住的一点是，像这样的深度过滤器会产生复制reports目录结构的副作用。如果只想复制没有目录结构的文件，则需要使用fileTree(*dir*) { *includes* }.files表达式。

## 创建归档文件

```
tasks.register('packageDistribution', Zip) {
    archiveFileName = "my-distribution.zip"
    destinationDirectory = layout.buildDirectory.dir('dist')

    from layout.buildDirectory.dir("toArchive")
}
```

每种类型的存档都有自己的任务类型，最常见的是Zip、Tar和Jar。它们都共享Copy的大部分配置选项，包括过滤和重命名。



## 更改文件内容

文件内容过滤允许您在复制文件时转换文件的内容。这可能涉及使用token替换的基本模板、删除文本行，或者使用成熟的模板引擎进行更复杂的过滤。

以下示例演示了几种形式的过滤，包括使用CopySpec.exexpand（java.util.Map）方法进行token替换，以及使用CopySpec.filter（java.lang.Class）和Ant过滤器进行token替换：

```groovy
import org.apache.tools.ant.filters.FixCrLfFilter
import org.apache.tools.ant.filters.ReplaceTokens

tasks.register('filter', Copy) {
    from 'src/main/webapp'
    into layout.buildDirectory.dir('explodedWar')
    // Substitute property tokens in files
    expand(copyright: '2009', version: '2.3.1')
    expand(project.properties)
    // Use some of the filters provided by Ant
    filter(FixCrLfFilter)
    filter(ReplaceTokens, tokens: [copyright: '2009', version: '2.3.1'])
    // Use a closure to filter each line
    filter { String line ->
        "[$line]"
    }
    // Use a closure to remove lines
    filter { String line ->
        line.startsWith('-') ? null : line
    }
    filteringCharset = 'UTF-8'
}
```

filter（）方法有两个变体，其行为不同：

* 一个采用FilterReader，设计用于Ant过滤器，如ReplaceTokens
* 一个是闭包或Transformer，它为源文件的每一行定义转换

当您将ReplaceTokens类与filter（）一起使用时，结果是一个模板引擎，它将@tokenName@（Ant样式的token）形式的token替换为您定义的值。

expand（）方法将源文件视为Groovy模板，用于计算和扩展${expression}形式的表达式。您可以传入属性名称和值，然后在源文件中展开。expand（）允许的不仅仅是基本的令牌替换，因为嵌入的表达式是完整的Groovy表达式。
