---
title: graphql-spring
tags:
  - java
  - spring
  - graphql
categories:
  - java
date: 2022-11-27 19:02:13
---

# 注解驱动

Spring for GraphQL提供了一个基于注释的编程模型，其中@Controller组件使用注释来声明具有灵活方法签名的处理程序方法，以获取特定GraphQL字段的数据。例如：

```java
@Controller
    public class GreetingController {

        @QueryMapping 
        public String hello() { 
            return "Hello, world!";
        }

    }
```

1. 将此方法绑定到查询，即查询类型下的字段。
2. 如果未在注释上声明，则根据方法名确定查询。

Spring使用RuntimeWiring.Builder将上述处理程序方法注册为名为“hello”的查询graphql.schema.DataFetcher。

![img](graphql-spring/1668918641210-4300ec25-8d15-4736-bc31-056b67ee95aa.png)

AnnotatedControllerConfigurer 检测 @Controller bean 并通过 RuntimeWiring.Builder 将标注的方法注册为 DataFetchers。 它是 RuntimeWiringConfigurer 的一个实现，可以添加到 GraphQlSource.Builder。 Spring Boot 自动将 AnnotatedControllerConfigurer 声明为 bean，并将所有 RuntimeWiringConfigurer bean 添加到 GraphQlSource.Builder 并启用对带注释的 DataFetchers 的支持。

## @SchemaMapping

@SchemaMapping 注解将方法映射到 GraphQL schema中的字段，并将其声明为该字段的 DataFetcher。 注解可以指定类型名称，以及字段名称：

```java
@Controller
public class BookController {

    @SchemaMapping(typeName="Book", field="author")
    public Author getAuthor(Book book) {
        // ...
    }
}
```

@SchemaMapping 注解也可以省略这些属性，在这种情况下，字段名称默认为方法名称，而类型名称默认为方法参数的简单类名称。 例如，下面默认键入Book和字段author：

```java
@Controller
public class BookController {

    @SchemaMapping
    public Author author(Book book) {
        // ...
    }
}
```

@SchemaMapping 注解可以在类级别声明，为类中的所有处理程序方法指定默认类型名称:

```java
@Controller
@SchemaMapping(typeName="Book")
public class BookController {

    // @SchemaMapping methods for fields of the "Book" type

}
```

@QueryMapping、@MutationMapping 和@SubscriptionMapping 是复合注解，它们本身使用@SchemaMapping 进行标注，并且typeName 分别预设为Query、Mutation 或Subscription。

### 方法签名

schema映射处理程序方法可以具有以下任何方法参数：

| Method Argument                | Description                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| @Argument                      | 用于访问绑定到更高级别、类型化对象的命名字段参数。           |
| @Arguments                     | 用于访问绑定到更高级别、类型化对象的所有字段参数。           |
| @Argument Map<String, Object>  | 用于访问参数的原始map，其中@Argument没有name属性。           |
| @Arguments Map<String, Object> | 用于访问参数的原始映射。                                     |
| @ProjectedPayload Interface    | 用于通过项目接口访问字段参数。                               |
| "Source"                       | 用于访问字段的源（即父/容器）实例。                          |
| DataLoader                     | 用于访问DataLoaderRegistry中的DataLoader。                   |
| @ContextValue                  | 用于从DataFetchingEnvironment中的主GraphQLContext访问属性。  |
| @LocalContextValue             | 用于从DataFetchingEnvironment中的本地GraphQLContext访问属性。 |
| GraphQLContext                 | 用于从DataFetchingEnvironment访问上下文。                    |
| java.security.Principal        | 从Spring Security上下文中获取（如果可用）。                  |
| @AuthenticationPrincipal       | 从Spring Security上下文访问Authentication#getPrincipal（）。 |
| DataFetchingFieldSelectionSet  | 通过DataFetchingEnvironment访问查询的选择集。                |
| Locale, Optional<Locale>       | 用于从DataFetchingEnvironment访问区域设置。                  |
| DataFetchingEnvironment        | 用于直接访问基础DataFetchingEnvironment。                    |

schema映射处理程序方法可以返回：

- 任何类型的解析值。
- Mono 和Flux 用于异步值。 支持控制器方法和响应式 DataFetcher 中描述的任何 DataFetcher。
- java.util.concurrent.Callable 以异步方式生成值。 为此，必须使用 Executor 配置 AnnotatedControllerConfigurer。

### @Argument

在 GraphQL Java 中，DataFetchingEnvironment 提供了对特定字段的参数值映射的访问。 这些值可以是简单的标量值（例如 String、Long）、用于更复杂输入的Map 或列表。

使用 @Argument 注释将参数绑定到目标对象并注入到处理程序方法中。 绑定是通过将参数值映射到预期方法参数类型的主数据构造函数来执行的，或者通过使用默认构造函数来创建对象，然后将参数值映射到它的属性。 这是递归重复的，使用所有嵌套的参数值并相应地创建嵌套的目标对象。 例如：

```java
query {
  bookById(id: "book-1") {
    id
    name
    author {
      id
      lastName
    }
  }
}
type Query {
    bookById(id:String):Book
}
@Controller
public class BookController {

    @QueryMapping
    public Book bookById(@Argument String id) {
        // ...
    }

    @MutationMapping
    public Book addBook(@Argument BookInput bookInput) {
        // ...
    }
}
```

默认情况下，如果方法参数名称可用，则它用于查找参数。 如果需要，您可以通过注解自定义名称，例如 @Argument("bookInput")。

如果绑定失败，则会引发 BindException，绑定问题累积为字段错误，其中每个错误的字段是发生问题的参数路径。

您可以将 @Argument 与 Map<String, Object> 参数一起使用，以获取所有参数值的原始映射。 不得设置 @Argument 上的 name 属性。

### @Arguments

@Arguments 绑定参数到对象类型，而 @Argument 则绑定到简单参数。

例如，@Argument BookInput bookInput 使用参数“bookInput”的值来初始化 BookInput对象。

您可以将 @Arguments 与 Map<String, Object> 参数一起使用，以获取所有参数值的原始映射。

### @ProjectedPayload

当 Spring Data 在类路径上时，参数投影由 Spring Data 的接口投影提供。

要使用它，请创建一个使用 @ProjectedPayload 注释的接口并将其声明为控制器方法参数。 如果参数使用@Argument 进行注释，则它适用于 DataFetchingEnvironment.getArguments() 映射中的单个参数。 在没有 @Argument 的情况下声明时，投影适用于完整参数映射中的顶级参数。

```java
@Controller
public class BookController {

    @QueryMapping
    public Book bookById(BookIdProjection bookId) {
        // ...
    }

    @MutationMapping
    public Book addBook(@Argument BookInputProjection bookInput) {
        // ...
    }
}

@ProjectedPayload
interface BookIdProjection {

    Long getId();
}

@ProjectedPayload
interface BookInputProjection {

    String getName();

    @Value("#{target.author + ' ' + target.name}")
    String getAuthorAndName();
}
```

### Source

在 GraphQL Java 中，DataFetchingEnvironment 提供对字段源（即父/容器）实例的访问。 要访问它，只需声明预期目标类型的方法参数。

```java
@Controller
public class BookController {

    @SchemaMapping
    public Author author(Book book) {
        // ...
    }
}
```

源方法参数还有助于确定映射的类型名称。 如果 Java 类的简单名称与 GraphQL 类型匹配，则无需在 @SchemaMapping 注解中显式指定类型名称。

### DataLoader

当您为实体注册批量加载功能时，如批量加载中所述，您可以通过声明 DataLoader 类型的方法参数来访问实体的 DataLoader 并使用它来加载实体：

```java
@Controller
public class BookController {

    public BookController(BatchLoaderRegistry registry) {
        registry.forTypePair(Long.class, Author.class).registerMappedBatchLoader((authorIds, env) -> {
            // return Map<Long, Author>
        });
    }

    @SchemaMapping
    public CompletableFuture<Author> author(Book book, DataLoader<Long, Author> loader) {
        return loader.load(book.getAuthorId());
    }

}
```

默认情况下，BatchLoaderRegistry 使用值类型的完整类名（例如 Author 的类名）作为注册键，因此只需使用泛型类型声明 DataLoader 方法参数即可提供足够的信息来在 DataLoaderRegistry 中定位它。 作为后备，DataLoader 方法参数解析器还将尝试将方法参数名称作为键，但通常这不是必需的。

请注意，对于许多加载相关实体的情况，@SchemaMapping 只是委托给 DataLoader，您可以使用下一节中描述的 @BatchMapping 方法来减少样板。

### Validation

当找到 javax.validation.Validator bean 时，AnnotatedControllerConfigurer 启用对带注释的控制器方法的 Bean Validation 的支持。 通常，bean 的类型为 LocalValidatorFactoryBean。

Bean 验证允许您声明类型的约束：

```java
public class BookInput {

    @NotNull
    private String title;

    @NotNull
    @Size(max=13)
    private String isbn;
}
```

然后，您可以使用 @Valid 注释控制器方法参数以在方法调用之前对其进行验证：

```java
@Controller
public class BookController {

    @MutationMapping
    public Book addBook(@Argument @Valid BookInput bookInput) {
        // ...
    }
}
```

如果在验证期间发生错误，则会引发 ConstraintViolationException。 您可以使用异常解决链来决定如何将其转换为错误以包含在 GraphQL 响应中，从而将其呈现给客户端。

Bean 验证对@Argument、@Arguments 和@ProjectedPayload 方法参数很有用，但更普遍地适用于任何方法参数。

## @BatchMapping

批量加载通过使用 org.dataloader.DataLoader 延迟加载单个实体实例来解决 N+1 选择问题，因此它们可以一起加载。 例如：

```java
@Controller
public class BookController {

    public BookController(BatchLoaderRegistry registry) {
        registry.forTypePair(Long.class, Author.class).registerMappedBatchLoader((authorIds, env) -> {
            // return Map<Long, Author>
        });
    }

    @SchemaMapping
    public CompletableFuture<Author> author(Book book, DataLoader<Long, Author> loader) {
        return loader.load(book.getAuthorId());
    }

}
```

对于加载关联实体的直接情况，如上所示，@SchemaMapping 方法只是委托给 DataLoader。 这是可以通过 @BatchMapping 方法避免的样板。 例如：

```java
@Controller
public class BookController {

    @BatchMapping
    public Mono<Map<Book, Author>> author(List<Book> books) {
        // ...
    }
}
```

以上成为 BatchLoaderRegistry 中的批量加载函数，其中键是 Book 实例，加载的值是它们的作者。 此外，DataFetcher 也透明地绑定到 Book 类型的 author 字段，它只是为作者委托 DataLoader，给定它的源/父 Book 实例。

默认情况下，字段名默认为方法名，而类型名默认为输入List元素类型的简单类名。 两者都可以通过注释属性进行自定义。 类型名称也可以从类级别@SchemaMapping 继承。



# 数据集成

Spring for GraphQL 允许您利用现有的 Spring 技术，遵循常见的编程模型公开底层数据源。

本节讨论 Spring Data 的集成层，它提供了一种将 Querydsl 或 Query by Example 存储库适应 DataFetcher 的简单方法，包括自动检测选项和标记为 @GraphQlRepository 的存储库的 GraphQL 查询注册。

## Querydsl

Spring for GraphQL 支持使用 Querydsl ，通过 Spring Data Querydsl 扩展来获取数据。 Querydsl 通过使用注释处理器生成元模型，提供了一种灵活但类型安全的方法来表达查询谓词。

例如，将存储库声明为 QuerydslPredicateExecutor：

```java
public interface AccountRepository extends Repository<Account, Long>,
            QuerydslPredicateExecutor<Account> {
}
```

然后用它来创建一个DataFetcher：

```java
// For single result queries
DataFetcher<Account> dataFetcher =
        QuerydslDataFetcher.builder(repository).single();

// For multi-result queries
DataFetcher<Iterable<Account>> dataFetcher =
        QuerydslDataFetcher.builder(repository).many();
```

现在可以通过 RuntimeWiringConfigurer 注册上述 DataFetcher。

DataFetcher 从 GraphQL 请求参数构建一个 Querydsl 谓词，并使用它来获取数据。 Spring Data 支持 JPA、MongoDB 和 LDAP 的 QuerydslPredicateExecutor。

如果存储库是 ReactiveQuerydslPredicateExecutor，则构建器返回 DataFetcher<Mono<Account>> 或 DataFetcher<Flux<Account>>。 Spring Data MongoDB 支持这种变体。

### 构建

要在您的构建中配置 Querydsl，请遵循[官方参考文档](https://querydsl.com/static/querydsl/latest/reference/html/ch02.html)：

```java
dependencies {
    annotationProcessor "com.querydsl:querydsl-apt:$querydslVersion:jpa",
            'org.hibernate.javax.persistence:hibernate-jpa-2.1-api:1.0.2.Final',
            'javax.annotation:javax.annotation-api:1.3.2'
}

compileJava {
     options.annotationProcessorPath = configurations.annotationProcessor
}
```

[webmvc-http](https://github.com/spring-projects/spring-graphql/tree/main/samples/webmvc-http) 示例将 Querydsl 用于 artifactRepositories。

### 自定义

QuerydslDataFetcher 支持自定义 GraphQL 参数如何绑定到属性以创建 Querydsl 谓词。 默认情况下，每个可用属性的参数都绑定为“等于”。 要自定义它，您可以使用 QuerydslDataFetcher 构建器方法来提供 QuerydslBinderCustomizer。

存储库本身可能是 QuerydslBinderCustomizer 的一个实例。 这是在自动注册期间自动检测并透明应用的。 但是，当手动构建 QuerydslDataFetcher 时，您将需要使用构建器方法来应用它。

QuerydslDataFetcher 支持接口和 DTO 投影来转换查询结果，然后再返回这些结果以进行进一步的 GraphQL 处理。

要将 Spring Data 投影与 Querydsl 存储库一起使用，请创建投影接口或目标 DTO 类，并通过 projectAs 方法对其进行配置，以获得生成目标类型的 DataFetcher：

```java
class Account {

    String name, identifier, description;

    Person owner;
}

interface AccountProjection {

    String getName();

    String getIdentifier();
}

// For single result queries
DataFetcher<AccountProjection> dataFetcher =
        QuerydslDataFetcher.builder(repository).projectAs(AccountProjection.class).single();

// For multi-result queries
DataFetcher<Iterable<AccountProjection>> dataFetcher =
        QuerydslDataFetcher.builder(repository).projectAs(AccountProjection.class).many();
```

### 自动注册

如果存储库使用@GraphQlRepository 进行注释，则会自动为尚未注册 DataFetcher 且返回类型与存储库域类型匹配的查询注册。 这包括单值和多值查询。

默认情况下，查询返回的 GraphQL 类型的名称必须与存储库域类型的简单名称匹配。 如果需要，您可以使用 @GraphQlRepository 的 typeName 属性来指定目标 GraphQL 类型名称。

自动注册检测给定存储库是否实现 QuerydslBinderCustomizer 并通过 QuerydslDataFetcher 构建器方法透明地应用它。

自动注册是通过可从 QuerydslDataFetcher 获得的内置 RuntimeWiringConfigurer 执行的。 Boot 启动器会自动检测 @GraphQlRepository bean 并使用它们来初始化 RuntimeWiringConfigurer。

自动注册不支持自定义。 如果需要，您需要使用 QueryByExampleDataFetcher 通过 RuntimeWiringConfigurer 手动构建和注册 DataFetcher。



## Query by Example

Spring Data 支持使用 Query by Example 来获取数据。 Query by Example (QBE) 是一种简单的查询技术，不需要您通过特定于store的查询语言编写查询。

首先声明一个存储库 QueryByExampleExecutor：

```java
public interface AccountRepository extends Repository<Account, Long>,
            QueryByExampleExecutor<Account> {
}
```

使用 QueryByExampleDataFetcher 将存储库转换为 DataFecher：

```java
// For single result queries
DataFetcher<Account> dataFetcher =
        QueryByExampleDataFetcher.builder(repository).single();

// For multi-result queries
DataFetcher<Iterable<Account>> dataFetcher =
        QueryByExampleDataFetcher.builder(repository).many();
```

现在可以通过 RuntimeWiringConfigurer 注册上述 DataFetcher。



DataFetcher 使用 GraphQL 参数映射来创建存储库的域类型，并将其用作示例对象来获取数据。 Spring Data 支持 JPA、MongoDB、Neo4j 和 Redis 的 QueryByExampleDataFetcher。

如果存储库是 ReactiveQueryByExampleExecutor，则构建器返回 DataFetcher<Mono<Account>> 或 DataFetcher<Flux<Account>>。 Spring Data 支持 MongoDB、Neo4j、Redis 和 R2dbc 的这种变体。



QueryByExampleDataFetcher 支持接口和 DTO 投影来转换查询结果，然后再返回这些结果以进行进一步的 GraphQL 处理。



要将 Spring Data 投影与 Query by Example 存储库一起使用，请创建投影接口或目标 DTO 类，并通过 projectAs 方法对其进行配置，以获得生成目标类型的 DataFetcher：

```java
class Account {

    String name, identifier, description;

    Person owner;
}

interface AccountProjection {

    String getName();

    String getIdentifier();
}

// For single result queries
DataFetcher<AccountProjection> dataFetcher =
        QueryByExampleDataFetcher.builder(repository).projectAs(AccountProjection.class).single();

// For multi-result queries
DataFetcher<Iterable<AccountProjection>> dataFetcher =
        QueryByExampleDataFetcher.builder(repository).projectAs(AccountProjection.class).many();
```

如果存储库使用@GraphQlRepository 进行注释，则会自动为尚未注册 DataFetcher 且返回类型与存储库域类型匹配的查询注册。 这包括单值和多值查询。

默认情况下，查询返回的 GraphQL 类型的名称必须与存储库域类型的简单名称匹配。 如果需要，您可以使用 @GraphQlRepository 的 typeName 属性来指定目标 GraphQL 类型名称。

自动注册是通过可从 QueryByExampleDataFetcher 获得的内置 RuntimeWiringConfigurer 执行的。 Boot 启动器会自动检测 @GraphQlRepository bean 并使用它们来初始化 RuntimeWiringConfigurer。

自动注册不支持自定义。 如果需要，您需要使用 QueryByExampleDataFetcher 通过 RuntimeWiringConfigurer 手动构建和注册 DataFetcher。



# 服务器传输

GraphQL后端支持三种协议：HTTP , WebSocket 和 RSocket.

## HTTP

GraphQlHttpHandler接受GraphQl请求，并委托给拦截链处理请求。 有两个变体，一个用于Spring MVC，另一个用于Spring WebFlux。 两者都异步处理请求，并且具有等效的功能，但是分别依靠阻塞和非阻塞I/O编写HTTP响应。

GraphQL必须使用POST请求，请求体使用json格式。一旦成功解码了JSON请求体，HTTP响应状态始终为200，并且GraphQL请求执行中的任何错误都会显示在GraphQL响应的error部分中。默认和首选的媒体类型是 application/graphql+json，但也支持application/json。

GraphQlHttpHandler 通过声明 RouterFunction bean 来公开为 HTTP 端点。 Boot starter 会执行此操作。

### 拦截器

服务器传输允许在GraphQl Java引擎被调用之前和之后拦截请求。

HTTP和WebSocket 调用 零个或多个WebGraphQlinterceptor的链条，然后是executionGraphQlService，其调用GraphQl Java引擎。 WebGraphQLinterceptor允许应用程序拦截传入请求并执行以下操作之一：

- 检查HTTP请求信息
- 自定义graphql.ExecutionInput
- 添加HTTP响应信息
- 自定义graphql.ExecutionResult

例如，拦截器可以将HTTP请求标头传递到DataFetcher：

```java
class HeaderInterceptor implements WebGraphQlInterceptor { 

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
        String value = request.getHeaders().getFirst("myHeader");
        request.configureExecutionInput((executionInput, builder) ->
                builder.graphQLContext(Collections.singletonMap("myHeader", value)).build());
        return chain.next(request);
    }
}

@Controller
class MyController { 

    @QueryMapping
    Person person(@ContextValue String myHeader) {
        // ...
    }
}
```

同样的，拦截器可以访问控制器添加到GraphQlContext的值：

```java
@Controller
class MyController {

    @QueryMapping
    Person person(GraphQLContext context) { 
        context.put("cookieName", "123");
    }
}

// Subsequent access from a WebGraphQlInterceptor

class HeaderInterceptor implements WebGraphQlInterceptor {

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) { 
        return chain.next(request).doOnNext(response -> {
            String value = response.getExecutionInput().getGraphQLContext().get("cookieName");
            ResponseCookie cookie = ResponseCookie.from("cookieName", value).build();
            response.getResponseHeaders().add(HttpHeaders.SET_COOKIE, cookie.toString());
        });
    }
}
```

WebGraphQlHandler 可以修改 ExecutionResult，例如，检查和修改在执行开始之前引发且无法使用 DataFetcherExceptionResolver 处理的请求验证错误：

```java
static class RequestErrorInterceptor implements WebGraphQlInterceptor {

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
        return chain.next(request).map(response -> {
            if (response.isValid()) {
                return response; 
            }

            List<GraphQLError> errors = response.getErrors().stream() 
                    .map(error -> {
                        GraphqlErrorBuilder<?> builder = GraphqlErrorBuilder.newError();
                        // ...
                        return builder.build();
                    })
                    .collect(Collectors.toList());

            return response.transform(builder -> builder.errors(errors).build()); 
        });
    }
}
```

使用WebGraphQlHandler配置WebGraphQlinterceptor链。 这是由启动器支持的，请参见Web端点。

# 请求执行过程

ExecutionGraphQlService是spring调用GraphQl Java执行请求的主要抽象。主要实现是DefaultExecutionGraphQlService，配置了GraphQlSource，以访问graphql.graphql实例进行调用。

## GraphQLSource

GraphQlSource 是一个核心 Spring 抽象，用于访问 graphql.GraphQL 实例以用于请求执行。 它提供了一个构建器 API 来初始化 GraphQL Java 并构建一个 GraphQlSource。

调用GraphQlSource.schemaResourceBuilder()创建默认的GraphQlSource 构建器，支持 Reactive DataFetcher, Context Propagation, 和 Exception Resolution.

Spring Boot启动器通过默认的GraphQlSource.builder初始化GraphQlSource实例，还可以启用以下内容：

- 从指定的位置加载schema文件
- 暴露适用于GraphQlSource.builder的属性。
- 探测 [RuntimeWiringConfigurer](https://docs.spring.io/spring-graphql/docs/current/reference/html/#execution-graphqlsource-runtimewiring-configurer) bean
- 探测 [Instrumentation](https://www.graphql-java.com/documentation/instrumentation) bean ，用于 [GraphQL metrics](https://docs.spring.io/spring-boot/docs/2.7.0-SNAPSHOT/reference/html/actuator.html#actuator.metrics.supported.spring-graphql).
- 探测 DataFetcherExceptionResolver bean
- 探测 SubscriptionExceptionResolver  bean

对于进一步的自定义，您可以声明自己的 GraphQlSourceBuilderCustomizer bean； 例如，用于配置您自己的 ExecutionIdProvider：

```java
@Configuration(proxyBeanMethods = false)
class GraphQlConfig {

    @Bean
    public GraphQlSourceBuilderCustomizer sourceBuilderCustomizer() {
        return (builder) ->
                builder.configureGraphQl(graphQlBuilder ->
                        graphQlBuilder.executionIdProvider(new CustomExecutionIdProvider()));
    }
}
```

### schema 资源加载

GraphQlSource.Builder 可以配置一个或多个 Resource 实例被解析，然后合并在一起。 这意味着可以从任何位置加载chema文件。

默认情况下，Spring Boot 启动器在位置 classpath:graphql/**（通常为 src/main/resources/graphql）下查找扩展名为“.graphqls”或“.gqls”的模式文件。 您还可以使用文件系统位置或 Spring 资源层次结构支持的任何位置，包括从远程位置、存储或内存加载模式文件的自定义实现。

使用 classpath*:graphql/**/ 跨多个类路径位置查找模式文件，例如 跨多个模块。

### chema 创建

默认情况下，GraphQlSource.builder使用GraphQl Java 的 GraphQLSchemaGenerato 创建graphql.schema.GraphQLSchema。 这适用于大多数应用程序，但是，如有必要，您可以通过构建器模式创建：

```java
GraphQlSource.Builder builder = ...

builder.schemaResources(..)
        .configureRuntimeWiring(..)
        .schemaFactory((typeDefinitionRegistry, runtimeWiring) -> {
            // create GraphQLSchema
        })
```

### RuntimeWiringConfigurer

你可以使用RuntimeWiringConfigurer 注册：

- 自定义标量类型
- 指令处理代码
- TypeResolver ，如果你想覆盖默认的TypeResolver
- 字段的DataFetcher，大多数程序会配置AnnotatedControllerConfigurer， 其将注解的方法配置成 DataFetcher . Spring Boot启动器默认情况下添加了AnnotatedControllerConfigurer

与 Web 框架不同，GraphQL 不使用 Jackson 注释来驱动 JSON 序列化/反序列化。 自定义数据类型及其序列化必须描述为标量。

Spring Boot启动器检测到类型为RuntimeWiringConfigurer的bean ，并将其注册在GraphQlSource.builder中。 这意味着在大多数情况下，您将在配置中具有以下内容：

```java
@Configuration
public class GraphQlConfig {

    @Bean
    public RuntimeWiringConfigurer runtimeWiringConfigurer(BookRepository repository) {

        GraphQLScalarType scalarType = ... ;
        SchemaDirectiveWiring directiveWiring = ... ;
        DataFetcher dataFetcher = QuerydslDataFetcher.builder(repository).single();

        return wiringBuilder -> wiringBuilder
                .scalar(scalarType)
                .directiveWiring(directiveWiring)
                .type("Query", builder -> builder.dataFetcher("book", dataFetcher));
    }
}
```

### 默认的TypeResolver

GraphQlSource.Builder 使用 ClassNameTypeResolver 作为默认的 TypeResolver ， 用于尚未通过RuntimeWiringConfigurer注册的GraphQL接口和联合。GraphQL Java中TypeResolver的目的是为从DataFetcher返回的GraphQL接口或Union字段的值确定GraphQL对象类型。

ClassNameTypeResolver尝试将值的简单类名与GraphQL对象类型匹配，如果不成功，它还会导航其超级类型，包括基类和接口，以查找匹配项。ClassNameTypeResolver提供了一个选项，用于配置名称提取函数以及Class-to-GraphQL对象类型名称映射，这将有助于覆盖更多的角点情况：

```java
GraphQlSource.Builder builder = ...
ClassNameTypeResolver classNameTypeResolver = new ClassNameTypeResolver();
classNameTypeResolver.setClassNameExtractor((klass) -> {
    // Implement Custom ClassName Extractor here
});
builder.defaultTypeResolver(classNameTypeResolver);
```

### 缓存

GraphQLJava必须在执行操作之前解析并验证该操作。这可能会显著影响性能。为了避免重新解析和验证的需要，应用程序可以配置一个缓存和重用Document实例的PreparsedDocumentProvider。GraphQL Java文档通过PreparsedDocumentProvider提供了有关查询缓存的更多详细信息。

在Spring GraphQL中，您可以通过 GraphQlSource.Builder#configureGraphQl 注册PreparsedDocumentProvider:

```java
// Typically, accessed through Spring Boot's GraphQlSourceBuilderCustomizer
GraphQlSource.Builder builder = ...

// Create provider
PreparsedDocumentProvider provider = ...

builder.schemaResources(..)
        .configureRuntimeWiring(..)
        .configureGraphQl(graphQLBuilder -> graphQLBuilder.preparsedDocumentProvider(provider))
```

### 指令

GraphQL Java提供SchemaDirectiveWiring契约，帮助应用程序检测和处理指令。您可以通过RuntimeWiringConfigurer注册SchemaDirectiveWiring：

```java
@Configuration
public class GraphQlConfig {

     @Bean
     public RuntimeWiringConfigurer runtimeWiringConfigurer() {
          return builder -> builder.directiveWiring(new MySchemaDirectiveWiring());
     }

}
```

## 上下文传播

Spring for GraphQL支持将上下文从服务器传输透明地传播到DataFetcher及其调用的其他组件。这包括来自Spring MVC请求处理线程的ThreadLocal上下文和来自WebFlux处理管道的Reactor上下文。

### WebMvc

由GraphQL Java调用的DataFetcher和其他组件可能不总是在与Spring MVC处理程序相同的线程上执行，例如，如果异步WebGraphQlInterceptor或DataFetchers切换到不同的线程。

Spring for GraphQL支持将ThreadLocal值从Servlet容器线程传播到DataFetcher和GraphQL Java调用的其他组件在其上执行的线程。为此，应用程序需要创建ThreadLocalAccessor以提取感兴趣的ThreadLocal值：

```java
public class RequestAttributesAccessor implements ThreadLocalAccessor {

    private static final String KEY = RequestAttributesAccessor.class.getName();

    @Override
    public void extractValues(Map<String, Object> container) {
        container.put(KEY, RequestContextHolder.getRequestAttributes());
    }

    @Override
    public void restoreValues(Map<String, Object> values) {
        if (values.containsKey(KEY)) {
            RequestContextHolder.setRequestAttributes((RequestAttributes) values.get(KEY));
        }
    }

    @Override
    public void resetValues(Map<String, Object> values) {
        RequestContextHolder.resetRequestAttributes();
    }

}
```

ThreadLocalAccessor可以在WebGraphHandler builder中注册。Boot starter检测这种类型的bean，并自动为Spring MVC应用程序注册它们，请参阅Web端点部分。

## 异常解析

GraphQL Java应用程序可以注册DataFetcherExceptionHandler，以决定如何在GraphQL响应的“error”部分中表示数据层的异常。

Spring for GraphQL有一个内置的DataFetcherExceptionHandler，它被配置为供默认GraphQLSource builder使用。它允许应用程序注册一个或多个按顺序调用的SpringDataFetcherExceptionResolver组件，直到将异常解析为graphql.GraphQLError对象（可能为空）列表。

DataFetcherExceptionResolver是一个异步协定。对于大多数实现，扩展DataFetcherExceptionResolverAdapter并重写其同步解析异常的resolveToSingleError或resolveToMultipleErrors方法之一就足够了。

可以通过graphql.ErrorClassification将GraphQLError分配给类别。在Spring GraphQL中，您还可以通过ErrorType进行分配，ErrorType具有以下常见分类，应用程序可以使用这些分类对错误进行分类：

- BAD_REQUEST
- UNAUTHORIZED
- FORBIDDEN
- NOT_FOUND
- INTERNAL_ERROR

如果异常仍未解决，则默认情况下，它被分类为INTERNAL_ERROR，并带有一条包含类别名称和DataFetchingEnvironment中的executionId的通用消息。该消息故意不透明，以避免泄漏实现细节。应用程序可以使用DataFetcherExceptionResolver自定义错误详细信息。

未解决的异常与executionId一起记录在ERROR级别，以与发送到客户端的错误相关。在DEBUG级别记录已解决的异常。

### 请求异常

GraphQL Java引擎在解析请求时可能会遇到验证或其他错误，从而阻止请求执行。在这种情况下，响应包含一个带有null的“data”键和一个或多个全局请求级“errors”，即没有字段路径。

DataFetcherExceptionResolver无法处理此类全局错误，因为它们是在执行开始之前和调用任何DataFetcher之前引发的。应用程序可以使用传输级拦截器来检查和转换ExecutionResult中的错误。请参见WebGraphQlInterceptor下的示例。

### 订阅异常

订阅请求的发布服务器可能会以错误信号完成，在这种情况下，底层传输（例如WebSocket）会发送带有GraphQL错误列表的最终“错误”类型消息。

DataFetcherExceptionResolver无法解决订阅发布服务器的错误，因为数据DataFetcher最初只创建发布服务器。之后，传输订阅发布服务器，然后发布服务器可能会出错。

应用程序可以注册SubscriptionExceptionResolver，以解决订阅发布服务器的异常，从而将这些异常解析为要发送给客户端的GraphQL错误。

## 批量加载

给定一本书及其作者，我们可以为一本书创建一个DataFetcher，为其作者创建另一个DataFetcher。这允许选择有作者或没有作者的书籍，但这意味着书籍和作者不会一起加载，这在查询多本书时尤其低效，因为每本书的作者都是单独加载的。这就是所谓的N+1选择问题。

### DataLoader

GraphQLJava提供了一种用于批量加载相关实体的DataLoader机制。您可以在GraphQL Java文档中找到完整的详细信息。以下是其工作原理的总结：

1. 在DataLoaderRegistry中注册DataLoader，它可以在给定唯一键的情况下加载实体。
2. DataFetcher可以访问DataLoader，并使用它们按id加载实体。
3. DataLoader通过返回future来延迟加载，以便可以在批处理中完成。
4. DataLoader维护加载实体的每个请求缓存，可以进一步提高效率。



### BatchLoaderRegistry

GraphQL Java中的完整批处理加载机制需要实现多个BatchLoader接口中的一个，然后用DataLoaderRegistry中的名称将这些接口包装并注册为DataLoader。

Spring GraphQL中的API略有不同。对于注册，只有一个中央BatchLoaderRegistry公开工厂方法和一个生成器来创建和注册任意数量的批加载函数：

```java
@Configuration
public class MyConfig {

    public MyConfig(BatchLoaderRegistry registry) {

        registry.forTypePair(Long.class, Author.class).registerMappedBatchLoader((authorIds, env) -> {
                // return Mono<Map<Long, Author>
        });

        // more registrations ...
    }

}
```

Spring Boot starter声明了一个BatchLoaderRegistrybean，您可以将其注入到您的配置中，如上所示，或者注入到任何组件中，如控制器，以便注册批加载函数。然后，BatchLoaderRegistry被注入DefaultExecutionGraphQlService，在那里它确保每个请求的DataLoader注册。

默认情况下，DataLoader名称基于目标实体的类名。这允许@SchemaMapping方法使用泛型类型声明DataLoader参数，而无需指定名称。但是，如果需要，可以通过BatchLoaderRegistry生成器以及其他DataLoader选项自定义名称。

在许多情况下，当加载相关实体时，您可以使用@BatchMapping控制器方法，这是一种快捷方式，可以替代直接使用BatchLoaderRegistry和DataLoader的需要。BatchLoaderRegistry还提供了其他重要的好处。它支持从批处理加载函数和@BatchMapping方法访问相同的GraphQLContext，并确保上下文传播到它们。这就是为什么应用程序需要使用它。可以直接执行自己的DataLoader注册，但这样的注册将放弃上述好处。

### 测试批量加载

首先让BatchLoaderRegistry在DataLoaderRegistry上执行注册：

```java
BatchLoaderRegistry batchLoaderRegistry = new DefaultBatchLoaderRegistry();
// perform registrations...

DataLoaderRegistry dataLoaderRegistry = DataLoaderRegistry.newRegistry().build();
batchLoaderRegistry.registerDataLoaders(dataLoaderRegistry, graphQLContext);
```

现在，您可以按如下方式访问和测试各个DataLoader：

```java
DataLoader<Long, Book> loader = dataLoaderRegistry.getDataLoader(Book.class.getName());
loader.load(1L);
loader.loadMany(Arrays.asList(2L, 3L));
List<Book> books = loader.dispatchAndJoin(); // actual loading

assertThat(books).hasSize(3);
assertThat(books.get(0).getName()).isEqualTo("...");
// ...
```
