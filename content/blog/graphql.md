---
title: graphql规范
tags:
  - graphql
categories:
  - 规范
date: 2022-11-27 19:02:18
---

GraphQL 是一个用于 API 的查询语言，是一个使用基于类型系统来执行查询的服务端运行时。GraphQL 并没有和任何特定数据库或者存储引擎绑定，而是依靠你现有的代码和数据支撑。

## schema定义

schema定义由普通对象类型和内置类型组成。其中内置类型是query(用于查询)和mutation（用于修改） ，两者之一必须存在schema文件中（ 因为其是GraphQL 查询的**入口**）。有必要记住的是，除了作为 schema 的入口，Query 和 Mutation 类型与其它 GraphQL 对象类型别无二致，它们的字段也是一样的工作方式。

下面是一个示例文件：

```graphql
scalar LocalDate

type Query { #必须类型
    queryUsers: [User]
    queryByBirth(birth:LocalDate):User
    queryByDetail(birth:LocalDate,name:String):User
}

type User { # 可选类型
    id: String
    name: String
    age: Int
    birth: LocalDate
}

```

### 类型

#### 标量类型

GraphQL 自带一组默认标量类型：

- Int：有符号 32 位整数。
- Float：有符号双精度浮点值。
- String：UTF‐8 字符序列。
- Boolean：true 或者 false。
- ID：ID 标量类型表示一个唯一标识符，通常用以重新获取对象或者作为缓存中的键。ID 类型使用和 String 一样的方式序列化；

当然我们也可以自定义标量类型，例如上面的 `LocalDate` 。 除此之外，还需要在我们的实现中定义序列化、反序列化和验证等方法。

#### 枚举类型

```
enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}
```

#### 列表类型

在 GraphQL schema 语言中，我们通过将类型包在方括号（[ 和 ]）中的方式来标记列表：

```
myField: [String]
```

#### 接口

**接口**是一个抽象类型，它包含某些字段，而对象类型必须包含这些字段，才能算实现了这个接口。

```graphql
interface Animal {
  id: ID!
  name: String!
}
```

具体的实现：

```
type Cat implements Animal{
  id: ID!
  name: String!
  color: String
}
```

#### 联合类型

```
union SearchResult = Human | Droid | Starship
```

联合类型的成员需要是具体对象类型；你不能使用接口或者其他联合类型来创造一个联合类型。

#### 输入类型（Input Types）

目前为止，我们只讨论过将例如枚举和字符串等标量值作为参数传递给字段，但是怎么传递复杂对象呢？这在变更（mutation）中特别有用。答案是输入对象，其看上去和常规对象一模一样，除了关键字是 input 而不是 type：

```
input ReviewInput {
  stars: Int!
  commentary: String
}
```

你可以像这样在变更（mutation）中使用输入对象类型：

```graphql
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}
```

输入：

```json
{
  "ep": "JEDI",
  "review": {
    "stars": 5,
    "commentary": "This is a great movie!"
  }
}
```

输出：

```json
{
  "data": {
    "createReview": {
      "stars": 5,
      "commentary": "This is a great movie!"
    }
  }
}
```

### 非空限制

```
type Character {
  name: String!
  appearsIn: [Episode]!
}
```

类型名后面添加一个感叹号!将其标注为**非空**，这表示我们的服务器对于这个字段，总是会返回一个非空值，如果它结果得到了一个空值，那么事实上将会触发一个 GraphQL 执行错误，以让客户端知道发生了错误。

非空和列表修饰符可以组合使用。例如你可以要求一个非空字符串的数组：

```
myField: [String!]
```

这表示**数组本身**可以为空，但是其不能有任何空值成员。用 JSON 举例如下：

```
myField: null // 有效
myField: [] // 有效
myField: ['a', 'b'] // 有效
myField: ['a', null, 'b'] // 错误
```

然后，我们来定义一个不可为空的字符串数组：

```
myField: [String]!
```

这表示数组本身不能为空，但是其可以包含空值成员：

```
myField: null // 错误
myField: [] // 有效
myField: ['a', 'b'] // 有效
myField: ['a', null, 'b'] // 有效
```

### 参数

GraphQL 对象类型上的每一个字段都可能有零个或者多个参数，表示可以在该字段上进行筛选。例如下面的 length 字段：

```
type Starship {
  id: ID!
  name: String!
  length(unit: LengthUnit = METER): Float
}
```

所有参数都是具名的,上面的例子中，unit是参数的名称，LengthUnit是参数的类型，METER是默认值。

## 查询

### 基础查询

```
{
  hero {
    name
    # 查询可以有注释
    friends {
      name
    }
  }
}
```

返回结果：

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friends": [
        {
          "name": "Luke Skywalker"
        },
        {
          "name": "Han Solo"
        },
        {
          "name": "Leia Organa"
        }
      ]
    }
  }
}
```

### 带参数的查询

单个参数：

```
{
  human(id: "1000") {
    name
    height
  }
}
```

多个参数：

```

{
  human(id: "1000") {
    name
    height(unit: FOOT)
  }
}
```

### 别名

通过不同参数值来查询相同字段。这便是为何你需要**别名** —— 这可以让你重命名结果中的字段为任意的名字。

```
{
  empireHero: hero(episode: EMPIRE) {
    name
  }
  jediHero: hero(episode: JEDI) {
    name
  }
}
```

```json
{
  "data": {
    "empireHero": {
      "name": "Luke Skywalker"
    },
    "jediHero": {
      "name": "R2-D2"
    }
  }
}
```

### 变量

上面讲述的所有例子中，参数是限定好的。有些情况下，我们需要动态选择参数。变量可以帮我实现这个功能。

使用变量之前，我们得做三件事：

1. 使用 $variableName 替代查询中的静态值。
2. 声明 $variableName 为查询接受的变量之一。
3. 将 variableName: value 传递到查询中。

```
{ "graphiql": true, "variables": { "episode": JEDI } } # 传递变量

query HeroNameAndFriends($episode: Episode = "JEDI") { #声明变量
  hero(episode: $episode) {# 使用变量
    name
    friends {
      name
    }
  }
}
```

这样一来，我们的客户端代码就只需要传入不同的变量，而不用构建一个全新的查询了。这事实上也是一个良好实践，意味着查询的参数将是动态的 —— 我们决不能使用用户提供的值来字符串插值以构建查询。

### 操作名称

上面的示例中，我们使用了操作名称：HeroNameAndFriends。这是详细写法，也鼓励这么做。因为它对于调试和服务器端日志记录非常有用。

### 片段

**片段**是可复用单元

```
{
  leftComparison: hero(episode: EMPIRE) {
    ...comparisonFields  # 使用片段
  }
  rightComparison: hero(episode: JEDI) {
    ...comparisonFields
  }
}
# 声明片段
fragment comparisonFields on Character {
  name
  appearsIn
  friends {
    name
  }
}
```

在片段中使用变量：

```
query HeroComparison($first: Int = 3) {
  leftComparison: hero(episode: EMPIRE) {
    ...comparisonFields
  }
  rightComparison: hero(episode: JEDI) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  friendsConnection(first: $first) {
    totalCount
    edges {
      node {
        name
      }
    }
  }
}
```

### 内联片段

如果你查询的字段返回的是接口或者联合类型，那么你可能需要使用**内联片段**来取出下层具体类型的数据：

```
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid {
      primaryFunction
    }
    ... on Human {
      height
    }
  }
}
```

传入变量：

```
{
  "ep": "JEDI"
}
```

返回数据

```
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "primaryFunction": "Astromech"
    }
  }
}
```

如果要请求具体类型上的字段，你需要使用一个类型条件**内联片段**。因为第一个片段标注为 ... on Droid，primaryFunction 仅在 hero 返回的 Character 为 Droid 类型时才会执行。

### 内省

我们有时候会需要去问 GraphQL Schema 它支持哪些查询。GraphQL 通过内省系统让我们可以做到这点！

查询所有类型：

```
{
  __schema {
    types {
      name
    }
  }
}
```

返回结果：

```json
{
  "data": {
    "__schema": {
      "types": [
        {
          "name": "Boolean"
        },
        {
          "name": "Int"
        },
        {
          "name": "LocalDate"
        },
        {
          "name": "Query"
        },
        {
          "name": "String"
        },
        {
          "name": "User"
        },
        {
          "name": "__Directive"
        },
        {
          "name": "__DirectiveLocation"
        },
        {
          "name": "__EnumValue"
        },
        {
          "name": "__Field"
        },
        {
          "name": "__InputValue"
        },
        {
          "name": "__Schema"
        },
        {
          "name": "__Type"
        },
        {
          "name": "__TypeKind"
        }
      ]
    }
  }
}
```

* String, Boolean - 这些是内建的标量，由类型系统提供。
* `__Schema`, `__Type`,`__TypeKind`, `__Field`, `__InputValue`, `__EnumValue`, `__Directive` - 这些有着两个下划线的类型是内省系统的一部分。

### 修改

GraphQL 的大部分讨论集中在数据获取，但是任何完整的数据平台也都需要一个改变服务端数据的方法。

```
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}
```

变量：

```
{
  "ep": "JEDI",
  "review": {
    "stars": 5,
    "commentary": "This is a great movie!"
  }
}
```

```
{
  "data": {
    "createReview": {
      "stars": 5,
      "commentary": "This is a great movie!"
    }
  }
}
```

注意 createReview 字段如何返回了新建的 review 的 stars 和 commentary 字段。

**查询字段时，是并行执行，而变更字段时，是线性执行，一个接着一个。**

这意味着如果我们一个请求中发送了两个 incrementCredits 变更，第一个保证在第二个之前执行，以确保我们不会出现竞态。



### 指令

我们上面讨论的变量使得我们可以避免手动字符串插值构建动态查询。传递变量给参数解决了一大堆这样的问题，但是我们可能也需要一个方式使用变量动态地改变我们查询的结构。

```
query Hero($episode: Episode, $withFriends: Boolean!) {
  hero(episode: $episode) {
    name
    friends @include(if: $withFriends) {
      name
    }
  }
}
```

传入变量：

```
{
  "episode": "JEDI",
  "withFriends": false
}
```

输出结果：

```
{
  "data": {
    "hero": {
      "name": "R2-D2"
    }
  }
}
```

尝试修改上面的变量，传递 true 给 withFriends，看看结果的变化。

我们用了 GraphQL 中一种称作**指令**的新特性。一个指令可以附着在字段或者片段包含的字段上，然后以任何服务端期待的方式来改变查询的执行。GraphQL 的核心规范包含两个指令，其必须被任何规范兼容的 GraphQL 服务器实现所支持：

- @include(if: Boolean) 仅在参数为 true 时，包含此字段。
- @skip(if: Boolean) 如果参数为 true，跳过此字段。



## HTTP请求

你的 GraphQL HTTP 服务器应当能够处理 HTTP GET 和 POST 方法。

### GET 请求

在收到一个 HTTP GET 请求时，应当在 “query” 查询字符串（query string）中指定 GraphQL 查询。例如，如果我们要执行以下 GraphQL 查询：

```graphql
{
  me {
    name
  }
}
```

此请求可以通过 HTTP GET 发送，如下所示：

```javascript
http://myapi/graphql?query={me{name}}
```

查询变量可以作为 JSON 编码的字符串发送到名为 `variables` 的附加查询参数中。如果查询包含多个具名操作，则可以使用一个 `operationName` 查询参数来控制哪一个应当执行。

### POST 请求

标准的 GraphQL POST 请求应当使用 `application/json` 内容类型（content type），并包含以下形式 JSON 编码的请求体：

```javascript
{
  "query": "...",
  "operationName": "...",
  "variables": { "myVariable": "someValue", ... }
}
```

`operationName` 和 `variables` 是可选字段。仅当查询中存在多个操作时才需要 `operationName`。

除了上边这种请求之外，我们还建议支持另外两种情况：

- 如果存在 “query” 这一查询字符串参数（如上面的 GET 示例中），则应当以与 HTTP GET 相同的方式进行解析和处理。
- 如果存在 “application/graphql” Content-Type 头，则将 HTTP POST 请求体内容视为 GraphQL 查询字符串。

如果你使用的是 express-graphql，那么你已经直接获得了这些支持。

### 响应

无论使用任何方法发送查询和变量，响应都应当以 JSON 格式在请求正文中返回。如规范中所述，查询结果可能会是一些数据和一些错误，并且应当用以下形式的 JSON 对象返回：

```javascript
{
  "data": { ... },
  "errors": [ ... ]
}
```

如果没有返回错误，响应中不应当出现 `"errors"` 字段。如果没有返回数据，则 [根据 GraphQL 规范](http://spec.graphql.cn//#sec-Data-)，只能在执行期间发生错误时才能包含 `"data"` 字段。
