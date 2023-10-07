---
title: spring-mvc
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:56:09
---

# spring mvc官方文档
## DispatcherServlet
与其他许多Web框架一样，Spring MVC围绕前端控制器模式进行设计，在该模式下，中央Servlet DispatcherServlet提供了用于请求处理的共享算法，而实际工作是由可配置的委托组件执行的。 该模型非常灵活，并支持多种工作流程。

与任何Servlet一样，都需要使用Java配置或在web.xml中根据Servlet规范声明和映射DispatcherServlet。 反过来，DispatcherServlet使用Spring配置发现请求映射，视图解析，异常处理等所需的委托组件。

以下Java配置示例注册并初始化DispatcherServlet，该容器由Servlet容器自动检测到（请参阅Servlet Config）：

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // 加载 Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // 创建并注册DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
```
除了使用 ServletContext API，你也可以继承`AbstractAnnotationConfigDispatcherServletInitializer`，覆盖指定的方法来配置DispatcherServlet,参考Conetxt层级关系中的例子

### Context的层级关系
DispatcherServlet需要配置WebApplicationContext（纯ApplicationContext的扩展）为自身属性。 WebApplicationContext具有指向ServletContext和与其关联的Servlet的链接。它还绑定到ServletContext，以便应用程序可以在RequestContextUtils上使用静态方法来查找WebApplicationContext（如果需要访问它们）。

对于许多应用程序来说，拥有一个WebApplicationContext很简单并且足够。也可能具有上下文层次结构，其中一个根WebApplicationContext在多个DispatcherServlet（或其他Servlet）实例之间共享，每个实例都有其自己的子WebApplicationContext配置。有关上下文层次结构功能的更多信息，请参见ApplicationContext的其他功能。

根WebApplicationContext通常包含基础结构bean，例如需要在多个Servlet实例之间共享的数据存储库和业务服务。这些Bean是有效继承的，并且可以在Servlet特定的子WebApplicationContext中重写（即重新声明），该子WebApplicationContext通常包含给定Servlet本地的Bean。下图显示了这种关系：

![image](https://docs.spring.io/spring-framework/docs/current/reference/html/images/mvc-context-hierarchy.png)

下面的例子配置`WebApplicationContext`

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
```
当不需要层级上下文时，只需要getRootConfigClasses返回配置类，getServletConfigClasses返回null。

### 特殊的spring bean
DispatcherServlet委托给特殊的bean处理请求并响应。 所谓“特殊bean”，是指实现框架协定的Spring管理对象实例。 这些通常带有内置合同，但是您可以自定义它们的属性并扩展或替换它们。

下表列出了DispatcherServlet检测到的特殊bean：

| Bean 类型                                                    | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `HandlerMapping`                                             | 将请求映射到带拦截器列表的处理程序，以进行预处理和后期处理。 映射基于一些标准，具体细节因HandlerMapping实现而异。两个主要的HandlerMapping实现是RequestMappingHandlerMapping（支持@RequestMapping注释方法）和SimpleUrlHandlerMapping（维护显式注册） 处理程序的URI路径模式）。 |
| `HandlerAdapter`                                             | 帮助“ DispatcherServlet”调用请求的具体handler，而不管该处理程序的实际调用方式如何。 例如，调用带注释的控制器需要解析注释。 HandlerAdapter的主要目的是保护DispatcherServlet免受此类细节的影响。 |
| `HandlerExceptionResolver`                                   | 解决异常的策略，可能将它们映射Handler，HTML错误视图或其他目标。 |
| `ViewResolver`                                               | 解析从handler返回的基于逻辑的字符串视图名称，以实际的视图呈现给响应。 |
| `LocaleResolver`, [LocaleContextResolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-timezone) | 解决客户正在使用的“语言环境”以及可能的时区问题，以便能够提供国际化的视图。 |
| `ThemeResolver`                                              | 解决Web应用程序可以使用的主题，例如提供个性化的布局。        |
| `MultipartResolver`                                          | 借助一些多部分解析库来解析多部分请求的抽象（例如，浏览器表单文件上传）。 |
| `FlashMapManager`                                            | 存储和检索“输入”和“输出”“ FlashMap”，可用于将属性从一个请求传递到另一个请求，通常跨重定向。 |

### Web MVC配置
应用程序可以声明处理请求所需的特殊Bean类型中列出的基础结构Bean。 DispatcherServlet检查每个特殊bean的WebApplicationContext。 如果没有匹配的bean类型，它将使用DispatcherServlet.properties（参考附录）中列出的默认类型。

在大多数情况下，MVC Config是最佳起点。 它使用Java或XML声明所需的bean，并提供更高级别的配置回调API对其进行自定义。

### Servlet配置
在Servlet 3.0+环境中，您可以选择以编程方式配置Servlet容器，以替代web.xml文件的方式。 下面的示例注册一个DispatcherServlet：

```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```
WebApplicationInitializer是Spring MVC提供的接口，可确保检测到您的实现并将其自动用于初始化任何Servlet 3容器。 WebApplicationInitializer的抽象基类实现名为AbstractDispatcherServletInitializer，它通过覆盖指定Servlet映射和DispatcherServlet配置location 的方法，使注册DispatcherServlet更容易。

对于使用基于Java的Spring配置的应用程序，建议这样做，如以下示例所示：

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```
如果使用基于XML的Spring配置，则应直接从AbstractDispatcherServletInitializer进行扩展，如以下示例所示：

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```
AbstractDispatcherServletInitializer还提供了一种方便的方法来添加Filter实例，并将其自动映射到DispatcherServlet，如以下示例所示：

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```
每个过滤器都会根据其具体类型添加一个默认名称，并自动映射到DispatcherServlet。

AbstractDispatcherServletInitializer的isAsyncSupported受保护方法提供了一个位置，以在DispatcherServlet及其映射的所有过滤器上启用异步支持。 默认情况下，此标志设置为true。

最后，如果您需要进一步自定义DispatcherServlet本身，则可以覆盖createDispatcherServlet方法。

### DispatcherServlet的处理流程
`DispatcherServlet`处理请求的流程如下：

* 搜索WebApplicationContext并将其绑定在请求中，作为控制器和流程中其他元素可以使用的属性。 默认情况下，它绑定在DispatcherServlet.WEB\_APPLICATION\_CONTEXT\_ATTRIBUTE键下。
* 语言环境解析器绑定到请求，以使流程中的元素在处理请求（渲染视图，准备数据等）时要解析使用的语言环境。 如果不需要语言环境解析，则不需要语言环境解析器。
* 主题解析器绑定到请求，以使诸如视图之类的元素确定要使用的主题。 如果不使用主题，则可以将其忽略。
* 如果指定多部分文件解析器，则将检查请求中是否有多部分。 如果找到多部分，则将该请求包装在MultipartHttpServletRequest中，以供流程中的其他元素进一步处理。 有关多部分处理的更多信息，请参见[Multipart Resolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart)。
* 搜索适当的Handler。 如果找到Handler，则将运行与该Handler（预处理器，后处理器和Handler）关联的执行链，以准备要渲染的模型。 另外，对于带注释的控制器，可以渲染响应（在HandlerAdapter中），而不是返回视图。
* 如果返回模型，则渲染视图。 如果没有返回任何模型（可能是由于预处理器或后处理器拦截了该请求，可能出于安全原因），则不会渲染任何视图，因为该请求可能已经被满足。

WebApplicationContext中声明的HandlerExceptionResolver Bean用于解决在请求处理期间引发的异常。 这些异常解析器允许定制逻辑以解决异常。 有关更多详细信息，请参见异常模块。

Spring DispatcherServlet还支持Servlet API所指定的last-modification-date的返回。 确定特定请求的最后修改日期的过程很简单：DispatcherServlet查找适当的handler ，并测试找到的handler 是否实现了LastModified接口。 如果是这样，则将LastModified接口的long getLastModified（request）方法的值返回给客户端。

您可以通过将Servlet初始化参数（init-param元素）添加到web.xml文件中的Servlet声明中，来定制各个DispatcherServlet实例。 下表列出了受支持的参数：

| Parameter                        | Explanation                                                  |
| -------------------------------- | ------------------------------------------------------------ |
| `contextClass`                   | 实现“ ConfigurableWebApplicationContext”的类，将由该Servlet实例化并在本地配置。 默认情况下，使用XmlWebApplicationContext。 |
| `contextConfigLocation`          | 传递给上下文实例的字符串（由`contextClass`指定），指示可以在哪里找到上下文。 该字符串可能包含多个字符串（使用逗号作为分隔符）以支持多个上下文。 对于具有两次定义的bean的多个上下文位置，以最新位置为准。 |
| `namespace`                      | WebApplicationContext的命名空间。 默认为`[servlet-name] -servlet`。 |
| `throwExceptionIfNoHandlerFound` | 在找不到请求的处理程序时是否引发`NoHandlerFoundException`。 然后可以使用HandlerExceptionResolver捕获异常（例如通过使用@ExceptionHandler控制器方法）并以其他方式进行处理。默认情况下，将其设置为false，在这种情况下DispatcherServlet会设置 响应状态为404（NOT\_FOUND），而不会引发异常。请注意，如果[默认servlet处理](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc -default-servlet-handler)，未解决的请求始终转发到默认servlet，并且永远不会引发404。 |

### 拦截器
所有HandlerMapping实现都支持处理拦截器，当您要将特定功能应用于某些请求时（例如，检查主体），该拦截器很有用。 HandlerInterceptor提供了三个方法来做拦截处理逻辑，提供了足够的灵活性来执行各种预处理和后处理：

* preHandle：在handler之前执行
* postHandle：在handler之后执行
* afterCompletion：完成请求处理后（即渲染视图之后）的回调。 无论handler执行是否有异常，都会被调用，从而允许适当的资源清理。

preHandle返回bool,你可以通过这个值来决定执行链是否继续。当返回false时，DispatcherServlet假定拦截器本身已经处理了请求（例如，渲染了适当的视图），并且不会继续执行其他拦截器和执行链中的实际handler。

请注意，对于@ResponseBody和ResponseEntity方法中，postHandle的用处不大，在postHandle执行之前，HandlerAdapter已经编写和提交响应。 这意味着对响应进行任何更改为时已晚，例如添加额外的标头。 对于这种情况，您可以实现ResponseBodyAdvice并将其声明为Controller Advice Bean或直接在RequestMappingHandlerAdapter上对其进行配置。

### 异常
如果异常在请求映射期间发生或从handler（例如@Controller）抛出，则DispatcherServlet委托给HandlerExceptionResolver Bean链来解决该异常并提供替代处理，通常是错误响应。

下表列出了可用的HandlerExceptionResolver实现：

| `HandlerExceptionResolver`          | Description                                                  |
| ----------------------------------- | ------------------------------------------------------------ |
| `SimpleMappingExceptionResolver`    | 异常类名称和错误视图名称之间的映射。 对于在浏览器应用程序中呈现错误页面很有用。 |
| `DefaultHandlerExceptionResolver`   | 解决Spring MVC引发的异常，并将其映射到HTTP状态代码。 另请参见替代的`ResponseEntityExceptionHandler`和[REST API异常](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-rest-exceptions)。 |
| `ResponseStatusExceptionResolver`   | 使用@ResponseStatus批注解决异常，并根据批注中的值将其映射到HTTP状态代码。 |
| `ExceptionHandlerExceptionResolver` | 通过调用@Controller或@ControllerAdvice类中的@ExceptionHandler方法来解决异常。 参见[@ExceptionHandler方法](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-exceptionhandler)。 |

#### 解析器链
您可以通过在Spring配置中声明多个HandlerExceptionResolver bean并根据需要设置其order属性来形成异常解析器链。 order属性越高，异常解析器的定位就越晚。

HandlerExceptionResolver的约定可以返回：

* `ModelAndView`指向错误页面
* 如果在resolver中处理了异常，则为空的ModelAndView。
* 如果该异常仍未解决，则为null，以供后续解析器尝试；如果该异常仍在末尾，则允许将其冒泡到Servlet容器。

MVC Config使用支持@ResponseStatus注释的异常解析器和对@ExceptionHandler方法的支持声明内置解析器作为默认的Spring MVC异常配置。 您可以自定义该列表或替换它。

#### 容器错误页
如果任何HandlerExceptionResolver都无法解决异常，因此该异常可以传播，或者如果响应状态设置为错误状态（即4xx，5xx），则Servlet容器可以在HTML中呈现默认错误页面。 要自定义容器的默认错误页面，可以在web.xml中声明错误页面映射。 以下示例显示了如何执行此操作：

```xml
<error-page>
    <location>/error</location>
</error-page>
```
给定前面的示例，当异常冒出或响应具有错误状态时，Servlet容器在容器内向配置的URL（例如/ error）进行ERROR调度。 然后由DispatcherServlet处理它，可能将其映射到@Controller，可以实现该错误以使用模型返回错误视图名称或呈现JSON响应，如以下示例所示：

```java
@RestController
public class ErrorController {

    @RequestMapping(path = "/error")
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));
        return map;
    }
}
```
### 视图解析
Spring MVC定义了ViewResolver和View接口，这些接口使您可以在浏览器中呈现模型，而无需将您与特定的视图技术联系在一起。 ViewResolver提供了视图名称和实际视图之间的映射。 视图在移交给特定的视图技术之前着手准备数据。

下表提供了有关ViewResolver层次结构的更多详细信息：

| ViewResolver                     | Description                                                  |
| -------------------------------- | ------------------------------------------------------------ |
| `AbstractCachingViewResolver`    | AbstractCachingViewResolver的子类缓存它们解析的视图实例。 缓存可以提高某些视图技术的性能。 您可以通过将cache属性设置为false来关闭缓存。 此外，如果必须在运行时刷新某个视图（例如，当修改FreeMarker模板时），则可以使用removeFromCache（String viewName，Locale loc）方法。 |
| `UrlBasedViewResolver`           | ViewResolver接口的简单实现会影响逻辑视图名称到URL的直接解析，而无需显式映射定义。 如果您的逻辑名称以直接的方式与视图资源的名称匹配，而不需要任意映射，则这是适当的。 |
| `InternalResourceViewResolver`   | UrlBasedViewResolver的方便子类，它支持InternalResourceView（实际上是Servlet和JSP）以及诸如JstlView和TilesView之类的子类。 您可以使用setViewClass（..）为该解析器生成的所有视图指定视图类。 有关详细信息，请参见UrlBasedViewResolver Javadoc。 |
| `FreeMarkerViewResolver`         | UrlBasedViewResolver的便捷子类，支持FreeMarkerView及其自定义子类。 |
| `ContentNegotiatingViewResolver` | ViewResolver接口的实现，该接口根据请求文件名或Accept标头解析视图。 请参阅内容协商。 |
| `BeanNameViewResolver`           | ViewResolver接口的实现，该接口将视图名称解释为当前应用程序上下文中的Bean名称。 这是一个非常灵活的变体，它允许根据不同的视图名称来混合和匹配不同的视图类型。 每个这样的View可以定义为一个bean，例如 在XML或配置类中。 |

#### 处理
您可以通过声明多个解析器bean以及必要时通过设置order属性以指定排序来链接视图解析器。 请记住，order属性越高，视图解析器在链中的定位就越晚。

ViewResolver的协议指定它可以返回null，以指示找不到该视图。 但是，对于JSP和InternalResourceViewResolver，确定JSP是否存在的唯一方法是通过RequestDispatcher进行调度。 因此，您必须始终将InternalResourceViewResolver配置为在视图解析器的总体顺序中排在最后。

配置视图解析器就像将ViewResolver bean添加到Spring配置中一样简单。 MVC Config为View解析器和添加无逻辑的View Controller提供了专用的配置API，这对于无需控制器逻辑的HTML模板呈现非常有用。

#### 重定向
视图名称中的特殊redirect：前缀使您可以执行重定向。 UrlBasedViewResolver（及其子类）将其识别为需要重定向的指令。 视图名称的其余部分是重定向URL。

最终效果与控制器返回RedirectView的效果相同，但是现在控制器本身可以根据逻辑视图名称进行操作。 逻辑视图名称（如redirect：/ myapp / some / resource）相对于当前Servlet上下文进行重定向，而名称如redirect：https：//myhost.com/some/arbitrary/path则重定向至绝对URL。

请注意，如果使用@ResponseStatus注释控制器方法，则注释值优先于RedirectView设置的响应状态。

#### 转发
您还可以对视图名称使用特殊的forward：前缀，这些视图名称最终由UrlBasedViewResolver和子类解析。 这将创建一个InternalResourceView，它执行RequestDispatcher.forward（）。 因此，此前缀在InternalResourceViewResolver和InternalResourceView（对于JSP）中没有用，但是如果您使用另一种视图技术但仍希望强制转发由Servlet / JSP引擎处理的资源，则该前缀很有用。 请注意，您也可以改为链接多个视图解析器。

#### 内容协商
ContentNegotiatingViewResolver不会解析视图本身，而是委派给其他视图解析器，并选择类似于客户端请求的表示形式的视图。可以从Accept标头或查询参数（例如，“ / path？format = pdf”）中确定表示形式。

ContentNegotiatingViewResolver通过将请求媒体类型与与其每个ViewResolver关联的View支持的媒体类型（也称为Content-Type）进行比较，从而选择合适的View处理该请求。列表中具有兼容Content-Type的第一个View将表示形式返回给客户端。如果ViewResolver链无法提供兼容的视图，请查阅通过DefaultViews属性指定的视图列表。后一个选项适用于可以呈现当前资源的适当表示形式的单例视图，而与逻辑视图名称无关。 Accept标头可以包含通配符（例如text / \*），在这种情况下，其Content-Type为text / xml的View是兼容的匹配。

### 语言环境
正如Spring Web MVC框架所做的那样，Spring体系结构的大多数部分都支持国际化。通过DispatcherServlet，您可以使用客户端的语言环境自动解析消息。这是通过LocaleResolver对象完成的。

收到请求时，DispatcherServlet会查找语言环境解析器，如果找到了它，则尝试使用它来设置语言环境。通过使用RequestContext.getLocale（）方法，您始终可以检索由语言环境解析器解析的语言环境。

除了自动的语言环境解析之外，您还可以在处理程序映射上附加一个拦截器（有关处理程序映射拦截器的更多信息，请参见拦截），以在特定情况下（例如，基于请求中的参数）更改语言环境。

语言环境解析器和拦截器在org.springframework.web.servlet.i18n包中定义，并以常规方式在应用程序上下文中进行配置。 Spring包含以下选择的语言环境解析器。

* [Time Zone](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-timezone)
* [Header Resolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-localeresolver-acceptheader)
* [Cookie Resolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-localeresolver-cookie)
* [Session Resolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-localeresolver-session)
* [Locale Interceptor](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-localeresolver-interceptor)

#### Time Zone
除了获取客户的语言环境外，了解其时区通常也很有用。 LocaleContextResolver接口提供了LocaleResolver的扩展，该扩展使解析程序可以提供更丰富的LocaleContext，其中可能包含时区信息。

如果可用，则可以使用RequestContext.getTimeZone（）方法获取用户的TimeZone。 通过Spring的ConversionService注册的任何日期/时间转换器和格式器对象都会自动使用时区信息。

#### Header Resolver
该语言环境解析器检查客户端（例如，Web浏览器）发送的请求中的accept-language标头。 通常，此标头字段包含客户端操作系统的语言环境。 请注意，此解析器不支持时区信息。

##### Cookie Resolver
此语言环境解析器检查客户端上可能存在的Cookie，以查看是否指定了语言环境或TimeZone。 如果是这样，它将使用指定的详细信息。 通过使用此语言环境解析器的属性，可以指定Cookie的名称以及最长期限。 以下示例定义了CookieLocaleResolver：

```xml
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">

    <property name="cookieName" value="clientlanguage"/>

    <!-- in seconds. If set to -1, the cookie is not persisted (deleted when browser shuts down) -->
    <property name="cookieMaxAge" value="100000"/>

</bean>
```
CookieLocaleResolver的几个属性

| Property       | Default                   | Description                                                  |
| -------------- | ------------------------- | ------------------------------------------------------------ |
| `cookieName`   | classname + LOCALE        | 名称                                                         |
| `cookieMaxAge` | Servlet container default | Cookie在客户端上保留的最长时间。 如果指定了-1，则cookie将不会保留。 它仅在客户端关闭浏览器之前可用。 |
| `cookiePath`   | /                         | 将Cookie的可见性限制在您网站的特定部分。 当指定`cookiePath`时，该cookie仅对该路径及其下方的路径可见。 |

#### Session Resolver
通过SessionLocaleResolver，您可以从可能与用户请求关联的会话中检索Locale和TimeZone。 与CookieLocaleResolver相比，此策略将本地选择的语言环境设置存储在Servlet容器的HttpSession中。 这些设置对于每个会话都是临时的，因此在每个会话结束时会丢失。

请注意，与外部会话管理机制（例如Spring Session项目）没有直接关系。 该SessionLocaleResolver针对当前HttpServletRequest评估并修改相应的HttpSession属性。

#### Locale Interceptor
您可以通过将LocaleChangeInterceptor添加到HandlerMapping定义之一来启用语言环境更改。 它检测请求中的参数并相应地更改语言环境，在调度程序的应用程序上下文中在LocaleResolver上调用setLocale方法。 下一个示例显示，对所有包含名为siteLanguage的参数的\* .view资源的调用现在都会更改语言环境。 因此，例如，对URL的请求https://www.sf.net/home.view?siteLanguage=nl会将站点语言更改为荷兰语。 以下示例显示如何拦截语言环境：

```java
<bean id="localeChangeInterceptor"
        class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
    <property name="paramName" value="siteLanguage"/>
</bean>

<bean id="localeResolver"
        class="org.springframework.web.servlet.i18n.CookieLocaleResolver"/>

<bean id="urlMapping"
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="localeChangeInterceptor"/>
        </list>
    </property>
    <property name="mappings">
        <value>/**/*.view=someController</value>
    </property>
</bean>
```
### 主题
您可以应用Spring Web MVC框架主题来设置应用程序的整体外观，从而增强用户体验。 主题是静态资源（通常是样式表和图像）的集合，这些资源会影响应用程序的视觉样式。

#### 定义主题
要在Web应用程序中使用主题，您必须设置org.springframework.ui.context.ThemeSource接口的实现。 WebApplicationContext接口扩展了ThemeSource，但将其职责委托给专用的实现。 默认情况下，委托是org.springframework.ui.context.support.ResourceBundleThemeSource实现，该实现从类路径的根加载属性文件。 要使用自定义ThemeSource实现或配置ResourceBundleThemeSource的基本名称前缀，可以在应用程序上下文中注册使名称themeSource的bean。 Web应用程序上下文会自动检测到具有该名称的bean并使用它。

当您使用ResourceBundleThemeSource时，将在一个简单的属性文件中定义一个主题。 属性文件列出了组成主题的资源，如以下示例所示：

```Plain Text
styleSheet=/themes/cool/style.css
background=/themes/cool/img/coolBg.jpg
```
属性的键是引用视图代码中主题元素的名称。 对于JSP，通常使用spring：theme定制标记来执行此操作，该标记与spring：message标记非常相似。 以下JSP片段使用上一示例中定义的主题来自定义外观：

```html
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
<html>
    <head>
        <link rel="stylesheet" href="<spring:theme code='styleSheet'/>" type="text/css"/>
    </head>
    <body style="background=<spring:theme code='background'/>">
        ...
    </body>
</html>
```
默认情况下，ResourceBundleThemeSource使用一个空的基本名称前缀。 结果，从类路径的根加载属性文件。 因此，您可以将cool.properties主题定义放在类路径的根目录中（例如，在/ WEB-INF / classes中）。 ResourceBundleThemeSource使用标准的Java资源束加载机制，允许主题的完全国际化。 例如，我们可以有一个/WEB-INF/classes/cool\_nl.properties，它引用带有荷兰文字的特殊背景图像。

#### 解析主题
在定义主题之后，如上一节所述，您可以确定要使用的主题。 DispatcherServlet查找一个名为themeResolver的bean，以找出要使用的ThemeResolver实现。 主题解析器的工作方式与LocaleResolver大致相同。 它可以检测用于特定请求的主题，还可以更改请求的主题。 下表描述了Spring提供的主题解析器：

| Class                  | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| `FixedThemeResolver`   | 选择一个固定的主题，通过使用defaultThemeName属性设置。       |
| `SessionThemeResolver` | 该主题在用户的HTTP会话中维护。 每个会话只需设置一次，但在会话之间不会保留。 |
| `CookieThemeResolver`  | 所选主题存储在客户端的cookie中。                             |

Spring还提供了ThemeChangeInterceptor，可以使用简单的请求参数对每个请求进行主题更改。

### Multipart Resolver
org.springframework.web.multipart包中的MultipartResolver是一种用于解析包括文件上传在内的多部分请求的策略。 有一种基于Commons FileUpload的实现，另一种基于Servlet 3.0多部分请求解析。

要启用多部分处理，您需要在DispatcherServlet Spring配置中声明一个名为multipartResolver的MultipartResolver bean。 DispatcherServlet检测到它并将其应用于传入的请求。 当收到内容类型为multipart / form-data的POST时，解析程序将解析内容并将当前的HttpServletRequest包装为MultipartHttpServletRequest，以提供对已解析部分的访问权，此外还可以将其公开为请求参数。

#### Apache Commons `FileUpload`
要使用Apache Commons FileUpload，可以配置名称为multipartResolver的CommonsMultipartResolver类型的Bean。 您还需要commons-fileupload作为对类路径的依赖。

#### Servlet 3.0
需要通过Servlet容器配置启用Servlet 3.0多部分解析。 为此：

在Java中，在Servlet注册上设置MultipartConfigElement。

在web.xml中，将“ ”部分添加到Servlet声明中。

以下示例显示了如何在Servlet注册上设置MultipartConfigElement：

```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ...

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {

        // Optionally also set maxFileSize, maxRequestSize, fileSizeThreshold
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

}
```
Servlet 3.0配置到位后，您可以添加名称为multipartResolver的StandardServletMultipartResolver类型的Bean。

### 日志
Spring MVC中的DEBUG级别的日志被设计为紧凑，最少且人性化的。 它侧重于一遍又一遍有用的高价值信息，而其他信息仅在调试特定问题时才有用。

TRACE级别的日志记录通常遵循与DEBUG相同的原则，但可用于调试任何问题。 此外，某些日志消息在TRACE和DEBUG上可能显示不同级别的详细信息。

良好的日志记录来自使用日志的经验。 如果发现任何不符合既定目标的东西，请告诉我们。

#### 敏感数据
调试和跟踪日志记录可能会记录敏感信息。 这就是默认情况下屏蔽请求参数和标头，并且必须通过DispatcherServlet上的enableLoggingRequestDetails属性显式启用它们的原因。打印日志效果参考附录

以下示例显示了如何通过使用Java配置来执行此操作：

```java
public class MyInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return ... ;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return ... ;
    }

    @Override
    protected String[] getServletMappings() {
        return ... ;
    }

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
        registration.setInitParameter("enableLoggingRequestDetails", "true");
    }

}
```
## 过滤器
### Form Data
浏览器只能通过HTTP GET或HTTP POST提交表单数据，但非浏览器客户端也可以使用HTTP PUT，PATCH和DELETE。 Servlet API的ServletRequest.getParameter 之类的方法来仅支持HTTP POST的表单字段访问。

spring-web模块提供FormContentFilter来拦截内容类型为application / x-www-form-urlencoded的HTTP PUT，PATCH和DELETE请求，从请求主体中读取表单数据，并包装ServletRequest以使表单数据可通过ServletRequest.getParameter 方法族获得。

### Forwarded Headers
当请求通过代理（例如负载平衡器）进行处理时，主机，端口和协议可能会更改，这会映射获取客户端的实际IP和端口等信息。

RFC 7239定义了用来提供有关原始请求的信息的HTTP转发头。还有其他非标准标头，包括X-Forwarded-Host，X-Forwarded-Port，X-Forwarded-Proto，X-Forwarded-Ssl和X-Forwarded-Prefix。

ForwardedHeaderFilter是一个Servlet过滤器，用于修改请求，以便

a）基于Forwarded标头更改主机，端口和协议

b）删除这些Forwarded标头以消除影响

该过滤器依赖于包装请求，因此必须在其他过滤器（例如RequestContextFilter）之前，其他过滤器应与修改后的请求而不是原始请求一起使用。

对于转发的标头，存在安全方面的考虑，因为应用程序无法知道标头是由代理添加的，还是由恶意客户端添加的。这就是为什么应配置信任边界处的代理以删除来自外部的不受信任的转发标头的原因。您还可以使用removeOnly = true配置ForwardedHeaderFilter，在这种情况下，它将删除但不使用标头。

为了支持异步请求和错误调度，此过滤器应与DispatcherType.ASYNC以及DispatcherType.ERROR映射。如果使用Spring Framework的AbstractAnnotationConfigDispatcherServletInitializer（请参阅Servlet Config），则会为所有调度类型自动注册所有过滤器。但是，如果通过web.xml或在Spring Boot中通过FilterRegistrationBean注册过滤器，请确保除了DispatcherType.REQUEST之外，还包括DispatcherType.ASYNC和DispatcherType.ERROR。

### Shallow ETag
ShallowEtagHeaderFilter过滤器通过缓存写入响应的内容并从中计算MD5哈希值来创建“shallow” ETag。客户端下一次发送时，它会执行相同的操作，但是还会将计算值与If-None-Match请求标头进行比较，如果两者相等，则返回304（NOT\_MODIFIED）。

此策略可节省网络带宽，但不能节省CPU，因为必须为每个请求计算完整响应。如前所述，控制器级别的其他策略可以避免计算。请参阅HTTP缓存。

该过滤器具有writeWeakETag参数，该参数将过滤器配置为写入弱ETag，类似于以下内容：W /“ 02a2d595e6ed9a0b24f027f2b63b134d6”（在RFC 7232第2.3节中定义）。

为了支持异步请求，此过滤器必须与DispatcherType.ASYNC映射，以便过滤器可以延迟并成功生成ETag到最后一个异步调度的末尾。如果使用Spring Framework的AbstractAnnotationConfigDispatcherServletInitializer（请参阅Servlet Config），则会为所有调度类型自动注册所有过滤器。但是，如果通过web.xml或在Spring Boot中通过FilterRegistrationBean注册过滤器，请确保包括DispatcherType.ASYNC。

### CORS
Spring MVC通过控制器上的注释为CORS配置提供了细粒度的支持。 但是，当与Spring Security一起使用时，我们建议您依赖内置的CorsFilter，该CorsFilter必须在Spring Security的过滤器链之前。

有关更多详细信息，请参见有关CORS和CORS过滤器的部分。

## 注解式控制器
Spring MVC提供了一个基于注释的编程模型，其中@Controller和@RestController组件使用注释来表达请求映射，请求输入，异常处理等。 带注释的控制器具有灵活的方法签名，无需扩展基类或实现特定的接口。 以下示例显示了由注释定义的控制器：

```java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
```
在前面的示例中，该方法接受Model并以String的形式返回视图名称，但是还存在许多其他选项，本章稍后将对其进行说明。

### 声明
您可以使用Servlet的WebApplicationContext中的标准Spring bean定义来定义控制器bean。 @Controller构造型允许自动检测，与Spring对在类路径中检测@Component类并为其自动注册Bean定义的常规支持保持一致。 它还充当带注释的类的原型，来表明其作为Web组件的作用。 

要启用对此类@Controller bean的自动检测，可以将组件扫描添加到Java配置中，如以下示例所示：

```java
@Configuration
@ComponentScan("org.example.web")
public class WebConfig {

    // ...
}
```
@RestController是一个组合的批注，其本身使用@Controller和@ResponseBody进行了元注释，以指示其每个方法都继承类型级别@ResponseBody批注的控制器，因此，将其直接写入响应主体（相对于视图解析器）并渲染HTML模板。

#### AOP代理
在某些情况下，您可能需要在运行时用AOP代理装饰控制器。 一个示例是，如果您选择直接在控制器上使用@Transactional批注。 在这种情况下，特别是对于控制器，我们建议使用基于类的代理。 这通常是控制器的默认选择。 但是，如果控制器必须实现不是Spring Context回调的接口（例如InitializingBean，\* Aware等），则可能需要显式配置基于类的代理。 例如，使用<tx：annotation-driven />，您可以更改为<tx:annotation-driven proxy-target-class="true"/>；使用@EnableTransactionManagement，您可以更改为@EnableTransactionManagement（proxyTargetClass = true）。

### 请求映射
您可以使用@RequestMapping批注将请求映射到控制器方法。 它具有多个属性，可以通过URL，HTTP方法，请求参数，标头和媒体类型进行匹配。 您可以在类级别使用它来表示共享的映射，也可以在方法级别使用它来缩小到特定的端点映射。

@RequestMapping还有HTTP方法特定的快捷方式：

* `@GetMapping`
* `@PostMapping`
* `@PutMapping`
* `@DeleteMapping`
* `@PatchMapping`

快捷方式是提供的“自定义注释”，因为，大多数控制器方法应该映射到特定的HTTP方法，而不是使用@RequestMapping，后者默认情况下与所有HTTP方法匹配。 同时，在类级别仍需要@RequestMapping来表示共享映射。

以下示例具有类型和方法级别的映射：

```java
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
```
#### URI匹配模式
可以使用URL模式映射@RequestMapping方法。 有两种选择：

* PathPattern-与URL路径匹配的预解析模式，该URL路径也预解析为PathContainer。 该解决方案专为Web使用而设计，可有效处理编码和路径参数，并有效匹配。
* AntPathMatcher-匹配字符串模式和字符串路径。 这是在Spring配置中还用于选择类路径，文件系统和其他位置上的资源的原始解决方案。 它效率较低，并且字符串路径输入对于有效处理URL的编码和其他问题是一个挑战。

PathPattern是Web应用程序的推荐解决方案，它是Spring WebFlux中的唯一选择。 在5.3版之前，AntPathMatcher是Spring MVC中的唯一选择，并且继续是默认设置。 但是，可以在MVC配置中启用PathPattern。

PathPattern支持与AntPathMatcher相同的模式语法。 此外，它还支持捕获模式，例如 {\* spring}，用于匹配路径末尾的0个或更多路径段。 PathPattern还限制了使用\*\*来匹配多个路径段，以便仅在模式末尾才允许使用。 当为给定请求选择最佳匹配模式时，这消除了很多歧义。 有关完整模式的语法，请参阅[PathPattern (Spring Framework 5.3.1 API)](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/web/util/pattern/PathPattern.html)和[AntPathMatcher (Spring Framework 5.3.1 API)](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/util/AntPathMatcher.html)。

一些例子：

* "/resources/ima?e.png"：匹配路径段中的一个字符
* "/resources/\*.png"：匹配路径段中的零个或多个字符
* "/resources/\*\*"：匹配多个路径段
* "/projects/{project}/versions"：匹配路径段并将其捕获为变量
* "/projects/{project:\[a-z\]+}/versions"：用正则表达式匹配并捕获变量

捕获的URI变量可以使用@PathVariable访问。 例如：

```java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
```
您可以在类和方法级别声明URI变量，如以下示例所示：

```java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
```
URI变量将自动转换为适当的类型，或者引发TypeMismatchException。 默认情况下，支持简单类型（int，long，Date等），您可以注册对任何其他数据类型的支持。 请参阅类型转换和DataBinder。

您可以显式地命名URI变量（例如，@PathVariable（“ customId”）），但是如果名称相同并且您的代码是使用调试信息或Java 8上的-parameters编译器标志编译的，则可以省略该详细信息。 。

语法{varName：regex}声明带有正则表达式的URI变量，语法为{varName：regex}。 例如，给定URL“ /spring-web-3.0.5 .jar”，以下方法将提取名称，版本和文件扩展名：

```java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String name, @PathVariable String version, @PathVariable String ext) {
    // ...
}
```
URI路径模式还可以嵌入\$ {…}占位符，这些占位符在启动时通过针对本地，系统，环境和其他属性源使用PropertyPlaceHolderConfigurer进行解析。 例如，您可以使用它来基于一些外部配置参数化基本URL。

#### 模式比较
当多个模式与URL匹配时，必须选择最佳匹配。 根据是否启用了已解析的“ PathPattern”，使用以下一种方法来完成此操作：

* `PathPattern.SPECIFICITY_COMPARATOR`
* [AntPathMatcher](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/util/AntPathMatcher.html#getPatternComparator-java.lang.String-)

两者都有助于对模式进行排序。 如果模式的URI变量（计数为1），单通配符（计数为1）和双通配符（计数为2）的数量较少，则模式的含义不太明确。 给定相等的分数，则选择更长的模式。 给定相同的分数和长度，将选择URI变量多于通配符的模式。

默认映射模式（/ \*\*）从评分中排除，并且始终排在最后。 同样，前缀模式（例如/ public / \*\*）被认为比其他没有双通配符的模式更不具体。

有关完整的详细信息，请单击上面的链接到模式比较器。

#### 后缀匹配
从5.3开始，默认情况下，Spring MVC不再执行`.*`后缀模式匹配，其中映射到/ person的控制器也隐式映射到/person.\*。 因此，路径扩展不再用于解释响应所请求的内容类型，例如/person.pdf、/person.xml等。

当浏览器用于发送难以一致解释的Accept标头时，以这种方式使用文件扩展名是必要的。 目前，这已不再是必须的，使用Accept标头应该是首选。

随着时间的流逝，文件扩展名的使用已经以各种方式证明是有问题的。 当使用URI变量，路径参数和URI编码进行覆盖时，可能会导致歧义。 关于基于URL的授权和安全性的推理（请参阅下一部分以了解更多详细信息）也变得更加困难。

要完全禁用5.3之前版本中的路径扩展，请设置以下内容：

* `useSuffixPatternMatching(false)`, see [PathMatchConfigurer](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-path-matching)
* `favorPathExtension(false)`, see [ContentNegotiationConfigurer](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-content-negotiation)

除了通过“ Accept”标头之外，还有一种请求内容类型的方法仍然很有用，例如 在浏览器中输入URL时。 路径扩展的一种安全替代方法是使用查询参数策略。 如果必须使用文件扩展名，请考虑通过ContentNegotiationConfigurer的mediaTypes属性将它们限制为显式注册的扩展名列表。

#### 后缀匹配和RDF
反射文件下载（RFD）攻击与XSS相似，因为它依赖反映在响应中的请求输入（例如，查询参数和URI变量）。 但是，RFD攻击不是将JavaScript插入HTML，而是依靠浏览器切换来执行下载，并在以后双击时将响应视为可执行脚本。

在Spring MVC中，@ ResponseBody和ResponseEntity方法存在风险，因为它们可以呈现不同的内容类型，客户端可以通过URL路径扩展请求这些内容类型。 禁用后缀模式匹配并使用路径扩展进行内容协商可以降低风险，但不足以防止RFD攻击。

为了防止RFD攻击，Spring MVC在呈现响应主体之前添加了Content-Disposition：inline; filename = f.txt标头，以建议提供固定且安全的下载文件。 仅当URL路径包含既不被认为安全也不被明确注册用于内容协商的文件扩展名时，才执行此操作。 但是，当直接在浏览器中键入URL时，它可能会产生副作用。

默认情况下，许多常见路径扩展都被视为安全。 具有自定义HttpMessageConverter实现的应用程序可以显式注册文件扩展名以进行内容协商，以避免为这些扩展名添加Content-Disposition标头。 请参阅内容类型。

有关RFD的其他建议，请参见[CVE-2015-5211 RFD Attack in Spring Framework | Security | VMware Tanzu](https://pivotal.io/security/cve-2015-5211)

#### Consumable Media Types
您可以根据请求的Content-Type缩小请求映射，如以下示例所示：

```java
@PostMapping(path = "/pets", consumes = "application/json") 
public void addPet(@RequestBody Pet pet) {
    // ...
}
```
消耗属性还支持否定表达式-例如，!text / plain表示除text / plain之外的任何内容类型。

您可以在类级别上声明一个共享的消耗属性。 但是，与大多数其他请求映射属性不同，在类级别已有，方法级别consumes 属性重写而不是扩展类级别声明。

#### Producible Media Types
您可以根据Accept 请求标头和控制器方法生成的内容类型列表来缩小请求映射，如以下示例所示：

```java
@GetMapping(path = "/pets/{petId}", produces = "application/json") 
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
```
媒体类型可以指定字符集。 支持否定的表达式-例如，！text / plain表示除“ text / plain”以外的任何内容类型。

您可以在类级别声明共享的Produces属性。 但是，与大多数其他请求映射属性不同，在类级使用时，方法级produces 属性会覆盖，而不是扩展类级声明。

#### 参数和请求头
您可以根据请求参数条件来缩小请求映射。 您可以测试是否存在请求参数（myParam），是否不存在一个请求参数（!myParam）或特定值（myParam = myValue）。 以下示例显示如何测试特定值：

```java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
```
您还可以将其与请求标头条件一起使用，如以下示例所示：

```java
@GetMapping(path = "/pets", headers = "myHeader=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
```
#### HTTP HEAD, OPTIONS
@GetMapping（和@RequestMapping（method = HttpMethod.GET））透明地支持HTTP HEAD来进行请求映射。 控制器方法不需要更改。 应用于javax.servlet.http.HttpServlet的响应包装器确保将Content-Length标头设置为写入的字节数（实际上未写入响应）。

@GetMapping（和@RequestMapping（method = HttpMethod.GET））被隐式映射到并支持HTTP HEAD。 像处理HTTP GET一样处理HTTP HEAD请求，不同的是，不是写入正文，而是计算字节数并设置Content-Length头。

默认情况下，通过将“Allow ”标头设置为所有具有匹配URL模式的@RequestMapping方法中列出的HTTP方法列表来处理HTTP OPTIONS。

对于没有HTTP方法声明的@RequestMapping，将Allow标头设置为GET，HEAD，POST，PUT，PATCH，DELETE，OPTIONS。 控制器方法应始终声明支持的HTTP方法（例如，通过使用HTTP方法特定的变体：@ GetMapping，@ PostMapping等）。

您可以将@RequestMapping方法显式映射到HTTP HEAD和HTTP OPTIONS，但这在通常情况下不是必需的。

#### 自定义注解
Spring MVC支持将组合注释用于请求映射。 这些注解本身使用@RequestMapping进行元注解，并且旨在以更狭窄，更具体的用途重新声明@RequestMapping属性的子集（或全部）。

@ GetMapping，@ PostMapping，@ PutMapping，@ DeleteMapping和@PatchMapping是组合注释的示例。 之所以提供它们，是因为大多数控制器方法应该映射到特定的HTTP方法，而不是使用@RequestMapping，后者默认情况下与所有HTTP方法都匹配。 如果需要组合注释的示例，请查看如何声明它们。

Spring MVC还支持带有自定义请求匹配逻辑的自定义请求映射属性。 这是一个更高级的选项，它需要子类化RequestMappingHandlerMapping并覆盖getCustomMethodCondition方法，您可以在其中检查自定义属性并返回自己的RequestCondition。

#### 显示注册
您可以以编程方式注册handler方法，这些方法可用于动态注册或高级案例，例如同一handler注册在不同URL下。 下面的示例注册一个处理程序方法：

```java
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) 
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); 

        Method method = UserHandler.class.getMethod("getUser", Long.class); 

        mapping.registerMapping(info, handler, method); 
    }
}
```
1. 注入目标处理程序和控制器的处理程序映射。
2. 准备请求映射元数据。
3. 获取处理程序方法。
4. 添加注册。

### handler方法
@RequestMapping处理程序方法具有灵活的签名，可以从一系列受支持的控制器方法参数和返回值中进行选择。

#### 方法参数
下表描述了受支持的控制器方法参数。 任何参数均不支持反应性类型。

支持JDK 8的java.util.Optional作为方法参数，并与具有required 属性（例如@ RequestParam，@ RequestHeader等）的注释结合在一起，等效于required = false。

| Controller method argument                                   | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `WebRequest`, `NativeWebRequest`                             | 对请求参数以及请求和会话属性的常规访问，而无需直接使用Servlet API。 |
| `javax.servlet.ServletRequest`, `javax.servlet.ServletResponse` | 选择任何特定的请求或响应类型-例如，ServletRequest，HttpServletRequest或Spring的MultipartRequest，MultipartHttpServletRequest。 |
| `javax.servlet.http.HttpSession`                             | 强制会话的存在。 永远不会为空。 请注意，会话访问不是线程安全的。 如果允许多个请求并发访问会话，请考虑将“ RequestMappingHandlerAdapter”实例的“ synchronizeOnSession”标志设置为“ true”。 |
| `javax.servlet.http.PushBuilder`                             | 用于程序化HTTP / 2资源推送的Servlet 4.0推送构建器API。 请注意，根据Servlet规范，如果客户端不支持HTTP / 2功能，则注入的`PushBuilder`实例可以为null。 |
| `java.security.Principal`                                    | 当前已认证的用户-可能是特定的`Principal`实现类（如果已知）。 |
| `HttpMethod`                                                 | 请求的HTTP方法。                                             |
| `java.util.Locale`                                           | 当前的请求区域设置，由最具体的可用LocaleResolver（实际上是配置的LocaleResolver或LocaleContextResolver）确定。 |
| `java.util.TimeZone` + `java.time.ZoneId`                    | 与当前请求关联的时区，由LocaleContextResolver确定。          |
| `java.io.InputStream`, `java.io.Reader`                      | 用于访问Servlet API公开的原始请求正文。                      |
| `java.io.OutputStream`, `java.io.Writer`                     | 用于访问Servlet API公开的原始响应正文。                      |
| `@PathVariable`                                              | 用于访问URI模板变量。 参见[URI模式](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-requestmapping-uri-templates)。 |
| `@MatrixVariable`                                            | 用于访问URI路径段中的名称/值对。 参见[矩阵变量](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-matrix-variables)。 |
| `@RequestParam`                                              | 用于访问Servlet请求参数，包括多部分文件。 参数值将转换为声明的方法参数类型。 参见`@ RequestParam`和[多部分](https：/ /docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart-forms)。请注意，对于简单的参数值，使用@RequestParam是可选的。 请参阅此表末尾的“其他任何参数”。 |
| `@RequestHeader`                                             | 用于访问请求标头。 标头值将转换为声明的方法参数类型。 参见`@ RequestHeader`。 |
| `@CookieValue`                                               | 用于访问cookie。 Cookies值将转换为声明的方法参数类型。 参见`@ CookieValue`。 |
| `@RequestBody`                                               | 用于访问HTTP请求正文。 正文内容通过使用`HttpMessageConverter`实现转换为声明的方法参数类型。 参见`@ RequestBody`。 |
| `HttpEntity<B>`                                              | 用于访问请求标头和正文。 主体是通过HttpMessageConverter转换的。 参见[HttpEntity](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-httpentity)。 |
| `@RequestPart`                                               | 要访问“ multipart / form-data”请求中的零件，请使用“ HttpMessageConverter”转换零件的主体。 参见[Multipart](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart-forms)。 |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | 用于访问HTML控制器中使用的模型，并作为视图渲染的一部分公开给模板。 |
| `RedirectAttributes`                                         | 指定在重定向的情况下使用的属性（即追加到查询字符串中），并指定要临时存储的属性，直到重定向后的请求为止。 参见[重定向属性](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-redirecting-passing-data)和[Flash属性](https：// docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-flash-attributes)。 |
| `@ModelAttribute`                                            | 用于访问已应用数据绑定和验证的模型中现有的属性（如果不存在，则进行实例化）。 参见`@ ModelAttribute` 。请注意，使用@ModelAttribute是可选的（例如，设置其属性）。 请参阅此表末尾的“其他任何参数”。 |
| `Errors`, `BindingResult`                                    | 用于访问命令对象的验证和数据绑定（即@ModelAttribute参数）中的错误，或访问@RequestBody或@RequestPart参数的验证中的错误。 您必须在经过验证的方法参数后立即声明“错误”或“ BindingResult”参数。 |
| `SessionStatus` + class-level `@SessionAttributes`           | 为了标记表单处理完成，将触发清除通过类级别的@SessionAttributes批注声明的会话属性。 有关更多详细信息，请参见`@ SessionAttributes`。 |
| `UriComponentsBuilder`                                       | 用于准备相对于当前请求的主机，端口，方案，上下文路径以及servlet映射的文字部分的URL。 参见[URI链接](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-uri-building)。 |
| `@SessionAttribute`                                          | 对于访问任何会话属性，与由于类级别的@SessionAttributes声明而存储在会话中的模型属性相反。 有关更多详细信息，请参见`@ SessionAttribute`。 |
| `@RequestAttribute`                                          | 用于访问请求属性。 有关更多详细信息，请参见`@ RequestAttribute`。 |
| Any other argument                                           | 如果方法参数与该表中的任何较早值都不匹配，并且是简单类型（由[BeanUtils＃isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.3确定。 1 / javadoc-api / org / springframework / beans / BeanUtils.html＃isSimpleProperty-java.lang.Class-)，将其解析为“ @RequestParam”；否则，将其解析为“ @ModelAttribute”。 |

#### 返回值
下表描述了受支持的控制器方法返回值。 所有返回值都支持反应性类型。

| Controller method return value                               | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `@ResponseBody`                                              | The return value is converted through `HttpMessageConverter` implementations and written to the response. See `@ResponseBody`. |
| `HttpEntity<B>`, `ResponseEntity<B>`                         | The return value that specifies the full response (including HTTP headers and body) is to be converted through `HttpMessageConverter` implementations and written to the response. See [ResponseEntity](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-responseentity). |
| `HttpHeaders`                                                | For returning a response with headers and no body.           |
| `String`                                                     | A view name to be resolved with `ViewResolver` implementations and used together with the implicit model — determined through command objects and `@ModelAttribute` methods. The handler method can also programmatically enrich the model by declaring a `Model` argument (see [Explicit Registrations](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-requestmapping-registration)). |
| `View`                                                       | A `View` instance to use for rendering together with the implicit model — determined through command objects and `@ModelAttribute` methods. The handler method can also programmatically enrich the model by declaring a `Model` argument (see [Explicit Registrations](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-requestmapping-registration)). |
| `java.util.Map`, `org.springframework.ui.Model`              | Attributes to be added to the implicit model, with the view name implicitly determined through a `RequestToViewNameTranslator`. |
| `@ModelAttribute`                                            | An attribute to be added to the model, with the view name implicitly determined through a `RequestToViewNameTranslator`.Note that `@ModelAttribute` is optional. See "Any other return value" at the end of this table. |
| `ModelAndView` object                                        | The view and model attributes to use and, optionally, a response status. |
| `void`                                                       | A method with a `void` return type (or `null` return value) is considered to have fully handled the response if it also has a `ServletResponse`, an `OutputStream` argument, or an `@ResponseStatus` annotation. The same is also true if the controller has made a positive `ETag` or `lastModified` timestamp check (see [Controllers](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-caching-etag-lastmodified) for details).If none of the above is true, a `void` return type can also indicate “no response body” for REST controllers or a default view name selection for HTML controllers. |
| `DeferredResult<V>`                                          | Produce any of the preceding return values asynchronously from any thread — for example, as a result of some event or callback. See [Asynchronous Requests](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async) and `DeferredResult`. |
| `Callable<V>`                                                | Produce any of the above return values asynchronously in a Spring MVC-managed thread. See [Asynchronous Requests](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async) and `Callable`. |
| `ListenableFuture<V>`, `java.util.concurrent.CompletionStage<V>`, `java.util.concurrent.CompletableFuture<V>` | Alternative to `DeferredResult`, as a convenience (for example, when an underlying service returns one of those). |
| `ResponseBodyEmitter`, `SseEmitter`                          | Emit a stream of objects asynchronously to be written to the response with `HttpMessageConverter` implementations. Also supported as the body of a `ResponseEntity`. See [Asynchronous Requests](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async) and [HTTP Streaming](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async-http-streaming). |
| `StreamingResponseBody`                                      | Write to the response `OutputStream` asynchronously. Also supported as the body of a `ResponseEntity`. See [Asynchronous Requests](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async) and [HTTP Streaming](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async-http-streaming). |
| Reactive types — Reactor, RxJava, or others through `ReactiveAdapterRegistry` | Alternative to `DeferredResult` with multi-value streams (for example, `Flux`, `Observable`) collected to a `List`.For streaming scenarios (for example, `text/event-stream`, `application/json+stream`), `SseEmitter` and `ResponseBodyEmitter` are used instead, where `ServletOutputStream` blocking I/O is performed on a Spring MVC-managed thread and back pressure is applied against the completion of each write.See [Asynchronous Requests](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async) and [Reactive Types](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async-reactive-types). |
| Any other return value                                       | Any return value that does not match any of the earlier values in this table and that is a `String` or `void` is treated as a view name (default view name selection through `RequestToViewNameTranslator` applies), provided it is not a simple type, as determined by [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-). Values that are simple types remain unresolved. |

#### 类型转化
如果参数声明为String以外的形式，则某些表示基于String的请求输入的带注释的控制器方法参数（例如@ RequestParam，@ RequestHeader，@ PathVariable，@ MatrixVariable和@CookieValue）可能需要类型转换。

在这种情况下，将根据配置的转换器自动应用类型转换。 默认情况下，支持简单类型（int，long，Date和其他）。 您可以通过WebDataBinder（请参阅DataBinder）或通过在FormattingConversionService中注册Formatter来自定义类型转换。 参见Spring字段格式。

#### 矩阵变量
RFC 3986讨论路径段中的名称/值对。 在Spring MVC中，我们根据Tim Berners-Lee的“旧帖子”将其称为“矩阵变量”，但它们也可以称为URI路径参数。

矩阵变量可以出现在任何路径段中，每个变量用分号分隔，多个值用逗号分隔（例如，/cars;color=red,green;year=2012）。 也可以通过重复的变量名称指定多个值（例如，color = red; color = green; color = blue）。

如果期望URL包含矩阵变量，则控制器方法的请求映射必须使用URI变量来屏蔽该变量内容，并确保可以成功地匹配请求，而与矩阵变量的顺序和状态无关。 以下示例使用矩阵变量：

```java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
```
鉴于所有路径段都可能包含矩阵变量，因此有时您可能需要消除矩阵变量应位于哪个路径变量的歧义。下面的示例演示了如何做到这一点：

```java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```
可以将矩阵变量定义为可选变量，并指定默认值，如以下示例所示：

```java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1
}
```
要获取所有矩阵变量，可以使用MultiValueMap，如以下示例所示：

```java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```
请注意，您需要启用矩阵变量的使用。 在MVC Java配置中，您需要通过Path Matching设置带有removeSemicolonContent = false的UrlPathHelper。 在MVC XML名称空间中，您可以设置<mvc：annotation-driven enable-matrix-variables =“ true” />。

#### @RequestParam
您可以使用@RequestParam批注将Servlet请求参数（即查询参数或表单数据）绑定到控制器中的方法参数。

```java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    // ...

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { 
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }

    // ...

}
```
默认情况下，使用此批注的方法参数是必需的，但是您可以通过将@RequestParam批注的required标志设置为false或使用java.util.Optional包装器声明该参数，来指定方法参数是可选的。

如果目标方法参数类型不是字符串，则将自动应用类型转换。 请参阅类型转换。

将参数类型声明为数组或列表，可以为同一参数名称解析多个参数值。

如果将@RequestParam批注声明为Map <String，String>或MultiValueMap <String，String>，但未在批注中指定参数名称，则将使用每个给定参数名称的请求参数值填充映射。

请注意，@ RequestParam的使用是可选的（例如，设置其属性）。 默认情况下，任何简单值类型的参数（由BeanUtils＃isSimpleProperty确定），并且没有被任何其他参数解析器解析，就如同使用@RequestParam进行了注释一样。

#### @RequestHeader
您可以使用@RequestHeader批注将请求标头绑定到控制器中的方法参数。

考虑以下带有标头的请求：

```bash
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
```
以下示例获取Accept-Encoding和Keep-Alive标头的值：

```java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
```
如果目标方法的参数类型不是String，则将自动应用类型转换。 请参阅类型转换。

在Map <String，String>，MultiValueMap <String，String>或HttpHeaders参数上使用@RequestHeader批注时，将使用所有标头值填充映射。

内置支持可用于将逗号分隔的字符串转换为数组或字符串集合或类型转换系统已知的其他类型。 例如，用@RequestHeader（“ Accept”）注释的方法参数可以是String类型，也可以是String \[\]或List 。

#### @CookieValue
您可以使用@CookieValue批注将HTTP cookie的值绑定到控制器中的方法参数。

考虑带有以下cookie的请求：

```bash
JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
```
```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
```
如果目标方法的参数类型不是String，则将自动应用类型转换。 请参阅类型转换。

#### @ModelAttribute
您可以在方法参数上使用@ModelAttribute批注，以从模型访问属性，或将其实例化（如果不存在）。 model属性会被请求参数中的值覆盖（如果参数名称相投同）。 这称为数据绑定，它自动帮您进行解析和转化的工作。 以下示例显示了如何执行此操作：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { }
```
Pet实例填充流程如下：

* 从模型（如果已通过使用模型添加，该模型是通过@ModelAttribute注解在方法上实现的）。
* 通过使用@SessionAttributes在HTTP会话中进行。
* 来自通过Converter传递的URI路径变量（请参见下一个示例）。
* 从默认构造函数的调用开始。
* 从请求参数中构造

尽管通常使用模型(Model章节)来填充示例，但另一种替代方法是依赖于Converter <String，T>与URI路径变量约定结合使用。 在以下示例中，模型属性名称account与URI路径变量account匹配，并且通过将String帐号传递给已注册的Converter <String，Account>来加载Account：

```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```
获取模型属性实例后，将应用数据绑定。 WebDataBinder类将Servlet请求参数名称（查询参数和表单字段）与目标Object上的字段名称匹配。 应用类型转换后，如有必要，将填充匹配字段。 有关数据绑定（和验证）的更多信息，请参见验证。 有关自定义数据绑定的更多信息，请参见DataBinder。

数据绑定可能导致错误。 默认情况下，引发BindException。 但是，要检查控制器方法中的此类错误，可以在@ModelAttribute旁边立即添加BindingResult参数，如以下示例所示：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```
在某些情况下，您可能希望只访问模型属性，而不把请求参数绑定到方法参数上。 对于这种情况，可以将模型注入控制器并直接访问它，或者设置@ModelAttribute（binding = false），如以下示例所示：

```java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { 
    // ...
}
```
您可以在数据绑定之后通过添加javax.validation.Valid注释或Spring的@Validated注释（Bean验证和Spring验证）自动应用验证。 以下示例显示了如何执行此操作：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { 
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```
请注意，使用@ModelAttribute是可选的（例如，设置其属性）。 默认情况下，任何不是简单值类型（由BeanUtils＃isSimpleProperty确定）且未被其他任何参数解析器解析的参数都将被视为使用@ModelAttribute进行了注释。

#### @SessionAttributes
@SessionAttributes用于在请求之间的HTTP Servlet会话中存储模型属性。 它是类型级别的注释，用于声明特定控制器使用的会话属性。 这通常列出应透明地存储在会话中以供后续访问请求的模型属性的名称或模型属性的类型。

以下示例使用@SessionAttributes批注：

```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {
    // ...
}
```
如果有请求，将名称为pet的模型属性添加到模型时，该属性会自动提升到HTTP Servlet会话并保存在该会话中。 它会一直保留在那里，直到另一个控制器方法使用SessionStatus方法参数来清除存储，如以下示例所示：

```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors) {
            // ...
        }
            status.setComplete(); 
            // ...
        }
    }
}
```
#### @SessionAttribute
如果您需要访问全局存在的会话属性，则可以在方法参数上使用@SessionAttribute批注，例如 以下示例显示：

```java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { 
    // ...
}
```
对于需要添加或删除会话属性的用例，请考虑将org.springframework.web.context.request.WebRequest或javax.servlet.http.HttpSession注入到控制器方法中。

要在控制器工作流中将模型属性临时存储在会话中，请考虑使用@SessionAttributes，如@SessionAttributes中所述。

#### @RequestAttribute
与@SessionAttribute类似，您可以使用@RequestAttribute批注来访问之前创建的预先存在的请求属性（例如，通过Servlet Filter或HandlerInterceptor）：

```java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { 
    // ...
}
```
#### Redirect Attributes
默认情况下，所有模型属性均被视为在重定向URL中作为URI模板变量公开。 其他属性中，那些属于原始类型或原始类型的集合或数组的属性会自动附加为查询参数。

如果专门为重定向准备了模型实例，则将原始类型属性作为查询参数附加可能是理想的结果。 但是，在带注释的控制器中，模型可以包含为渲染目的添加的其他属性（例如，下拉字段值）。 为避免此类属性出现在URL中的可能性，@ RequestMapping方法可以声明RedirectAttributes类型的参数，并使用它指定可用于RedirectView的确切属性。 如果该方法确实重定向，则使用RedirectAttributes的内容。 否则，将使用模型的内容。

RequestMappingHandlerAdapter提供了一个名为ignoreDefaultModelOnRedirect的标志，您可以使用该标志指示如果控制器方法重定向，则绝不要使用默认Model的内容。 相反，控制器方法应声明一个RedirectAttributes类型的属性，或者，如果没有这样做，则不应将任何属性传递给RedirectView。 MVC命名空间和MVC Java配置都将此标志设置为false，以保持向后兼容性。 但是，对于新应用程序，我们建议将其设置为true。

请注意，展开重定向URL时，当前请求中的URI模板变量会自动变为可用，并且您无需通过Model或RedirectAttributes显式添加它们。 以下示例显示了如何定义重定向：

```java
@PostMapping("/files/{path}")
public String upload(...) {
    // ...
    return "redirect:files/{path}";
}
```
将数据传递到重定向目标的另一种方法是使用Flash属性。 与其他重定向属性不同，Flash属性保存在HTTP会话中（因此不会出现在URL中）。 有关更多信息，请参见Flash属性。

#### Flash Attributes
Flash属性为一个请求提供了一种存储打算在另一个请求中使用的属性的方式。 重定向时最常需要此操作，例如Post-Redirect-Get模式。 Flash属性在重定向之前（通常在会话中）被临时保存，以便在重定向之后可供请求使用，并立即被删除。

Spring MVC有两个主要的抽象来支持Flash属性。 FlashMap用于保存Flash属性，而FlashMapManager用于存储，检索和管理FlashMap实例。

Flash属性支持始终处于“打开”状态，无需显式启用。 但是，如果不使用它，则永远不会导致HTTP会话创建。 在每个请求上，都有一个“input” FlashMap，其属性是从上一个请求（如果有）传递过来的，而“output” FlashMap的属性是为后续请求保存的。 可以通过RequestContextUtils中的静态方法从Spring MVC中的任何位置访问这两个FlashMap实例。

带注释的控制器通常不需要直接使用FlashMap。 相反，@ RequestMapping方法可以接受RedirectAttributes类型的参数，并使用它为重定向方案添加Flash属性。 通过RedirectAttributes添加的Flash属性会自动传播到“output” FlashMap。 同样，重定向后，来自“input” FlashMap的属性会自动添加到服务于目标URL的控制器的模型中。

#### Multipart
启用MultipartResolver后，将解析具有multipart / form-data的POST请求的内容，并将其作为常规请求参数进行访问。 以下示例访问一个常规表单字段和一个上载文件：

```java
@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```
将参数类型声明为List 允许解析相同参数名称的多个文件。

如果将@RequestParam批注声明为Map <String，MultipartFile>或MultiValueMap <String，MultipartFile>，但未在批注中指定参数名称，则将使用每个给定参数名称的多部分文件填充该映射。

通过Servlet 3.0多部分解析，您还可以声明javax.servlet.http.Part而不是Spring的MultipartFile作为方法参数或集合值类型。

您还可以将多部分内容用作绑定到命令对象的数据的一部分。 例如，前面示例中的表单字段和文件可以是表单对象上的字段，如以下示例所示：

```java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...
}

@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        if (!form.getFile().isEmpty()) {
            byte[] bytes = form.getFile().getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```
在RESTful服务方案中，也可以从非浏览器客户端提交多部分请求。 以下示例显示了带有JSON的文件：

```bash
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
    "name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...
```
您可以使用@RequestParam作为字符串访问“meta-data”部分，但您可能希望将其从JSON反序列化（类似于@RequestBody）。 在使用HttpMessageConverter进行转换之后，使用@RequestPart批注来访问多部分：

```java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {
    // ...
}
```
您可以将@RequestPart与javax.validation.Valid结合使用，也可以使用Spring的@Validated注释，这两种注释都会导致应用标准Bean验证。 默认情况下，验证错误会导致MethodArgumentNotValidException，该异常将转换为400（BAD\_REQUEST）响应。 另外，您可以通过Errors或BindingResult参数在控制器内本地处理验证错误，如以下示例所示：

```java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") MetaData metadata,
        BindingResult result) {
    // ...
}
```
#### @RequestBody
您可以使用@RequestBody批注使请求正文通过HttpMessageConverter读取并反序列化为Object。 以下示例使用@RequestBody参数：

```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```
您可以使用MVC Config的“消息转换器”选项来配置或自定义消息转换。

您可以将@RequestBody与javax.validation.Valid或Spring的@Validated批注结合使用，这两种方法都会导致应用标准Bean验证。 默认情况下，验证错误会导致MethodArgumentNotValidException，该异常将转换为400（BAD\_REQUEST）响应。 另外，您可以通过Errors或BindingResult参数在控制器内本地处理验证错误，如以下示例所示：

```java
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Account account, BindingResult result) {
    // ...
}
```
#### HttpEntity
HttpEntity或多或少与使用@RequestBody相同，但它基于公开请求标头和正文的容器对象。 以下清单显示了一个示例：

```java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
```
#### @ResponseBody
您可以在方法上使用@ResponseBody批注，以将返回值通过HttpMessageConverter序列化为响应主体。 以下清单显示了一个示例：

```java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
```
在类级别还支持@ResponseBody，在这种情况下，所有控制器方法都将继承它。 这就是@RestController的效果，它只不过是带有@Controller和@ResponseBody标记的元注释。

您可以将@ResponseBody与反应类型一起使用。 有关更多详细信息，请参见异步请求和响应类型。

您可以使用MVC Config的“消息转换器”选项来配置或自定义消息转换。

您可以将@ResponseBody方法与JSON序列化视图结合使用。 有关详细信息，请参见Jackson JSON。

#### ResponseEntity
ResponseEntity类似于@ResponseBody，但具有状态和标头。 例如：

```java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).build(body);
}
```
Spring MVC支持使用单值反应类型来异步生成ResponseEntity，和/或为主体使用单值和多值反应类型。

#### JSON Views
Spring MVC为Jackson的序列化视图提供了内置支持，该视图仅可呈现Object中所有字段的一部分。 要将其与@ResponseBody或ResponseEntity控制器方法一起使用，可以使用Jackson的@JsonView批注来激活序列化视图类，如以下示例所示：

```java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```
@JsonView允许一组视图类，但是每个控制器方法只能指定一个。 如果需要激活多个视图，则可以使用复合接口。

如果要以编程方式执行上述操作，而不是声明@JsonView批注，请使用MappingJacksonValue包装返回值，并使用其提供序列化视图：

```java
@RestController
public class UserController {

    @GetMapping("/user")
    public MappingJacksonValue getUser() {
        User user = new User("eric", "7!jd#h23");
        MappingJacksonValue value = new MappingJacksonValue(user);
        value.setSerializationView(User.WithoutPasswordView.class);
        return value;
    }
}
```
对于依赖视图解析器的控制器，您可以将序列化视图类添加到模型中，如以下示例所示：

```java
@Controller
public class UserController extends AbstractController {

    @GetMapping("/user")
    public String getUser(Model model) {
        model.addAttribute("user", new User("eric", "7!jd#h23"));
        model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
        return "userView";
    }
}
```
### DataBinder
@Controller或@ControllerAdvice类可以具有用于初始化WebDataBinder实例的@InitBinder方法，而这些方法又可以：

* 将请求参数（即表单或查询数据）绑定到模型对象。
* 将基于字符串的请求值（例如请求参数，路径变量，标头，Cookie等）转换为控制器方法参数的目标类型。
* 呈现HTML表单时，将模型对象的值格式化为String值。

@InitBinder方法可以注册特定于控制器的java.beans.PropertyEditor或Spring Converter和Formatter组件。 此外，您可以使用MVC配置在全局共享的FormattingConversionService中注册Converter和Formatter类型。

@InitBinder方法支持与@RequestMapping方法相同的许多参数，除了@ModelAttribute（命令对象）参数。 通常，它们使用WebDataBinder参数（用于注册）和无效的返回值声明。 以下清单显示了一个示例：

```java
@Controller
public class FormController {

    @InitBinder 
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}
```
另外，当通过共享的FormattingConversionService使用基于Formatter的设置时，可以重新使用相同的方法并注册特定于控制器的Formatter实现，如以下示例所示：

```java
@Controller
public class FormController {

    @InitBinder 
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }

    // ...
}
```
### 异常
@Controller和@ControllerAdvice类可以具有@ExceptionHandler方法来处理控制器方法的异常，如以下示例所示：

```java
@Controller
public class SimpleController {

    // ...

    @ExceptionHandler
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
```
该异常可能与正在传播的顶级异常（即，直接IOException被抛出）匹配，也可能与顶级包装程序异常（例如，包装在IllegalStateException内部的IOException）内的直接原因匹配。

对于匹配的异常类型，如前面的示例所示，最好将目标异常声明为方法参数。 当多个异常方法匹配时，根源异常匹配通常比原因异常匹配更可取。 更具体地说，ExceptionDepthComparator用于根据从引发的异常类型开始的深度对异常进行排序。

另外，注释声明可以缩小异常类型以使其匹配，如以下示例所示：

```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(IOException ex) {
    // ...
}
```
您甚至可以使用带有非常通用的参数签名的特定异常类型的列表，如以下示例所示：

```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(Exception ex) {
    // ...
}
```
我们通常建议您在参数签名中尽可能具体，以减少根类型和原因异常类型之间不匹配的可能性。 考虑将多重匹配方法分解为单独的@ExceptionHandler方法，每个方法均通过其签名匹配单个特定的异常类型。

在多@ControllerAdvice安排中，我们建议异常的范围根据根据@ControllerAdvice顺序声明，什么意思呢？假如A,B两个顺序advice，在A中声明了FileSystemException异常，在B中声明Exception.

最后但并非最不重要的一点是，@ExceptionHandler方法实现可以选择通过以原始形式重新抛出异常来退出处理给定异常实例。 在仅对根级别匹配或无法静态确定的特定上下文中的匹配感兴趣的情况下，这很有用。 重新抛出的异常会在其余的解决方案链中传播，就像给定的@ExceptionHandler方法最初不会匹配一样。

Spring MVC中对@ExceptionHandler方法的支持建立在DispatcherServlet级别HandlerExceptionResolver机制上。

#### 方法参数
`@ExceptionHandler` methods support the following arguments:

| Method argument                                              | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Exception type                                               | For access to the raised exception.                          |
| `HandlerMethod`                                              | For access to the controller method that raised the exception. |
| `WebRequest`, `NativeWebRequest`                             | Generic access to request parameters and request and session attributes without direct use of the Servlet API. |
| `javax.servlet.ServletRequest`, `javax.servlet.ServletResponse` | Choose any specific request or response type (for example, `ServletRequest` or `HttpServletRequest` or Spring’s `MultipartRequest` or `MultipartHttpServletRequest`). |
| `javax.servlet.http.HttpSession`                             | Enforces the presence of a session. As a consequence, such an argument is never `null`.<br>Note that session access is not thread-safe. Consider setting the `RequestMappingHandlerAdapter` instance’s `synchronizeOnSession` flag to `true` if multiple requests are allowed to access a session concurrently. |
| `java.security.Principal`                                    | Currently authenticated user — possibly a specific `Principal` implementation class if known. |
| `HttpMethod`                                                 | The HTTP method of the request.                              |
| `java.util.Locale`                                           | The current request locale, determined by the most specific `LocaleResolver` available — in effect, the configured `LocaleResolver` or `LocaleContextResolver`. |
| `java.util.TimeZone`, `java.time.ZoneId`                     | The time zone associated with the current request, as determined by a `LocaleContextResolver`. |
| `java.io.OutputStream`, `java.io.Writer`                     | For access to the raw response body, as exposed by the Servlet API. |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | For access to the model for an error response. Always empty. |
| `RedirectAttributes`                                         | Specify attributes to use in case of a redirect — (that is to be appended to the query string) and flash attributes to be stored temporarily until the request after the redirect. See [Redirect Attributes](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-redirecting-passing-data) and [Flash Attributes](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-flash-attributes). |
| `@SessionAttribute`                                          | For access to any session attribute, in contrast to model attributes stored in the session as a result of a class-level `@SessionAttributes` declaration. See `@SessionAttribute` for more details. |
| `@RequestAttribute`                                          | For access to request attributes. See `@RequestAttribute` for more details. |

#### 返回值
`@ExceptionHandler` methods support the following return values:

| Return value                                    | Description                                                  |
| ----------------------------------------------- | ------------------------------------------------------------ |
| `@ResponseBody`                                 | The return value is converted through `HttpMessageConverter` instances and written to the response. See `@ResponseBody`. |
| `HttpEntity<B>`, `ResponseEntity<B>`            | The return value specifies that the full response (including the HTTP headers and the body) be converted through `HttpMessageConverter` instances and written to the response. See [ResponseEntity](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-responseentity). |
| `String`                                        | A view name to be resolved with `ViewResolver` implementations and used together with the implicit model — determined through command objects and `@ModelAttribute` methods. The handler method can also programmatically enrich the model by declaring a `Model` argument (described earlier). |
| `View`                                          | A `View` instance to use for rendering together with the implicit model — determined through command objects and `@ModelAttribute` methods. The handler method may also programmatically enrich the model by declaring a `Model` argument (descried earlier). |
| `java.util.Map`, `org.springframework.ui.Model` | Attributes to be added to the implicit model with the view name implicitly determined through a `RequestToViewNameTranslator`. |
| `@ModelAttribute`                               | An attribute to be added to the model with the view name implicitly determined through a `RequestToViewNameTranslator`.Note that `@ModelAttribute` is optional. See “Any other return value” at the end of this table. |
| `ModelAndView` object                           | The view and model attributes to use and, optionally, a response status. |
| `void`                                          | A method with a `void` return type (or `null` return value) is considered to have fully handled the response if it also has a `ServletResponse` an `OutputStream` argument, or a `@ResponseStatus` annotation. The same is also true if the controller has made a positive `ETag` or `lastModified` timestamp check (see [Controllers](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-caching-etag-lastmodified) for details).If none of the above is true, a `void` return type can also indicate “no response body” for REST controllers or default view name selection for HTML controllers. |
| Any other return value                          | If a return value is not matched to any of the above and is not a simple type (as determined by [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.3.1/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)), by default, it is treated as a model attribute to be added to the model. If it is a simple type, it remains unresolved. |

#### REST API exceptions
REST服务的常见要求是在响应正文中包含错误详细信息。 Spring框架不会自动执行此操作，因为响应主体中错误详细信息的表示是特定于应用程序的。 但是，@ RestController可以将@ExceptionHandler方法与ResponseEntity返回值一起使用，以设置响应的状态和主体。 也可以在@ControllerAdvice类中声明此类方法，以将其全局应用。

在响应主体中使用错误详细信息实现全局异常处理的应用程序应考虑扩展ResponseEntityExceptionHandler，该处理程序可处理Spring MVC引发的异常，并提供用于自定义响应主体的钩子。 要使用此功能，请创建ResponseEntityExceptionHandler的子类，并使用@ControllerAdvice对其进行注释，重写必需的方法，并将其声明为Spring bean。

### Model
@ModelAttribute注解可以用于：

* @RequestMapping标注的方法参数上，访问（没有则创建）Model里面对应的对象并使用WebDataBinder将该对象绑定到请求中。
* 标注在`@Controller` 或 `@ControllerAdvice`注解的方法上，初始化Model对象，该方法会在任何@RequestMapping方法之前执行。
* 标注在@RequestMapping方法上，表明该方法的返回值会作为Model属性。

本节讨论@ModelAttribute方法-前面列表中的第二项。 控制器可以具有任意数量的@ModelAttribute方法。 所有此类方法均在同一控制器中的@RequestMapping方法之前调用。 也可以通过@ControllerAdvice在控制器之间共享@ModelAttribute方法。 有关更多详细信息，请参见“控制器Advice”部分。

@ModelAttribute方法具有灵活的方法签名。 它们支持许多与@RequestMapping方法相同的参数，除了@ModelAttribute本身或与请求主体相关的任何东西。

下面的例子是@ModelAttribute注解的Controller方法

```java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
```
以下示例仅添加一个属性：

```java
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
```
如果未明确指定名称，则根据“对象”类型选择默认名称，如javadoc中所述。 您始终可以使用重载的addAttribute方法或通过@ModelAttribute上的name属性（用于返回值）来分配显式名称。

您也可以将@ModelAttribute用作@RequestMapping方法上的方法级注释，在这种情况下，@ RequestMapping方法的返回值将解释为模型属性。 通常不需要这样做，因为它是HTML控制器的默认行为，除非返回值是一个String，这时，返回值被解释为视图名称。 @ModelAttribute还可以自定义模型属性名称，如以下示例所示：

```java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```
```java

 @ModelAttribute
 @RequestMapping("/model/name")
    public String testModelName(Model model) {
        Map<String, Object> stringObjectMap = model.asMap();
        //{myUser=MyUser{id=123, name='???', birth=null, password='null', createTime=null}}
        return stringObjectMap.toString();
    }
```
这个时候，会把返回值解析成视图名称，找不到视图就会出错。

### Controller Advice
通常，@ ExceptionHandler，@ InitBinder和@ModelAttribute方法在声明它们的@Controller类（或类层次结构）中应用。 如果要使此类方法更全局地应用（跨控制器），则可以在带有@ControllerAdvice或@RestControllerAdvice注释的类中声明它们。

@ControllerAdvice带有@Component注释，这意味着可以通过组件扫描将此类注册为Spring Bean。 @RestControllerAdvice是由@ControllerAdvice和@ResponseBody注释的组合注释，这实际上意味着@ExceptionHandler方法通过消息转换（与视图分辨率或模板渲染）呈现给响应主体。

启动时，@ RequestMapping和@ExceptionHandler方法的基础结构类将检测使用@ControllerAdvice注释的Spring bean，然后在运行时应用其方法。 全局@ExceptionHandler方法（来自@ControllerAdvice）在本地方法（来自@Controller）之后应用。 相比之下，全局@ModelAttribute和@InitBinder方法在本地方法之前应用。

默认情况下，@ ControllerAdvice方法适用于每个请求（即所有控制器），但是您可以通过使用批注上的属性将其范围缩小到控制器的子集，如以下示例所示：

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```
前面示例中的选择器在运行时进行评估，如果广泛使用，可能会对性能产生负面影响。 有关更多详细信息，请参见@ControllerAdvice javadoc。

### CORS
#### 简介
出于安全原因，浏览器禁止AJAX调用当前来源以外的资源。 例如，您可以将您的银行帐户放在一个浏览器标签中，将evil.com放在另一个标签中。 来自evil.com的脚本不应使用您的凭据向您的银行API发出AJAX请求，例如从您的帐户中取钱！

跨域资源共享（CORS）是由大多数浏览器实现的W3C规范，可让您指定授权哪种类型的跨域请求，而不是使用基于IFRAME或JSONP的安全性较低且功能较弱的变通办法。

#### 处理
CORS规范区分预检（例如Options），简单和实际要求。 要了解CORS的工作原理，您可以阅读本文[Cross-Origin Resource Sharing (CORS) - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)，或者参阅规范以获取更多详细信息。

Spring MVC HandlerMapping实现为CORS提供内置支持。 成功将请求映射到处理程序后，HandlerMapping实现将检查给定请求和处理程序的CORS配置，并采取进一步的措施。 遇检请求直接处理，而简单和实际的CORS请求被拦截，验证并设置了必需的CORS响应标头。

为了启用跨域请求（即存在Origin标头，并且与请求的主机不同），您需要具有一些显式声明的CORS配置。 如果找不到匹配的CORS配置，则预检请求将被拒绝。 没有将CORS标头添加到简单和实际CORS请求的响应中，因此，浏览器拒绝了它们。

可以使用基于URL模式的CorsConfiguration映射分别配置每个HandlerMapping。 在大多数情况下，应用程序使用MVC Java配置或XML名称空间声明此类映射，这导致将单个全局映射传递给所有HandlerMappping实例。

您可以将HandlerMapping级别的全局CORS配置与更细粒度的处理程序级别的CORS配置结合使用。 例如，带注释的控制器可以使用类或方法级别的@CrossOrigin注释（其他处理程序可以实现CorsConfigurationSource）。

全局和本地配置组合的规则通常是相加的。 对于那些只能接受单个值的属性，例如 allowCredentials和maxAge，则局部值覆盖全局值。 有关更多详细信息，请参见CorsConfiguration＃combine（CorsConfiguration）。

如果你想高度定制化，请参考下面几个类源码：

* `CorsConfiguration`
* `CorsProcessor`, `DefaultCorsProcessor`
* `AbstractHandlerMapping`

#### @CrossOrigin
@CrossOrigin批注启用带注释的控制器方法上的跨域请求，如以下示例所示：

```java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```
@CrossOrigin默认允许所有：

* All origins.
* All headers.
* 控制器方法映射到的所有HTTP方法。

默认情况下，allowCredentials未启用，因为它将建立一个信任级别，以公开敏感的用户特定信息（例如cookie和CSRF令牌），并且仅在适当的地方使用。 启用后，必须将allowOrigins设置为一个或多个特定域（而不是特殊值“ \*”），或者可将allowOriginPatterns属性用于匹配动态的一组origins。

maxAge设置为30分钟。

@CrossOrigin在类级别也受支持，并且被所有方法继承，如以下示例所示：

```java
@CrossOrigin(origins = "https://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```
#### 全局配置
除了细粒度的控制器方法级别配置之外，您可能还想定义一些全局CORS配置。 您可以在任何HandlerMapping上分别设置基于URL的CorsConfiguration映射。 但是，大多数应用程序都使用MVC Java配置或MVC XML名称空间来执行此操作。

默认情况下，全局配置启用以下功能：

* All origins.
* All headers.
* `GET`, `HEAD`, and `POST` methods.

默认情况下，allowCredentials未启用，因为它将建立一个信任级别，以公开敏感的用户特定信息（例如cookie和CSRF令牌），并且仅在适当的地方使用。 启用后，必须将allowOrigins设置为一个或多个特定域（而不是特殊值“ \*”），或者可将allowOriginPatterns属性用于匹配动态的一组origins。

maxAge设置为30分钟。

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("https://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);

        // Add more mappings...
    }
}
```
#### CORS过滤器
您可以通过内置的CorsFilter应用CORS支持。

要配置过滤器，请将CorsConfigurationSource传递给其构造函数，如以下示例所示：

```java
CorsConfiguration config = new CorsConfiguration();

// Possibly...
// config.applyPermitDefaultValues()

config.setAllowCredentials(true);
config.addAllowedOrigin("https://domain1.com");
config.addAllowedHeader("*");
config.addAllowedMethod("*");

UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
source.registerCorsConfiguration("/**", config);

CorsFilter filter = new CorsFilter(source);
```
### HTTP缓存
HTTP缓存可以显着提高Web应用程序的性能。 HTTP缓存围绕Cache-Control响应标头以及随后的条件请求标头（例如Last-Modified和ETag）。 缓存控制为私有（例如浏览器）和公共（例如代理）缓存，提供有关如何缓存和重用响应的建议。 ETag标头用于发出条件请求，如果内容未更改，则可能导致没有主体的304（NOT\_MODIFIED）。 ETag可以看作是Last-Modified头的更复杂的后继者。

#### CacheControl
CacheControl支持配置与Cache-Control标头相关的设置，并在许多地方作为参数被接受：

* `WebContentInterceptor`
* `WebContentGenerator`
* [Controllers](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-caching-etag-lastmodified)
* [Static Resources](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-caching-static-resources)

尽管RFC 7234描述了Cache-Control响应标头的所有可能的指令，但CacheControl类型采用了面向用例的方法，着重于常见方案：

```java
// 缓存一个小时"Cache-Control: max-age=3600"
CacheControl ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS);

//阻止缓存"Cache-Control: no-store"
CacheControl ccNoStore = CacheControl.noStore();

// 缓存10天，无论共有私有,
// public缓存不应改变响应
// "Cache-Control: max-age=864000, public, no-transform"
CacheControl ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS).noTransform().cachePublic();
```
WebContentGenerator还接受一个更简单的cachePeriod属性（以秒为单位定义），该属性的工作方式如下：

* \-1值不会生成Cache-Control响应标头。
* 值为0使用“ Cache-Control: no-store”指令进行缓存。
* n> 0值通过使用'Cache-Control：max-age = n'指令将给定响应缓存n秒。

#### 控制器
控制器可以添加对HTTP缓存的显式支持。 我们建议您这样做，因为需要先计算资源的lastModified或ETag值，然后才能将其与条件请求标头进行比较。 控制器可以将ETag标头和Cache-Control设置添加到ResponseEntity，如以下示例所示：

```java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {

    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book);
}
```
如果与条件请求标头的比较表明内容未更改，则前面的示例发送带有空正文的304（NOT\_MODIFIED）响应。 否则，ETag和Cache-Control标头将添加到响应中。

您还可以在控制器中针对条件请求标头进行检查，如以下示例所示：

```java
@RequestMapping
public String myHandleMethod(WebRequest request, Model model) {

    long eTag = ... 

    if (request.checkNotModified(eTag)) {
        return null; 
    }

    model.addAttribute(...); 
    return "myViewName";
}
```
可以使用三种变体来检查针对eTag值，lastModified值或两者的条件请求。 对于有条件的GET和HEAD请求，可以将响应设置为304（NOT\_MODIFIED）。 对于条件POST，PUT和DELETE，您可以将响应设置为412（PRECONDITION\_FAILED），以防止并发修改。

#### 静态资源
您应该为静态资源提供Cache-Control和条件响应标头，以实现最佳性能。 请参阅有关配置静态资源的部分。

#### `ETag` Filter
您可以使用ShallowEtagHeaderFilter来添加根据响应内容计算的“浅” eTag值，从而节省带宽，但不节省CPU时间。 请参阅浅ETag。

## 附录
#### DispatcherServlet.properties
![image](index_files/76a8eba1-3fe6-4b61-a5c8-67669d147c33.png)

### spring mvc中获取Context
```java
        Object attribute = request.getAttribute(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE);
        System.err.println(attribute.getClass().getName());
        WebApplicationContext webApplicationContext = RequestContextUtils.findWebApplicationContext(request);
        System.err.println(webApplicationContext.equals(attribute));//true

        Object attribute1 = RequestContextHolder.currentRequestAttributes().getAttribute(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);

        System.err.println(attribute.equals(attribute1));//true
        return attribute.getClass().getName();
```
### LastModify实现
继承Controller方式

```java
public class TestController implements Controller,LastModified{
    private long lastModified = System.currentTimeMillis();

    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response)
            throws Excepti回视图的话
        return new ModelAndView("returnView");
    }

    @Override
    public long getLastModified(HttpServletRequest request) {
        if (lastModified == 0L) {
        // 只用返回历史时间戳即可
        return lastModified;
    }
}
```
Webrequest方式

```java
@RestController
@RequestMapping("/last")
public class LastModifyControllerTest {

    private long lastModify = System.currentTimeMillis();

    @GetMapping("/t1")
    public String test1(WebRequest webRequest) {
        System.err.println(lastModify);
        if (webRequest.checkNotModified(lastModify)) {
            return null;
        }
        System.err.println("test1");
        return "test1";
    }

```
为什么会出现这两种方式呢？我们来看一下源码：

```java
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }

```
ha是HandlerAdapter，@RequestMapping对应的实现是RequestMappingHandlerAdapter，该类的getLastModified方法直接返回-1（表示已修改，不走304）.继承Controller方式的方式实现类是SimpleControllerHandlerAdapter，该类会判断handler是否实现LastModified接口，实现则调用实现接口。

### log-request-details
springboot只需要在属性文件中开启即可：

```Plain Text
spring.mvc.log-request-details=true
debug=true
```
```bash
DispatcherServlet        : enableLoggingRequestDetails='false': request parameters and headers will be masked to prevent unsafe logging of potentially sensitive data
DispatcherServlet        : Completed initialization in 4 ms
DispatcherServlet        : POST "/last/t1?password=122323", parameters={masked}
RequestMappingHandlerMapping : Mapped to com.example.dateformat.web.LastModifyControllerTest#test1(WebRequest)
1605243469961
test1
RequestResponseBodyMethodProcessor : Using 'text/plain', given [*/*] and supported [text/plain, */*, application/json, application/*+json]
RequestResponseBodyMethodProcessor : Writing ["test1"]
DispatcherServlet        : Completed 200 OK




DispatcherServlet        : enableLoggingRequestDetails='true': request parameters and headers will be shown which may lead to unsafe logging of potentially sensitive data
DispatcherServlet        : Completed initialization in 6 ms
DispatcherServlet        : POST "/last/t1?password=122323", parameters={password:[122323], name:[zhao]}
RequestMappingHandlerMapping : Mapped to com.example.dateformat.web.LastModifyControllerTest#test1(WebRequest)
1605243584655
test1
RequestResponseBodyMethodProcessor : Using 'text/plain', given [*/*] and supported [text/plain, */*, application/json, application/*+json]
RequestResponseBodyMethodProcessor : Writing ["test1"]
DispatcherServlet        : Completed 200 OK

```
