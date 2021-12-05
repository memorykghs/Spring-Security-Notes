# 02 - Authentication 架構

以下是一些比較常見的驗證機制：
* [Username and Password](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/index.html#servlet-authentication-unpwd) - how to authenticate with a username/password

* [OAuth 2.0 Logi](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/index.html#oauth2login) - OAuth 2.0 Log In with OpenID Connect and non-standard OAuth 2.0 Login (i.e. GitHub)

* SAML 2.0 Login - SAML 2.0 Log In

* Central Authentication Server (CAS) - Central Authentication Server (CAS) Support

* [Remember Me](https://docs.spring.io/spring-security/reference/servlet/authentication/rememberme.html#servlet-rememberme) - how to remember a user past session expiration

* [JAAS Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/jaas.html#servlet-jaas) - authenticate with JAAS

* OpenID - OpenID Authentication (not to be confused with OpenID Connect)

* [Pre-Authentication Scenarios](https://docs.spring.io/spring-security/reference/servlet/authentication/preauth.html#servlet-preauth) - authenticate with an external mechanism such as SiteMinder or Java EE security but still use Spring Security for authorization and protection against common exploits.

* X509 Authentication - X509 Authentication

上面有用到再去看就好，接下來我們要看一些 Spring Security 中驗證機制會提到的名詞。

## Authentication Architecture
* `SecurityContextHolder` - Spring Security 儲存身分驗證訊息的地方。
<br/>

* `SecurityContext` - 從 SecurityContextHolder 獲得併包含當前經過身份驗證的用戶的身份驗證 ( Authentication )。
<br/>

* `Authentication` - 可以是 AuthenticationManager 的傳入、包含已經驗證過的用戶資訊憑證的物件，或是由 `SecurityContext` 提供的當前用戶的資訊。
<br/>

* `GrantedAuthority` - 授予用戶權限的身份驗證權限物件 ( 角色、範圍等 )。
<br/>

* `AuthenticationManager` - 定義 Spring Security 的過濾器如何執行身份驗證的 API，是一個介面。
<br/>

* `ProviderManager` - `AuthenticationManager` 最常見的實例類別。
<br/>

* `AuthenticationProvider` - 存於 `ProviderManager` 中，實際執行特定類型身份驗證物件。
<br/>

* Request Credentials with `AuthenticationEntryPoint` - 用於從客戶端請求憑據 ( 即重定向到登錄頁面、發送 WWW-Authenticate 響應等 )。
<br/>

* `AbstractAuthenticationProcessingFilter` - 用於身份驗證的基本過濾器。 這也很好地了解了高級身份驗證流程以及各個部分如何協同工作。

![](/images/2-2.png)

## SecurityContextHolder
`SecurityContextHolder` 是 Spring Security 驗證模型的核心，Spring Security 會將驗證過的使用者資訊 ( `Principle` ) 儲存於 `SecurityContextHolder`。Spring Security 不關心 `SecurityContextHolder` 是如何被初始化及建立的，只要裡面有值，就會被認為是最近通過的驗證的使用者資訊 ( by ThreadLocal，該條執行緒 )。所以要讓使用者被視為已通過驗證，最快的方式就是直接設定 `SecurityContextHolder`。

如何設定詳情洽[官網](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html) ☆⌒(≧▽​° )。<br/>
![](/images/2-1.png)

默認情況下，Spring Security 是使用不同的 ThreadLocal 去判定不同使用者的
，並確保每次新的 ThreadLocal 進來都會將 `SecurityContextHolder` 清空，確保使用者資訊的安全。不過有些應用程式並不完全適用 ThreadLocal 機制，例如 Swing 的客戶端。

## SecurityContext
是一個從 `SecurityContextHolder` 中獲得，並且包含 `Authentication` 物件的對象。

## Authentication
在 Spring Security 中可以分為兩塊：
1. 傳入 `AuthenticationManager` 並提供使用者的 `credentials` ( 通常是密碼 )。
2. 提供當前執行緒的使用者的資料

而 `Authentication` 中帶有的資訊又可以細分為：
* `principle` - 識別使用者的物件，代表一位使用者。如果是使用一般的帳號密碼驗證，通常用的會是 `UserDetails` 物件。
<br/>

* `credentials` - 通常是密碼，在使用者通過身分驗證後會被清除，以避免外洩。
<br/>

* `authorities` - 由 `Authentication` 的 `getAuthorities()` 方法回傳 `GrantedAuthorities` 集合。`GrantedAuthority` 是用來授予使用者權限 ( `principle` ) 的物件，默認情況下，使用帳號密碼驗證的部分會由 `UserDetailsService` 負責提供 `GrantedAuthorities`。

## GrantedAuthority
用來授予使用者權限 ( `principle` ) 的物件，可以由 `Authentication` 的 `getAuthorities()` 方法回傳取得。此方法會回傳一個 `GrantedAuthority` 的集合物件。比較常見是確認使用者的權限，例如 `ROLE_ADMINISTRATOR` 或是 `ROLE_HR_SUPERVISOR` 等等。

`GrantedAuthority` 的作用範圍是整個應用程式 ( application-wide )，在整個應用程式環境中，不太可能為每一個使用者建立特定的物件去紀錄他的權限，這樣內存的消耗非常的高。

## AuthenticationManager
`AuthenticationManager` 是定義 Spring Security 的 `Filter` 如何執行身份驗證的 API。不做任何設定、默認的情況下，`ProviderManager` 是最常見的實例物件，負責調用 `AuthenticationProvider` 集合物件。`ProviderManager` 也可以設定 Parent AuthenticationManager，當沒有 `AuthenticationProvider` 可以提供驗證流程時，就由 Parent AuthenticationManager 提供驗證方法。

`ProviderManager` 也也可以設定相同的 Parent AuthenticationManager，因為可能有多個 SecurityFilterChain 物件會使用相同的驗證方法。<br/>

![](/images/2-3.png)

## ProviderManager
是 `AuthenticationManager` 最常見了實例物件，`ProviderManager` 內存有 `AuthenticationProvider` 集合物件，並將驗證委派給 `AuthenticationProvider` 執行。

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

    private static final Log logger = LogFactory.getLog(ProviderManager.class);

    private AuthenticationEventPublisher eventPublisher = new NullEventPublisher();

    private List<AuthenticationProvider> providers = Collections.emptyList();

    protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();

    private AuthenticationManager parent;

    private boolean eraseCredentialsAfterAuthentication = true;
    ...
    ...
}
```

![](/images/2-4.png)

每個 `AuthenticationProvider` 都會給出在這關是否驗證成功、失敗或無法決定的結果，供下游的 `AuthenticationProvider` 參考。如果沒有 `AuthenticationProvider` 可以進行驗證，則驗證失敗並顯示 `ProviderNotFoundException`，它是一個特殊的 `AuthenticationProvider`，代表 `ProviderManager` 沒有支援符合的身分驗證邏輯。

`ProviderManager` 預設會把從 `Authentication` 拿到的 `credentials` 移除，避免資料外洩。但若 `Authentication` 含有做為快取的物件參考，當移除 `credentials` 時，快取就擁有沒有值可以使用。

因此有兩個解決辦法：
1. 在實現快取的物件或是 `AuthenticationProvider` 中直接複製`Authentication`。
2. 將 `eraseCredentialsAfterAuthentication` 設定為 `disable`。

## AuthenticationProvider
`ProviderManager` 中含有 `AuthenticationProviders` 的物件集合，每個 `AuthenticationProvider` 可以執行特定類型的身份驗證。例如，`DaoAuthenticationProvider` 支援基於帳號密碼的身份驗證，而 `JwtAuthenticationProvider` 支援對 JWT Token 進行身份驗證。

## Request Credentials with AuthenticationEntryPoint

## AbstractAuthenticationProcessingFilter

## 參考
* https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html
* [MP治療師的筆記](https://github.com/ChiungjuCheng/SpringSecurity/blob/main/0_%E5%8E%9F%E7%90%86/0_2_Authentication%E6%9E%B6%E6%A7%8B.md)