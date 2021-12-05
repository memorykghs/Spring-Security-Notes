# 01 - Spring Security 簡介與架構 

Spring Security 通過使用標準 Servlet 與 Servlet Container 集成 Filter，代表他可以應用於 Servlet 容器為基礎開發的任何應用程序上。也就是說不需要適用 Spring 開發的環境也可以使用。

* 利用 Servlet Filter 過濾目標 Request。

## 目的
* Spring 底下的一個框架，用來來幫助實現 Java 應用程式的驗證 ( Authentication ) 和授權 ( Authorization )。
* 可以搭配第三方資源 OAuth 進行驗證與授權。

總之，就是驗證與授權!<br/>
![](/images/就是這樣.png)

## Spring Security 架構
由於 Spring Security 事可以在 Servlet 的架構上支援 MVC Servlet Filters 的，所以我們先大致了解一下 Http Request 是怎麼被處理的，對於之後理解 Spring Security 架構也有幫助。
<br/>

### Servlet MVC Filter Chain
![](/images/1-1.png)

客戶端向應用程式發送請求，容器 ( Web Container ) 會創建一個包含許多設定好的 `Filter` 的 `FilterChain`，並依照請求的目的 `url` 處理。在 Spring MVC 中，`Servlet` 是 `DispatcherSerlet` 的實例，一個 `Servlet` 最多只能處理一組 `HttpServletRequest` 和 `HttpServletResponse`，而 `Filter` 可以設定多個並檢查傳入的資訊。

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// do something before the rest of the application
    chain.doFilter(request, response); // invoke the rest of the application
    // do something after the rest of the application
}
```
`Filter` 在進行過濾的時候會呼叫 `doFilter` 方法，而這些要使用的 `Filter` 都會在 `FilterChain` 當中，要往下繼續過濾，使用 `chain.doFilter()` 方法即可。由於 `Filter` 是一層一層往下過濾的，所以順序就相對地非常重要。

---

### DelegatingFilterProxy

`Servlet` 容器環境允許使用自訂的標準註冊 `Filter`，但是 Spring 有自己的 Container 與 Bean，所以在一般情況下 `Servlet` 不知道 Spring 的 Beans 有哪些。所以 Spring 提供了一個 `Filter` 實例 `DelegatingFilterProxy`，它可以在 `Servlet` 容器的生命週期與 Spring ApplicationContext 中橋接，這樣一來就可以將內部的過濾邏輯交由 Spring Bean 處理。<pr/>

![](/images/1-2.png)

`DelegatingFilterProxy` 會從 `ApplicationContext` 中查找要過濾的 Bean Filter ( 此處例子為 Bean Filter0 )，然後調用它。

> `DelegatingFilterProxy Pseudo Code`
```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// Lazily get Filter that was registered as a Spring Bean
	// For the example in DelegatingFilterProxy 
delegate
 is an instance of Bean Filter0
	Filter delegate = getFilterBean(someBeanName);
	// delegate work to the Spring Bean
	delegate.doFilter(request, response);
}
```
---

### FilterChainProxy
`FilterChainProxy` 是 Spring Security 提供的一個特殊的 `Filter`，可以透過 `SecurityFilterChain` 將請求委派給多個 `Filter` 實例。而 `FilterChainProxy` 本身是一個 Bean，所以會被包裝在 `DelegatingFilterProxy` 中。<br/>

![](/images/1-3.png)

`FilterChainProxy` 在確定何時應該調用 `SecurityFilterChain` 方面提供了更大的靈活性。 在 Servlet 容器中，僅根據 URL 調用過濾器。但是，`FilterChainProxy` 可以利用 `RequestMatcher` 接口根據 `HttpServletRequest` 中的任何內容確定調用。

---

##### 好處
* 允許延遲查找 Filterbean 實例，因為容器需要Filter在容器啟動之前註冊實例。
* Spring 通常使用 ContextLoaderListener 來加載 Spring Bean，所以要等 Filter 實例註冊完成後才會執行。

### SecurityFilterChain
`FilterChainProxy` 使用 `SecurityFilterChain` 來決定此請求要調用哪些 Spring Filters，`SecurityFilterChain` 中會有屬性紀錄要用的 `Filter`，預設實作 `SecurityFilterChain` 的類別是 `DefaultSecurityFilterChain`，程式碼如下：

```java
public final class DefaultSecurityFilterChain implements SecurityFilterChain {

	private static final Log logger = LogFactory.getLog(DefaultSecurityFilterChain.class);

	private final RequestMatcher requestMatcher;

	private final List<Filter> filters;

	public DefaultSecurityFilterChain(RequestMatcher requestMatcher, Filter... filters) {
		this(requestMatcher, Arrays.asList(filters));
	}

	public DefaultSecurityFilterChain(RequestMatcher requestMatcher, List<Filter> filters) {
		logger.info(LogMessage.format("Will secure %s with %s", requestMatcher, filters));
		this.requestMatcher = requestMatcher;
		this.filters = new ArrayList<>(filters);
	}
	...
	...
}
```
<br/>

![](/images/1-4.png)

由於 `FilterChainProxy` 是 Spring Security 使用的核心，它可以清除 `SecurityContext` 以避免內存洩漏，或是使用 Spring Security 的 `HttpFirewall` 來保護應用程序免受某些類型的攻擊。
<br/>

下面的圖代表說。Spring Security 可以有多個 `SecurityFilterChain`，`FilterChainProxy` 會調用第一個符合 URL 的 `SecurityFilterChain`，例如請求 `/api/messages` 的 URL ，它會先交由第一個匹配到的 `SecurityFilterChain 0` 而不是 `SecurityFilterChain n`。

但如果是 `/messages`，`FilterChainProxy` 會一直嘗試匹配每個 `SecurityFilterChain`，如果沒有符合的話，`SecurityFilterChain n` 就會被調用。<br/>
![](/images/1-5.png)

另外，每個 `SecurityFilterChain` 內可以有不等數量的 `Filter`，當有東西不想要被過濾的時候，也可以建立一個沒有 `Filter` 的 `SecurityFilterChain`。

## Security Filters
以下提供一些 `SecurityFilterChain` API 插入 `FilterChain` 中的 Security Filters，順序可以大概有個印象就好：
* ChannelProcessingFilter
* WebAsyncManagerIntegrationFilter
* SecurityContextPersistenceFilter
* HeaderWriterFilter
* CorsFilter
* CsrfFilter
* LogoutFilter
* OAuth2AuthorizationRequestRedirectFilter
* Saml2WebSsoAuthenticationRequestFilter
* X509AuthenticationFilter
* AbstractPreAuthenticatedProcessingFilter
* CasAuthenticationFilter
* OAuth2LoginAuthenticationFilter
* Saml2WebSsoAuthenticationFilter
* [UsernamePasswordAuthenticationFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html#servlet-authentication-usernamepasswordauthenticationfilter)
* OpenIDAuthenticationFilter
* DefaultLoginPageGeneratingFilter
* DefaultLogoutPageGeneratingFilter
* ConcurrentSessionFilter
* [DigestAuthenticationFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/digest.html#servlet-authentication-digest)
* BearerTokenAuthenticationFilter
* [BasicAuthenticationFilter](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html#servlet-authentication-basic)
* RequestCacheAwareFilter
* SecurityContextHolderAwareRequestFilter
* JaasApiIntegrationFilter
* RememberMeAuthenticationFilter
* AnonymousAuthenticationFilter
* OAuth2AuthorizationCodeGrantFilter
* SessionManagementFilter
* [ExceptionTranslationFilter](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-exceptiontranslationfilter)
* [FilterSecurityInterceptor](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-requests.html#servlet-authorization-filtersecurityinterceptor)
* SwitchUserFilter

## 小結
![](/images/1-6.png)

## Dependency
總之要之前，除了一些 Spring Boot 的依賴，再多注入這個就可以了。
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

