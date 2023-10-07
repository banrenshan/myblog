---
title: graphql java实现
tags:
  - graphql
categories:
  - java
date: 2022-11-27 19:02:18
---

## 执行

### 查询

要对schema执行查询，请使用适当的参数构建一个新的GraphQL对象，然后调用execute方法。查询的结果是ExecutionResult，它是查询数据和/或错误列表。

```java
GraphQLSchema schema = GraphQLSchema.newSchema()
        .query(queryType)
        .build();

GraphQL graphQL = GraphQL.newGraphQL(schema)
        .build();

ExecutionInput executionInput = ExecutionInput.newExecutionInput().query("query { hero { name } }")
        .build();

ExecutionResult executionResult = graphQL.execute(executionInput);

Object data = executionResult.getData();
List<GraphQLError> errors = executionResult.getErrors();
```

### Data Fetchers

每个graphql字段类型都有一个graphql.schema.DataFetcher与其关联的。通常，您可以依赖graphql.schema.PropertyDataFetcher检查Java POJO对象，以从中提供字段值。如果您没有在字段上指定数据获取器，则将使用此选项。

```
DataFetcher userDataFetcher = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment environment) {
        return fetchUserFromDatabase(environment.getArgument("userId"));
    }
};
```

在上面的示例中，执行将等待数据获取器返回，然后再继续。您可以通过向数据返回CompletionStage来实现DataFetcher的异步执行。

### 获取数据的时候发生异常

如果在数据获取器调用期间发生异常，则默认情况下执行策略将生成一个graphql.ExceptionWhileDataFetching错误，并将其添加到结果的错误列表中。请记住，graphql允许有错误的部分结果。这是标准行为的代码：

```java
public class SimpleDataFetcherExceptionHandler implements DataFetcherExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(SimpleDataFetcherExceptionHandler.class);

    @Override
    public void accept(DataFetcherExceptionHandlerParameters handlerParameters) {
        Throwable exception = handlerParameters.getException();
        SourceLocation sourceLocation = handlerParameters.getField().getSourceLocation();
        ExecutionPath path = handlerParameters.getPath();

        ExceptionWhileDataFetching error = new ExceptionWhileDataFetching(path, exception, sourceLocation);
        handlerParameters.getExecutionContext().addError(error);
        log.warn(error.getMessage(), exception);
    }
}
```

如果抛出的异常本身是GraphqlError，那么它会将该异常的消息和自定义扩展属性传输到ExceptionWhileDataFetching对象中。这允许您将自己的自定义属性放入发送回调用者的graphql错误中。

例如，假设您的数据获取器抛出了此异常。foo和fizz属性将包含在生成的graphql错误中。

```java
class CustomRuntimeException extends RuntimeException implements GraphQLError {
    @Override
    public Map<String, Object> getExtensions() {
        Map<String, Object> customAttributes = new LinkedHashMap<>();
        customAttributes.put("foo", "bar");
        customAttributes.put("fizz", "whizz");
        return customAttributes;
    }

    @Override
    public List<SourceLocation> getLocations() {
        return null;
    }

    @Override
    public ErrorType getErrorType() {
        return ErrorType.DataFetchingException;
    }
}
```

您可以通过创建自己的graphql.execution.DataFetcherExceptionHandler来更改此行为。

例如，上面的代码记录了底层异常和堆栈跟踪。有些人可能不希望在输出错误列表中看到这一点。所以你可以使用这个机制来改变这种行为。

```
DataFetcherExceptionHandler handler = new DataFetcherExceptionHandler() {
    @Override
    public void accept(DataFetcherExceptionHandlerParameters handlerParameters) {
        //
        // do your custom handling here.  The parameters have all you need
    }
};
ExecutionStrategy executionStrategy = new AsyncExecutionStrategy(handler);
```

### 返回数据和错误

graphql.execution.DataFetcherResult同时返回数据和多个错误.。或包装在CompletableFuture实例中，用于异步执行。当DataFetcher可能需要从多个源或另一个GraphQL资源检索数据时，这非常有用。

在本例中，DataFetcher从另一个GraphQL资源中检索用户并返回其数据和错误。

```
DataFetcher userDataFetcher = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment environment) {
        Map response = fetchUserFromRemoteGraphQLResource(environment.getArgument("userId"));
        List<GraphQLError> errors = response.get("errors").stream()
            .map(MyMapGraphQLError::new)
            .collect(Collectors.toList());
        return new DataFetcherResult(response.get("data"), errors);
    }
};
```

### 序列化结果成JSON

调用graphql最常见的方法是通过HTTP，并期望返回JSON响应。所以你需要将graphql.ExecutionResult转换为JSON负载。
一种常见的方法是使用JSON序列化库，如Jackson或GSON。然而，他们具体如何解释数据结果是他们所特有的。例如，null在graphql结果中很重要，因此必须设置json映射器以包含它们。

为了确保获得100%符合graphql规范的JSON结果，应该对结果调用toSpecification，然后将其作为JSON发送回去。

```
ExecutionResult executionResult = graphQL.execute(executionInput);

Map<String, Object> toSpecificationResult = executionResult.toSpecification();

sendAsJson(toSpecificationResult);
```







graphql java库专注于查询引擎，不关心其他高级应用程序问题，例如：

* 数据库访问
* 数据缓存
* 数据认证
* 数据分页
* JSON编码
* 依赖注入

您需要将这些关注点推入业务逻辑层。

### 上下文对象

您可以在查询执行期间传入上下文对象，以便更好地调用该业务逻辑。例如，应用程序可能正在执行用户检测，您需要在graphql执行过程中提供这些信息来执行授权。下面的示例显示了如何传递信息来帮助执行查询。

```java
// 将用户信息设置到上下文，
UserContext contextForUser = YourGraphqlContextBuilder.getContextForUser(getCurrentUser());

ExecutionInput executionInput = ExecutionInput.newExecutionInput()
        .context(contextForUser)
        .build();

ExecutionResult executionResult = graphQL.execute(executionInput);

// 在DataFetcher 获取上下文中的内容
DataFetcher dataFetcher = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment environment) {
        UserContext userCtx = environment.getContext();
        Long businessObjId = environment.getArgument("businessObjId");

        return invokeBusinessLayerMethod(userCtx, businessObjId);
    }
};
```

## 数据获取

### graphql 如何获取数据

graphql 中的每个字段都有一个 `graphql.schema.DataFetcher` 与之关联 。一些字段将使用专门的数据获取器代码，该代码知道如何去数据库获取字段信息，而大多数字段只需使用字段名和普通旧Java对象（POJO）模式从返回的内存对象中获取数据即可。

设想一个如下的类型声明：

```
type Query {
  products(match : String) : [Product]   # a list of products
}

type Product {
  id : ID
  name : String
  description : String
  cost : Float
  tax : Float
  launchDate(dateFormat : String = "dd, MMM, yyyy") : String
}
```

Query.products 有专门的 DataFetcher  ， 例如下面的代码：

```java
DataFetcher productsDataFetcher = new DataFetcher<List<ProductDTO>>() {
    @Override
    public List<ProductDTO> get(DataFetchingEnvironment environment) {
        DatabaseSecurityCtx ctx = environment.getContext();

        List<ProductDTO> products;
        String match = environment.getArgument("match");
        if (match != null) {
            products = fetchProductsFromDatabaseWithMatching(ctx, match);
        } else {
            products = fetchAllProductsFromDatabase(ctx);
        }
        return products;
    }
};
```

每个DataFetcher都传递了一个`graphql.schema.DataFetchingEnvironment`对象，其中包含要获取的字段、已向字段提供的参数以及其他信息，例如字段的类型、其父类型、查询根对象或查询上下文对象。

一旦我们有了ProductDTO对象的列表，我们通常不需要在每个字段上使用专门的数据获取器。graphql-java附带了一个智能`graphql.schema.PropertyDataFetcher`，它知道如何根据字段名遵循POJO模式。在上面的示例中，有一个name字段，因此它将尝试查找公共`String getName() `POJO方法来获取数据。

然而，您可以在DTO方法中访问`graphql.schema.DataFetchingEnvironment`对象。这允许您在发送值之前对其进行调整。例如，上面我们有一个launchDate字段，它接受可选的dateFormat参数。我们可以让ProductDTO具有将此日期格式应用于所需格式的逻辑：

```java
class ProductDTO {

    private ID id;
    private String name;
    private String description;
    private Double cost;
    private Double tax;
    private LocalDateTime launchDate;

    // ...

    public String getName() {
        return name;
    }

    // ...

    public String getLaunchDate(DataFetchingEnvironment environment) {
        String dateFormat = environment.getArgument("dateFormat");
        return yodaTimeFormatter(launchDate,dateFormat);
    }
}
```

### 自定义PropertyDataFetcher

如上所述，`graphql.schema.PropertyDataFetcher`是graphql-java中字段的默认数据获取器，它将使用标准模式获取对象字段值。

它支持POJO方法和Map方法。默认情况下，对于graphql字段fieldX，则可以找到名为fieldX的POJO属性或名为fieldX的map 键。

但是，graphql模式命名和运行时对象命名之间可能有一些小差异。例如，想象一下`Product.description`实际上在运行时对应 Java对象中的getDesc()。如果使用SDL指定schema，则可以使用@fetch指令来重新映射:

```
directive @fetch(from : String!) on FIELD_DEFINITION

type Product {
    id : ID
    name : String
    description : String @fetch(from:"desc")
    cost : Float
    tax : Float
}
```

这将告诉`graphql.schema.PropertyDataFetcher`在获取名为description的graphql字段的数据时使用属性名称desc。

如果您是手动编码模式，那么您可以直接通过在字段数据获取器中进行连接:

```java
GraphQLFieldDefinition descriptionField = GraphQLFieldDefinition.newFieldDefinition()
        .name("description")
        .type(Scalars.GraphQLString)
        .build();

GraphQLCodeRegistry codeRegistry = GraphQLCodeRegistry.newCodeRegistry()
        .dataFetcher(
                coordinates("ObjectType", "description"),
                PropertyDataFetcher.fetching("desc"))
        .build();
```

### DataFetchingEnvironment核心方法

* <T> T getSource()：source对象用于获取字段的信息。在常见情况下，它是一个内存中的DTO对象，因此简单的POJO getter将用于字段值。在更复杂的情况下，您可以检查它以了解如何获取当前字段的特定信息。在执行graphql字段树时，每个返回的字段值都会成为子字段的源对象。
* <T> T getRoot()： 对于顶级字段，root和source是相同的。根对象在查询过程中从不更改，它可能为空，因此不使用
* Map<String, Object> getArguments():这表示在字段上提供的参数以及从传入的变量、AST文本和默认参数值解析的参数值。您可以使用字段的参数来控制它返回的值。
* <T> T getContext(): 上下文对象在首次查询时设置，并在查询的整个生命周期内保持不变。上下文可以是任何值，通常用于在尝试获取字段数据时为每个数据获取器提供一些所需的调用上下文。例如，当前用户凭证或数据库连接参数可以包含在上下文对象中，以便数据获取器可以进行业务层调用。作为一名graphql系统设计师，您的一个关键设计决定是，如果需要，您将如何在提取器中使用上下文。有些人使用一个依赖框架，该框架将上下文自动注入数据获取器，因此不需要使用它。
* ExecutionStepInfo getExecutionStepInfo(): 执行graphql查询会创建字段及其类型的调用树。graphql.execution.ExecutionStepInfo.getParentTypeInfo允许您向上导航，查看导致当前字段执行的类型和字段。由于这在执行期间形成了树路径，因此graphql.execution.ExecutionStepInfo.getPath方法返回该路径的表示。这对于记录和调试查询非常有用。
* DataFetchingFieldSelectionSet getSelectionSet(): 选择集表示在当前执行的字段下“选择”的子字段。这有助于提前了解客户需要的子字段信息。
* ExecutionId getExecutionId():每个查询执行都有一个唯一的id。您可以在日志中使用它来标记每个单独的查询。

## 数据映射

graphql的核心是声明一个类型schema，并将其映射到运行时数据上。例如下面的schema:

```
type Query {
  products(match : String) : [Product]   # a list of products
}

type Product {
  id : ID
  name : String
  description : String
  cost : Float
  tax : Float
}
```

然后，我们可以通过类似以下的方式在这个简单模式上运行查询：

```
query ProductQuery {
  products(match : "Paper*")
  {
    id, name, cost, tax
  }
}
```

我们将有一个DataFetcher匹配 Query的products字段，负责查找与传入参数匹配的产品列表。
现在假设我们有3个下游服务。一个用于获取产品信息，一个用于获得产品cost 信息，另一个用于计算产品tax信息。

graphql-java通过在对象上运行数据获取器获取所有信息并将其映射回模式中指定的类型来工作。我们面临的挑战是将这三种信息来源作为一种统一的类型。

我们可以在cost 和 tax 字段上指定数据获取器，但这需要更多的维护，可能会导致N+1性能问题。我们最好在Query.products 的 data fetcher 中完成所有这些工作，并在此时创建数据的统一视图:

```java
DataFetcher productsDataFetcher = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment env) {
        String matchArg = env.getArgument("match");

        List<ProductInfo> productInfo = getMatchingProducts(matchArg);

        List<ProductCostInfo> productCostInfo = getProductCosts(productInfo);

        List<ProductTaxInfo> productTaxInfo = getProductTax(productInfo);

        return mapDataTogether(productInfo, productCostInfo, productTaxInfo);
    }
};
```

因此，查看上面的代码，我们有3种类型的信息需要以某种方式组合，以便上面的graphql查询可以访问字段id、name、cost、tax。

我们有两种方法来创建此映射。一种是通过使用非类型安全的List＜Map＞结构，另一种是创建封装此数据的类型安全类List＜ProductDTO＞。

Map 的方式如下：

```java
private List<Map> mapDataTogetherViaMap(List<ProductInfo> productInfo, List<ProductCostInfo> productCostInfo, List<ProductTaxInfo> productTaxInfo) {
    List<Map> unifiedView = new ArrayList<>();
    for (int i = 0; i < productInfo.size(); i++) {
        ProductInfo info = productInfo.get(i);
        ProductCostInfo cost = productCostInfo.get(i);
        ProductTaxInfo tax = productTaxInfo.get(i);

        Map<String, Object> objectMap = new HashMap<>();
        objectMap.put("id", info.getId());
        objectMap.put("name", info.getName());
        objectMap.put("description", info.getDescription());
        objectMap.put("cost", cost.getCost());
        objectMap.put("tax", tax.getTax());

        unifiedView.add(objectMap);
    }
    return unifiedView;
}
```

DTO 方式如下：

```java
class ProductDTO {
    private final String id;
    private final String name;
    private final String description;
    private final Float cost;
    private final Float tax;

    public ProductDTO(String id, String name, String description, Float cost, Float tax) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.cost = cost;
        this.tax = tax;
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getDescription() {
        return description;
    }

    public Float getCost() {
        return cost;
    }

    public Float getTax() {
        return tax;
    }
}

private List<ProductDTO> mapDataTogetherViaDTO(List<ProductInfo> productInfo, List<ProductCostInfo> productCostInfo, List<ProductTaxInfo> productTaxInfo) {
    List<ProductDTO> unifiedView = new ArrayList<>();
    for (int i = 0; i < productInfo.size(); i++) {
        ProductInfo info = productInfo.get(i);
        ProductCostInfo cost = productCostInfo.get(i);
        ProductTaxInfo tax = productTaxInfo.get(i);

        ProductDTO productDTO = new ProductDTO(
                info.getId(),
                info.getName(),
                info.getDescription(),
                cost.getCost(),
                tax.getTax()
        );
        unifiedView.add(productDTO);
    }
    return unifiedView;
}
```

## 异常

如果遇到某些异常情况，graphql引擎可以抛出运行时异常。以下是可以在graphql.execute()调用中抛出的异常列表。

这些不是执行中的graphql错误，而是执行graphql查询时完全不可接受的条件:

* graphql.schema.CoercingSerializeException:当一个值不能通过标量类型序列化时，例如String值被强制为Int。
* graphql.schema.CoercingParseValueException：当标量类型无法解析值（例如，字符串输入值被解析为Int）时引发。
* graphql.execution.UnresolvedTypeException：如果graphql.schema.TypeResolver无法提供给定接口或联合类型的具体对象类型。
* graphql.execution.NonNullableValueCoercedAsNullException:如果在执行期间将非空变量参数强制为空值时引发。
* graphql.execution.InputMapDefinesTooManyFieldsException:如果用于输入类型对象的映射包含的键多于该输入类型中定义的键，则引发。
* graphql.schema.validation.InvalidSchemaException:如果通过graphql.schema.GraphQLSchema.Builder#build()生成schema无效时引发
* `graphql.execution.UnknownOperationException`：如果查询中定义了多个操作，并且缺少操作名称，或者GraphQL查询中没有包含匹配的操作名称。
* graphql.GraphQLException：作为通用运行时异常抛出，例如，如果代码在检查POJO时无法访问命名字段，它类似于RuntimeException。
* graphql.AssertException：

## 字段选择

当您具有复合类型（对象或接口类型）并且从该类型中选择了一组子字段时，将发生字段选择。

例如下面的查询：

```
query {
  user(userId : "xyz") {
    name
    age
    weight
    friends {
      name
    }
  }
}
```

了解字段选择集有助于提高DataFetchers的效率。例如，在上面的查询中，假设用户字段由SQL数据库系统支持。数据获取器可以查看字段选择集并使用不同的查询，因为它知道调用者需要朋友信息和用户信息。

graphql.schema.DataFetchingFieldSelectionSet 用于表示选择的字段。他提供了schema中字段/参数和graphql.schema.GraphQLFieldDefinition的映射：

```java
DataFetcher smartUserDF = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment env) {
        String userId = env.getArgument("userId");

        DataFetchingFieldSelectionSet selectionSet = env.getSelectionSet();
        if (selectionSet.contains("friends/*")) {
            return getUserAndTheirFriends(userId);
        } else {
            return getUser(userId);
        }
    }
};
```

全局路径匹配系统用于对选择中的字段进行寻址。它基于java.nio.file.FileSystem#getPathMatcher作为实现。这将允许您使用 `*`, `**` and `?` 作为特殊匹配字符，例如`invoice/customer*`将invoice字段与以customer开头的子字段相匹配。每个级别的字段由`/`分隔，就像文件系统路径一样。

有一些方法允许您获取有关选择集中字段的更详细信息。例如，如果您经常使用Relay，您希望知道查询的Connection部分中请求了哪些字段:

```
query {
  users(first:10)  {
    edges {
      node {
        name
        age
        weight
        friends {
          name
        }
      }
    }
  }
}
```

您可以编写代码来获取与glob匹配的每个特定字段的详细信息:

```java
DataFetchingFieldSelectionSet selectionSet = env.getSelectionSet();
List<SelectedField> nodeFields = selectionSet.getFields("edges/nodes/*");
nodeFields.forEach(selectedField -> {
    System.out.println(selectedField.getName());
    System.out.println(selectedField.getFieldDefinition().getType());

    DataFetchingFieldSelectionSet innerSelectionSet = selectedField.getSelectionSet();
    // this forms a tree of selection and you can get very fancy with it
}
```

## 字段的可见性

默认情况下，GraphqlSchema中定义的每个字段都可用。在某些情况下，您可能需要根据用户限制某些字段。

您可以使用graphql.schema.visibility.GraphqlFieldVisibility来实现这一点。

一个简单的graphql.schema.visibility.BlockedFields实现基于全限定字段名称：

```java
GraphqlFieldVisibility blockedFields = BlockedFields.newBlock()
        .addPattern("Character.id")
        .addPattern("Droid.appearsIn")
        .addPattern(".*\\.hero") // it uses regular expressions
        .build();
GraphQLCodeRegistry codeRegistry = GraphQLCodeRegistry.newCodeRegistry()
        .fieldVisibility(blockedFields)
        .build();

GraphQLSchema schema = GraphQLSchema.newSchema()
        .query(StarWarsSchema.queryType)
        .codeRegistry(codeRegistry)
        .build();
```

如果需要，还有另一种实现可以阻止在模式上执行检测。请注意，这会使您的服务器违反graphql规范和大多数客户端的期望，因此请谨慎使用。

```java
GraphQLCodeRegistry codeRegistry = GraphQLCodeRegistry.newCodeRegistry()
        .fieldVisibility(NoIntrospectionGraphqlFieldVisibility.NO_INTROSPECTION_FIELD_VISIBILITY)
        .build();
GraphQLSchema schema = GraphQLSchema.newSchema()
        .query(StarWarsSchema.queryType)
        .codeRegistry(codeRegistry)
        .build();
```

您可以创建自己的GraphqlFieldVisibility派生，以检查需要做什么来确定哪些字段应该可见或不可见。

```java
class CustomFieldVisibility implements GraphqlFieldVisibility {

    final YourUserAccessService userAccessService;

    CustomFieldVisibility(YourUserAccessService userAccessService) {
        this.userAccessService = userAccessService;
    }

    @Override
    public List<GraphQLFieldDefinition> getFieldDefinitions(GraphQLFieldsContainer fieldsContainer) {
        if ("AdminType".equals(fieldsContainer.getName())) {
            if (!userAccessService.isAdminUser()) {
                return Collections.emptyList();
            }
        }
        return fieldsContainer.getFieldDefinitions();
    }

    @Override
    public GraphQLFieldDefinition getFieldDefinition(GraphQLFieldsContainer fieldsContainer, String fieldName) {
        if ("AdminType".equals(fieldsContainer.getName())) {
            if (!userAccessService.isAdminUser()) {
                return null;
            }
        }
        return fieldsContainer.getFieldDefinition(fieldName);
    }
}
```

## 标量

GraphQL类型系统的叶节点称为标量。一旦达到标量类型，就不能再下降到类型层次结构中。标量类型表示不可分割的值。

GraphQL规范规定，所有实现都必须具有以下标量类型：

* String aka `GraphQLString` - UTF‐8 字符串
* Boolean aka `GraphQLBoolean` - true 或者 false.
* Int aka `GraphQLInt` 
* Float aka `GraphQLFloat` 
* ID aka `GraphQLID`：以与字符串相同的方式序列化的唯一标识符。然而，将其定义为ID意味着它不是人类可读的。

[graphql-java-extended-scalars](https://github.com/graphql-java/graphql-java-extended-scalars)扩展了以下标量：

* Long aka `GraphQLLong`
* Short aka `GraphQLShort`
* Byte aka `GraphQLByte`
* BigDecimal aka `GraphQLBigDecimal` 
* BigInteger aka `GraphQLBigInteger`

### 自定义标量

您可以自定义标量。在这样做时，您将承担在运行时强制值的责任，稍后我们将对此进行解释。
假设我们决定需要一个电子邮件标量类型。它将以电子邮件地址作为输入和输出。

我们会创建下面的graphql.schema.GraphQLScalarType实例：

```java
public static class EmailScalar {

    public static final GraphQLScalarType EMAIL = GraphQLScalarType.newScalar()
            .name("email")
            .description("A custom scalar that handles emails")
            .coercing(new Coercing() {
                @Override
                public Object serialize(Object dataFetcherResult) {
                    return serializeEmail(dataFetcherResult);
                }

                @Override
                public Object parseValue(Object input) {
                    return parseEmailFromVariable(input);
                }

                @Override
                public Object parseLiteral(Object input) {
                    return parseEmailFromAstLiteral(input);
                }
            })
            .build();

    private static boolean looksLikeAnEmailAddress(String possibleEmailValue) {
        // ps.  I am not trying to replicate RFC-3696 clearly
        return Pattern.matches("[A-Za-z0-9]@[.*]", possibleEmailValue);
    }

    private static Object serializeEmail(Object dataFetcherResult) {
        String possibleEmailValue = String.valueOf(dataFetcherResult);
        if (looksLikeAnEmailAddress(possibleEmailValue)) {
            return possibleEmailValue;
        } else {
            throw new CoercingSerializeException("Unable to serialize " + possibleEmailValue + " as an email address");
        }
    }

    private static Object parseEmailFromVariable(Object input) {
        if (input instanceof String) {
            String possibleEmailValue = input.toString();
            if (looksLikeAnEmailAddress(possibleEmailValue)) {
                return possibleEmailValue;
            }
        }
        throw new CoercingParseValueException("Unable to parse variable value " + input + " as an email address");
    }

    private static Object parseEmailFromAstLiteral(Object input) {
        if (input instanceof StringValue) {
            String possibleEmailValue = ((StringValue) input).getValue();
            if (looksLikeAnEmailAddress(possibleEmailValue)) {
                return possibleEmailValue;
            }
        }
        throw new CoercingParseLiteralException(
                "Value is not any email address : '" + String.valueOf(input) + "'"
        );
    }
}
```

任何自定义标量实现的真正工作都是graphql.schema.Coercing执行。它负责3个功能：

* parseValue : 获取变量输入对象并转换为Java运行时表示
* parseLiteral : 采用AST文字graphql.language.Value作为输入并转换为Java运行时表示
* serialize : 获取Java对象并将其转换为该标量的输出形状

因此，自定义标量代码必须处理2种形式的输入（parseValue/parseLiteral）和1种形式的输出（serialize）。

假如有下面的输入：

```
mutation Contact($mainContact: Email!) {
  makeContact(mainContactEmail: $mainContact, backupContactEmail: "backup@company.com") {
    id
    mainContactEmail
  }
}
```

* 通过parseValue调用，将$mainContact变量值转换为运行时对象
* 通过parseLiteral调用以转换AST graphql.language.StringValue “backup@company.com“转换为运行时对象
* 通过serialize调用，将mainContactEmail的运行时表示转换为可输出的表单

### 验证输入输出

这些方法可以验证所接收的输入是否有意义。例如，我们的电子邮件标量将尝试验证输入和输出是否确实是电子邮件地址。
graphql.schema.Coercing的规定如下：

* serialize 只能允许抛出 graphql.schema.SerializeException，这表示该值无法序列化为适当的形式。您不能允许其他运行时异常转义此方法以获得用于验证的正常graphql行为。
* parseValue只能允许抛出graphql.schema.ParseValueException,这表示无法将值作为输入解析为适当的形式。您不能允许其他运行时异常逃脱此方法以获得用于验证的正常graphql行为。
* parseLiteral只能允许抛出graphql.schema.ParseLiteralException，这表明AST值不能作为输入解析为适当的形式。您不能允许任何运行时异常逃脱此方法以获得正常的graphql行为进行验证。

有些人试图依赖运行时异常进行验证，并希望它们以graphql错误的形式出现。事实并非如此。您必须遵循强制方法契约，以允许graphqljava引擎根据标量类型的graphql规范工作。

## 创建schema

GraphQL API有一个schema，它定义了可以查询或修改的每个字段以及这些字段的类型。graphql-java提供了两种不同的schema定义方式：编程或通过特殊的graphql-dsl（称为SDL）。

SDL方式:

```
type Foo {
  bar: String
}
```

代码方式：

```
GraphQLObjectType fooType = newObject()
    .name("Foo")
    .field(newFieldDefinition()
            .name("bar")
            .type(GraphQLString))
    .build();
```

### DataFetcher 和 TypeResolver

DataFetcher为字段提供数据（如果是mutation，则会进行更改）。每个字段定义都有一个DataFetcher。如果未配置，则使用PropertyDataFetch。

PropertyDataFetcher从Map和JavaBeans获取数据。因此，当字段名与源对象的Map键或属性名匹配时，不需要DataFetcher。

TypeResolver帮助graphql-java决定具体值属于哪种类型。这是Interface和Union所需要的。

例如，假设您有一个名为MagicUserType的接口，它可以解析回一系列名为Wizard、Witch和Necromancer的Java类。类型解析器负责检查运行时对象，并决定应该使用什么GraphqlObjectType来表示它，从而决定将调用哪些数据获取器和字段。

```java
new TypeResolver() {
    @Override
    public GraphQLObjectType getType(TypeResolutionEnvironment env) {
        Object javaObject = env.getObject();
        if (javaObject instanceof Wizard) {
            return env.getSchema().getObjectType("WizardType");
        } else if (javaObject instanceof Witch) {
            return env.getSchema().getObjectType("WitchType");
        } else {
            return env.getSchema().getObjectType("NecromancerType");
        }
    }
};
```

### 使用SDL的方式创建Schema

通过SDL定义模式时，在创建可执行模式时，需要提供所需的DataFetcher和TypeResolver。
以以下静态模式定义文件starWarsSchema.graphqls为例：

```
schema {
  query: QueryType
}

type QueryType {
  hero(episode: Episode): Character
  human(id : String) : Human
  droid(id: ID!): Droid
}


enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}

interface Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
}

type Human implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
  homePlanet: String
}

type Droid implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
  primaryFunction: String
}
```

静态架构定义文件starWarsSchema.graphqls包含字段和类型定义，但需要运行时连接才能使其成为真正的可执行模式。

运行时连接包含DataFetcher、TypeResolver和自定义Scalar，它们是生成完全可执行架构所需的。

使用此生成器模式将其连接在一起：

```java
RuntimeWiring buildRuntimeWiring() {
    return RuntimeWiring.newRuntimeWiring()
            .scalar(CustomScalar)
            // this uses builder function lambda syntax
            .type("QueryType", typeWiring -> typeWiring
                    .dataFetcher("hero", new StaticDataFetcher(StarWarsData.getArtoo()))
                    .dataFetcher("human", StarWarsData.getHumanDataFetcher())
                    .dataFetcher("droid", StarWarsData.getDroidDataFetcher())
            )
            .type("Human", typeWiring -> typeWiring
                    .dataFetcher("friends", StarWarsData.getFriendsDataFetcher())
            )
            // you can use builder syntax if you don't like the lambda syntax
            .type("Droid", typeWiring -> typeWiring
                    .dataFetcher("friends", StarWarsData.getFriendsDataFetcher())
            )
            // or full builder syntax if that takes your fancy
            .type(
                    newTypeWiring("Character")
                            .typeResolver(StarWarsData.getCharacterTypeResolver())
                            .build()
            )
            .build();
}
```

最后，您可以通过将静态模式和连接组合在一起来生成可执行模式，如以下示例所示：

```java
SchemaParser schemaParser = new SchemaParser();
SchemaGenerator schemaGenerator = new SchemaGenerator();

File schemaFile = loadSchema("starWarsSchema.graphqls");

TypeDefinitionRegistry typeRegistry = schemaParser.parse(schemaFile);
RuntimeWiring wiring = buildRuntimeWiring();
GraphQLSchema graphQLSchema = schemaGenerator.makeExecutableSchema(typeRegistry, wiring);
```

除了上面显示的生成器样式之外，TypeResolver和DataFetcher也可以使用WiringFactory接口进行连接。这允许更动态的运行时连接，因为可以检查SDL定义以决定连接什么。例如，您可以查看SDL指令或SDL定义的其他方面，以帮助您决定创建什么运行时。

```java
RuntimeWiring buildDynamicRuntimeWiring() {
    WiringFactory dynamicWiringFactory = new WiringFactory() {
        @Override
        public boolean providesTypeResolver(TypeDefinitionRegistry registry, InterfaceTypeDefinition definition) {
            return getDirective(definition,"specialMarker") != null;
        }

        @Override
        public boolean providesTypeResolver(TypeDefinitionRegistry registry, UnionTypeDefinition definition) {
            return getDirective(definition,"specialMarker") != null;
        }

        @Override
        public TypeResolver getTypeResolver(TypeDefinitionRegistry registry, InterfaceTypeDefinition definition) {
            Directive directive  = getDirective(definition,"specialMarker");
            return createTypeResolver(definition,directive);
        }

        @Override
        public TypeResolver getTypeResolver(TypeDefinitionRegistry registry, UnionTypeDefinition definition) {
            Directive directive  = getDirective(definition,"specialMarker");
            return createTypeResolver(definition,directive);
        }

        @Override
        public boolean providesDataFetcher(TypeDefinitionRegistry registry, FieldDefinition definition) {
            return getDirective(definition,"dataFetcher") != null;
        }

        @Override
        public DataFetcher getDataFetcher(TypeDefinitionRegistry registry, FieldDefinition definition) {
            Directive directive = getDirective(definition, "dataFetcher");
            return createDataFetcher(definition,directive);
        }
    };
    return RuntimeWiring.newRuntimeWiring()
            .wiringFactory(dynamicWiringFactory).build();
}
```

## 指令

假如有下面的类型：

```
type Employee
  id : ID
  name : String!
  startDate : String!
  salary : Float
}
```

向每个能看到该员工姓名的人发布工资信息可能不是您想要的。相反，你需要某种访问控制，这样如果你的角色是经理，你就可以看到薪水，否则你就得不到数据。

指令可以帮助您更容易地声明这一点。我们的上述声明大致如下：

```
directive @auth(role : String!) on FIELD_DEFINITION

type Employee
  id : ID
  name : String!
  startDate : String!
  salary : Float @auth(role : "manager")
}
```

```java
class AuthorisationDirective implements SchemaDirectiveWiring {

    @Override
    public GraphQLFieldDefinition onField(SchemaDirectiveWiringEnvironment<GraphQLFieldDefinition> environment) {
        String targetAuthRole = (String) environment.getDirective().getArgument("role").getValue();

        GraphQLFieldDefinition field = environment.getElement();
        GraphQLFieldsContainer parentType = environment.getFieldsContainer();
        //
        // build a data fetcher that first checks authorisation roles before then calling the original data fetcher
        //
        DataFetcher originalDataFetcher = environment.getCodeRegistry().getDataFetcher(parentType, field);
        DataFetcher authDataFetcher = new DataFetcher() {
            @Override
            public Object get(DataFetchingEnvironment dataFetchingEnvironment) throws Exception {
                Map<String, Object> contextMap = dataFetchingEnvironment.getContext();
                AuthorisationCtx authContext = (AuthorisationCtx) contextMap.get("authContext");

                if (authContext.hasRole(targetAuthRole)) {
                    return originalDataFetcher.get(dataFetchingEnvironment);
                } else {
                    return null;
                }
            }
        };
        //
        // now change the field definition to have the new authorising data fetcher
        environment.getCodeRegistry().dataFetcher(parentType, field, authDataFetcher);
        return field;
    }
}

//
// we wire this into the runtime by directive name
//
RuntimeWiring.newRuntimeWiring()
        .directive("auth", new AuthorisationDirective())
        .build();
```

这修改了GraphQLFieldDefinition，使得只有在当前授权上下文具有管理器角色时才会调用其原始数据获取器。您使用什么机制进行授权取决于您。例如，您可以使用SpringSecurity，graphql-java并不真正关心。

您将在graphql输入的执行“上下文”对象中提供此授权检查器，以便稍后可以在DataFetchingEnvironment中访问它:

```java
AuthorisationCtx authCtx = AuthorisationCtx.obtain();

ExecutionInput executionInput = ExecutionInput.newExecutionInput()
        .query(query)
        .context(authCtx)
        .build();
```

### 声明指令

为了在SDL中使用指令，graphql规范要求在使用它之前必须声明它的形状。我们上面的@auth指令示例在使用之前需要这样声明。

```
# This is a directive declaration
directive @auth(role : String!) on FIELD_DEFINITION

type Employee
  id : ID

  # and this is a usage of that declared directive
  salary : Float @auth(role : "manager")
}
```

一个例外是@deprecated指令，它为您隐式声明如下：

```
directive @deprecated(reason: String = "No longer supported") on FIELD_DEFINITION | ENUM_VALUE
```

有效的SDL指令位置如下：

- SCHEMA,
- SCALAR,
- OBJECT,
- FIELD_DEFINITION,
- ARGUMENT_DEFINITION,
- INTERFACE,
- UNION,
- ENUM,
- ENUM_VALUE,
- INPUT_OBJECT,
- INPUT_FIELD_DEFINITION

指令通常应用于字段定义，但正如您所看到的，有许多地方可以应用它们。

日期格式是一个贯穿各领域的问题，我们只需要编写一次并在许多领域应用它。
下面演示了一个示例模式指令，它可以将日期格式应用于LocaleDate对象的字段。
这个例子中最棒的是，它为应用它的每个字段添加了一个额外的格式参数。因此，客户端可以选择您为每个请求提供的日期格式。

```
directive @dateFormat on FIELD_DEFINITION

type Query {
  dateField : String @dateFormat
}
```

那么我们的运行时代码可以是：

```JAVA
public static class DateFormatting implements SchemaDirectiveWiring {
    @Override
    public GraphQLFieldDefinition onField(SchemaDirectiveWiringEnvironment<GraphQLFieldDefinition> environment) {
        GraphQLFieldDefinition field = environment.getElement();
        GraphQLFieldsContainer parentType = environment.getFieldsContainer();
        //
        // DataFetcherFactories.wrapDataFetcher is a helper to wrap data fetchers so that CompletionStage is handled correctly
        // along with POJOs
        //
        DataFetcher originalFetcher = environment.getCodeRegistry().getDataFetcher(parentType, field);
        DataFetcher dataFetcher = DataFetcherFactories.wrapDataFetcher(originalFetcher, ((dataFetchingEnvironment, value) -> {
            DateTimeFormatter dateTimeFormatter = buildFormatter(dataFetchingEnvironment.getArgument("format"));
            if (value instanceof LocalDateTime) {
                return dateTimeFormatter.format((LocalDateTime) value);
            }
            return value;
        }));

        //
        // This will extend the field by adding a new "format" argument to it for the date formatting
        // which allows clients to opt into that as well as wrapping the base data fetcher so it
        // performs the formatting over top of the base values.
        //
        FieldCoordinates coordinates = FieldCoordinates.coordinates(parentType, field);
        environment.getCodeRegistry().dataFetcher(coordinates, dataFetcher);

        return field.transform(builder -> builder
                .argument(GraphQLArgument
                        .newArgument()
                        .name("format")
                        .type(Scalars.GraphQLString)
                        .defaultValueProgrammatic("dd-MM-YYYY")
                )
        );
    }

    private DateTimeFormatter buildFormatter(String format) {
        String dtFormat = format != null ? format : "dd-MM-YYYY";
        return DateTimeFormatter.ofPattern(dtFormat);
    }
}

static GraphQLSchema buildSchema() {

    String sdlSpec = "directive @dateFormat on FIELD_DEFINITION\n" +
                  "type Query {\n" +
                  "    dateField : String @dateFormat \n" +
                  "}";

    TypeDefinitionRegistry registry = new SchemaParser().parse(sdlSpec);

    RuntimeWiring runtimeWiring = RuntimeWiring.newRuntimeWiring()
            .directive("dateFormat", new DateFormatting())
            .build();

    return new SchemaGenerator().makeExecutableSchema(registry, runtimeWiring);
}

public static void main(String[] args) {
    GraphQLSchema schema = buildSchema();
    GraphQL graphql = GraphQL.newGraphQL(schema).build();

    Map<String, Object> root = new HashMap<>();
    root.put("dateField", LocalDateTime.of(1969, 10, 8, 0, 0));

    String query = "" +
            "query {\n" +
            "    default : dateField \n" +
            "    usa : dateField(format : \"MM-dd-YYYY\") \n" +
            "}";

    ExecutionInput executionInput = ExecutionInput.newExecutionInput()
            .root(root)
            .query(query)
            .build();

    ExecutionResult executionResult = graphql.execute(executionInput);
    Map<String, Object> data = executionResult.getData();

    // data['default'] == '08-10-1969'
    // data['usa'] == '10-08-1969'
}
```

请注意，SDL定义没有格式参数，一旦应用了指令连接，它就被添加到字段定义中，因此客户端可以开始使用它。
请注意，graphql-java没有随此实现一起提供。这里只提供了一个示例，说明您可以自己添加什么。

### 指令链

指令按照遇到的顺序应用。例如，想象一下改变字段值大小写的指令。

```java
directive @uppercase on FIELD_DEFINITION
directive @lowercase on FIELD_DEFINITION
directive @mixedcase on FIELD_DEFINITION
directive @reversed on FIELD_DEFINITION

type Query {
  lowerCaseValue : String @uppercase
  upperCaseValue : String @lowercase
  mixedCaseValue : String @mixedcase

  #
  # directives are applied in order hence this will be lower, then upper, then mixed then reversed
  #
  allTogetherNow : String @lowercase @uppercase @mixedcase @reversed
}
```

当执行上述指令时，每个指令将一个应用于另一个之上。每个指令实现都应该小心地保留以前的数据获取器以保持行为（当然，除非您打算放弃它）
