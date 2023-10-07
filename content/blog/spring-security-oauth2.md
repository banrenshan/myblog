---
title: spring-security-oauth2
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:43:31
---

### OAuth2的配置属性

| Spring Boot 2.x                                              | ClientRegistration                                     |
| ------------------------------------------------------------ | ------------------------------------------------------ |
| spring.security.oauth2.client.registration.*[registrationId]* | registrationId                                         |
| spring.security.oauth2.client.registration.*[registrationId]*.client-id | clientId                                               |
| spring.security.oauth2.client.registration.*[registrationId]*.client-secret | clientSecret                                           |
| spring.security.oauth2.client.registration.*[registrationId]*.client-authentication-method | clientAuthenticationMethod                             |
| spring.security.oauth2.client.registration.*[registrationId]*.authorization-grant-type | authorizationGrantType                                 |
| spring.security.oauth2.client.registration.*[registrationId]*.redirect-uri | redirectUri                                            |
| spring.security.oauth2.client.registration.*[registrationId]*.scope | scopes                                                 |
| spring.security.oauth2.client.registration.*[registrationId]*.client-name | clientName                                             |
| spring.security.oauth2.client.provider.*[providerId]*.authorization-uri | providerDetails.authorizationUri                       |
| spring.security.oauth2.client.provider.*[providerId]*.token-uri | providerDetails.tokenUri                               |
| spring.security.oauth2.client.provider.*[providerId]*.jwk-set-uri | providerDetails.jwkSetUri                              |
| spring.security.oauth2.client.provider.*[providerId]*.issuer-uri | providerDetails.issuerUri                              |
| spring.security.oauth2.client.provider.*[providerId]*.user-info-uri | providerDetails.userInfoEndpoint.uri                   |
| spring.security.oauth2.client.provider.*[providerId]*.user-info-authentication-method | providerDetails.userInfoEndpoint.authenticationMethod  |
| spring.security.oauth2.client.provider.*[providerId]*.user-name-attribute | providerDetails.userInfoEndpoint.userNameAttributeName |



配置举例：

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          okta:
            client-id: okta-client-id
            client-secret: okta-client-secret
        provider:
          okta:	
            authorization-uri: https://your-subdomain.oktapreview.com/oauth2/v1/authorize
            token-uri: https://your-subdomain.oktapreview.com/oauth2/v1/token
            user-info-uri: https://your-subdomain.oktapreview.com/oauth2/v1/userinfo
            user-name-attribute: sub
            jwk-set-uri: https://your-subdomain.oktapreview.com/oauth2/v1/keys
```

OAuth2ClientAutoConfiguration 类主要做了下面的工作：

- 注册 ClientRegistrationRepository@Bean，该Bean由配置的OAuth客户端属性中的ClientRegistry组成。
- 注册 SecurityFilterChain@Bean，并通过httpSecurity.oauth2Login()启用OAuth 2.0登录。

如果你想覆盖自动配置可以通过下面的方式：

- 注册 ClientRegistrationRepository bean
- 注册 SecurityFilterChain bean
- 完全覆盖自动注册类



### 注册 ClientRegistrationRepository bean

```java
@Configuration
public class OAuth2LoginConfig {

	@Bean
	public ClientRegistrationRepository clientRegistrationRepository() {
		return new InMemoryClientRegistrationRepository(this.googleClientRegistration());
	}

	private ClientRegistration googleClientRegistration() {
		return ClientRegistration.withRegistrationId("google")
			.clientId("google-client-id")
			.clientSecret("google-client-secret")
			.clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
			.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
			.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
			.scope("openid", "profile", "email", "address", "phone")
			.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
			.tokenUri("https://www.googleapis.com/oauth2/v4/token")
			.userInfoUri("https://www.googleapis.com/oauth2/v3/userinfo")
			.userNameAttributeName(IdTokenClaimNames.SUB)
			.jwkSetUri("https://www.googleapis.com/oauth2/v3/certs")
			.clientName("Google")
			.build();
	}
}
```

### 注册 SecurityFilterChain bean

以下示例显示如何使用@EnableWebSecurity注册SecurityFilterChain@Bean，并通过httpSecurity.oauth2Login() 启用OAuth 2.0登录：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.authorizeHttpRequests(authorize -> authorize
				.anyRequest().authenticated()
			)
			.oauth2Login(withDefaults());
		return http.build();
	}
}
```

### 完全覆盖自动注册类

以下示例显示了如何通过注册ClientRegistrationRepository@Bean和SecurityFilterChain@Bean来完全覆盖自动配置。

```java
@Configuration
public class OAuth2LoginConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.authorizeHttpRequests(authorize -> authorize
				.anyRequest().authenticated()
			)
			.oauth2Login(withDefaults());
		return http.build();
	}

	@Bean
	public ClientRegistrationRepository clientRegistrationRepository() {
		return new InMemoryClientRegistrationRepository(this.googleClientRegistration());
	}

	private ClientRegistration googleClientRegistration() {
		return ClientRegistration.withRegistrationId("google")
			.clientId("google-client-id")
			.clientSecret("google-client-secret")
			.clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
			.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
			.redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
			.scope("openid", "profile", "email", "address", "phone")
			.authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
			.tokenUri("https://www.googleapis.com/oauth2/v4/token")
			.userInfoUri("https://www.googleapis.com/oauth2/v3/userinfo")
			.userNameAttributeName(IdTokenClaimNames.SUB)
			.jwkSetUri("https://www.googleapis.com/oauth2/v3/certs")
			.clientName("Google")
			.build();
	}
}
```

如果您无法使用Spring Boot 2.x，并且希望在CommonOAuth2Provider（例如，Google）中配置一个预定义的提供程序，请应用以下配置：

```java
@EnableWebSecurity
public class OAuth2LoginConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.authorizeHttpRequests(authorize -> authorize
				.anyRequest().authenticated()
			)
			.oauth2Login(withDefaults());
		return http.build();
	}

	@Bean
	public ClientRegistrationRepository clientRegistrationRepository() {
		return new InMemoryClientRegistrationRepository(this.googleClientRegistration());
	}

	@Bean
	public OAuth2AuthorizedClientService authorizedClientService(
			ClientRegistrationRepository clientRegistrationRepository) {
		return new InMemoryOAuth2AuthorizedClientService(clientRegistrationRepository);
	}

	@Bean
	public OAuth2AuthorizedClientRepository authorizedClientRepository(
			OAuth2AuthorizedClientService authorizedClientService) {
		return new AuthenticatedPrincipalOAuth2AuthorizedClientRepository(authorizedClientService);
	}

	private ClientRegistration googleClientRegistration() {
		return CommonOAuth2Provider.GOOGLE.getBuilder("google")
			.clientId("google-client-id")
			.clientSecret("google-client-secret")
			.build();
	}
}
```

HttpSecurity.oauth2Login() 为自定义OAuth 2.0登录提供了许多配置选项。主要配置选项被分组到其协议端点对应项中。例如，oauth2Login().authorizationEndpoint()允许配置授权端点，而oauth2Login().tokenEndpoin()允许配置令牌端点。如下面代码：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .authorizationEndpoint(authorization -> authorization
			            ...
			    )
			    .redirectionEndpoint(redirection -> redirection
			            ...
			    )
			    .tokenEndpoint(token -> token
			            ...
			    )
			    .userInfoEndpoint(userInfo -> userInfo
			            ...
			    )
			);
		return http.build();
	}
}
```

oauth2Login() DSL的主要目标是与规范中定义的命名紧密一致。

OAuth 2.0授权框架将协议端点定义如下：

授权过程使用授权服务器两个端点（HTTP资源）：

- 授权端点：由客户端使用，通过用户代理重定向，从资源所有者获得授权。
- 令牌端点：客户端用于获取访问令牌，通常携带客户端身份信息。

此外还有一个客户端端点:

- 重定向端点：授权服务器使用它通过资源所有者用户代理向客户端返回包含授权凭据的响应。

OpenID Connect Core 1.0规范对UserInfo端点的定义如下：

UserInfo端点是一个OAuth 2.0受保护的资源，它返回关于经过身份验证的最终用户的声明。为了获得有关最终用户的请求声明，客户端使用通过OpenID Connect身份验证获得的访问令牌向UserInfo端点发出请求。这些声明通常由一个JSON对象表示，该对象包含声明的键-值对集合。

以下代码显示了oauth2Login() DSL可用的完整配置选项：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .clientRegistrationRepository(this.clientRegistrationRepository())
			    .authorizedClientRepository(this.authorizedClientRepository())
			    .authorizedClientService(this.authorizedClientService())
			    .loginPage("/login")
			    .authorizationEndpoint(authorization -> authorization
			        .baseUri(this.authorizationRequestBaseUri())
			        .authorizationRequestRepository(this.authorizationRequestRepository())
			        .authorizationRequestResolver(this.authorizationRequestResolver())
			    )
			    .redirectionEndpoint(redirection -> redirection
			        .baseUri(this.authorizationResponseBaseUri())
			    )
			    .tokenEndpoint(token -> token
			        .accessTokenResponseClient(this.accessTokenResponseClient())
			    )
			    .userInfoEndpoint(userInfo -> userInfo
			        .userAuthoritiesMapper(this.userAuthoritiesMapper())
			        .userService(this.oauth2UserService())
			        .oidcUserService(this.oidcUserService())
			    )
			);
		return http.build();
	}
}
```

### 登录页配置

默认情况下，OAuth 2.0登录页面由DefaultLoginPageGeneratingFilter自动生成。默认登录页面显示每个配置的OAuth客户端及其ClientRegistration.clientName作为链接，能够启动授权请求（或OAuth 2.0登录）。

为了让DefaultLoginPageGeneratingFilter显示配置的OAuth客户端的链接，注册的ClientRegistrationRepository还需要实现Iterable<ClientRegistration>。有关参考信息，请参阅InMemoryClientRegistrationRepository。

每个OAuth客户端的链接目标默认为：

OAuth2AuthorizationRequestRedirectFilter.DEFAULT_AUTHORIZATION_REQUEST_BASE_URI + "/{registrationId}"

例如下面的示例：

```html
<a href="/oauth2/authorization/google">Google</a>
```

要覆盖默认登录页面，请配置oauth2Login().loginPage() 和（可选）oauth2Login().authizationEndpoint().baseUri()。例如：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .loginPage("/login/oauth2")
			    ...
			    .authorizationEndpoint(authorization -> authorization
			        .baseUri("/login/oauth2/authorization")
			        ...
			    )
			);
		return http.build();
	}
}
```

您需要提供一个带有@RequestMapping（“/login/oauth2”）的@Controller，它能够呈现自定义登录页面。

### 重定向端点

授权服务器使用重定向端点通过资源所有者用户代理向客户端返回授权响应（包含授权凭据）。

OAuth 2.0登录利用授权代码授予。因此，授权凭证就是授权码。

默认的授权响应baseUri（重定向端点）是/login/oauth2/code/*，它在OAuth2LoginAuthenticationFilter.default_FILTER_PROCESSES_URI中定义。

如果要自定义授权响应baseUri，请按以下示例所示进行配置：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

    @Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .redirectionEndpoint(redirection -> redirection
			        .baseUri("/login/oauth2/callback/*")
			        ...
			    )
			);
		return http.build();
	}
}
```

您还需要确保ClientRegistration.redirectUri与自定义授权响应baseUri匹配。

以下列表显示了一个示例：

```java
return CommonOAuth2Provider.GOOGLE.getBuilder("google")
	.clientId("google-client-id")
	.clientSecret("google-client-secret")
	.redirectUri("{baseUrl}/login/oauth2/callback/{registrationId}")
	.build();
```

### 用户信息端点

UserInfo端点包括许多配置选项，我们会在下面的小结中描述。

#### 映射用户权限

用户成功使用OAuth 2.0 Provider进行身份验证后，OAuth2User.getAuthorities（）（或OidcUser.getauthorites（））可能会映射到一组新的GrantedAuthority实例，在完成身份验证时，这些实例将提供给OAuth2AuthenticationToken。

OAuth2AuthenticationToken.getAuthorities（）用于授权请求，例如在hasRole（'USER'）或hasRoel（'ADMIN'）中。

映射用户权限时有几个选项可供选择：

- 使用GrantedAuthoritiesMapper
- OAuth2UserService基于委派的策略

##### 使用GrantedAuthoritiesMapper

提供GrantedAuthoritiesMapper的实现并对其进行配置，如以下示例所示：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

    @Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .userInfoEndpoint(userInfo -> userInfo
			        .userAuthoritiesMapper(this.userAuthoritiesMapper())
			        ...
			    )
			);
		return http.build();
	}

	private GrantedAuthoritiesMapper userAuthoritiesMapper() {
		return (authorities) -> {
			Set<GrantedAuthority> mappedAuthorities = new HashSet<>();

			authorities.forEach(authority -> {
				if (OidcUserAuthority.class.isInstance(authority)) {
					OidcUserAuthority oidcUserAuthority = (OidcUserAuthority)authority;

					OidcIdToken idToken = oidcUserAuthority.getIdToken();
					OidcUserInfo userInfo = oidcUserAuthority.getUserInfo();

					// Map the claims found in idToken and/or userInfo
					// to one or more GrantedAuthority's and add it to mappedAuthorities

				} else if (OAuth2UserAuthority.class.isInstance(authority)) {
					OAuth2UserAuthority oauth2UserAuthority = (OAuth2UserAuthority)authority;

					Map<String, Object> userAttributes = oauth2UserAuthority.getAttributes();

					// Map the attributes found in userAttributes
					// to one or more GrantedAuthority's and add it to mappedAuthorities

				}
			});

			return mappedAuthorities;
		};
	}
}
```

或者，您可以注册GrantedAuthoritiesMapper@Bean，使其自动应用于配置，如以下示例所示：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
		    .oauth2Login(withDefaults());
		return http.build();
	}

	@Bean
	public GrantedAuthoritiesMapper userAuthoritiesMapper() {
		...
	}
}
```

##### OAuth2UserService基于委派的策略

与使用GrantedAuthoritiesMapper相比，此策略是先进的，但是，它也更灵活，因为它允许您访问OAuth2UserRequest和OAuth2User（当使用OAuth 2.0 UserService时）或OidcUserRequest与OidcUser（在使用OpenID Connect 1.0 UserServices时）。

OAuth2UserRequest（和OidcUserRequest）为您提供了对关联OAuth2AccessToken的访问，这在委托人需要从受保护的资源中获取权限信息，然后才能映射用户的自定义权限的情况下非常有用。

以下示例显示如何使用OpenID Connect 1.0 UserService实施和配置基于委派的策略：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .userInfoEndpoint(userInfo -> userInfo
			        .oidcUserService(this.oidcUserService())
			        ...
			    )
			);
		return http.build();
	}

	private OAuth2UserService<OidcUserRequest, OidcUser> oidcUserService() {
		final OidcUserService delegate = new OidcUserService();

		return (userRequest) -> {
			// Delegate to the default implementation for loading a user
			OidcUser oidcUser = delegate.loadUser(userRequest);

			OAuth2AccessToken accessToken = userRequest.getAccessToken();
			Set<GrantedAuthority> mappedAuthorities = new HashSet<>();

			// TODO
			// 1) Fetch the authority information from the protected resource using accessToken
			// 2) Map the authority information to one or more GrantedAuthority's and add it to mappedAuthorities

			// 3) Create a copy of oidcUser but use the mappedAuthorities instead
			oidcUser = new DefaultOidcUser(mappedAuthorities, oidcUser.getIdToken(), oidcUser.getUserInfo());

			return oidcUser;
		};
	}
}
```

### OAuth 2.0 UserService

DefaultOAuth2UserService是支持标准OAuth 2.0 Provider的OAuth2UserService的实现。

OAuth2UserService从UserInfo端点获取最终用户（资源所有者）的用户属性（通过使用在授权流期间授予客户端的访问令牌），并以OAuth2User的形式返回AuthenticatedPrincipal。

DefaultOAuth2UserService在UserInfo端点请求用户属性时使用RestOperations。

如果需要自定义UserInfo请求的预处理，可以提供DefaultOAuth2UserService.setRequestEntityConverter（），带有自定义Converter<OAuth2UserRequest，RequestEntity<？>>。默认实现OAuth2UserRequestEntityConverter构建UserInfo请求的RequestEntity表示，默认情况下，该表示在Authorization头中设置OAuth2AccessToken。

另一方面，如果需要自定义UserInfo响应的后期处理，则需要提供DefaultOAuth2UserService。带有自定义配置的RestOperations的setRestOperation（）。默认RestOperations配置如下：

```java
RestTemplate restTemplate = new RestTemplate();
restTemplate.setErrorHandler(new OAuth2ErrorResponseErrorHandler());
```


OAuth2ErrorResponseErrorHandler是可以处理OAuth 2.0错误（400错误请求）的Response ErrorHandlder。它使用OAuth2ErrorHttpMessageConverter将OAuth 2.0 Error参数转换为OAuth2错误。

无论您是自定义DefaultOAuth2UserService还是提供自己的OAuth2UserService实现，都需要按以下示例所示进行配置：

```java
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Login(oauth2 -> oauth2
			    .userInfoEndpoint(userInfo -> userInfo
			        .userService(this.oauth2UserService())
			        ...
			    )
			);
		return http.build();
	}

	private OAuth2UserService<OAuth2UserRequest, OAuth2User> oauth2UserService() {
		...
	}
}
```

DSL 提供了许多配置选项，用于自定义 OAuth 2.0 客户端使用的核心组件。此外，授权码允许自定义授权码授予。



下面的代码显示了 DSL 提供的完整配置选项：

```java
@EnableWebSecurity
public class OAuth2ClientSecurityConfig {

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
			.oauth2Client(oauth2 -> oauth2
				.clientRegistrationRepository(this.clientRegistrationRepository())
				.authorizedClientRepository(this.authorizedClientRepository())
				.authorizedClientService(this.authorizedClientService())
				.authorizationCodeGrant(codeGrant -> codeGrant
					.authorizationRequestRepository(this.authorizationRequestRepository())
					.authorizationRequestResolver(this.authorizationRequestResolver())
					.accessTokenResponseClient(this.accessTokenResponseClient())
				)
			);
		return http.build();
	}
}
```

OAuth2AuthorizedClientManager负责与一个或多个OAuth2AauthorizedClientProvider协作管理OAuth 2.0客户端的授权（或重新授权）。

以下代码显示了如何注册OAuth2AuthorizedClientManager@Bean并将其与OAuth2AauthorizedClientProvider组合关联的示例，该组合提供对authorization_code、refresh_token、client_credentials和密码授权授予类型的支持：

```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
		ClientRegistrationRepository clientRegistrationRepository,
		OAuth2AuthorizedClientRepository authorizedClientRepository) {

	OAuth2AuthorizedClientProvider authorizedClientProvider =
			OAuth2AuthorizedClientProviderBuilder.builder()
					.authorizationCode()
					.refreshToken()
					.clientCredentials()
					.password()
					.build();

	DefaultOAuth2AuthorizedClientManager authorizedClientManager =
			new DefaultOAuth2AuthorizedClientManager(
					clientRegistrationRepository, authorizedClientRepository);
	authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

	return authorizedClientManager;
}
```

# Spring Security OAuth2.0



# OAuth2.0协议



# Spring Security OAuth2.0客户端



OAuth 2.0客户端特性为OAuth 2.0授权框架中定义的客户端角色提供支持。



在高层，可用的核心功能包括：



- 授权支持 

- - [Authorization Code](https://tools.ietf.org/html/rfc6749#section-1.3.1)
  - [Refresh Token](https://tools.ietf.org/html/rfc6749#section-6)
  - [Client Credentials](https://tools.ietf.org/html/rfc6749#section-1.3.4)
  - [Resource Owner Password Credentials](https://tools.ietf.org/html/rfc6749#section-1.3.3)
  - [JWT Bearer](https://datatracker.ietf.org/doc/html/rfc7523#section-2.1)

- 客户端认证支持 

- - [JWT Bearer](https://datatracker.ietf.org/doc/html/rfc7523#section-2.2)

- HTTP客户端支持 

- - `WebClient`[ integration for Servlet Environments（请求受保护的资源）](https://docs.spring.io/spring-security/reference/6.0.0-M1/servlet/oauth2/client/authorized-clients.html#oauth2Client-webclient-servlet)



`HttpSecurity.oauth2Client()`DSL提供了许多配置选项，用于定制OAuth 2.0客户端使用的核心组件。此外，`HttpSecurity.oauth2Client().authorizationCodeGrant()自定义`授权码模式。



```java
@EnableWebSecurity
public class OAuth2ClientSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http
			.oauth2Client(oauth2 -> oauth2
				.clientRegistrationRepository(this.clientRegistrationRepository())
				.authorizedClientRepository(this.authorizedClientRepository())
				.authorizedClientService(this.authorizedClientService())
				.authorizationCodeGrant(codeGrant -> codeGrant
					.authorizationRequestRepository(this.authorizationRequestRepository())
					.authorizationRequestResolver(this.authorizationRequestResolver())
					.accessTokenResponseClient(this.accessTokenResponseClient())
				)
			);
	}
}
```



OAuth2AuthorizedClientManager负责与一个或多个OAuth2AuthorizedClientProvider协作管理OAuth 2.0客户端的授权（或重新授权）。



以下代码显示了如何注册OAuth2AuthorizedClientManager @Bean并将其与OAuth2AuthorizedClientProvider组合关联的示例，该组合提供对`authorization_code`, `refresh_token`, `client_credentials`, 和`password`授予类型的支持：



```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
		ClientRegistrationRepository clientRegistrationRepository,
		OAuth2AuthorizedClientRepository authorizedClientRepository) {

	OAuth2AuthorizedClientProvider authorizedClientProvider =
			OAuth2AuthorizedClientProviderBuilder.builder()
					.authorizationCode()
					.refreshToken()
					.clientCredentials()
					.password()
					.build();

	DefaultOAuth2AuthorizedClientManager authorizedClientManager =
			new DefaultOAuth2AuthorizedClientManager(
					clientRegistrationRepository, authorizedClientRepository);
	authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

	return authorizedClientManager;
}
```



## 核心类



### ClientRegistration



ClientRegistration是向OAuth 2.0或OpenID Connect 1.0提供程序注册的客户端的表示。



```java
public final class ClientRegistration {
	private String registrationId;	//表示客户端注册的唯一id
	private String clientId; //客户端id	
	private String clientSecret;//客户端密码
       //Provider验证客户端的方法。
       //支持的值为client_secret_basic、client_secret_post、private_key_jwt、client_secret_jwt和none
	private ClientAuthenticationMethod clientAuthenticationMethod;	
       //OAuth2.0授权框架定义了四种授权授予类型。
       //支持的值包括authorization_code, client_credentials, password
       //以及扩展授权类型urn:ietf:params:oauth:grant-type:jwt-bearer。
	private AuthorizationGrantType authorizationGrantType;	
	//客户端的注册重定向URI，最终用户对客户端进行身份验证和授权访问后，
       // 授权服务器在将最终用户的用户代理重定向到该URI。
        private String redirectUri;	
        //客户端在授权请求流期间请求的范围，例如openid、电子邮件或概要文件。
	private Set<String> scopes;
	
	private ProviderDetails providerDetails;
        //用于客户端的描述性名称。该名称可用于某些场景，
        //例如在自动生成的登录页面中显示客户端名称时。
	private String clientName;	

	public class ProviderDetails {
            //授权服务器的授权端点URI。
		private String authorizationUri;
            //获取token的端点
		private String tokenUri;
              //获取用户信息的端点		
		private UserInfoEndpoint userInfoEndpoint;
		//从授权服务器获取JSON Web Key (JWK) Set，该信息
//包括	验证JSON Web Signature (JWS) ID token的加密key和可选的用户信息
               private String jwkSetUri;	
		//返回OpenID Connect 1.0提供程序或OAuth 2.0授权服务器的颁发者标识符URI。	
                private String issuerUri;
// OPen ID配置信息	
        private Map<String, Object> configurationMetadata;

		public class UserInfoEndpoint {
//用于访问经过身份验证的最终用户的声明和属性的UserInfo端点URI。
			private String uri;	
      //将访问令牌发送到UserInfo端点时使用的身份验证方法。
 /// 支持的值包括header, form, 和 query.。
      private AuthenticationMethod authenticationMethod;
	//UserInfo响应中返回的属性的名称，该响应引用最终用户的名称或标识符。	
	private String userNameAttributeName;	

		}
	}
}
```



ClientRegistration提供了以这种方式配置ClientRegistration的方便方法，如下所示：



```java
ClientRegistration clientRegistration =
    ClientRegistrations.fromIssuerLocation("https://idp.example.com/issuer").build();
```



### ClientRegistrationRepository



ClientRegistrationRepository用作OAuth 2.0/OpenID Connect 1.0 ClientRegistration的存储库。



客户端注册信息最终由关联的授权服务器存储和拥有。此存储库提供检索主客户端注册信息子集的能力，该信息存储在授权服务器中。



Spring Boot2.x绑定`spring.security.oauth2.client.registration.[registrationId]`下的属性到`ClientRegistration` 实例，然后由`ClientRegistrationRepository`组装



ClientRegistration存储库的默认实现是InMemoryClientRegistrationRepository。



自动配置还将ClientRegistrationRepository注册为ApplicationContext中的@Bean，以便在应用程序需要时可用于依赖项注入。



```java
@Controller
public class OAuth2ClientController {

	@Autowired
	private ClientRegistrationRepository clientRegistrationRepository;

	@GetMapping("/")
	public String index() {
		ClientRegistration oktaRegistration =
			this.clientRegistrationRepository.findByRegistrationId("okta");

		...

		return "index";
	}
}
```



### OAuth2AuthorizedClient



OAuth2AuthorizedClient是已授权客户端的表示。当最终用户（资源所有者）已授权客户端访问其受保护的资源时，客户端被视为已授权。



OAuth2AuthorizedClient用于将OAuth2AccessToken（以及可选的OAuth2RefreshToken）关联到ClientRegistration（客户端）和资源所有者，后者是授予授权的主要最终用户。



### OAuth2AuthorizedClientRepository and OAuth2AuthorizedClientService



OAuth2AuthorizedClientRepository 负责在web请求中持久化`OAuth2AuthorizedClient`(s)  ，OAuth2AuthorizedClientservice的主要角色是在应用程序级别管理OAuth2AuthorizedClient。



从开发人员的角度来看，OAuth2AuthorizedClientRepository或OAuth2AuthorizedClientService提供了查找与客户端关联的OAuth2AccessToken的功能，以便可以使用它来访问受保护的资源。



下面是一个实例：



```java
@Controller
public class OAuth2ClientController {

    @Autowired
    private OAuth2AuthorizedClientService authorizedClientService;

    @GetMapping("/")
    public String index(Authentication authentication) {
        OAuth2AuthorizedClient authorizedClient =
            this.authorizedClientService.loadAuthorizedClient("okta", authentication.getName());

        OAuth2AccessToken accessToken = authorizedClient.getAccessToken();

        ...

        return "index";
    }
}
```



OAuth2AuthorizedClientservice的默认实现是InMemoryAuth2AuthorizedClientservice，它在内存中存储OAuth2AuthorizedClient对象。或者，您可以使用JDBCOAuth2AuthorizedClient。



### OAuth2AuthorizedClientManager and OAuth2AuthorizedClientProvider



OAuth2AuthorizedClient管理器负责OAuth2AuthorizedClient的总体管理。主要职责包括：



- 使用OAuth2AuthorizedClientProvider授权（或重新授权）OAuth 2.0客户端。
- 存储OAuth2AuthorizedClient，通常委托给`OAuth2AuthorizedClientService` or `OAuth2AuthorizedClientRepository`.
- 当OAuth 2.0客户端已成功授权（或重新授权）时，委托给OAuth2AuthorizationSuccessHandler。
- 当OAuth 2.0客户端无法授权（或重新授权）时，委托给OAuth2AuthorizationFailureHandler。



OAuth2AuthorizedClientProvider实现了授权（或重新授权）OAuth2.0客户端的策略。实现通常实现授权授予类型，例如`authorization_code`, `client_credentials`等。



OAuth2AuthorizedClientManager的默认实现是DefaultOAuth2AuthorizedClientManager，它与OAuth2AuthorizedClientProvider关联，OAuth2AuthorizedClientManager可以使用基于委托的组合支持多种授权授予类型。您可以使用OAuth2AuthorizedClientProviderBuilder来配置和构建基于委托的组合。



以下代码显示了如何配置和构建OAuth2AuthorizedClient Provider组合的示例，该组合提供对`authorization_code`, `refresh_token`, `client_credentials`, and `password`的支持：



```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
		ClientRegistrationRepository clientRegistrationRepository,
		OAuth2AuthorizedClientRepository authorizedClientRepository) {

	OAuth2AuthorizedClientProvider authorizedClientProvider =
			OAuth2AuthorizedClientProviderBuilder.builder()
					.authorizationCode()
					.refreshToken()
					.clientCredentials()
					.password()
					.build();

	DefaultOAuth2AuthorizedClientManager authorizedClientManager =
			new DefaultOAuth2AuthorizedClientManager(
					clientRegistrationRepository, authorizedClientRepository);
	authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

	return authorizedClientManager;
}
```



当授权成功时，`DefaultOAuth2AuthorizedClientManager` 委托给OAuth2AuthorizationSuccessHandler 进行后续处理 。 OAuth2AuthorizationSuccessHandler 默认使用 OAuth2AuthorizedClientRepository 保存`OAuth2AuthorizedClient` 。



在重新授权失败的情况下（例如，刷新令牌不再有效），RemoveAuthorizedClientOAuth2AuthorizationFailureHandler 会将先前保存的`OAuth2AuthorizedClient 从` `OAuth2AuthorizedClientRepository` 移除。



你可以使用`setAuthorizationSuccessHandler(OAuth2AuthorizationSuccessHandler)` 和`setAuthorizationFailureHandler(OAuth2AuthorizationFailureHandler)` 自定义这些handler。



DefaultOAuth2AuthorizedClientManager还与类型为Function<OAuth2AuthorizeRequest，Map<String，Object>>的contextAttributesMapper相关联，后者负责将属性从OAuth2AuthorizeRequest映射到要关联到OAuth2AuthorizationContext。当您需要为OAuth2AuthorizedClientProvider提供所需（支持的）属性时，这非常有用，例如，OAuth2AuthorizedClientProvider要求资源所有者的用户名和密码在OAuth2AuthorizationContext.getAttributes（）中可用。



```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
		ClientRegistrationRepository clientRegistrationRepository,
		OAuth2AuthorizedClientRepository authorizedClientRepository) {

	OAuth2AuthorizedClientProvider authorizedClientProvider =
			OAuth2AuthorizedClientProviderBuilder.builder()
					.password()
					.refreshToken()
					.build();

	DefaultOAuth2AuthorizedClientManager authorizedClientManager =
			new DefaultOAuth2AuthorizedClientManager(
					clientRegistrationRepository, authorizedClientRepository);
	authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

	// Assuming the `username` and `password` are supplied as `HttpServletRequest` parameters,
	// map the `HttpServletRequest` parameters to `OAuth2AuthorizationContext.getAttributes()`
	authorizedClientManager.setContextAttributesMapper(contextAttributesMapper());

	return authorizedClientManager;
}

private Function<OAuth2AuthorizeRequest, Map<String, Object>> contextAttributesMapper() {
	return authorizeRequest -> {
		Map<String, Object> contextAttributes = Collections.emptyMap();
		HttpServletRequest servletRequest = authorizeRequest.getAttribute(HttpServletRequest.class.getName());
		String username = servletRequest.getParameter(OAuth2ParameterNames.USERNAME);
		String password = servletRequest.getParameter(OAuth2ParameterNames.PASSWORD);
		if (StringUtils.hasText(username) && StringUtils.hasText(password)) {
			contextAttributes = new HashMap<>();

			// `PasswordOAuth2AuthorizedClientProvider` requires both attributes
			contextAttributes.put(OAuth2AuthorizationContext.USERNAME_ATTRIBUTE_NAME, username);
			contextAttributes.put(OAuth2AuthorizationContext.PASSWORD_ATTRIBUTE_NAME, password);
		}
		return contextAttributes;
	};
}
```



DefaultOAuth2AuthorizedClientManager设计用于HttpServletRequest的上下文中。在HttpServletRequest上下文之外操作时，请改用AuthorizedClientServiceOAuth2AuthorizedClientManager。



无交互的后台程序经常使用AuthorizedClientServiceOAuth2AuthorizedClientManager。这种程序经常使用`client_credentials` 授权模式，下面是使用举例：



```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
		ClientRegistrationRepository clientRegistrationRepository,
		OAuth2AuthorizedClientService authorizedClientService) {

	OAuth2AuthorizedClientProvider authorizedClientProvider =
			OAuth2AuthorizedClientProviderBuilder.builder()
					.clientCredentials()
					.build();

	AuthorizedClientServiceOAuth2AuthorizedClientManager authorizedClientManager =
			new AuthorizedClientServiceOAuth2AuthorizedClientManager(
					clientRegistrationRepository, authorizedClientService);
	authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

	return authorizedClientManager;
}
```



# OAuth 2.0 Resource Server



Spring Security 支持使用两种形式的 OAuth 2.0 Bearer Tokens 来保护端点：



-  [JWT](https://tools.ietf.org/html/rfc7519) 
-  Opaque Tokens
  这在应用程序将其权限管理委托给授权服务器（例如，Okta 或 Ping Identity）的情况下很方便。 资源服务器可以咨询此授权服务器以授权请求。
  让我们看看 Bearer Token Authentication 在 Spring Security 中是如何工作的。 首先，我们看到，与基本身份验证一样，WWW-Authenticate 标头被发送回未经身份验证的客户端。 



![img](../../../App/Typora/images/eStgFkMPuOzjFBPtJ6ygpT4j3gFAUp4uCnSgcVPRmbg.png)



1. 首先，用户向需要授权的资源/private 发出未经身份验证的请求。
2. Spring Security 的 FilterSecurityInterceptor 指示抛出 AccessDeniedException， 拒绝未经身份验证的请求。
3. 由于用户未通过身份验证，因此 ExceptionTranslationFilter 会启动 Start Authentication。 配置的 AuthenticationEntryPoint 是 BearerTokenAuthenticationEntryPoint 的一个实例，它发送 WWW-Authenticate 标头。 RequestCache 通常是一个不保存请求的 NullRequestCache，因为客户端能够重放它最初请求的请求。



当客户端收到 WWW-Authenticate: Bearer 标头时，它知道应该使用bearer令牌重试。 以下是正在处理的bearer令牌的流程:



![img](../../../App/Typora/images/-lLq5leI6K7lT7_qDsDVsQC_fL6FvPWHjs4aEYEGn2c.png)



1. 当用户提交其持有者令牌时，BearerTokenAuthenticationFilter 通过从 HttpServletRequest 中提取令牌来创建一个 BearerTokenAuthenticationToken，这是一种身份验证。
2. 接下来，HttpServletRequest 被传递给 AuthenticationManagerResolver，它选择 AuthenticationManager。 BearerTokenAuthenticationToken 被传入 AuthenticationManager 进行认证。 AuthenticationManager 的详细信息取决于您是否配置了 JWT 或[opaque token](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/opaque-token.html#oauth2resourceserver-opaque-minimalconfiguration).。
3. 如果认证失败： 

1. 1. SecurityContextHolder 被清除。
   2. 调用 AuthenticationEntryPoint 以触发再次发送 WWW-Authenticate 标头。

1. 如果认证成功： 

1. 1. Authentication 在 SecurityContextHolder 上设置。
   2. BearerTokenAuthenticationFilter 调用 FilterChain.doFilter(request,response) 以继续应用程序逻辑的其余部分。
