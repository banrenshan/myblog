---
title: jackson
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:41:07
---

# ObjectMapper

## 基本使用

```java
public class Car {

    private String color;
    private String type;

    // standard getters setters
}
```

**序列化:**

```java
ObjectMapper objectMapper = new ObjectMapper();
Car car = new Car("yellow", "renault");
objectMapper.writeValue(new File("target/car.json"), car);
{"color":"yellow","type":"renault"}
```

ObjectMapper类的writeValueAsString和writeValueAsBytes方法从Java对象生成JSON，并将生成的JSON作为字符串或字节数组返回：

```java
String carAsString = objectMapper.writeValueAsString(car);
```

反序列化：

```java
String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\" }";
Car car = objectMapper.readValue(json, Car.class);	
```

readValue函数还接受其他形式的输入，例如包含JSON字符串的文件：

```java
Car car = objectMapper.readValue(new File("src/test/resources/json_car.json"), Car.class);
```

或者从URL中：

```java
Car car = 
  objectMapper.readValue(new URL("file:src/test/resources/json_car.json"), Car.class);
```

## JsonNode：json的表示对象

或者，可以将JSON解析为JsonNode对象，并用于从特定节点检索数据：

```java
String json = "{ \"color\" : \"Black\", \"type\" : \"FIAT\" }";
JsonNode jsonNode = objectMapper.readTree(json);
String color = jsonNode.get("color").asText();
// Output: color -> Black
```

更低层次的转换：

```java
@Test
public void givenUsingLowLevelApi_whenParsingJsonStringIntoJsonNode_thenCorrect() 
  throws JsonParseException, IOException {
    String jsonString = "{"k1":"v1","k2":"v2"}";

    ObjectMapper mapper = new ObjectMapper();
    JsonFactory factory = mapper.getFactory();
    JsonParser parser = factory.createParser(jsonString);
    JsonNode actualObj = mapper.readTree(parser);

    assertNotNull(actualObj);
}
```

## 转换成集合类型

我们可以使用TypeReference将数组形式的JSON解析为Java对象列表：

```java
String jsonCarArray = 
  "[{ \"color\" : \"Black\", \"type\" : \"BMW\" }, { \"color\" : \"Red\", \"type\" : \"FIAT\" }]";
List<Car> listCar = objectMapper.readValue(jsonCarArray, new TypeReference<List<Car>>(){});
String jsonCarArray = 
  "[{ \"color\" : \"Black\", \"type\" : \"BMW\" }, { \"color\" : \"Red\", \"type\" : \"FIAT\" }]";
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(DeserializationFeature.USE_JAVA_ARRAY_FOR_JSON_ARRAY, true);
Car[] cars = objectMapper.readValue(jsonCarArray, Car[].class);
// print cars
```



类似地，我们可以将JSON解析为Java Map：

```java
String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\" }";
Map<String, Object> map 
  = objectMapper.readValue(json, new TypeReference<Map<String,Object>>(){});
```































# 注解

## 常规注解

### ***@JsonProperty***

我们可以添加@JsonProperty注释，表示JSON中的属性名称。

```java
public class MyBean {
    public int id;
    private String name;

    @JsonProperty("name")
    public void setTheName(String name) {
        this.name = name;
    }

    @JsonProperty("name")
    public String getTheName() {
        return name;
    }
}
@Test
public void whenUsingJsonProperty_thenCorrect()
  throws IOException {
    MyBean bean = new MyBean(1, "My bean");

    String result = new ObjectMapper().writeValueAsString(bean);
    
    assertThat(result, containsString("My bean"));
    assertThat(result, containsString("1"));

    MyBean resultBean = new ObjectMapper()
      .readerFor(MyBean.class)
      .readValue(result);
    assertEquals("My bean", resultBean.getTheName());
}
```

### ***@JsonFormat***

@JsonFormat注释指定序列化日期/时间值时的格式。

```java
public class EventWithFormat {
    public String name;

    @JsonFormat(
      shape = JsonFormat.Shape.STRING,
      pattern = "dd-MM-yyyy hh:mm:ss")
    public Date eventDate;
}
@Test
public void whenSerializingUsingJsonFormat_thenCorrect()
  throws JsonProcessingException, ParseException {
    SimpleDateFormat df = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");
    df.setTimeZone(TimeZone.getTimeZone("UTC"));

    String toParse = "20-12-2014 02:30:00";
    Date date = df.parse(toParse);
    EventWithFormat event = new EventWithFormat("party", date);
    
    String result = new ObjectMapper().writeValueAsString(event);
    
    assertThat(result, containsString(toParse));
}
```

### ***@JsonUnwrapped***

@JsonUnwrapped定义了序列化/反序列化时应展开/展平的值。

```java
public class UnwrappedUser {
    public int id;

    @JsonUnwrapped
    public Name name;

    public static class Name {
        public String firstName;
        public String lastName;
    }
}
@Test
public void whenSerializingUsingJsonUnwrapped_thenCorrect()
  throws JsonProcessingException, ParseException {
    UnwrappedUser.Name name = new UnwrappedUser.Name("John", "Doe");
    UnwrappedUser user = new UnwrappedUser(1, name);

    String result = new ObjectMapper().writeValueAsString(user);
    
    assertThat(result, containsString("John"));
    assertThat(result, not(containsString("name")));
}
{
    "id":1,
    "firstName":"John",
    "lastName":"Doe"
}
```

### ***@JsonView***

@JsonView指示将在其中包含属性以进行序列化/反序列化的视图。

```java
public class Views {
    public static class Public {}
    public static class Internal extends Public {}
}
public class Item {
    @JsonView(Views.Public.class)
    public int id;

    @JsonView(Views.Public.class)
    public String itemName;

    @JsonView(Views.Internal.class)
    public String ownerName;
}
@Test
public void whenSerializingUsingJsonView_thenCorrect()
  throws JsonProcessingException {
    Item item = new Item(2, "book", "John");

    String result = new ObjectMapper()
      .writerWithView(Views.Public.class)
      .writeValueAsString(item);

    assertThat(result, containsString("book"));
    assertThat(result, containsString("2"));
    assertThat(result, not(containsString("John")));
}
```

### ***@JsonManagedReference, @JsonBackReference***

@JsonManagedReference和@JsonBackReference注释可以处理父/子关系并处理循环依赖。

```java
public class ItemWithRef {
    public int id;
    public String itemName;

    @JsonManagedReference
    public UserWithRef owner;
}
public class UserWithRef {
    public int id;
    public String name;

    @JsonBackReference
    public List<ItemWithRef> userItems;
}
@Test
public void whenSerializingUsingJacksonReferenceAnnotation_thenCorrect()
  throws JsonProcessingException {
    UserWithRef user = new UserWithRef(1, "John");
    ItemWithRef item = new ItemWithRef(2, "book", user);
    user.addItem(item);

    String result = new ObjectMapper().writeValueAsString(item);

    assertThat(result, containsString("book"));
    assertThat(result, containsString("John"));
    assertThat(result, not(containsString("userItems")));
}
{"id":2,"itemName":"book","owner":{"id":1,"name":"zzq"}}
{"id":1,"name":"zzq"}
```



### ***@JsonIdentityInfo***

@JsonIdentityInfo指出在序列化/反序列化值时应该使用对象标识，例如在处理无限递归类型的问题时。

```java
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id")
public class ItemWithIdentity {
    public int id;
    public String itemName;
    public UserWithIdentity owner;
}
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id")
public class UserWithIdentity {
    public int id;
    public String name;
    public List<ItemWithIdentity> userItems;
}
```



```java
@Test
public void whenSerializingUsingJsonIdentityInfo_thenCorrect()
  throws JsonProcessingException {
    UserWithIdentity user = new UserWithIdentity(1, "John");
    ItemWithIdentity item = new ItemWithIdentity(2, "book", user);
    user.addItem(item);

    String result = new ObjectMapper().writeValueAsString(item);

    assertThat(result, containsString("book"));
    assertThat(result, containsString("John"));
    assertThat(result, containsString("userItems"));
}
{
    "id": 2,
    "itemName": "book",
    "owner": {
        "id": 1,
        "name": "John",
        "userItems": [
            2
        ]
    }
}
```

### ***@JsonFilter***

@JsonFilter注释指定了在序列化过程中使用的过滤器。

```java
@JsonFilter("myFilter")
public class BeanWithFilter {
    public int id;
    public String name;
}
@Test
public void whenSerializingUsingJsonFilter_thenCorrect()
  throws JsonProcessingException {
    BeanWithFilter bean = new BeanWithFilter(1, "My bean");

    FilterProvider filters 
      = new SimpleFilterProvider().addFilter(
        "myFilter", 
        SimpleBeanPropertyFilter.filterOutAllExcept("name"));

    String result = new ObjectMapper()
      .writer(filters)
      .writeValueAsString(bean);

    assertThat(result, containsString("My bean"));
    assertThat(result, not(containsString("id")));
}
```

## 序列化

### ***@JsonAnyGetter***

@JsonAnyGetter允许灵活使用Map字段作为标准属性。



```java
public class User {
    private String name;
    private Map<String, Object> others;
}


User user = new User();
user.setName("zzq");
user.setOthers(Map.of("age", 12, "height", 178));
String s = objectMapper.writeValueAsString(user);
System.err.println(s);
{"name":"zzq","others":{"age":12,"height":178}}
```

### ***@JsonGetter***

@JsonGetter注释是@JsonProperty注释的替代，将方法标记为getter方法。

```java
public class MyBean {
    public int id;
    private String name;

    @JsonGetter("name")
    public String getTheName() {
        return name;
    }
}


MyBean bean = new MyBean();
bean.setName("zzq");
bean.setId(1);
String s = objectMapper.writeValueAsString(bean);
System.err.println(s);
{"id":1,"name":"zzq"}
```

### ***@JsonPropertyOrder***

我们可以使用@JsonPropertyOrder注释来指定序列化时属性的顺序。

```java
@JsonPropertyOrder({ "name", "id" })
public class MyBean {
    public int id;
    public String name;
}

MyBean bean = new MyBean();
bean.setName("zzq");
bean.setId(1);
String s = objectMapper.writeValueAsString(bean);
System.err.println(s);
{"name":"zzq","id":1}
```

### ***@JsonRawValue***

@JsonRawValue注释可以指示Jackson按原样序列化属性。

在下面的示例中，我们使用@JsonRawValue嵌入一些自定义JSON作为实体s属性的值：

```java
public class RawBean {
    public String name;

    @JsonRawValue
    public String json;
}

RawBean bean=new RawBean();
bean.setName("zzq");
bean.setJson("{\"age\":33}");
String s = objectMapper.writeValueAsString(bean);
System.err.println(s);
{"name":"zzq","json":{"age":33}}
```

### ***@JsonValue***

@JsonValue使用标注的方法序列化整个实例。例如，在枚举中，我们用@JsonValue注释getName，以便通过其名称序列化任何这样的实体：

```java
public enum TypeEnumWithValue {
    TYPE1(1, "Type A"), TYPE2(2, "Type 2");

    private Integer id;
    private String name;

    TypeEnumWithValue(int i, String s) {
        this.id = i;
        this.name = s;
    }

    @JsonValue
    public String getName() {
        return name;
    }
}

String s = objectMapper.writeValueAsString(TypeEnumWithValue.TYPE1);
System.err.println(s);
"Type A"
```

### ***@JsonRootName***

如果启用了包装，则使用@JsonRootName注释来指定要使用的根包装的名称。

```java
@JsonRootName(value = "user")

public class MyBean {

    public int id;
    private String name;

}


objectMapper.enable(SerializationFeature.WRAP_ROOT_VALUE);
MyBean bean = new MyBean();
bean.setName("zzq");
bean.setId(1);
String s = objectMapper.writeValueAsString(bean);
System.err.println(s);
{"user":{"name":"zzq","id":1}}
```

### ***@JsonSerialize***

@JsonSerialize指示编组实体时要使用的自定义序列化程序。

```java
public class EventWithSerializer {
    public String name;

    @JsonSerialize(using = CustomDateSerializer.class)
    public Date eventDate;
}

public class CustomDateSerializer extends StdSerializer<Date> {

    private static SimpleDateFormat formatter 
      = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

    public CustomDateSerializer() { 
        this(null); 
    } 

    public CustomDateSerializer(Class<Date> t) {
        super(t); 
    }

    @Override
    public void serialize(
      Date value, JsonGenerator gen, SerializerProvider arg2) 
      throws IOException, JsonProcessingException {
        gen.writeString(formatter.format(value));
    }
}
@Test
public void whenSerializingUsingJsonSerialize_thenCorrect()
  throws JsonProcessingException, ParseException {
 
    SimpleDateFormat df
      = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

    String toParse = "20-12-2014 02:30:00";
    Date date = df.parse(toParse);
    EventWithSerializer event = new EventWithSerializer("party", date);

    String result = new ObjectMapper().writeValueAsString(event);
    assertThat(result, containsString(toParse));
}
```

## 反序列化

### ***@JsonCreator***

我们可以使用@JsonCreator注释来调整反序列化中使用的构造函数或构造工厂。当我们需要反序列化与我们需要获取的目标实体不完全匹配的JSON时，这非常有用。

让我们看一个例子。假设我们需要反序列化以下JSON：

```json
{
    "id":1,
    "theName":"My bean"
}
```

目标实体中没有*theName* 字段。只有*name* 字段。 现在我们不想改变实体本身，我们只需要用@JsonCreator注释构造函数，并使用@JsonProperty注释来对解组过程进行更多的控制：

```java
public class BeanWithCreator {
    public int id;
    public String name;

    @JsonCreator
    public BeanWithCreator(
        @JsonProperty("id") int id, 
        @JsonProperty("theName") String name) {
        this.id = id;
        this.name = name;
    }
}
```

### ***@JacksonInject***

@JacksonInject表示属性将从注入而不是从JSON数据中获取其值。

在下面的示例中，我们使用@JacksonInject注入属性id：

```java
public class BeanWithInject {
    @JacksonInject
    public int id;
    
    public String name;
}
@Test
public void whenDeserializingUsingJsonInject_thenCorrect()
  throws IOException {
 
    String json = "{\"name\":\"My bean\"}";
    
    InjectableValues inject = new InjectableValues.Std()
      .addValue(int.class, 1);
    BeanWithInject bean = new ObjectMapper().reader(inject)
      .forType(BeanWithInject.class)
      .readValue(json);
    
    assertEquals("My bean", bean.name);
    assertEquals(1, bean.id);
}
```

### ***@JsonAnySetter***

@JsonAnySetter允许我们灵活地使用Map作为标准属性。在反序列化时，JSON中的没有匹配的属性将简单地添加到Map中。

```java
public class ExtendableBean {
    public String name;
    private Map<String, String> properties;

    @JsonAnySetter
    public void add(String key, String value) {
        properties.put(key, value);
    }
}
@Test
public void whenDeserializingUsingJsonAnySetter_thenCorrect()
  throws IOException {
    String json
      = "{\"name\":\"My bean\",\"attr2\":\"val2\",\"attr1\":\"val1\"}";

    ExtendableBean bean = new ObjectMapper()
      .readerFor(ExtendableBean.class)
      .readValue(json);
    
    assertEquals("My bean", bean.name);
    assertEquals("val2", bean.getProperties().get("attr2"));
}
```

### ***@JsonSetter***

@JsonSetter是@JsonProperty的替代方法，它将该方法标记为setter方法。

当我们需要读取一些JSON数据时，这非常有用，目标实体类与该数据不完全匹配，因此我们需要调整流程以使其适合。

```java
public class MyBean {
    public int id;
    private String name;

    @JsonSetter("name")
    public void setTheName(String name) {
        this.name = name;
    }
}
@Test
public void whenDeserializingUsingJsonSetter_thenCorrect()
  throws IOException {
 
    String json = "{\"id\":1,\"name\":\"My bean\"}";

    MyBean bean = new ObjectMapper()
      .readerFor(MyBean.class)
      .readValue(json);
    assertEquals("My bean", bean.getTheName());
}
```

### ***@JsonDeserialize***

@JsonDeserialize表示使用自定义反序列化程序。

```java
public class EventWithSerializer {
    public String name;

    @JsonDeserialize(using = CustomDateDeserializer.class)
    public Date eventDate;
}
public class CustomDateDeserializer
  extends StdDeserializer<Date> {

    private static SimpleDateFormat formatter
      = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

    public CustomDateDeserializer() { 
        this(null); 
    } 

    public CustomDateDeserializer(Class<?> vc) { 
        super(vc); 
    }

    @Override
    public Date deserialize(
      JsonParser jsonparser, DeserializationContext context) 
      throws IOException {
        
        String date = jsonparser.getText();
        try {
            return formatter.parse(date);
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    }
}
@Test
public void whenDeserializingUsingJsonDeserialize_thenCorrect()
  throws IOException {
 
    String json
      = "{"name":"party","eventDate":"20-12-2014 02:30:00"}";

    SimpleDateFormat df
      = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");
    EventWithSerializer event = new ObjectMapper()
      .readerFor(EventWithSerializer.class)
      .readValue(json);
    
    assertEquals(
      "20-12-2014 02:30:00", df.format(event.eventDate));
}
```

### *@JsonAlias*

@JsonAlias在反序列化期间为属性定义了一个或多个备选名称。

```java
public class AliasBean {
    @JsonAlias({ "fName", "f_name" })
    private String firstName;   
    private String lastName;
}
```

这里我们有一个POJO，我们想将带有fName、f_name和firstName等值的JSON反序列化到POJO的firstName变量中。

```java
@Test
public void whenDeserializingUsingJsonAlias_thenCorrect() throws IOException {
    String json = "{\"fName\": \"John\", \"lastName\": \"Green\"}";
    AliasBean aliasBean = new ObjectMapper().readerFor(AliasBean.class).readValue(json);
    assertEquals("John", aliasBean.getFirstName());
}
```

## 包含

### ***@JsonIgnoreProperties***

@JsonIgnoreProperties是一个类级注释，用于标记忽略的属性或属性列表。序列化和反序列化都支持

```java
@JsonIgnoreProperties({ "id" })
public class BeanWithIgnore {
    public int id;
    public String name;
}
@Test
public void whenSerializingUsingJsonIgnoreProperties_thenCorrect()
  throws JsonProcessingException {
 
    BeanWithIgnore bean = new BeanWithIgnore(1, "My bean");

    String result = new ObjectMapper()
      .writeValueAsString(bean);
    
    assertThat(result, containsString("My bean"));
    assertThat(result, not(containsString("id")));
}
```

忽略JSON输入中的任何未知属性，我们可以设置@JsonIgnoreProperties注释的ignoreUnknown=true。

### ***@JsonIgnore***

@JsonIgnore注释用于在字段级别标记要忽略的属性。序列化和反序列化都支持

```java
public class BeanWithIgnore {
    @JsonIgnore
    public int id;

    public String name;
}
@Test
public void whenSerializingUsingJsonIgnore_thenCorrect()
  throws JsonProcessingException {
 
    BeanWithIgnore bean = new BeanWithIgnore(1, "My bean");

    String result = new ObjectMapper()
      .writeValueAsString(bean);
    
    assertThat(result, containsString("My bean"));
    assertThat(result, not(containsString("id")));
}
```

### ***@JsonIgnoreType***

@JsonIgnoreType将注释类型的所有属性标记为忽略。序列化和反序列化都支持

```java
public class User {
    public int id;
    public Name name;

    @JsonIgnoreType
    public static class Name {
        public String firstName;
        public String lastName;
    }
}
@Test
public void whenSerializingUsingJsonIgnoreType_thenCorrect()
  throws JsonProcessingException, ParseException {
 
    User.Name name = new User.Name("John", "Doe");
    User user = new User(1, name);

    String result = new ObjectMapper()
      .writeValueAsString(user);

    assertThat(result, containsString("1"));
    assertThat(result, not(containsString("name")));
    assertThat(result, not(containsString("John")));
}
```

### ***@JsonInclude***

我们可以使用@JsonInclude排除具有空/null/default值的属性。

```java
@JsonInclude(Include.NON_NULL)
public class MyBean {
    public int id;
    public String name;
}
public void whenSerializingUsingJsonInclude_thenCorrect()
  throws JsonProcessingException {
 
    MyBean bean = new MyBean(1, null);

    String result = new ObjectMapper()
      .writeValueAsString(bean);
    
    assertThat(result, containsString("1"));
    assertThat(result, not(containsString("name")));
}
```

### ***@JsonAutoDetect***

@JsonAutoDetect可以覆盖哪些属性可见，哪些属性不可见的默认语义。

```java
@JsonAutoDetect(fieldVisibility = Visibility.ANY)
public class PrivateBean {
    private int id;
    private String name;
}
@Test
public void whenSerializingUsingJsonAutoDetect_thenCorrect()
  throws JsonProcessingException {
 
    PrivateBean bean = new PrivateBean(1, "My bean");

    String result = new ObjectMapper()
      .writeValueAsString(bean);
    
    assertThat(result, containsString("1"));
    assertThat(result, containsString("My bean"));
}
```

## 多态

- @JsonTypeInfo–指示要在序列化中包含的类型信息的详细信息
- @JsonSubTypes–指示带注释类型的子类型
- @JsonTypeName–定义用于带注释类的逻辑类型名称

```java
public class Zoo {
    public Animal animal;

    @JsonTypeInfo(
      use = JsonTypeInfo.Id.NAME, 
      include = As.PROPERTY, 
      property = "type")
    @JsonSubTypes({
        @JsonSubTypes.Type(value = Dog.class, name = "dog"),
        @JsonSubTypes.Type(value = Cat.class, name = "cat")
    })
    public static class Animal {
        public String name;
    }

    @JsonTypeName("dog")
    public static class Dog extends Animal {
        public double barkVolume;
    }

    @JsonTypeName("cat")
    public static class Cat extends Animal {
        boolean likesCream;
        public int lives;
    }
}
@Test
public void whenSerializingPolymorphic_thenCorrect()
  throws JsonProcessingException {
    Zoo.Dog dog = new Zoo.Dog("lacy");
    Zoo zoo = new Zoo(dog);

    String result = new ObjectMapper()
      .writeValueAsString(zoo);

    assertThat(result, containsString("type"));
    assertThat(result, containsString("dog"));
}
```

以下是使用Dog序列化Zoo实例的结果：

```json
{
    "animal": {
        "type": "dog",
        "name": "lacy",
        "barkVolume": 0
    }
}
```

现在进行反序列化。让我们从以下JSON输入开始：

```json
{
    "animal":{
        "name":"lacy",
        "type":"cat"
    }
}
@Test
public void whenDeserializingPolymorphic_thenCorrect()
throws IOException {
    String json = "{\"animal\":{\"name\":\"lacy\",\"type\":\"cat\"}}";

    Zoo zoo = new ObjectMapper()
      .readerFor(Zoo.class)
      .readValue(json);

    assertEquals("lacy", zoo.animal.name);
    assertEquals(Zoo.Cat.class, zoo.animal.getClass());
}
```

## 自定义注解

接下来，让我们看看如何创建自定义Jackson注释。我们可以使用@JacksonAnnotationsInside注释：

```java
    @Retention(RetentionPolicy.RUNTIME)
    @JacksonAnnotationsInside
    @JsonInclude(Include.NON_NULL)
    @JsonPropertyOrder({ "name", "id", "dateCreated" })
    public @interface CustomAnnotation {}
```

现在，如果我们在实体上使用新注释：

```java
@CustomAnnotation
public class BeanWithCustomAnnotation {
    public int id;
    public String name;
    public Date dateCreated;
}
```

我们可以看到它如何将现有注释组合成一个简单的自定义注释，我们可以将其用作速记：

```java
@Test
public void whenSerializingUsingCustomAnnotation_thenCorrect()
  throws JsonProcessingException {
    BeanWithCustomAnnotation bean 
      = new BeanWithCustomAnnotation(1, "My bean", null);

    String result = new ObjectMapper().writeValueAsString(bean);

    assertThat(result, containsString("My bean"));
    assertThat(result, containsString("1"));
    assertThat(result, not(containsString("dateCreated")));
}
{
    "name":"My bean",
    "id":1
}
```

## 混入

我们使用MixIn注释忽略User类型的属性：

```java
public class Item {
    public int id;
    public String itemName;
    public User owner;
}
@JsonIgnoreType
public class MyMixInForIgnoreType {}
@Test
public void whenSerializingUsingMixInAnnotation_thenCorrect() 
  throws JsonProcessingException {
    Item item = new Item(1, "book", null);

    String result = new ObjectMapper().writeValueAsString(item);
    assertThat(result, containsString("owner"));

    ObjectMapper mapper = new ObjectMapper();
    mapper.addMixIn(User.class, MyMixInForIgnoreType.class);

    result = mapper.writeValueAsString(item);
    assertThat(result, not(containsString("owner")));
}
```

## 其他

### @JsonIdentityReference

@JsonIdentityReference用于自定义对对象的引用，这些对象将序列化为对象标识，而不是完整的POJO。它与@JsonIdentityInfo协作，强制在每次序列化中使用对象标识，这与除了第一次缺少@JsonIdentityReference之外的所有序列化不同。当处理对象之间的循环依赖关系时，这两个注释非常有用。

为了演示使用@JsonIdentityReference，我们将定义两个不同的bean类，不使用和使用该注释。

```java
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class BeanWithoutIdentityReference {
    private int id;
    private String name;

    // constructor, getters and setters
}
BeanWithoutIdentityReference bean 
  = new BeanWithoutIdentityReference(1, "Bean Without Identity Reference Annotation");
String jsonString = mapper.writeValueAsString(bean);
{
    "id": 1,
    "name": "Bean Without Identity Reference Annotation"
}
```

------

```java
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
@JsonIdentityReference(alwaysAsId = true)
public class BeanWithIdentityReference {
    private int id;
    private String name;
    
    // constructor, getters and setters
}
BeanWithIdentityReference bean 
  = new BeanWithIdentityReference(1, "Bean With Identity Reference Annotation");
String jsonString = mapper.writeValueAsString(bean);
assertEquals("1", jsonString);
```

### @JsonAppend

当对象序列化时，@JsonAppend注释用于将虚拟属性添加到对象中，而不是常规属性。当我们想将补充信息直接添加到JSON字符串中，而不是更改类定义时，这是必要的。例如，将bean的版本元数据插入相应的JSON文档可能比为其提供附加属性更方便。

假设我们有一个没有@JsonAppend的bean，如下所示：

```java
public class BeanWithoutAppend {
    private int id;
    private String name;

    // constructor, getters and setters
}
```

测试将确认，在缺少@JsonAppend注释的情况下，序列化输出不包含关于补充版本属性的信息，尽管我们试图将其添加到ObjectWriter对象：

```java
BeanWithoutAppend bean = new BeanWithoutAppend(2, "Bean Without Append Annotation");
ObjectWriter writer 
  = mapper.writerFor(BeanWithoutAppend.class).withAttribute("version", "1.0");
String jsonString = writer.writeValueAsString(bean);
{
    "id": 2,
    "name": "Bean Without Append Annotation"
}
```

现在，假设我们有一个用@JsonAppend注释的bean：

```java
@JsonAppend(attrs = { 
  @JsonAppend.Attr(value = "version") 
})
public class BeanWithAppend {
    private int id;
    private String name;

    // constructor, getters and setters
}
```

与前一个类似的测试将验证当应用@JsonAppend注释时，在序列化后是否包含补充属性：

```java
BeanWithAppend bean = new BeanWithAppend(2, "Bean With Append Annotation");
ObjectWriter writer 
  = mapper.writerFor(BeanWithAppend.class).withAttribute("version", "1.0");
String jsonString = writer.writeValueAsString(bean);
```

该序列化的输出显示已添加版本属性：

```json
{
    "id": 2,
    "name": "Bean With Append Annotation",
    "version": "1.0"
}
```

### @JsonNaming

@JsonNaming注释用于选择序列化中属性的命名策略，覆盖默认值。使用value元素，我们可以指定任何策略，包括自定义策略。

除了默认值LOWER_CAMEL_CASE之外，Jackson库还为我们提供了其他四种内置属性命名策略：

- KEBAB_CASE：名称元素用连字符`(-)`隔开，例如*kebab-case*.。
- LOWER_CASE：所有字母都是小写，没有分隔符，例如小写。
- SNAKE_CASE：所有字母均为小写，名称元素之间用下划线分隔，例如SNAKE_CASE。
- UPPER_CAMEL_CASE：所有名称元素，包括第一个，都以大写字母开头，后跟小写字母，并且没有分隔符，例如UpperCamelCase。

此示例将说明使用s*SNAKE_CASE* 名称序列化属性的方法，其中名为beanName的属性被序列化为bean_name。

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class NamingBean {
    private int id;
    private String beanName;

    // constructor, getters and setters
}
NamingBean bean = new NamingBean(3, "Naming Bean");
String jsonString = mapper.writeValueAsString(bean);        
assertThat(jsonString, containsString("bean_name"));
{
    "id": 3,
    "bean_name": "Naming Bean"
}
```

### @JsonPropertyDescription

Jackson库能够在名为JSON schema的单独模块的帮助下为Java类型创建JSON模式。当我们希望在序列化Java对象时指定预期输出，或者在反序列化之前验证JSON文档时，该模式非常有用。

@JsonPropertyDescription注释允许通过提供描述字段将人类可读的描述添加到创建的JSON模式中。

本节使用下面声明的bean来演示@JsonPropertyDescription的功能：

```java
public class PropertyDescriptionBean {
    private int id;
    @JsonPropertyDescription("This is a description of the name property")
    private String name;

    // getters and setters
}
```

通过添加描述字段生成JSON模式的方法如下所示：

```java
SchemaFactoryWrapper wrapper = new SchemaFactoryWrapper();
mapper.acceptJsonFormatVisitor(PropertyDescriptionBean.class, wrapper);
JsonSchema jsonSchema = wrapper.finalSchema();
String jsonString = mapper.writeValueAsString(jsonSchema);
assertThat(jsonString, containsString("This is a description of the name property"));
```

如我们所见，JSON模式的生成是成功的：

```json
{
    "type": "object",
    "id": "urn:jsonschema:com:baeldung:jackson:annotation:extra:PropertyDescriptionBean",
    "properties": 
    {
        "name": 
        {
            "type": "string",
            "description": "This is a description of the name property"
        },

        "id": 
        {
            "type": "integer"
        }
    }
}
```

### @JsonPOJOBuilder

@JsonPOJOBuider注释用于配置构建器类，以便在命名约定与默认约定不同时自定义JSON文档的反序列化以恢复POJO。

假设我们需要反序列化以下JSON字符串：

```json
{
    "id": 5,
    "name": "POJO Builder Bean"
}
```

该JSON源将用于创建POJOBuilderBean的实例：

```java
@JsonDeserialize(builder = BeanBuilder.class)
public class POJOBuilderBean {
    private int identity;
    private String beanName;

    // constructor, getters and setters
}
```

bean属性的名称与JSON字符串中字段的名称不同。这就是@JsonPOJOBuider前来救援的地方。
 

@JsonPOJOBuider注释附带两个属性：

- *buildMethodName:* 将JSON字段绑定到预期bean的属性后，用于实例化该bean的无参数方法的名称。默认名称为build。
- *withPrefix:* 用于自动检测JSON和bean属性之间匹配的名称前缀。默认前缀为with。

```java
@JsonPOJOBuilder(buildMethodName = "createBean", withPrefix = "construct")
public class BeanBuilder {
    private int idValue;
    private String nameValue;

    public BeanBuilder constructId(int id) {
        idValue = id;
        return this;
    }

    public BeanBuilder constructName(String name) {
        nameValue = name;
        return this;
    }

    public POJOBuilderBean createBean() {
        return new POJOBuilderBean(idValue, nameValue);
    }
}
```

在上面的代码中，我们配置了@JsonPOJOBuilder以使用名为createBean的构建方法和匹配属性的构造前缀。

@JsonPOJOBuilder对bean的应用程序描述和测试如下：

```java
String jsonString = "{\"id\":5,\"name\":\"POJO Builder Bean\"}";
POJOBuilderBean bean = mapper.readValue(jsonString, POJOBuilderBean.class);

assertEquals(5, bean.getIdentity());
assertEquals("POJO Builder Bean", bean.getBeanName());
```

### @JsonTypeId

@JsonTypeId注释用于指示在包含多态类型信息时，应将带注释的属性序列化为类型id，而不是常规属性。该多态元数据在反序列化期间用于重新创建与序列化前相同子类型的对象，而不是声明的超类型。

```java
public class TypeIdBean {
    private int id;
    @JsonTypeId
    private String name;

    // constructor, getters and setters
}
mapper.enableDefaultTyping(DefaultTyping.NON_FINAL);
TypeIdBean bean = new TypeIdBean(6, "Type Id Bean");
String jsonString = mapper.writeValueAsString(bean);
        
assertThat(jsonString, containsString("Type Id Bean"));
[
    "Type Id Bean",
    {
        "id": 6
    }
]
```

### @JsonTypeIdResolver

@JsonTypeIdResolver注释用于表示序列化和反序列化中的自定义类型标识处理程序。该处理程序负责Java类型和JSON文档中包含的类型id之间的转换。

假设在处理以下类层次结构时，我们希望在JSON字符串中嵌入类型信息。

```java
@JsonTypeInfo(
  use = JsonTypeInfo.Id.NAME, 
  include = JsonTypeInfo.As.PROPERTY, 
  property = "@type"
)
@JsonTypeIdResolver(BeanIdResolver.class)
public class AbstractBean {
    private int id;

    protected AbstractBean(int id) {
        this.id = id;
    }

    // no-arg constructor, getter and setter
}
public class FirstBean extends AbstractBean {
    String firstName;

    public FirstBean(int id, String name) {
        super(id);
        setFirstName(name);
    }

    // no-arg constructor, getter and setter
}
public class LastBean extends AbstractBean {
    String lastName;

    public LastBean(int id, String name) {
        super(id);
        setLastName(name);
    }

    // no-arg constructor, getter and setter
}
```

这些类的实例用于填充BeanContainer对象：

```java
public class BeanContainer {
    private List<AbstractBean> beans;

    // getter and setter
}
```

我们可以看到，AbstractBean类用@JsonTypeIdResolver进行了注释，这表明它使用自定义TypeIdResorver来决定如何在序列化中包含子类型信息，以及如何反过来使用该元数据。

```java
public class BeanIdResolver extends TypeIdResolverBase {
    
    private JavaType superType;

    @Override
    public void init(JavaType baseType) {
        superType = baseType;
    }

    @Override
    public Id getMechanism() {
        return Id.NAME;
    }

    @Override
    public String idFromValue(Object obj) {
        return idFromValueAndType(obj, obj.getClass());
    }

    @Override
    public String idFromValueAndType(Object obj, Class<?> subType) {
        String typeId = null;
        switch (subType.getSimpleName()) {
        case "FirstBean":
            typeId = "bean1";
            break;
        case "LastBean":
            typeId = "bean2";
        }
        return typeId;
    }

    @Override
    public JavaType typeFromId(DatabindContext context, String id) {
        Class<?> subType = null;
        switch (id) {
        case "bean1":
            subType = FirstBean.class;
            break;
        case "bean2":
            subType = LastBean.class;
        }
        return context.constructSpecializedType(superType, subType);
    }
}
```

最值得注意的两个方法是idFromValueAndType和typeFromId，前者告诉序列化POJO时如何包含类型信息，后者使用该元数据确定重新创建的对象的子类型。

为了确保序列化和反序列化都能正常工作，让我们编写一个测试来验证完整的进度。

首先，我们需要实例化一个bean容器和bean类，然后用bean实例填充该容器：

```java
FirstBean bean1 = new FirstBean(1, "Bean 1");
LastBean bean2 = new LastBean(2, "Bean 2");

List<AbstractBean> beans = new ArrayList<>();
beans.add(bean1);
beans.add(bean2);

BeanContainer serializedContainer = new BeanContainer();
serializedContainer.setBeans(beans);
```

接下来，BeanContainer对象被序列化，我们确认结果字符串包含类型信息：

```java
String jsonString = mapper.writeValueAsString(serializedContainer);
assertThat(jsonString, containsString("bean1"));
assertThat(jsonString, containsString("bean2"));
{
    "beans": 
    [
        {
            "@type": "bean1",
            "id": 1,
            "firstName": "Bean 1"
        },

        {
            "@type": "bean2",
            "id": 2,
            "lastName": "Bean 2"
        }
    ]
}
```

该JSON结构将用于重新创建与序列化之前相同子类型的对象。以下是反序列化的实现步骤：

```json
BeanContainer deserializedContainer = mapper.readValue(jsonString, BeanContainer.class);
List<AbstractBean> beanList = deserializedContainer.getBeans();
assertThat(beanList.get(0), instanceOf(FirstBean.class));
assertThat(beanList.get(1), instanceOf(LastBean.class));
```

# How To

## 编组时如何忽略属性？

- @JsonIgnoreProperties ： 在类上指定忽略的属性
- @JsonIgnore： 在字段上指定忽略的属性
- @JsonIgnoreType：忽略类型的所有属性
- @JsonFilter： 动态忽略属性
- @JsonInclude(Include.NON_NULL)：忽略特定值类型的属性

参考： https://www.baeldung.com/jackson-ignore-properties-on-serialization



## jackson处理Optional 字段

```java
public class Book {
   String title;
   Optional<String> subTitle;
   
   // getters and setters omitted
}
```

序列化该对象：

```java
Book book = new Book();
book.setTitle("Oliver Twist");
book.setSubTitle(Optional.of("The Parish Boy's Progress"));
String result = mapper.writeValueAsString(book);
{"title":"Oliver Twist","subTitle":{"present":true}}
```

我们将看到，Optional字段的输出不包含其值，而是包含一个嵌套的JSON对象，该对象具有一个名为present的字段.

在本例中，isPresent是Optional类上的公共getter。这意味着它将被序列化为true或false，具体取决于它是否为空。这是jackson的默认序列化行为。



现在，让我们反转上一个示例，这次尝试将对象反序列化为可选对象。现在我们将看到JsonMappingException：

```java
@Test(expected = JsonMappingException.class)
public void givenFieldWithValue_whenDeserializing_thenThrowException
    String bookJson = "{ \"title\": \"Oliver Twist\", \"subTitle\": \"foo\" }";
    Book result = mapper.readValue(bookJson, Book.class);
}
com.fasterxml.jackson.databind.JsonMappingException:
  Can not construct instance of java.util.Optional:
  no String-argument constructor/factory method to deserialize from String value ('The Parish Boy's Progress')
```

这种行为再次有意义。本质上，Jackson需要一个构造函数，它可以将*subtitle* 的值作为参数。我们的可选字段并非如此。



**如何解决呢？**

```xml
<dependency>
   <groupId>com.fasterxml.jackson.datatype</groupId>
   <artifactId>jackson-datatype-jdk8</artifactId>
   <version>2.13.3</version>
</dependency>
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new Jdk8Module());
```

## 反序列化集合类型？

### 转换成数组

```java
@Test
public void givenJsonArray_whenDeserializingAsArray_thenCorrect() 
  throws JsonParseException, JsonMappingException, IOException {
 
    ObjectMapper mapper = new ObjectMapper();
    List<MyDto> listOfDtos = Lists.newArrayList(
      new MyDto("a", 1, true), new MyDto("bc", 3, false));
    String jsonArray = mapper.writeValueAsString(listOfDtos);
 
    // [{"stringValue":"a","intValue":1,"booleanValue":true},
    // {"stringValue":"bc","intValue":3,"booleanValue":false}]

    MyDto[] asArray = mapper.readValue(jsonArray, MyDto[].class);
    assertThat(asArray[0], instanceOf(MyDto.class));
}
```

### 转换成集合

将同一个JSON数组读入Java集合有点困难——默认情况下，Jackson将无法获得完整的泛型类型信息，而是创建一个LinkedHashMap实例集合：

```java
@Test
public void givenJsonArray_whenDeserializingAsListWithNoTypeInfo_thenNotCorrect() 
  throws JsonParseException, JsonMappingException, IOException {
 
    ObjectMapper mapper = new ObjectMapper();

    List<MyDto> listOfDtos = Lists.newArrayList(
      new MyDto("a", 1, true), new MyDto("bc", 3, false));
    String jsonArray = mapper.writeValueAsString(listOfDtos);

    List<MyDto> asList = mapper.readValue(jsonArray, List.class);
    assertThat((Object) asList.get(0), instanceOf(LinkedHashMap.class));
}
```

有两种方法可以帮助Jackson理解正确的类型信息——我们可以为此使用库提供的TypeReference：

```java
@Test
public void givenJsonArray_whenDeserializingAsListWithTypeReferenceHelp_thenCorrect() 
  throws JsonParseException, JsonMappingException, IOException {
 
    ObjectMapper mapper = new ObjectMapper();

    List<MyDto> listOfDtos = Lists.newArrayList(
      new MyDto("a", 1, true), new MyDto("bc", 3, false));
    String jsonArray = mapper.writeValueAsString(listOfDtos);

    List<MyDto> asList = mapper.readValue(
      jsonArray, new TypeReference<List<MyDto>>() { });
    assertThat(asList.get(0), instanceOf(MyDto.class));
}
```

或者我们可以使用接受JavaType的重载readValue方法：

```java
@Test
public void givenJsonArray_whenDeserializingAsListWithJavaTypeHelp_thenCorrect() 
  throws JsonParseException, JsonMappingException, IOException {
    ObjectMapper mapper = new ObjectMapper();

    List<MyDto> listOfDtos = Lists.newArrayList(
      new MyDto("a", 1, true), new MyDto("bc", 3, false));
    String jsonArray = mapper.writeValueAsString(listOfDtos);

    CollectionType javaType = mapper.getTypeFactory()
      .constructCollectionType(List.class, MyDto.class);
    List<MyDto> asList = mapper.readValue(jsonArray, javaType);
 
    assertThat(asList.get(0), instanceOf(MyDto.class));
}
```

最后一个注意事项是，MyDto类需要有无参数的默认构造函数——如果没有，Jackson将无法实例化它：

```latex
com.fasterxml.jackson.databind.JsonMappingException: 
No suitable constructor found for type [simple type, class org.baeldung.jackson.ignore.MyDto]: 
can not instantiate from JSON object (need to add/enable type information?)
```

## 如何比较两个json对象是否相等？

使用 [JsonNode.equals](https://static.javadoc.io/com.fasterxml.jackson.core/jackson-databind/2.9.8/com/fasterxml/jackson/databind/JsonNode.html#equals-java.lang.Object-) 方法执行完全（深度）比较。

假设我们有一个JSON字符串定义为s1变量：

```json
{
  "employee":
  {
    "id": "1212",
    "fullName": "John Miles",
    "age": 34
  }
}
```

我们有另一个JSON字符串定义为s2变量：

```json
{   
    "employee":
    {
        "id": "1212",
        "age": 34,
        "fullName": "John Miles"
    }
}
```

让我们将输入JSON读取为JsonNode，并进行比较：

```java
assertEquals(mapper.readTree(s1), mapper.readTree(s2));
```

需要注意的是，即使输入JSON变量s1和s2中属性的顺序不同，equals（）方法也会忽略顺序并将它们视为相等。

## 如何对Map进行序列化和反序列化

序列化将Java对象转换为字节流，可以根据需要进行持久化或共享。Java Map是将键对象映射到值对象的集合，通常是最不直观的序列化对象。

### ***Map<String, String>*** **序列化**

对于一个简单的例子，让我们创建一个Map＜String，String＞并将其序列化为JSON：

```json
Map<String, String> map = new HashMap<>();
map.put("key", "value");

ObjectMapper mapper = new ObjectMapper();
String jsonResult = mapper.writerWithDefaultPrettyPrinter()
  .writeValueAsString(map);
```

ObjectMapper是Jackson的序列化映射器。它允许我们序列化我们的Map，并使用String中的toString方法将其写成一个打印精美的JSON字符串：

```json
{
  "key" : "value"
}
```

### ***Map<Object, String>*** **序列化**
