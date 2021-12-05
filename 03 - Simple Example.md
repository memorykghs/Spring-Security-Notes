# 03 - Simple Example
接下來我們先建立一個最簡單的例子，並觀察 Spring Security 的架構與驗證的過程。先在 `pom.xml` 加入依賴。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

在專案中建立 Spring Security 要用的 Config。
```
|--com.security.springSecurityExample.config
  |--WebSecurityConfig.java
```

* `WebSecurityConfig.java`
```java
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests()
		.antMatchers("/", "/login").permitAll()
			.anyRequest()
			.authenticated()
			.and()
		.formLogin()
			.failureUrl("/error")
			.and();
	}
}
```




* `WebSecurityConfig.java`
```java
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests()
		.antMatchers("/", "/test", "/login").permitAll()
			.anyRequest()
			.authenticated()
			.and()
		.formLogin()
			.defaultSuccessUrl("https:///google.com")
			.failureUrl("/error")
			.and();
	}
}
```