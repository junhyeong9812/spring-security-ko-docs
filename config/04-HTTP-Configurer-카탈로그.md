# 04. HTTP Configurer 카탈로그

## 무엇을 / 왜

`http.formLogin(...)`, `http.csrf(...)`, `http.authorizeHttpRequests(...)` 의 실제 일은 각각의 **Configurer** 가 한다. Configurer는 [01장](01-빌더와-Configurer-기반구조.md)의 `SecurityConfigurer` 구현체로서, "한 가지 보안 기능에 필요한 필터(들)를 만들어 `HttpSecurity`의 올바른 위치에 끼우는" 전략 객체다. 이 장은 그 공통 베이스(`AbstractHttpConfigurer`)와 대표 Configurer들이 `init/configure`에서 무엇을 하는지를 본다.

`config.annotation.web.configurers` 패키지에는 25개가 넘는 Configurer가 있다. 전부 나열하는 대신, **유형별 대표 한둘**을 깊게 본다.

## 핵심 타입

```
   SecurityConfigurerAdapter<DefaultSecurityFilterChain, B>
        ▲
   AbstractHttpConfigurer<T, B>          disable(), getSecurityContextHolderStrategy(),
        ▲                                 getRequestMatcherBuilder() 등 공통 헬퍼
        │
        ├── AuthorizeHttpRequestsConfigurer   → AuthorizationFilter (인가)
        ├── CsrfConfigurer                    → CsrfFilter
        ├── ExceptionHandlingConfigurer       → ExceptionTranslationFilter
        ├── SessionManagementConfigurer       → SessionManagementFilter 류
        ├── LogoutConfigurer                  → LogoutFilter
        ├── HeadersConfigurer                 → HeaderWriterFilter
        │
        └── AbstractAuthenticationFilterConfigurer<B,T,F>   "로그인" 류 공통 베이스
                 ▲
                 ├── FormLoginConfigurer        → UsernamePasswordAuthenticationFilter
                 ├── OAuth2LoginConfigurer      → OAuth2LoginAuthenticationFilter
                 └── Saml2LoginConfigurer, OneTimeTokenLoginConfigurer ...
```

- 소스: `config/src/main/java/org/springframework/security/config/annotation/web/configurers/AbstractHttpConfigurer.java`
- 소스: `.../configurers/AbstractAuthenticationFilterConfigurer.java`
- 소스: `.../configurers/FormLoginConfigurer.java`
- 소스: `.../configurers/AuthorizeHttpRequestsConfigurer.java`

### AbstractHttpConfigurer — 공통 베이스

```java
public abstract class AbstractHttpConfigurer<T extends AbstractHttpConfigurer<T, B>, B extends HttpSecurityBuilder<B>>
        extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, B> {

    public B disable() {                            // csrf(c -> c.disable())
        getBuilder().removeConfigurer(getClass());
        return getBuilder();
    }
    protected SecurityContextHolderStrategy getSecurityContextHolderStrategy() { ... } // 공유/빈 조회
    protected PathPatternRequestMatcher.Builder getRequestMatcherBuilder() {
        return getBuilder().getSharedObject(PathPatternRequestMatcher.Builder.class);
    }
}
```

`disable()`이 흥미롭다. "이 기능을 끈다"는 곧 "내 Configurer를 빌더에서 제거한다"이다. 그러면 `build()` 때 그 Configurer의 `configure()`가 호출되지 않아 해당 필터가 만들어지지 않는다. 매우 단순하지만 일관된 메커니즘이다.

### getOrApply — 같은 Configurer 누적

`HttpSecurity`의 각 DSL 메서드는 `getOrApply`로 Configurer를 가져온다.

```java
private <C extends SecurityConfigurerAdapter<...>> C getOrApply(C configurer) {
    C existingConfig = (C) getConfigurer(configurer.getClass());
    if (existingConfig != null) return existingConfig;   // 이미 있으면 재사용
    return apply(configurer);                            // 없으면 새로 등록
}
```

그래서 `http.formLogin(c -> c.loginPage("/a")).formLogin(c -> c.loginProcessingUrl("/b"))`처럼 나눠 호출해도 **하나의 `FormLoginConfigurer`** 에 설정이 누적된다.

## 동작 흐름: 폼 로그인 Configurer 가 필터를 끼우기까지

`AbstractAuthenticationFilterConfigurer`는 "필터 하나로 로그인을 처리하는" 모든 Configurer의 공통 골격이다. `init`과 `configure` 두 단계를 본다.

```
build() 진행 중
  │
  ├─ init 단계 (모든 Configurer.init 먼저)
  │    AbstractAuthenticationFilterConfigurer.init(http)
  │      ├ updateAuthenticationDefaults()        // loginProcessingUrl 기본값 등
  │      ├ updateAccessDefaults()
  │      └ registerDefaultAuthenticationEntryPoint(http)
  │           └ ExceptionHandlingConfigurer 의 공유 EntryPoint 맵에
  │             LoginUrlAuthenticationEntryPoint 등록  ← 미인증 시 로그인 페이지로
  │
  └─ configure 단계
       AbstractAuthenticationFilterConfigurer.configure(http)
         ├ authFilter.setAuthenticationManager(
         │     http.getSharedObject(AuthenticationManager.class))   ← 05장에서 만든 매니저
         ├ authFilter.setAuthenticationSuccessHandler(successHandler)
         ├ authFilter.setAuthenticationFailureHandler(failureHandler)
         ├ authFilter.setSecurityContextRepository(...)
         ├ F filter = postProcess(this.authFilter)   ← 빈 초기화/Aware (01장)
         └ http.addFilter(filter)                    ← FilterOrderRegistration 이 위치 결정 (03장)
```

코드의 핵심 두 토막:

```java
// init: 미인증 요청을 어디로 보낼지 ExceptionHandling 에 등록
protected final void registerDefaultAuthenticationEntryPoint(B http) {
    ExceptionHandlingConfigurer<B> exceptionHandling = http.getConfigurer(ExceptionHandlingConfigurer.class);
    if (exceptionHandling == null) return;
    exceptionHandling.defaultAuthenticationEntryPointFor(
        postProcess(this.authenticationEntryPoint), getAuthenticationEntryPointMatcher(http));
}

// configure: 필터를 완성해 끼운다
public void configure(B http) {
    ...
    this.authFilter.setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));
    this.authFilter.setAuthenticationSuccessHandler(this.successHandler);
    this.authFilter.setAuthenticationFailureHandler(this.failureHandler);
    ...
    F filter = postProcess(this.authFilter);
    http.addFilter(filter);
}
```

여기서 **Configurer 간 협력**이 잘 드러난다. 로그인 Configurer는 자기 필터만 만드는 게 아니라, `ExceptionHandlingConfigurer`의 공유 상태에 "미인증이면 로그인 페이지로"라는 EntryPoint를 등록한다. 이것이 [01장](01-빌더와-Configurer-기반구조.md)에서 말한 "init에서 공유 상태, configure에서 속성 확정"의 실제 사례다.

`FormLoginConfigurer`는 이 베이스에 `UsernamePasswordAuthenticationFilter`와 폼 기본값(`/login` 처리 URL, username/password 파라미터)만 채운 얇은 서브클래스다.

```java
public final class FormLoginConfigurer<H extends HttpSecurityBuilder<H>>
        extends AbstractAuthenticationFilterConfigurer<H, FormLoginConfigurer<H>, UsernamePasswordAuthenticationFilter> {
    public FormLoginConfigurer() {
        super(new UsernamePasswordAuthenticationFilter(), null);
        usernameParameter("username");
        passwordParameter("password");
    }
}
```

## 인가 Configurer 의 특수성

`AuthorizeHttpRequestsConfigurer`는 로그인 류와 달리 **요청 URL → 권한 규칙**을 누적해 마지막에 `AuthorizationFilter` 하나를 만든다.

```java
public final class AuthorizeHttpRequestsConfigurer<H extends HttpSecurityBuilder<H>>
        extends AbstractHttpConfigurer<...> {
    private final AuthorizationManagerRequestMatcherRegistry registry;
    private final AuthorizationManagerFactory<? super RequestAuthorizationContext> authorizationManagerFactory;
    ...
}
```

`http.authorizeHttpRequests(a -> a.requestMatchers("/admin/**").hasRole("ADMIN").anyRequest().authenticated())`에서 각 `requestMatchers(...).hasRole(...)`은 `registry`에 (매처, `AuthorizationManager`) 한 쌍을 더한다. `configure()` 때 이 매핑들로 `RequestMatcherDelegatingAuthorizationManager`를 만들고, 그것을 품은 `AuthorizationFilter`를 체인 끝부분에 추가한다. `AuthorizationManagerFactory`/`DefaultAuthorizationManagerFactory`는 `hasRole`/`authenticated` 같은 규칙을 실제 `AuthorizationManager`로 변환하는 책임을 진다(상세 인가 로직은 [../core/README.md](../core/README.md)).

이 인가 매핑은 [03장](03-HttpSecurity-와-WebSecurity.md)에서 본 대로 `WebSecurity`가 다시 긁어모아 `WebInvocationPrivilegeEvaluator`를 구성하는 데도 재사용된다.

## 핵심 메서드

- **`init(B)`** — 공유 객체(EntryPoint 맵 등)에 등록만. 다른 Configurer가 참조할 것을 준비.
- **`configure(B)`** — 필터 생성 → `postProcess` → `http.addFilter`. 실제 산출.
- **`postProcess(filter)`** — `SecurityConfigurerAdapter`가 보유한 `CompositeObjectPostProcessor`로 위임해 빈 초기화/Aware/autowire 보장.
- **`disable()`** — `removeConfigurer(getClass())`로 기능 끄기.

## 설계 포인트 / 확장점

- **기능 = Configurer 1개**: 추가/제거/교체가 객체 단위. `csrf(c -> c.disable())`이 곧 Configurer 제거.
- **상속 계층으로 중복 제거**: 로그인 류는 `AbstractAuthenticationFilterConfigurer`가 인증 매니저 주입·성공/실패 핸들러·EntryPoint 등록의 공통부를 처리하고, 서브클래스는 필터와 기본값만.
- **Configurer 간 협력은 공유 객체로**: 로그인 Configurer ↔ ExceptionHandlingConfigurer가 EntryPoint를 공유. 직접 참조 대신 칠판(shared object)을 통한 느슨한 결합.
- **확장점**: `AbstractHttpConfigurer`를 상속해 커스텀 Configurer를 만들고 `http.with(myConfigurer, customizer)`로 끼운다. 기존 Configurer의 핸들러/저장소는 DSL 메서드로 교체.

## 정리

각 보안 기능은 하나의 Configurer로 표현되며, 모두 `AbstractHttpConfigurer`(→`SecurityConfigurerAdapter`) 위에 선다. `init`에서 공유 상태를 준비하고 `configure`에서 필터를 만들어 `postProcess` 후 `http.addFilter`로 끼운다 — 위치는 `FilterOrderRegistration`이 결정한다. 로그인 류는 `AbstractAuthenticationFilterConfigurer`로 인증 매니저 주입과 EntryPoint 협력을 공유한다. 그 인증 매니저가 어떻게 만들어지는지가 다음 장이다.
