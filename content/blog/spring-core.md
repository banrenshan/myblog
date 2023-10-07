---
title: spring容器
tags:
  - java
  - spring
categories:
  - java
  - spring
date: 2022-12-02 12:02:30
---

## 1.容器和 Bean 介绍
控制反转也叫依赖注入(DI)，这是一个过程。对象通过下面的方式，知道自己的依赖项：

* 构造函数参数
* 工厂方法创建对象时的参数
* 对象的setter方法参数

然后容器在创建bean时，注入这些依赖。在这个过程中，bean控制着自身的创建，通过类上的构造函数等机制搜索所需的依赖，因此称为控制反转。

`org.springframework.beans` 和 `org.springframework.context` 包是 Spring Framework 的 IoC 容器的基础。 `BeanFactory` 接口管理容器中的bean。 `ApplicationContext` 是 `BeanFactory` 的扩展子接口，扩展项如下：

* 更容易与 Spring 的 AOP 特性集成
* 消息资源处理（用于国际化）
* 事件发布
* 应用层特定上下文，例如用于 Web 应用程序的 `WebApplicationContext`。

简而言之，`BeanFactory` 提供了配置框架和基本功能，`ApplicationContext` 增加了更多企业特定的功能。 

> 在 Spring 中，构成应用程序主干并由 Spring IoC 容器管理的对象称为 bean。 bean 是由 Spring IoC 容器实例化、组装和管理的对象。 



## 2.容器概览

org.springframework.context.ApplicationContext接口表示Spring IoC容器，并负责实例化，配置和组装Bean。 容器通过读取配置元数据获取有关要实例化，配置和组装哪些对象的指令。 配置元数据以XML，Java批注或Java代码表示。 它使您能够表达组成应用程序的对象以及这些对象之间的丰富相互依赖关系。

Spring提供了ApplicationContext接口的几种实现。 在独立应用程序中，通常创建ClassPathXmlApplicationContext或FileSystemXmlApplicationContext的实例。 尽管XML是定义配置元数据的传统格式，但是您可以通过提供少量XML配置来声明性地启用对这些其他元数据格式的支持，从而指示容器将Java注释或代码用作元数据格式。

在大多数应用场景中，不需要显式用户代码即可实例化一个Spring IoC容器的一个或多个实例。 例如，在Web应用程序场景中，应用程序的web.xml文件中配置简单八行样板XML就足够了（请参阅Web应用程序的便捷ApplicationContext实例化）。 

下图显示了Spring的工作原理的高级视图。 您的应用程序类与配置元数据结合在一起，以便在创建和初始化ApplicationContext之后，您将拥有一个完全配置且可执行的系统或应用程序。

![container magic](spring-core/container-magic.png)

### 配置元数据

如上图所示，Spring IoC容器使用一种形式的配置元数据。 此配置元数据表示您作为应用程序开发人员如何告诉Spring容器实例化，配置和组装应用程序中的对象。

传统上，配置元数据以简单直观的XML格式提供，这是本章大部分内容用来传达Spring IoC容器的关键概念和功能的内容。

有关在Spring容器中使用其他形式的元数据的信息，请参见：

- 基于注释的配置：Spring 2.5引入了对基于注释的配置元数据的支持。

- 基于Java的配置：从Spring 3.0开始，Spring JavaConfig项目提供的许多功能成为核心Spring Framework的一部分。 因此，您可以使用Java而不是XML文件来定义应用程序类外部的bean。 要使用这些新功能，请参见@Configuration，@Bean，@Import和@DependsOn批注。

基于XML的配置元数据将这些bean配置为<beans/>内的元素。 Java配置通常在@Configuration类中使用@Bean注释的方法。

这些bean定义对应于组成应用程序的实际对象。 通常，您定义服务层对象，数据访问对象（DAO），表示层对象（例如Struts Action实例），基础结构对象（例如JPA `EntityManagerFactory`，JMS队列）等等。 通常，不会在容器中配置细粒度的域对象，因为DAO和业务逻辑通常负责创建和加载域对象。 但是，您可以使用Spring与AspectJ的集成来配置在IoC容器控制之外创建的对象。 请参阅使用AspectJ与Spring依赖注入域对象。

以下示例显示了基于XML的配置元数据的基本结构：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">  
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>
```

- id属性是标识单个bean定义的字符串。

- class属性定义bean的类型并使用完全限定类名称。

### 实例化容器

提供给ApplicationContext构造函数的一个或多个位置路径是资源字符串，这些资源字符串使容器可以从各种外部资源（例如本地文件系统，Java CLASSPATH等）加载配置元数据。

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

下面是services.xml文件的内容:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->

    <bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
        <property name="accountDao" ref="accountDao"/>
        <property name="itemDao" ref="itemDao"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->

</beans>
```

下面是daos.xml文件的内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountDao"
        class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>
```

在前面的示例中，服务层由PetStoreServiceImpl类和两个JpaAccountDao和JpaItemDao类型的数据访问对象组成（基于JPA对象关系映射标准）。 property 标签的names属性引用JavaBean属性的名称，而ref元素引用另一个bean定义的名称。 id和ref元素之间的这种联系表达了协作对象之间的依赖性。 有关配置对象的依存关系的详细信息，请参阅依存关系。

### 基于XML的配置元数据的组成

使bean定义跨越多个XML文件可能很有用。 通常，每个单独的XML配置文件都代表体系结构中的逻辑层或模块。

您可以使用应用程序上下文构造函数从所有这些XML片段中加载bean定义。 如上一节中所示，此构造函数具有多个Resource位置。 或者，使用元素的一个或多个实例从另一个文件中加载bean定义。 以下示例显示了如何执行此操作：

```xml
<beans>
    <import resource="services.xml"/>
    <import resource="resources/messageSource.xml"/>
    <import resource="/resources/themeSource.xml"/>

    <bean id="bean1" class="..."/>
    <bean id="bean2" class="..."/>
</beans>
```

在前面的示例中，外部bean定义是从三个文件加载的：services.xml，messageSource.xml和themeSource.xml。 所有位置路径都相对于进行导入的定义文件，因此，services.xml必须与进行导入的文件位于同一目录或类路径位置，而messageSource.xml和themeSource.xml必须位于该位置下方的resource目录下 。 如您所见，斜杠被忽略。 但是，鉴于这些路径是相对的，最好不要使用任何斜线。 根据Spring Schema，导入的文件的内容（包括顶级元素）必须是有效的XML bean定义。

可以但不建议使用相对的“ ../”路径引用父目录中的文件。 这样做会让当前应用程序依赖外部文件。 特别是，不建议使用classpath：URL（例如，classpath：../ services.xml）引用，在URL中，运行时解析过程会选择“最近”的classpath根目录，然后查看其父目录。 类路径配置的更改可能导致选择其他错误的目录。

您始终可以使用标准资源位置而不是相对路径：例如，file:C:/config/services.xml或classpath:/config/services.xml。 但是请注意，您正在将应用程序的配置耦合到特定的绝对位置。 通常，最好为这样的绝对位置保留一个间接寻址，例如，通过在运行时针对JVM系统属性解析的“ $ {…}”占位符。

### 使用容器

ApplicationContext是高级工厂的接口，该工厂能够维护不同bean及其依赖关系的注册表。 通过使用方法`T getBean（String name，Class requiredType）`，可以检索bean的实例。

通过ApplicationContext，您可以读取Bean并访问它们，如以下示例所示：

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

最灵活的变体是GenericApplicationContext,与具体的reader委托结合使用，例如，与XML文件的XmlBeanDefinitionReader结合使用，如以下示例所示：

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();
```

然后可以使用getBean检索bean的实例。 ApplicationContext接口还有其他几种检索bean的方法，但是理想情况下，您的应用程序代码永远不要使用它们。 确实，您的应用程序代码应该不调用getBean（）方法，这样不会耦合Spring API。 例如，Spring与Web框架的集成为各种Web框架组件（例如控制器和JSF管理的Bean）提供了依赖项注入，使您可以通过元数据（例如自动装配注释）声明对特定Bean的依赖项。

## 3.bean概览
在spring中，bean 定义表示为 `BeanDefinition` 对象，其中包含以下元数据：

* 包限定的类名：通常是定义的 bean 的实际实现类。
* Bean 行为配置元素，它说明 Bean 在容器中的行为方式（范围、生命周期回调等）。
* 对 bean 执行其工作所需的其他 bean 的引用。 这些引用也称为协作者或依赖项。
* 要在新创建的对象中设置的其他配置设置 — 例如，管理连接池的 bean ，池的大小限制或使用的连接数。

这些元数据由下面的属性描述：

| Property                 | Explained in…                                                |
| :----------------------- | :----------------------------------------------------------- |
| Class                    | [Instantiating Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-class) |
| Name                     | [Naming Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beanname) |
| Scope                    | [Bean Scopes](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-scopes) |
| Constructor arguments    | [Dependency Injection](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-collaborators) |
| Properties               | [Dependency Injection](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-collaborators) |
| Autowiring mode          | [Autowiring Collaborators](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-autowire) |
| Lazy initialization mode | [Lazy-initialized Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lazy-init) |
| Initialization method    | [Initialization Callbacks](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lifecycle-initializingbean) |
| Destruction method       | [Destruction Callbacks](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lifecycle-disposablebean) |

除了包含有关如何创建特定bean的定义信息之外，ApplicationContext实现还允许注册在容器外部（由用户）创建的现有对象。 这是通过ApplicationContext的getBeanFactory（）方法返回的BeanFactory来完成的，该方法返回BeanFactory DefaultListableBeanFactory实现。 DefaultListableBeanFactory通过registerSingleton（..）和registerBeanDefinition（..）方法支持此注册。 但是，典型的应用程序只能与通过常规bean定义元数据定义的bean一起使用。

Bean元数据和手动提供的单例实例需要尽早注册，以便容器在自动装配和其他自省步骤中正确地推理它们。 尽管在某种程度上支持覆盖现有元数据和现有单例实例，但官方不支持在运行时（与对工厂的实时访问同时）对新bean的注册，并且可能导致并发访问异常，bean容器中的状态不一致。

### 3.1 bean命名
每个bean具有一个或多个标识符。 这些标识符在承载Bean的容器内必须唯一。 一个bean通常只有一个标识符。 但是，如果需要多个，则可以将多余的别名视为别名。

在基于XML的配置元数据中，可以使用id属性和（或）name属性来指定Bean标识符。 id属性可让您精确指定一个id。 按照惯例，这些名称是字母加数字（“ myBean”，“ someService”等），但它们也可以包含特殊字符。 如果要为bean引入其他别名，则还可以在name属性中指定它们，并用逗号（，）分号（;）或空格分隔。 作为历史记录，在Spring 3.1之前的版本中，id属性定义为xsd：ID类型，该类型限制了可能的字符。 从3.1开始，它被定义为xsd：string类型。 请注意，bean ID唯一性仍由容器强制执行，尽管不再由XML解析器执行。

您不需要提供bean的名称或ID。 如果未明确提供名称或ID，则容器将为该bean生成一个唯一的名称。 但是，如果要通过名称引用该bean，则必须通过使用ref元素或服务定位器样式查找，您必须提供一个名称。 

通过在类路径中进行组件扫描，Spring会按照前面描述的规则为未命名的组件生成Bean名称：从本质上讲，采用简单的类名称并将其初始字符转换为小写。 但是，在（不寻常的）特殊情况下，如果有多个字符并且第一个和第二个字符均为大写字母，则会保留原始大小写。 这些规则与java.beans.Introspector.decapitalize（Spring在此使用）定义的规则相同。

#### 3.1.1 在Bean定义之外别名Bean
在bean定义本身中，可以通过使用id属性指定的最多一个名称和name属性中任意数量的其他名称的组合来为bean提供多个名称。 这些名称可以是同一个bean的等效别名，并且在某些情况下很有用，例如，通过使用特定于该组件本身的bean名称，让应用程序中的每个组件都引用一个公共依赖项。

但是，在实际定义bean的地方指定所有别名并不总是足够的。 有时需要为在别处定义的bean引入别名。 这在大型系统中通常是这种情况，在大型系统中，配置在每个子系统之间分配，每个子系统都有自己的对象定义集。 在基于XML的配置元数据中，可以使用元素来完成此操作。 以下示例显示了如何执行此操作：

```xml
<alias name="fromName" alias="toName"/>
```
在这种情况下，在使用该别名定义之后，也可以将名为fromName的bean（在同一容器中）称为toName。

例如，子系统A的配置元数据可以通过子系统A-dataSource的名称引用数据源。 子系统B的配置元数据可以通过子系统B-dataSource的名称引用数据源。 组成使用这两个子系统的主应用程序时，主应用程序通过myApp-dataSource的名称引用数据源。 要使所有三个名称都引用同一个对象，可以将以下别名定义添加到配置元数据中：

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```
现在，每个组件和主应用程序都可以通过唯一的名称引用数据源，并且可以保证不与任何其他定义冲突（有效地创建名称空间），但是它们引用的是同一bean。

如果使用Javaconfiguration，则@Bean批注可用于提供别名。 有关详细信息，请参见使用@Bean注释。

### 3.2 实例化Bean
Bean定义实质上是创建一个或多个对象的方法。 容器在被询问时会查看命名bean的配方，并使用该bean定义封装的配置元数据来创建（或获取）实际对象。

如果使用基于XML的配置元数据，请在元素的class属性中指定要实例化的对象的类型（或类）。 这个类属性（在内部是BeanDefinition实例的Class属性）通常是必需的。 （有关异常，请参见使用实例工厂方法实例化和Bean定义继承。）可以通过以下两种方式之一使用Class属性：

* 通常，在容器本身通过反射性地调用其构造函数直接创建Bean的情况下，指定要构造的Bean类，这在某种程度上等同于使用new运算符的Java代码。
* 要指定包含用于创建对象的静态工厂方法的实际类，在不太常见的情况下，容器将在类上调用静态工厂方法以创建Bean。 从静态工厂方法的调用返回的对象类型可以是同一类，也可以是完全不同的另一类。

> 如果要为静态嵌套类配置Bean定义，则必须使用嵌套类的二进制名称。

> 例如，如果您在com.example包中有一个名为SomeThing的类，并且此SomeThing类具有一个名为OtherThing的静态嵌套类，则bean定义上的class属性的值为com.example.SomeThing \$ OtherThing。

> 请注意，名称中使用\$字符将嵌套的类名与外部类名分开。

#### 使用构造函数创建
当通过构造方法创建一个bean时，所有普通类都可以被Spring使用并兼容。 也就是说，正在开发的类不需要实现任何特定的接口或以特定的方式进行编码。 只需指定bean类就足够了。 但是，根据您用于该特定bean的IoC的类型，您可能需要一个默认（空）构造函数。

Spring IoC容器几乎可以管理您要管理的任何类。 它不仅限于管理真正的JavaBean。 大多数Spring用户更喜欢实际的JavaBean，它们仅具有默认的（无参数）构造函数，并具有根据容器中的属性建模的适当的setter和getter。 您还可以在容器中具有更多奇特的非Bean样式类。 例如，如果您需要使用绝对不符合JavaBean规范的旧式连接池，则Spring也可以对其进行管理。

使用基于XML的配置元数据，您可以如下指定bean类：

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>

<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```
有关向构造函数提供参数（如果需要）并在构造对象之后设置对象实例属性的机制的详细信息，请参见注入依赖项。

#### 使用静态工厂方法创建实例
定义使用静态工厂方法创建的bean时，请使用class属性指定包含静态工厂方法的类，并使用名为factory-method的属性指定工厂方法本身的名称。 您应该能够调用此方法（使用可选参数，如稍后所述）并返回一个活动对象，该对象随后将被视为已通过构造函数创建。 这种bean定义的一种用法是在旧版代码中调用静态工厂。

以下bean定义指定通过调用工厂方法来创建bean。 该定义不指定返回对象的类型（类），而仅指定包含工厂方法的类。 在此示例中，createInstance（）方法必须是静态方法。 以下示例显示如何指定工厂方法：

```xml
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/>
```
以下示例显示了可与前面的bean定义一起使用的类：

```java
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        return clientService;
    }
}
```
有关为工厂方法提供（可选）参数并在从工厂返回对象后设置对象实例属性的机制的详细信息，请参阅详细信息中的依赖关系和配置。

#### 使用实例工厂方法创建对象
类似于通过静态工厂方法进行实例化，使用实例工厂方法进行实例化会从容器中调用现有bean的非静态方法来创建新bean。 要使用此机制，请将class属性保留为空，并在factory-bean属性中，在当前（或父或祖先）容器中指定包含要创建该对象的实例方法的bean的名称。 使用factory-method属性设置工厂方法本身的名称。 以下示例显示了如何配置此类Bean：

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }
}
```
一个工厂类也可以包含一个以上的工厂方法，如以下示例所示：

```xml
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>

<bean id="accountService"
    factory-bean="serviceLocator"
    factory-method="createAccountServiceInstance"/>
```
```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    private static AccountService accountService = new AccountServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }

    public AccountService createAccountServiceInstance() {
        return accountService;
    }
}
```
这种方法表明，工厂Bean本身可以通过依赖项注入（DI）进行管理和配置。 详细信息，请参见依赖性和配置。

在Spring文档中，“ factory bean”是指在Spring容器中配置并通过实例或静态工厂方法创建对象的bean。 相比之下，FactoryBean（注意大小写）是指特定于Spring的FactoryBean实现类。

#### 确定Bean的运行时类型
确定特定bean的运行时类型并非易事。 Bean元数据定义中的指定类只是初始类引用，可能与声明的工厂方法结合使用，或者是FactoryBean类，这可能导致Bean的运行时类型不同，或者在实例工厂方法的情况下根本不进行设置 （通过指定的factory-bean名称解析）。 此外，AOP代理可以使用基于接口的代理包装bean实例，而目标Bean的实际类型（仅是其实现的接口）的暴露程度有限。

找出特定bean的实际运行时类型的推荐方法是对指定bean名称的BeanFactory.getType调用。 这考虑了上述所有情况，并返回了针对相同bean名称的BeanFactory.getBean调用将返回的对象的类型。

## 依赖
典型的企业应用程序不包含单个对象（或Spring术语中的bean）。 即使是最简单的应用程序，也有一些对象可以协同工作，以呈现最终用户视为一致的应用程序。 下一部分将说明如何从定义多个独立的Bean定义到实现对象协作以实现目标的完全实现的应用程序。

### bean的命名

在基于 XML 的配置元数据中，您可以使用 id 属性（唯一）、name 属性（相当于别名，多个用逗号分隔）或两者来指定 bean 标识符。如果没有给bean指定id，会默认生成一个。通常我们建议bean的名称是小写开头的简单类名。

此外alias标签也可以创建别名：

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```
### bean实例化
构造函数方式：

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>
```
静态工厂：

```xml
<bean id="clientService" class="examples.ClientService" factory-method="createInstance"/>
```
实例工厂：

### bean定义继承
使用父子 bean 定义可以节省大量输入。 实际上，这是一种模板形式。

```xml
<bean id="inheritedTestBean" abstract="true"
        class="org.springframework.beans.TestBean">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>
<bean id="inheritsWithDifferentClass"
        class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBean" init-method="initialize">  
    <property name="name" value="override"/>
    <!-- the age property value of 1 will be inherited from parent -->
</bean>
```
子 bean 定义从父 bean 继承范围、构造函数参数值、属性值和方法覆盖，并可以选择添加新值。 您指定的任何范围、初始化方法、销毁方法或静态工厂方法设置都会覆盖相应的父设置。

前面的示例使用抽象属性将父 bean 定义显式标记为抽象。 如果父定义未指定类，则需要将父 bean 定义显式标记为抽象。

父 bean 不能单独实例化，因为它是不完整的，并且它也被显式标记为抽象。 当定义是抽象的时，它只能用作纯模板 bean 定义，作为子定义的父定义。 通过将其作为另一个 bean 的 ref 属性引用或使用父 bean ID 执行显式 getBean() 调用，尝试单独使用此类抽象父 bean 会返回错误。 类似地，容器的内部 preInstantiateSingletons() 方法忽略定义为抽象的 bean 定义。

## 依赖
### 构造函数注入
多个构造参数的构造函数，默认的注入方式：

```xml
<beans>
    <bean id="beanOne" class="x.y.ThingOne">
        <constructor-arg ref="beanTwo"/>
        <constructor-arg ref="beanThree"/>
    </bean>
    <bean id="beanTwo" class="x.y.ThingTwo"/>
    <bean id="beanThree" class="x.y.ThingThree"/>
</beans>
```
类型匹配的注入方式：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
```
索引匹配的注入方式：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg index="0" value="7500000"/>
    <constructor-arg index="1" value="42"/>
</bean>
```
构造参数名称匹配的方式：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```
请记住，要使这项工作开箱即用，您的代码必须在启用调试标志的情况下进行编译，以便 Spring 可以从构造函数中查找参数名称。 如果您不能或不想使用调试标志编译代码，则可以使用 @ConstructorProperties JDK 注释显式命名构造函数参数。 示例类必须如下所示：

```java
public class ExampleBean {
    @ConstructorProperties({"years", "ultimateAnswer"})
    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```
### setter注入
基于 Setter 的 DI 是容器在调用无参数构造函数或无参数静态工厂方法来实例化 bean 后调用 bean 上的 setter 方法来完成的。

以下示例显示了一个只能使用纯 setter 注入进行依赖注入的类。 这个类是传统的Java。 它是一个不依赖于容器特定接口、基类或注解的 POJO。

```java
public class SimpleMovieLister {
    // the SimpleMovieLister has a dependency on the MovieFinder
    private MovieFinder movieFinder;
    // a setter method so that the Spring container can inject a MovieFinder
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
    // business logic that actually uses the injected MovieFinder is omitted...
}
```
由于您可以混合使用基于构造函数和基于 setter 的 DI，因此根据经验，对强制依赖项使用构造函数，对可选依赖项使用 setter 方法是一个很好的经验法则。 请注意，在 setter 方法上使用 @Required 注释可用于使属性成为必需的依赖项； 但是，最好使用构造函数注入。

Spring 团队通常提倡构造函数注入，因为它可以让您将应用程序组件实现为不可变对象，并确保所需的依赖项不为空。 此外，构造函数注入的组件总是以完全初始化的状态返回给客户端（调用）代码。 

> 大量的构造函数参数是一种糟糕的代码，这意味着该类可能有太多的责任，应该重构以更好地解决适当的关注点分离问题。

### 循环依赖
使用构造函数注入，则可能会出现循环依赖。例如：A类通过构造函数注入B类的实例，B类通过构造函数注入A类的实例。 如果您将类 A 和 B 的 bean 配置为相互注入，则 Spring IoC 容器在运行时检测到此循环引用，并抛出 BeanCurrentlyInCreationException。

可能的解决方案是编辑一些类的源代码，以便由 setter 而不是构造函数来配置。 

Spring 在真正创建 bean 时尽可能晚地设置属性并解析依赖项。 这意味着，如果创建该对象或其依赖项之一时出现问题，spring 容器仍然正常启动，只在你调用改bean时，才会告诉你异常。

在容器启动时就创建bean,虽然会花费启动时间和系统内存，但是可以提前发现配置文件中的错误。你可以覆盖此默认的early方式，以便bean延时初始化。

### `depends-on`
如果一个 bean 是另一个 bean 的依赖项，这通常意味着一个 bean 被设置为另一个 bean 的属性。  但是，有时 bean 之间的依赖关系不那么直接。 例如，当需要触发类中的静态初始化程序时(数据库驱动程序注册)。 在初始化使用此元素的 bean 之前，depends-on 属性可以显式地强制初始化一个或多个 bean。 以下示例使用depends-on 属性来表达对bean 的依赖：

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
    <property name="manager" ref="manager" />
</bean>
<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```
### 配置细节
p标签方式：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource"
        destroy-method="close"
        p:driverClassName="com.mysql.jdbc.Driver"
        p:url="jdbc:mysql://localhost:3306/mydb"
        p:username="root"
        p:password="misterkaoli"/>
</beans>
```
**idref 元素**

idref 元素只是一种将容器中另一个 bean 的 id（字符串值 - 而不是引用）传递给元素的防错方式。 以下示例显示了如何使用它：

```xml
<bean id="theTargetBean" class="..."/>
<bean id="theClientBean" class="...">
    <property name="targetName">
        <idref bean="theTargetBean"/>
    </property>
</bean>
```
完全等同下面的代码：

```xml
<bean id="theTargetBean" class="..." />
<bean id="client" class="...">
    <property name="targetName" value="theTargetBean"/>
</bean>
```
ref**元素**

ref等同与property标签上的ref属性，表示对bean的引用。但是ref更强大，支持引用父容器中的bean。例如：

内部**bean**

内部 bean 定义不需要定义的 ID 或name。 如果指定，容器不会使用这样的值作为标识符。 容器在创建时也会忽略范围标志，因为内部 bean 始终是匿名的，并且始终与外部 bean 一起创建。 不可能独立访问内部 bean 或将它们注入除封闭 bean 之外的协作 bean 中:

```xml
<bean id="outer" class="...">
    <!-- instead of using a reference to a target bean, simply define the target bean inline -->
    <property name="target">
        <bean class="com.example.Person"> <!-- this is the inner bean -->
            <property name="name" value="Fiona Apple"/>
            <property name="age" value="25"/>
        </bean>
    </property>
</bean>
```
**设置集合类型的属性**

, , , 和

```xml
<bean id="moreComplexObject" class="example.ComplexObject">
    <!-- results in a setAdminEmails(java.util.Properties) call -->
    <property name="adminEmails">
        <props>
            <prop key="administrator">administrator@example.org</prop>
            <prop key="support">support@example.org</prop>
            <prop key="development">development@example.org</prop>
        </props>
    </property>
    <!-- results in a setSomeList(java.util.List) call -->
    <property name="someList">
        <list>
            <value>a list element followed by a reference</value>
            <ref bean="myDataSource" />
        </list>
    </property>
    <!-- results in a setSomeMap(java.util.Map) call -->
    <property name="someMap">
        <map>
            <entry key="an entry" value="just some string"/>
            <entry key="a ref" value-ref="myDataSource"/>
        </map>
    </property>
    <!-- results in a setSomeSet(java.util.Set) call -->
    <property name="someSet">
        <set>
            <value>just some string</value>
            <ref bean="myDataSource" />
        </set>
    </property>
</bean>
```
集合合并

子集合的值是合并父集合和子集合的元素的结果，子集合元素覆盖父集合中指定的值

```xml

<beans>
    <bean id="parent" abstract="true" class="example.ComplexObject">
        <property name="adminEmails">
            <props>
                <prop key="administrator">administrator@example.com</prop>
                <prop key="support">support@example.com</prop>
            </props>
        </property>
    </bean>
    <bean id="child" parent="parent">
        <property name="adminEmails">
            <!-- the merge is specified on the child collection definition -->
            <props merge="true">
                <prop key="sales">sales@example.com</prop>
                <prop key="support">support@example.co.uk</prop>
            </props>
        </property>
    </bean>
<beans>
```
**集合上的泛型**

```java
public class SomeClass {

    private Map<String, Float> accounts;

    public void setAccounts(Map<String, Float> accounts) {
        this.accounts = accounts;
    }
}
```
```xml
<beans>
    <bean id="something" class="x.y.SomeClass">
        <property name="accounts">
            <map>
                <entry key="one" value="9.99"/>
                <entry key="two" value="2.75"/>
                <entry key="six" value="3.99"/>
            </map>
        </property>
    </bean>
</beans>
```
**空值处理**

```xml
<bean class="ExampleBean">
    <property name="email" value=""/>
</bean>
```
等同于：

```java
exampleBean.setEmail("");
```
```xml
<bean class="ExampleBean">
    <property name="email">
        <null/>
    </property>
</bean>
```
等同于：

```java
exampleBean.setEmail(null);
```
### bean的懒加载机制
默认情况下，ApplicationContext 实现会在初始化过程中急切地创建和配置所有单例 bean。 因为可以立即发现配置或周围环境中的错误。 当这种行为不可取时，您可以通过将 bean 定义标记为延迟初始化。 一个延迟初始化的 bean 告诉 IoC 容器在它第一次被请求时创建一个 bean 实例，而不是在容器启动时。

在 XML 中，此行为由 元素上的 lazy-init 属性控制，如以下示例所示：

```Plain Text
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
```
当延迟初始化的 bean 是未延迟初始化的单例 bean 的依赖项时，ApplicationContext 在启动时创建延迟初始化的 bean，因为它必须满足单例的依赖项。

您还可以通过使用 元素上的 default-lazy-init 属性在容器级别控制延迟初始化，如以下示例所示：

```xml
<beans default-lazy-init="true">
    <!-- no beans will be pre-instantiated... -->
</beans>
```
### 自动装配
使用基于 XML 的配置元数据时，您可以使用标签的 autowire 属性为 bean 定义指定自动装配模式。 自动装配功能有四种模：

* no:（默认）没有自动装配。 Bean 引用必须由 ref 元素定义。 对于较大的部署，不建议更改默认设置，因为明确指定协作者可以提供更好的控制和清晰度。 在某种程度上，它记录了系统的结构。
* byName:按属性名称自动装配。 Spring 查找与需要自动装配的属性同名的 bean。 例如，如果一个 bean 定义被设置为按名称自动装配并且它包含一个master属性（即它有一个 setMaster(..) 方法），Spring 会查找一个名为 master 的 bean 定义并使用它来设置属性。
* byType:如果容器中只存在一个属性类型的 bean，则让属性自动装配。 如果存在多个，则会引发致命异常，这表明您不能为该 bean 使用 byType 自动装配。 如果没有匹配的 bean，则不会发生任何事情（未设置属性）。
* constructor:类似于 byType 但适用于构造函数参数。 如果容器中没有一个构造函数参数类型的 bean，则会引发致命错误。

### 方法注入
单例 bean A 需要使用非单例bean B时，容器只创建单例 bean A 一次，因此只有一次设置属性的机会，容器无法在每次需要时为bean A 提供 bean B 的新实例。

一个解决方案是放弃一些控制反转。 您可以通过实现 ApplicationContextAware 接口使 bean A 了解容器，并在每次使用 bean A 时通过对容器进行 getBean("B") 调用来请求（通常是新的）bean B 实例。 以下示例显示了这种方法：

```java
// a class that uses a stateful Command-style class to perform some processing
package fiona.apple;

// Spring-API imports
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class CommandManager implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public Object process(Map commandState) {
        // grab a new instance of the appropriate Command
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    protected Command createCommand() {
        // notice the Spring API dependency!
        return this.applicationContext.getBean("command", Command.class);
    }

    public void setApplicationContext(
            ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```
但是，这样的代码是不可取的，因为业务代码耦合到 Spring Framework。 方法注入是 Spring IoC 容器的高级的特性，可以让你干净地处理这个用例。

```java
package fiona.apple;

// no more Spring imports!

public abstract class CommandManager {

    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}
```
spring使用CGLIB 生成子类实现覆盖指定的方法。等同的注解方式：

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup("myCommand")
    protected abstract Command createCommand();
}
```
### bean的scope
Spring Framework 支持六个scope，其中四个仅在使用 web 时可用。 您还可以创建自定义scope。

* singleton：同一容器容器有且只有一个bean。
* prototype：容器会创建bean，但是不会添加到容器中管理，因此每次使用就会创建新的。也就说，只管创建，不管销毁，因此只会走初始化流程，销毁流程不会触发，例如销毁的回调机制。
* request：将单个 bean 定义范围限定为单个 HTTP 请求的生命周期。 也就是说，每个 HTTP 请求都有自己的 bean 实例。 @RequestScope
* session：将单个 bean 定义范围限定为 HTTP 会话的生命周期。@SessionScope
* application：将单个 bean 定义范围限定为 ServletContext 的生命周期。 @ApplicationScope
* websocket：将单个 bean 定义范围限定为 WebSocket 的生命周期。 

如果你想将一个 HTTP 请求范围的 bean 注入到另一个生命周期更长的 bean 中，可以使用AOP 代理来代替该范围的 bean。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- an HTTP Session-scoped bean exposed as a proxy -->
    <bean id="userPreferences" class="com.something.UserPreferences" scope="session">
        <!-- instructs the container to proxy the surrounding bean -->
        <aop:scoped-proxy/> 
    </bean>

    <!-- a singleton-scoped bean injected with a proxy to the above bean -->
    <bean id="userService" class="com.something.SimpleUserService">
        <!-- a reference to the proxied userPreferences bean -->
        <property name="userPreferences" ref="userPreferences"/>
    </bean>
</beans>
```
在这个例子中，当一个 UserManager 实例在依赖注入的 UserPreferences 对象上调用一个方法时，它实际上是在调用代理上的一个方法。然后代理从（在这种情况下）HTTP 会话中获取真实的 UserPreferences 对象，并将方法调用委托给检索到的真实 UserPreferences 对象。

#### 自定义scope
1. 实现 org.springframework.beans.factory.config.Scope 接口：

```java
public interface Scope {
    Object get(String name, ObjectFactory<?> objectFactory);

    @Nullable
    Object remove(String name);

    void registerDestructionCallback(String name, Runnable callback);

    @Nullable
    Object resolveContextualObject(String key);

    @Nullable
    String getConversationId();
}
```
1. 使用自定义scope:

```java
Scope threadScope = new SimpleThreadScope();
beanFactory.registerScope("thread", threadScope);
```
```xml
<bean id="..." class="..." scope="thread">
```
## 自定义bean
### 生命周期回调
在内部，Spring 框架使用 `BeanPostProcessor` 实现来处理它可以找到的任何回调接口并调用适当的方法。

初始化：

1. 实现`org.springframework.beans.factory.InitializingBean`接口：耦合spring，不建议
2. @PostConstruct 注解
3. 指定init-method方法

销毁：

1. 实现 org.springframework.beans.factory.DisposableBean 接口：耦合spring，不建议
2. @PreDestroy 注解：
3. 指定destroy-method方法：

> 如果没有指定生命周期方法，spring默认查找 init()、initialize()、dispose() 等名称的方法作为生命周期回调方法，你可以修改默认名称： 

为同一个 bean 配置的多个生命周期机制，具有不同的初始化方法，调用如下：

* 用@PostConstruct 注释的方法
* afterPropertiesSet() 由 InitializingBean 回调接口定义
* 自定义配置的 init() 方法

如果为一个 bean 配置了多个生命周期机制，并且每个机制都配置了不同的方法名称，那么每个配置的方法将按照上面列出的顺序运行。 但是，如果配置了相同的方法名称 ，则该方法将运行一次。

### Lifecycle
Lifecycle 接口为任何具有自己生命周期要求的对象定义了基本方法：

```java
public interface Lifecycle {
    void start();
    void stop();
    boolean isRunning();
}
```
任何 Spring 管理的对象都可以实现 Lifecycle 接口。 然后，当 ApplicationContext 本身接收到启动和停止信号时（例如，对于运行时的停止/重启场景），它会将这些调用级联到该上下文中定义的所有 Lifecycle 实现。 它通过委托给 LifecycleProcessor 来做到这一点，如下面所示：

```java
public interface LifecycleProcessor extends Lifecycle {
    void onRefresh();
    void onClose();
}
```
启动和关闭调用的顺序可能很重要。 如果任何两个对象之间存在“依赖”关系，则依赖方在其依赖之后开始，并在其依赖之前停止。 然而，有时，直接依赖是未知的。 您可能只知道某种类型的对象应该在另一种类型的对象之前开始。 在这些情况下，SmartLifecycle 接口定义了另一个选项，即在其超级接口 Phased 上定义的 getPhase() 方法。 以下清单显示了 Phased 接口的定义：

```java
public interface Phased {
    int getPhase();
}
```
以下清单显示了 SmartLifecycle 接口的定义：

```java
public interface SmartLifecycle extends Lifecycle, Phased {
    boolean isAutoStartup();
    void stop(Runnable callback);
}
```
启动时，Phased最低的对象首先启动。 停止时，遵循相反的顺序。 因此，一个实现 SmartLifecycle 并且其 getPhase() 方法返回 Integer.MIN\_VALUE 的对象将是最先启动和最后一个停止的对象。在考虑Phased值时，重要的是要知道任何未实现 SmartLifecycle 的“正常”生命周期对象的默认Phased是 0。因此，任何负Phased值表示对象应该在这些标准组件之前开始（ 在他们之后停止）。

### `ApplicationContextAware` 和`BeanNameAware`
当 ApplicationContext 创建一个实现 org.springframework.context.ApplicationContextAware 接口的对象实例时，该实例提供了对该 ApplicationContext 的引用。 以下清单显示了 ApplicationContextAware 接口的定义：

```java
public interface ApplicationContextAware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```
当 ApplicationContext 创建一个实现 org.springframework.beans.factory.BeanNameAware 接口的类时，该类被提供了对在其关联对象定义中定义的名称的引用。 以下清单显示了 BeanNameAware 接口的定义：

```java
public interface BeanNameAware {
    void setBeanName(String name) throws BeansException;
}
```
| Name                             | Injected Dependency                                          | Explained in…                                                |
| :------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `ApplicationContextAware`        | Declaring `ApplicationContext`.                              | `ApplicationContextAware`[ and ](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware)`BeanNameAware` |
| `ApplicationEventPublisherAware` | Event publisher of the enclosing `ApplicationContext`.       | [Additional Capabilities of the ](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction)`ApplicationContext` |
| `BeanClassLoaderAware`           | Class loader used to load the bean classes.                  | [Instantiating Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-class) |
| `BeanFactoryAware`               | Declaring `BeanFactory`.                                     | [The ](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beanfactory)`BeanFactory` |
| `BeanNameAware`                  | Name of the declaring bean.                                  | `ApplicationContextAware`[ and ](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware)`BeanNameAware` |
| `LoadTimeWeaverAware`            | Defined weaver for processing class definition at load time. | [Load-time Weaving with AspectJ in the Spring Framework](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-aj-ltw) |
| `MessageSourceAware`             | Configured strategy for resolving messages (with support for parametrization and internationalization). | [Additional Capabilities of the ](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction)`ApplicationContext` |
| `NotificationPublisherAware`     | Spring JMX notification publisher.                           | [Notifications](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#jmx-notifications) |
| `ResourceLoaderAware`            | Configured loader for low-level access to resources.         | [Resources](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#resources) |
| `ServletConfigAware`             | Current `ServletConfig` the container runs in. Valid only in a web-aware Spring `ApplicationContext`. | [Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc) |
| `ServletContextAware`            | Current `ServletContext` the container runs in. Valid only in a web-aware Spring `ApplicationContext`. | [Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc) |

## 容器扩展点
### 使用`BeanPostProcessor`自定义bean
> 实例化：new创建了对象

> 配置: 完成了依赖解析

> 初始化：执行了生命周期方法

```java
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
BeanPostProcessor 接口定义了回调方法，您可以实现这些方法来提供您自己的（或覆盖容器的默认）实例化逻辑、依赖解析逻辑等。 如果要在 Spring 容器完成对 bean 的实例化、配置和初始化之后实现一些自定义逻辑，可以插入一个或多个自定义 BeanPostProcessor 实现。

您可以配置多个 BeanPostProcessor 实例，并且可以通过实现 Ordered 接口控制这些 BeanPostProcessor 实例的运行顺序。

拦截bean的init callback方法，postProcessBeforeInitialization在init方法之前执行，postProcessAfterInitialization在init方法之后执行。 后处理器可以对 bean 实例执行任何操作，包括完全忽略初始化回调。 bean 后处理器通常检查回调接口，或者用代理包装 bean。 一些 Spring AOP 基础设施类被实现为 bean 后处理器，以提供代理包装逻辑。

ApplicationContext 自动检测实现 BeanPostProcessor 接口的 bean，然后 注册，以便稍后在 bean 创建时调用它们。 Bean 后处理器可以以与任何其他 Bean 相同的方式部署在容器中。

**以编程方式注册 BeanPostProcessor 实例**

 虽然推荐的 BeanPostProcessor 注册方法是通过 ApplicationContext 自动检测，但您可以使用 ConfigurableBeanFactory 的addBeanPostProcessor 方法以编程方式注册它们。 当您需要在注册之前评估条件逻辑时，甚至需要在层次结构中的上下文之间复制 bean 后处理器时，这会很有用。 但是请注意，以编程方式添加的 BeanPostProcessor 实例不遵守 Ordered 接口。 在这里，注册的顺序决定了执行的顺序。 另请注意，以编程方式注册的 BeanPostProcessor 实例始终在通过自动检测注册的实例之前处理，而不管任何显式排序。

**BeanPostProcessor 实例和 AOP 自动代理** 

实现 BeanPostProcessor 接口的类是特殊的，容器会对其进行不同的处理。 所有 BeanPostProcessor 实例和它们直接引用的 bean 在启动时被实例化，作为 ApplicationContext 的特殊启动阶段的一部分。 接下来，所有 BeanPostProcessor 实例都以排序方式注册并应用于容器中的所有其他 bean。

 因为 AOP 自动代理是作为 BeanPostProcessor 本身实现的，所以 BeanPostProcessor 实例和它们直接引用的 bean 都没有资格进行自动代理，因此，它们没有编织切面。对于任何此类 bean，您应该看到一条信息性日志消息：Bean someBean is not eligible for getting processed by all BeanPostProcessor interfaces (for example: not eligible for auto-proxying)

如果您使用自动装配或 @Resource将 bean 连接到 BeanPostProcessor，则 Spring 在搜索类型匹配依赖项时可能会访问意外的 bean（它们有可能还没有被自动代理或应用其他 bean后处理）。

将回调接口或注解与自定义 BeanPostProcessor 实现结合使用是扩展 Spring IoC 容器的常用方法。 一个例子是 Spring 的 AutowiredAnnotationBeanPostProcessor 

### 使用 BeanFactoryPostProcessor 自定义配置元数据
```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```
我们要查看的下一个扩展点是 org.springframework.beans.factory.config.BeanFactoryPostProcessor。 此接口的语义与 BeanPostProcessor 的语义相似，但有一个主要区别：BeanFactoryPostProcessor 对 bean 配置元数据进行操作。 也就是说，Spring IoC 容器让 BeanFactoryPostProcessor 读取配置元数据，并可能在容器实例化除 BeanFactoryPostProcessor 实例之外的任何 bean 之前更改它。

您可以配置多个 BeanFactoryPostProcessor 实例，您可以通过实现实现 Ordered 接口 来控制运行顺序。

bean 工厂后处理器在 ApplicationContext 中声明时会自动运行，以便将更改应用于定义容器的配置元数据。 Spring 包含许多预定义的 bean 工厂后处理器，例如 PropertyOverrideConfigurer 和 PropertySourcesPlaceholderConfigurer。

### 使用 FactoryBean 自定义实例化逻辑
您可以为本身是工厂的对象实现 org.springframework.beans.factory.FactoryBean 接口。

FactoryBean 接口是 Spring IoC 容器实例化逻辑的可插入点。 如果您有复杂的初始化代码，可以用 Java 更好地表达，而不是（可能）冗长的 XML，您可以创建自己的 FactoryBean，在该类中编写复杂的初始化，然后将您的自定义 FactoryBean 插入到容器中。

FactoryBean 接口提供了三种方法：

* T getObject()：返回此工厂创建的对象的实例。 实例可能会被共享，这取决于这个工厂是返回单例还是原型。
* boolean isSingleton()： 如果此 FactoryBean 返回单例，则返回 true，否则返回 false。 此方法的默认实现返回 true。
* Class<?> getObjectType()： 返回 getObject() 方法返回的对象类型，如果类型事先未知，则返回 null。

FactoryBean 概念和接口在 Spring Framework 中的许多地方使用。 超过 50 种 FactoryBean 接口的实现与 Spring 本身一起提供。

当您需要向容器请求实际的 FactoryBean 实例本身而不是它生成的 bean 时，请在调用 ApplicationContext 的 getBean() 方法时在 bean 的 id 前面加上与符号 (&)。 因此，对于具有 myBean id 的给定 FactoryBean，在容器上调用 getBean("myBean") 会返回 FactoryBean 的产品，而调用 getBean("&myBean") 会返回 FactoryBean 实例本身。

## 基于注解的方式配置容器
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

</beans>
```
[context:annotation-config/](context:annotation-config/) 元素隐式注册以下后处理器：

* ConfigurationClassPostProcessor
* AutowiredAnnotationBeanPostProcessor
* CommonAnnotationBeanPostProcessor
* PersistenceAnnotationBeanPostProcessor
* EventListenerMethodProcessor

>   @Required ,spring5.1以后废弃

### `@Autowired`
**用在构造函数上：**

```java
public class MovieRecommender {
  private final CustomerPreferenceDao customerPreferenceDao;
  @Autowired
  public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
      this.customerPreferenceDao = customerPreferenceDao;
  }
  // ...
}
```
从 Spring Framework 4.3 开始，如果目标 bean 只定义了一个构造函数，则不再需要在此类构造函数上添加 @Autowired 注释。 但是，如果有多个构造函数可用并且没有默认构造函数，则必须至少用 @Autowired 注释构造函数之一，以便指示容器使用哪个构造函数。 

**@Autowired 应用于 setter 方法**

```java
public class SimpleMovieLister {
  private MovieFinder movieFinder;
  @Autowired
  public void setMovieFinder(MovieFinder movieFinder) {
      this.movieFinder = movieFinder;
  }
  // ...
}
```
**应用在任意方法上**

```java
public class MovieRecommender {
  private MovieCatalog movieCatalog;
  private CustomerPreferenceDao customerPreferenceDao;
  @Autowired
  public void prepare(MovieCatalog movieCatalog,
          CustomerPreferenceDao customerPreferenceDao) {
      this.movieCatalog = movieCatalog;
      this.customerPreferenceDao = customerPreferenceDao;
  }
  // ...
}
```
**应用于字段，甚至将其与构造函数混合使用**

```java
public class MovieRecommender {
  private final CustomerPreferenceDao customerPreferenceDao;
  @Autowired
  private MovieCatalog movieCatalog;
  @Autowired
  public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
      this.customerPreferenceDao = customerPreferenceDao;
  }
  // ...
}
```
您还可以通过将 @Autowired 注释添加到需要该类型数组的字段或方法来指示 Spring 从 ApplicationContext 提供特定类型的所有 bean：

```java
public class MovieRecommender {
  @Autowired
  private MovieCatalog[] movieCatalogs;

  // ...
}
```
这同样适用于类型化集合，如以下示例所示：

```java
public class MovieRecommender {
  private Set<MovieCatalog> movieCatalogs;

  @Autowired
  public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
      this.movieCatalogs = movieCatalogs;
  }

  // ...
}
```
如果您希望数组或列表中的项目按特定顺序排序，您的目标 bean 可以实现 org.springframework.core.Ordered 接口或使用 @Order 或标准 @Priority 注释。 否则，它们的顺序遵循容器中相应目标 bean 定义的注册顺序。

您可以在目标类级别和 @Bean 方法上声明 @Order 注释。

请注意，标准 javax.annotation.Priority 注释在 @Bean 级别不可用，因为它不能在方法上声明。 它的语义可以通过@Order 值结合@Primary 在每个类型的单个bean 上建模。

只要预期的键类型是字符串，即使是类型化的 Map 实例也可以自动装配。 映射值包含预期类型的所有 bean，键包含相应的 bean 名称，如以下示例所示：



```java
public class MovieRecommender {
  private Map<String, MovieCatalog> movieCatalogs;

  @Autowired
  public void setMovieCatalogs(Map<String, MovieCatalog> movieCatalogs) {
      this.movieCatalogs = movieCatalogs;
  }

  // ...
}
```
默认情况下，当给定注入点没有匹配的候选 bean 时，自动装配失败。 对于声明的数组、集合或映射，至少需要一个匹配元素。

默认行为是将带注释的方法和字段视为指示所需的依赖项。 您可以更改此行为，如下例所示，通过将不可满足的注入点标记为非必需（即，通过将 @Autowired 中的 required 属性设置为 false），使框架能够跳过不可满足的注入点：

```java
public class SimpleMovieLister {
  private MovieFinder movieFinder;

  @Autowired(required = false)
  public void setMovieFinder(MovieFinder movieFinder) {
      this.movieFinder = movieFinder;
  }
}
```
如果非必需方法的依赖项（或其依赖项之一，如果有多个参数）不可用，则根本不会调用它。 在这种情况下，根本不会填充非必填字段，而是保留其默认值。

注入的构造函数和工厂方法参数是一种特殊情况，因为由于 Spring 的构造函数解析算法可能潜在地处理多个构造函数，@Autowired 中的 required 属性具有一些不同的含义。 构造函数和工厂方法参数在默认情况下是有效的，但在单构造函数场景中有一些特殊规则，例如多元素注入点（数组、集合、映射）在没有匹配的 bean 可用时解析为空实例。 这允许一种通用的实现模式，其中所有依赖项都可以在唯一的多参数构造函数中声明——例如，声明为没有 @Autowired 注释的单个公共构造函数。

或者，您可以通过 Java 8 的 java.util.Optional 表达特定依赖项的非必需性质，如以下示例所示：



```java
public class SimpleMovieLister {
  @Autowired
  public void setMovieFinder(Optional<MovieFinder> movieFinder) {
      ...
  }
}
```
从 Spring Framework 5.0 开始，您还可以使用 @Nullable 注释（任何包中的任何类型 — 例如，来自 JSR-305 的 javax.annotation.Nullable）或仅利用 Kotlin 内置的空安全支持：

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        ...
    }
}
```
您还可以将 @Autowired 用于众所周知的可解析依赖项的接口：BeanFactory、ApplicationContext、Environment、ResourceLoader、ApplicationEventPublisher 和 MessageSource。 这些接口及其扩展接口（例如 ConfigurableApplicationContext 或 ResourcePatternResolver）会自动解析，无需特殊设置。 以下示例自动装配 ApplicationContext 对象：

```java
public class MovieRecommender {
  @Autowired
  private ApplicationContext context;

  public MovieRecommender() {
  }

  // ...
}
```
@Autowired、@Inject、@Value 和 @Resource 注释由 Spring BeanPostProcessor 实现处理。 这意味着您不能在自己的 BeanPostProcessor 或 BeanFactoryPostProcessor 类型（如果有）中应用这些注释。 这些类型必须使用 XML 或 Spring @Bean 方法显式“连接”。

### @Primary
@Primary 表示当多个 bean 是自动装配到单值依赖项的候选者时，应优先考虑特定 bean。 如果候选中恰好存在一个主要 bean，则它成为自动装配的值。

```java
@Configuration
public class MovieConfiguration {
    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }
    @Bean
    public MovieCatalog secondMovieCatalog() { ... }
    // ...
}
```
```java
public class MovieRecommender {
   // MovieRecommender 自动装配到 firstMovieCatalog
    @Autowired
    private MovieCatalog movieCatalog;
    // ...
}
```
### @Qualifier
自动注入时，发现多个候选项，会根据注入项的名称匹配候选bean的名称，如果相同，则注入，否则失败。@Qualifier可以指定这个名称，使两者一致：

```java
public class MovieRecommender {
    @Autowired
    @Qualifier("secondMovieCatalog")
    private MovieCatalog movieCatalog;
    // ...
}
```
@Qualifier是元注解。

### `@Value`
@Value 通常用于注入外化属性：

```Plain Text
@Configuration
@PropertySource("classpath:application.properties") //引入属性文件
public class AppConfig { }

@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name}") String catalog) {
        this.catalog = catalog;
    }
}
```
Spring 提供了一个默认的宽松嵌入值解析器。 它将尝试解析属性值，如果无法解析，则属性名称（例如 \${catalog.name}）将作为值注入。 如果你想对不存在的值保持严格的控制，你应该声明一个 PropertySourcesPlaceholderConfigurer bean，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```
> 使用 JavaConfig 配置 PropertySourcesPlaceholderConfigurer 时，@Bean 方法必须是静态的。

如果无法解析任何 \${} 占位符，则使用上述配置可确保 Spring 初始化失败。 也可以使用 setPlaceholderPrefix、setPlaceholderSuffix 或 setValueSeparator 等方法来自定义占位符。

Spring Boot 默认配置一个 PropertySourcesPlaceholderConfigurer bean，它将从 application.properties 和 application.yml 文件中获取属性。

Spring 提供的内置转换器支持允许自动处理简单的类型转换（例如到 Integer 或 int）。 多个逗号分隔的值可以自动转换为 String 数组，无需额外的努力。可以提供如下默认值：

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name:defaultCatalog}") String catalog) {
        this.catalog = catalog;
    }
}
```
Spring BeanPostProcessor 在幕后使用 ConversionService 来处理将 @Value 中的 String 值转换为目标类型的过程。 如果您想为您自己的自定义类型提供转换支持，您可以提供您自己的 ConversionService bean 实例，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
        conversionService.addConverter(new MyCustomConverter());
        return conversionService;
    }
}
```
当 @Value 包含 SpEL 表达式时，该值将在运行时动态计算，如下例所示：

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("#{systemProperties['user.catalog'] + 'Catalog' }") String catalog) {
        this.catalog = catalog;
    }
}
```
SpEL 还支持使用更复杂的数据结构：

```Plain Text
@Component
public class MovieRecommender {

    private final Map<String, Integer> countOfMoviesPerCatalog;

    public MovieRecommender(
            @Value("#{{'Thriller': 100, 'Comedy': 300}}") Map<String, Integer> countOfMoviesPerCatalog) {
        this.countOfMoviesPerCatalog = countOfMoviesPerCatalog;
    }
}
```
### 扫描并注册bean
您可以使用注释（例如@Component）、AspectJ 类型表达式或您自己的自定义过滤条件来选择哪些类具有向容器注册的 bean 定义。

#### `@Component`
`@Component`是元注解，基于此派生了@Service 和 @Controller和 @Repository。被 `@Component`及其派生注解标注的类，都会被扫描，然后被spring注册成bean，则默认 bean 名称是开头小写的非限定类名称,开启扫描：

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    // ...
}
```
xml替代：

```xml
    <context:component-scan base-package="org.example"/>
```
> [context:component-scan](context:component-scan) 的使用隐式启用了 [context:annotation-config](context:annotation-config) 的功能。 使用 [context:component-scan](context:component-scan) 时，通常不需要包含 [context:annotation-config](context:annotation-config) 元素。

此外，当您使用组件扫描元素时， AutowiredAnnotationBeanPostProcessor 和 CommonAnnotationBeanPostProcessor 都隐式包含在内。 您可以通过包含值为 false 的 annotation-config 属性来禁用 AutowiredAnnotationBeanPostProcessor 和 CommonAnnotationBeanPostProcessor 的注册。

@ComponentScan 注释的 includeFilters 或 excludeFilters 属性定义扫描的范围（或作为 XML 配置中 [context:component-scan](context:component-scan) 元素的 <context:include-filter /> 或 <context:exclude-filter /> 子元素）。 每个过滤器元素都需要 type 和 expression 属性。 下表描述了过滤选项：

| Filter Type          | Example Expression           | Description                                                  |
| :------------------- | :--------------------------- | :----------------------------------------------------------- |
| annotation (default) | `org.example.SomeAnnotation` | An annotation to be *present* or *meta-present* at the type level in target components. |
| assignable           | `org.example.SomeClass`      | A class (or interface) that the target components are assignable to (extend or implement). |
| aspectj              | `org.example..*Service+`     | An AspectJ type expression to be matched by the target components. |
| regex                | `org\.example\.Default.*`    | A regex expression to be matched by the target components' class names. |
| custom               | `org.example.MyTypeFilter`   | A custom implementation of the `org.springframework.core.type.TypeFilter` interface. |

```java
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    // ...
}
```
#### 在组件中定义 Bean 元数据
可以在@Component标注的类中定义bean元数据：

```java
@Component
public class FactoryMethodComponent {

    private static int i;

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    // use of a custom qualifier and autowiring of method parameters
    @Bean
    protected TestBean protectedInstance(
            @Qualifier("public") TestBean spouse,
            @Value("#{privateInstance.age}") String country) {
        TestBean tb = new TestBean("protectedInstance", 1);
        tb.setSpouse(spouse);
        tb.setCountry(country);
        return tb;
    }

    @Bean
    private TestBean privateInstance() {
        return new TestBean("privateInstance", i++);
    }

    @Bean
    @RequestScope
    public TestBean requestScopedInstance() {
        return new TestBean("requestScopedInstance", 3);
    }
}
```
您可以将@Bean 方法声明为静态方法，允许在不将其包含的配置类创建为实例的情况下调用它们。 这在定义后处理器 bean（例如，BeanFactoryPostProcessor 或 BeanPostProcessor 类型）时特别有意义，因为此类 bean 在容器生命周期的早期被初始化，并且应避免在那时触发配置的其他部分。

由于技术限制，对静态 @Bean 方法的调用永远不会被容器拦截，即使在 @Configuration 类中也不行：CGLIB 子类化只能覆盖非静态方法。 

@Bean 方法的 Java 语言可见性不会对 Spring 容器中生成的 bean 定义产生直接影响。 您可以在非@Configuration 类以及任何地方的静态方法中自由地声明您认为合适的工厂方法。 但是，@Configuration 类中的常规@Bean 方法需要是可覆盖的 — 也就是说，它们不能被声明为private 或final。

@Bean 方法也可以在给定组件或配置类的基类上发现，以及在组件或配置类实现的接口中声明的 Java 8 默认方法上。 这为组合复杂的配置安排提供了很大的灵活性，甚至可以通过 Java 8 默认方法（从 Spring 4.2 开始）进行多重继承。

最后，单个类可以为同一个 bean 保存多个 @Bean 方法，作为多个工厂方法的安排，根据运行时的可用依赖项使用。 这与在其他配置场景中选择“最贪婪”的构造函数或工厂方法的算法相同：在构造时选择具有最多可满足依赖项的变体，类似于容器如何在多个 @Autowired 构造函数之间进行选择。

实现BeanNameGenerator接口，重新定义注册的bean名称 ：

```java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
    // ...
}
```
由于bean名称是简单类名，可能出现重复的情况，您可能需要配置一个 BeanNameGenerator，该 BeanNameGenerator 默认为该类的完全限定类名，spring提供了FullyQualifiedAnnotationBeanNameGenerator。

#### @Scope
@Scope 用在类上或@Bean方法上，要为范围解析提供自定义策略而不是依赖基于注释的方法，您可以实现 ScopeMetadataResolver 接口：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
    // ...
}
```
> 虽然类路径扫描非常快，但可以通过在编译时创建一个静态候选列表来提高大型应用程序的启动性能。 需要添加依赖：

```bash

dependencies {
    annotationProcessor "org.springframework:spring-context-indexer:5.3.10"
}
```
> spring-context-indexer 生成一个 META-INF/spring.components 文件，该文件包含在 jar 文件中。当在类路径上找到 META-INF/spring.components 文件时，索引会自动启用。

## 基于java代码配置bean
> 完整的@Configuration 与“轻量级”@Bean 模式？

> 当 @Bean 方法在没有用 @Configuration 注释的类中声明时，它们被称为在“精简”模式下处理。 在@Component 声明的 Bean 方法被认为是“轻量级”的。 例如，@Component类上声明的@Bean 方法向容器公开管理视图。 在这种情况下，@Bean 方法是一种通用的工厂方法机制。

> 与完整的@Configuration 不同，lite @Bean 方法不能声明 bean 间的依赖关系。  因此，这样的 @Bean 方法不应调用其他 @Bean 方法。 每个这样的方法实际上只是特定 bean 引用的工厂方法，没有任何特殊的运行时语义。 这样的好处是声明的bean不会被 CGLIB 代理。

> 在常见情况下，@Bean 方法将在 @Configuration 类中声明，确保始终使用“完整”模式，多次调用bean方法，只会生成一个实例。

 Spring 3.0 中引入的  AnnotationConfigApplicationContext，不仅能够接受 @Configuration 类作为输入，还能够接受普通的 @Component 类和用 JSR-330 元数据注释的类。

@Configuration 类本身被注册为一个 bean 定义，并且该类中所有声明的 @Bean 方法也被注册为 bean 定义。

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```
### `@Configuration`
@Configuration 是一个类级别的注解，通过@Bean 注释的方法声明bean，对@Bean 方法的调用也可用于定义bean 间的依赖关系：

```java
@Configuration
public class AppConfig {
    @Bean
    public BeanOne beanOne() {
        return new BeanOne(beanTwo());
    }
    @Bean
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```
这种声明 bean 间依赖关系的方法只有在 @Configuration 类中声明了 @Bean 方法时才有效。 您不能使用普通的 @Component 类来声明 bean 间的依赖关系。

查找方法注入是一项您应该很少使用的高级功能。 在单例范围的 bean 依赖于原型范围的 bean 的情况下，它很有用。 

```java
public abstract class CommandManager {
    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}
```
```java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
    AsyncCommand command = new AsyncCommand();
    // inject dependencies here as required
    return command;
}

@Bean
public CommandManager commandManager() {
    // return new anonymous implementation of CommandManager with createCommand()
    // overridden to return a new prototype Command object
    return new CommandManager() {
        protected Command createCommand() {
            return asyncCommand();
        }
    }
}
```
```java
@Configuration
public class AppConfig {

    @Bean
    public ClientService clientService1() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientService clientService2() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientDao clientDao() {
        return new ClientDaoImpl();
    }
}
```
clientDao()方法被调用了两次，但是返回了相同的实例。：所有@Configuration 类在启动时使用 CGLIB 进行子类化。 在子类中，子方法在调用父方法并创建新实例之前，首先检查容器中是否有任何缓存的（作用域）bean。

### `@Import`注解
@Import 注释允许从另一个配置类加载 @Bean 定义：

```java
@Configuration
public class ConfigA {

    @Bean
    public A a() {
        return new A();
    }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```
现在，无需在实例化上下文时同时指定 ConfigA.class 和 ConfigB.class，只需显式提供 ConfigB，如下例所示：

```Plain Text
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

    // now both beans A and B will be available...
    A a = ctx.getBean(A.class);
    B b = ctx.getBean(B.class);
}
```
从 Spring Framework 4.2 开始，@Import 还支持对常规组件类的引用，类似于 AnnotationConfigApplicationContext.register 方法。 如果您想避免组件扫描，这将特别有用，通过使用一些配置类作为入口点来显式定义所有组件。

跨类引用bean:

```java
@Configuration
public class ServiceConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    @Bean
    public AccountRepository accountRepository(DataSource dataSource) {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```
还有另一种方法可以达到相同的结果，@Configuration 类最终只是容器中的另一个 bean：这意味着它们可以利用 @Autowired 和 @Value 注入配置类，然后调用相关bean方法。

确保您以这种方式注入的依赖项只是最简单的类型。 @Configuration 类在上下文的初始化过程中被处理得很早，并且强制以这种方式注入依赖项可能会导致意外的早期初始化。 尽可能使用基于参数的注入，如前面的示例所示。

另外，通过@Bean 定义 BeanPostProcessor 和 BeanFactoryPostProcessor 时要特别小心。 这些通常应该声明为静态@Bean 方法，而不是触发包含它们的配置类的实例化。 否则，@Autowired 和@Value 可能不适用于配置类本身，因为可能在 AutowiredAnnotationBeanPostProcessor 之前将其创建为 bean 实例。

以下示例显示了如何将一个 bean 自动装配到另一个 bean：

```Plain Text
@Configuration
public class ServiceConfig {

    @Autowired
    private AccountRepository accountRepository;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    private final DataSource dataSource;

    public RepositoryConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```
但是上面方式，开发人员无法直接看到AccountRepository注入的详情，可以通过下面的方式：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        // navigate 'through' the config class to the @Bean method!
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}
```
### java配置和xml配置混合使用
Spring 的 @Configuration 类支持并不旨在 100% 完全替代 Spring XML，大多数情况下，两者都是混合使用的。

xml为主，@Configuration 类辅助：

```java
@Configuration
public class AppConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }

    @Bean
    public TransferService transferService() {
        return new TransferService(accountRepository());
    }
}
```
```xml
<beans>
    <!-- enable processing of annotations such as @Autowired and @Configuration -->
    <context:annotation-config/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="com.acme.AppConfig"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```
@Configuration为主，xml为辅：

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}
```
```xml
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```
## 环境抽象
Environment 接口是集成在容器中的抽象，它对应用程序环境的两个关键方面进行建模：配置文件和属性。

* 配置文件是一个命名的、逻辑的 bean 定义组，仅当给定的配置文件处于活动状态时才向容器注册。 Bean 可以分配给配置文件，无论是在 XML 中定义还是使用注释。 与配置文件相关的环境对象的作用是确定哪些配置文件（如果有）当前是活动的，以及默认情况下哪些配置文件（如果有）应该是活动的。
* 属性在几乎所有应用程序中都扮演着重要的角色，并且可能来自各种来源：属性文件、JVM 系统属性、系统环境变量、JNDI、servlet 上下文参数、ad-hoc Properties 对象、Map 对象等等。 与属性相关的 Environment 对象的作用是为用户提供方便的服务接口，用于配置属性源并从中解析属性。

### @Profile
@Profile（元注解） 注解实际上是通过@Conditional 来实现的:

```java
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    // Read the @Profile annotation attributes
    MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
    if (attrs != null) {
        for (Object value : attrs.get("value")) {
            if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
                return true;
            }
        }
        return false;
    }
    return true;
}
```
示例：

```java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}
```
```java
@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```
@Profile支持一些简单的表达式：

* `!`: A logical “not” of the profile
* `&`: A logical “and” of the profiles
* `|`: A logical “or” of the profiles

@Profile可以声明在方法级别上：

```java
@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Profile("development") 
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Profile("production") 
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```
代码操作profile:

```Plain Text
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```
jvm参数指定profile:

```Plain Text
    -Dspring.profiles.active="profile1,profile2"
```
默认配置文件代表默认启用的配置文件。 考虑以下示例：

```Plain Text
@Configuration
@Profile("default")
public class DefaultDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .build();
    }
}
```
您可以通过在 Environment 上使用 setDefaultProfiles() 或声明性地使用 spring.profiles.default 属性来更改默认配置文件的名称。

#### `PropertySource` 抽象
spring对属性操作是通过Environment接口实现的：

```java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```
Spring 的 StandardEnvironment 配置了两个 PropertySource 对象 : 一个表示 JVM 系统属性集（System.getProperties()）和一个表示系统环境变量集（ System.getenv())。

执行的搜索是分层的。 默认情况下，系统属性优先于环境变量。 因此，如果在调用 env.getProperty("my-property") 期间碰巧在两个地方都设置了 my-property 属性，则系统属性值“获胜”并返回。 请注意，属性值不会合并，而是被前面的条目完全覆盖。

对于常见的 StandardServletEnvironment，完整的层次结构如下，最高优先级的条目位于顶部：

* ServletConfig 参数（如果适用 — 例如，在 DispatcherServlet 上下文的情况下）
* ServletContext 参数（web.xml 上下文参数条目）
* JNDI 环境变量（java:comp/env/ 条目）
* JVM 系统属性（-D 命令行参数）
* JVM系统环境（操作系统环境变量）

最重要的是，整个机制是可配置的。 也许您有想要集成到此搜索中的自定义属性源。 为此，请实现并实例化您自己的 PropertySource，并将其添加到当前环境的 PropertySource 集中。 以下示例显示了如何执行此操作：

```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```
@PropertySource 注解可以代替上面的代码：

```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```
@PropertySource支持占位符：

```java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```
## `ApplicationContext`其他功能
### 使用 MessageSource 进行国际化
ApplicationContext 接口扩展了一个名为 MessageSource 的接口，因此提供了国际化（“i18n”）功能。 Spring 还提供了 HierarchicalMessageSource 接口，可以分层解析消息。 这些接口一起提供了 Spring 影响消息解析的基础。 在这些接口上定义的方法包括：

* String getMessage(String code, Object\[\] args, String default, Locale loc)：用于从 MessageSource 检索消息的基本方法。 如果没有找到指定语言环境的消息，则使用默认消息。 使用标准库提供的 MessageFormat 功能，传入的任何参数都成为替换值。
* String getMessage(String code, Object\[\] args, Locale loc)：与前面的方法基本相同，但有一个区别：不能指定默认消息。 如果找不到消息，则抛出 NoSuchMessageException。
* String getMessage(MessageSourceResolvable resolvable, Locale locale)：上述方法中使用的所有属性也都封装在一个名为 MessageSourceResolvable 的类中，您可以在此方法中使用该类。

加载 ApplicationContext 时，它会自动搜索上下文中定义的 MessageSource bean。 bean 必须具有名称 messageSource。 如果找到这样的 bean，则对上述方法的所有调用都将委托给消息源。 如果未找到消息源，则 ApplicationContext 会尝试查找包含同名 bean 的父级。 如果是，则使用该 bean 作为 MessageSource。 如果 ApplicationContext 找不到任何消息源，则会实例化一个空的 DelegatingMessageSource，以便能够接受对上面定义的方法的调用。

Spring 提供了三个 MessageSource 实现，ResourceBundleMessageSource、ReloadableResourceBundleMessageSource 和 StaticMessageSource。 它们都实现 HierarchicalMessageSource 以进行嵌套消息传递。 StaticMessageSource 很少使用，但提供了向源添加消息的编程方式。 以下示例显示了 ResourceBundleMessageSource：

```xml
<beans>
    <bean id="messageSource"
            class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>
                <value>exceptions</value>
                <value>windows</value>
            </list>
        </property>
    </bean>
</beans>
```
该示例假设您在类路径中定义了三个名为 format、exceptions 和 windows 的资源包。 任何解析消息的请求都以 JDK 标准的方式通过 ResourceBundle 对象解析消息来处理。 出于示例的目的，假设上述两个资源包文件的内容如下：

```Plain Text
    # in format.properties
    message=Alligators rock!
```
```Plain Text
# in exceptions.properties
argument.required=The {0} argument is required.
```
下一个示例显示了一个运行 MessageSource 功能的程序。 请记住，所有 ApplicationContext 实现也是 MessageSource 实现，因此可以转换为 MessageSource 接口:

```java
public static void main(String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("message", null, "Default", Locale.ENGLISH);
    System.out.println(message);
}
```
总而言之，MessageSource 是在名为 beans.xml 的文件中定义的，该文件位于类路径的根目录中。 messageSource bean 定义通过其 basenames 属性引用了许多资源包。 在列表中传递给 basenames 属性的三个文件作为文件存在于类路径的根目录中，分别称为 format.properties、exceptions.properties 和 windows.properties。

下一个示例显示传递给消息查找的参数。 这些参数被转换为 String 对象并插入到查找消息中的占位符中。

```xml
<beans>

    <!-- this MessageSource is being used in a web application -->
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basename" value="exceptions"/>
    </bean>

    <!-- lets inject the above MessageSource into this POJO -->
    <bean id="example" class="com.something.Example">
        <property name="messages" ref="messageSource"/>
    </bean>

</beans>
```
```java
public class Example {

    private MessageSource messages;

    public void setMessages(MessageSource messages) {
        this.messages = messages;
    }

    public void execute() {
        String message = this.messages.getMessage("argument.required",
            new Object [] {"userDao"}, "Required", Locale.ENGLISH);
        System.out.println(message);
    }
}
```
调用 execute() 方法的结果输出如下：

```Plain Text
The userDao argument is required.
```
关于国际化（“i18n”），Spring 的各种 MessageSource 实现遵循与标准 JDK ResourceBundle 相同的区域设置解析和回退规则。 简而言之，并继续之前定义的示例 messageSource，如果您想根据英国 (en-GB) 语言环境解析消息，您将分别创建名为 format\_en\_GB.properties、exceptions\_en\_GB.properties 和 windows\_en\_GB.properties 的文件。

通常，区域设置解析由应用程序的周围环境管理。 在以下示例中，手动指定解析（英国）消息所针对的区域设置：

```Plain Text
# in exceptions_en_GB.properties
argument.required=Ebagum lad, the ''{0}'' argument is required, I say, required.
```
```java
public static void main(final String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("argument.required",
        new Object [] {"userDao"}, "Required", Locale.UK);
    System.out.println(message);
}
```
运行上述程序的结果如下：

```Plain Text
Ebagum lad, the 'userDao' argument is required, I say, required.
```
您还可以使用 MessageSourceAware 接口获取对已定义的任何 MessageSource 的引用。 创建和配置 bean 时，在实现 MessageSourceAware 接口的 ApplicationContext 中定义的任何 bean 都会注入应用程序上下文的 MessageSource。

因为 Spring 的 MessageSource 基于 Java 的 ResourceBundle，所以它不会合并具有相同基名的 bundle，而只会使用找到的第一个 bundle。 具有相同基本名称的后续消息包将被忽略。

作为 ResourceBundleMessageSource 的替代方案，Spring 提供了一个 ReloadableResourceBundleMessageSource 类。 此变体支持相同的包文件格式，但比基于标准 JDK 的 ResourceBundleMessageSource 实现更灵活。 特别是，它允许从任何 Spring 资源位置（不仅从类路径）读取文件，并支持包属性文件的热重载（同时在它们之间有效地缓存它们）。 有关详细信息，请参阅 ReloadableResourceBundleMessageSource javadoc。

### 标准和自定义事件
ApplicationContext 中的事件处理是通过 ApplicationEvent 类和 ApplicationListener 接口提供的。 如果将实现 ApplicationListener 接口的 bean 部署到上下文中，则每次将 ApplicationEvent 发布到 ApplicationContext 时，都会通知该 bean。 本质上，这是标准的观察者设计模式。

| Event                        | Explanation                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| `ContextRefreshedEvent`      | 在 ApplicationContext 初始化或刷新时发布（例如，通过使用 ConfigurableApplicationContext 接口上的 refresh() 方法）。 这里，“初始化”意味着所有 bean 都被加载，后处理器 bean 被检测并激活，单例被预实例化，ApplicationContext 对象准备好使用。 只要上下文尚未关闭，就可以多次触发刷新，前提是所选的 ApplicationContext 实际上支持这种“热”刷新。 例如，XmlWebApplicationContext 支持热刷新，但 GenericApplicationContext 不支持。 |
| `ContextStartedEvent`        | 使用 `ConfigurableApplicationContext` 接口上的 `start()` 方法启动 `ApplicationContext` 时发布。 在这里，“已启动”意味着所有“Lifecycle”bean 都收到一个明确的启动信号。 通常，此信号用于在显式停止后重新启动 bean，但它也可用于启动尚未配置为自动启动的组件（例如，尚未在初始化时启动的组件）。 |
| `ContextStoppedEvent`        | 使用 `ConfigurableApplicationContext` 接口上的 `stop()` 方法停止 `ApplicationContext` 时发布。 在这里，“停止”意味着所有“Lifecycle”bean 都收到一个明确的停止信号。 停止的上下文可以通过`start()` 调用重新启动。 |
| `ContextClosedEvent`         | 当使用`ConfigurableApplicationContext` 接口上的`close()` 方法或通过JVM 关闭挂钩关闭`ApplicationContext` 时发布。 在这里，“关闭”意味着所有的单例 bean 都将被销毁。 一旦上下文关闭，它就会到达生命的尽头，无法刷新或重新启动。 |
| `RequestHandledEvent`        | 一个网络特定的事件，告诉所有的bean一个HTTP请求已经被服务。请求完成后，该事件被发布。此事件仅适用于使用Spring的`DispatcherServlet` Web应用程序。 |
| `ServletRequestHandledEvent` | `RequestHandledEvent` 的子类，用于添加特定于 Servlet 的上下文信息。 |

您还可以创建和发布自己的自定义事件。 以下示例显示了一个扩展 Spring 的 ApplicationEvent 基类的简单类：

```Plain Text
public class BlockedListEvent extends ApplicationEvent {

    private final String address;
    private final String content;

    public BlockedListEvent(Object source, String address, String content) {
        super(source);
        this.address = address;
        this.content = content;
    }

    // accessor and other methods...
}
```
要发布自定义 ApplicationEvent，请在 ApplicationEventPublisher 上调用 publishEvent() 方法。 通常，这是通过创建一个实现 ApplicationEventPublisherAware 的类并将其注册为 Spring bean 来完成的。 以下示例显示了这样一个类：

```java
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blockedList;
    private ApplicationEventPublisher publisher;

    public void setBlockedList(List<String> blockedList) {
        this.blockedList = blockedList;
    }

    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String content) {
        if (blockedList.contains(address)) {
            publisher.publishEvent(new BlockedListEvent(this, address, content));
            return;
        }
        // send email...
    }
}
```
配置时，Spring容器检测到EmailService实现了ApplicationEventPublisherAware，并自动调用setApplicationEventPublisher()。 实际上，传入的参数是Spring容器本身。 您正在通过其 ApplicationEventPublisher 接口与应用程序上下文进行交互。

要接收自定义 ApplicationEvent，您可以创建一个实现 ApplicationListener 的类并将其注册为 Spring bean。 以下示例显示了这样一个类：

```java
public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```
请注意，ApplicationListener 通常使用自定义事件的类型（在前面的示例中为 BlockedListEvent）进行参数化。 这意味着 onApplicationEvent() 方法可以保持类型安全，避免任何向下转换的需要。 您可以根据需要注册任意数量的事件侦听器，但请注意，默认情况下，事件侦听器会同步接收事件。 这意味着 publishEvent() 方法会阻塞，直到所有侦听器都完成对事件的处理。 这种同步和单线程方法的一个优点是，当侦听器接收到事件时，如果事务上下文可用，它就会在发布者的事务上下文中运行。 如果需要另一种事件发布策略，请参阅 Spring 的 ApplicationEventMulticaster 接口和 SimpleApplicationEventMulticaster 实现的 javadoc 以了解配置选项。

以下示例显示了用于注册和配置上述每个类的 bean 定义：

```java
<bean id="emailService" class="example.EmailService">
    <property name="blockedList">
        <list>
            <value>known.spammer@example.org</value>
            <value>known.hacker@example.org</value>
            <value>john.doe@example.org</value>
        </list>
    </property>
</bean>

<bean id="blockedListNotifier" class="example.BlockedListNotifier">
    <property name="notificationAddress" value="blockedlist@example.org"/>
</bean>
```
总而言之，当调用 emailService bean 的 sendEmail() 方法时，如果有任何电子邮件消息应该被阻止，则会发布一个 BlockedListEvent 类型的自定义事件。 BlockedListNotifier bean 注册为 ApplicationListener 并接收 BlockedListEvent，此时它可以通知适当的各方。

Spring 的事件机制是为同一应用程序上下文中 Spring bean 之间的简单通信而设计的。 然而，对于更复杂的企业集成需求，单独维护的 Spring Integration 项目为构建基于众所周知的 Spring 编程模型的轻量级、面向模式、事件驱动的体系结构提供了完整的支持。

#### 基于注解的事件监听器
您可以使用 @EventListener 注释在托管 bean 的任何方法上注册事件侦听器。 BlockedListNotifier 可以重写如下：

```java
@Compoent
public class BlockedListNotifier {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    @EventListener
    public void processBlockedListEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```
方法签名再次声明了它侦听的事件类型，但这次使用了灵活的名称并且没有实现特定的侦听器接口。 只要实际事件类型在其实现层次结构中解析您的泛型参数，也可以通过泛型缩小事件类型。

如果您的方法应该侦听多个事件，或者您想定义它时根本不带参数，则还可以在注释本身上指定事件类型。 以下示例显示了如何执行此操作：

```java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
    // ...
}
```
还可以通过使用定义 SpEL 表达式的注释的条件属性添加额外的运行时过滤，该表达式应该匹配以实际调用特定事件的方法。以下示例显示了如何重写我们的通知程序以仅在事件的内容属性等于 my-event 时才被调用：

```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlockedListEvent(BlockedListEvent blEvent) {
    // notify appropriate parties via notificationAddress...
}
```
每个 SpEL 表达式都针对专用上下文进行评估。 下表列出了可用于上下文的项目，以便您可以将它们用于条件事件处理：

| Name            | Location           | Description                                                  | Example                                                      |
| :-------------- | :----------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Event           | root object        | The actual `ApplicationEvent`.                               | `#root.event` or `event`                                     |
| Arguments array | root object        | The arguments (as an object array) used to invoke the method. | `#root.args` or `args`; `args[0]` to access the first argument, etc. |
| *Argument name* | evaluation context | The name of any of the method arguments. If, for some reason, the names are not available (for example, because there is no debug information in the compiled byte code), individual arguments are also available using the `#a<#arg>` syntax where `<#arg>` stands for the argument index (starting from 0). | `#blEvent` or `#a0` (you can also use `#p0` or `#p<#arg>` parameter notation as an alias) |

请注意，#root.event 可让您访问底层事件，即使您的方法签名实际上是指已发布的任意对象。

如果您需要发布一个事件作为处理另一个事件的结果，您可以更改方法签名以返回应该发布的事件，如以下示例所示：

```java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```
handleBlockedListEvent() 方法为它处理的每个 BlockedListEvent 发布一个新的 ListUpdateEvent。 如果您需要发布多个事件，则可以返回一个集合或事件数组。

#### 异步监听
如果您希望特定的侦听器异步处理事件，则可以重用常规的 @Async 支持。 以下示例显示了如何执行此操作：

```java
@EventListener
@Async
public void processBlockedListEvent(BlockedListEvent event) {
    // BlockedListEvent is processed in a separate thread
}
```
使用异步事件时请注意以下限制：

如果异步事件侦听器抛出异常，则不会将其传播给调用者。 有关更多详细信息，请参阅 AsyncUncaughtExceptionHandler。

异步事件侦听器方法无法通过返回值来发布后续事件。 如果您需要发布另一个事件作为处理结果，请注入 ApplicationEventPublisher 以手动发布事件。

#### 监听器的顺序
如果您需要在另一个侦听器之前调用一个侦听器，则可以在方法声明中添加 @Order 注释，如下例所示：

```java
@EventListener
@Order(42)
public void processBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress...
}
```
#### 泛型事件
您还可以使用泛型进一步定义事件的结构。 考虑使用 EntityCreatedEvent ，其中 T 是创建的实际实体的类型。 例如，您可以创建以下侦听器定义以仅接收 Person 的 EntityCreatedEvent：

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    // ...
}
```
由于类型擦除，这仅在被触发的事件解析事件侦听器过滤的通用参数时才有效（即类 PersonCreatedEvent extends EntityCreatedEvent { ... }）。

在某些情况下，如果所有事件都遵循相同的结构（如前面示例中的事件应该是这种情况），这可能会变得非常乏味。 在这种情况下，您可以实现 ResolvableTypeProvider 来指导框架超出运行时环境提供的范围。 以下事件显示了如何执行此操作：

```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

    public EntityCreatedEvent(T entity) {
        super(entity);
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
    }
}
```
### 便捷访问底层资源
为了最佳地使用和理解应用程序上下文，您应该熟悉 Spring 的 Resource 抽象，如参考资料中所述。

应用程序上下文是一个 ResourceLoader，可用于加载 Resource 对象。 资源本质上是 JDK java.net.URL 类的功能更丰富的版本。 事实上，Resource 的实现在适当的地方包装了一个 java.net.URL 的实例。 Resource 可以以透明的方式从几乎任何位置获取低级资源，包括从类路径、文件系统位置、用标准 URL 描述的任何位置以及其他一些变体。 如果资源位置字符串是一个没有任何特殊前缀的简单路径，那么这些资源的来源是特定的并且适合实际的应用程序上下文类型。

您可以将部署到应用程序上下文中的 bean 配置为实现特殊的回调接口 ResourceLoaderAware，以在初始化时自动回调应用程序上下文本身作为 ResourceLoader 传入。 您还可以公开 Resource 类型的属性，用于访问静态资源。 它们像任何其他属性一样被注入其中。 您可以将这些 Resource 属性指定为简单的 String 路径，并在部署 bean 时依赖于从这些文本字符串到实际 Resource 对象的自动转换。

提供给 ApplicationContext 构造函数的位置路径或路径实际上是资源字符串，并且以简单的形式根据特定的上下文实现进行适当处理。 例如 ClassPathXmlApplicationContext 将简单的位置路径视为类路径位置。 您还可以使用带有特殊前缀的位置路径（资源字符串）来强制从类路径或 URL 加载定义，而不管实际的上下文类型。





```java
package cn.zhao.beans;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.SmartLifecycle;
import org.springframework.stereotype.Service;

@Service
public class TestImpl implements ITest, ApplicationContextAware, InitializingBean, SmartLifecycle {

    private User user;

    public TestImpl() {
        System.err.println("constructor ");
    }

    @Autowired
    public void setUser(User user) {
        System.err.println("setter ");
        this.user = user;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.err.println("set context");
    }


    @Override
    public void afterPropertiesSet() throws Exception {
        System.err.println("init ");
    }

    @Override
    public void start() {
        System.err.println("start ... ");
    }

    @Override
    public void stop() {
        System.err.println("stop");
    }

    @Override
    public boolean isRunning() {
        return true;
    }
}


public class ApplicationTest2 {
    public static void main(String[] args) throws InterruptedException {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        applicationContext.start();
        applicationContext.getBean(ITest.class);
        System.err.println("get bean end");
        Thread.sleep(10000);

        applicationContext.stop();
        System.err.println("容器 stop");

        applicationContext.start();
        System.err.println("容器 start2");

        applicationContext.close();
        System.err.println("容器 close");

        Thread.sleep(10000);

    }
}

```
```bash
appConfig before...
appConfig after...
constructor 
user before...
user after...
setter 
set context
testImpl before...
init 
testImpl after...
get bean end
stop
容器 stop
容器 start2
stop
容器 close
```
