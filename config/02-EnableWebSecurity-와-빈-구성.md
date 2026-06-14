# 02. @EnableWebSecurity 와 빈 구성

## 무엇을 / 왜

`@EnableWebSecurity` 한 줄(혹은 Spring Boot 자동설정)이 어떻게 동작 가능한 보안 필터를 띄우는가? 이 장은 애너테이션이 import하는 `@Configuration`들과, 최종적으로 서블릿 컨테이너에 등록되는 `springSecurityFilterChain` 필터 빈이 만들어지는 경로를 추적한다. [01장](01-빌더와-Configurer-기반구조.md)의 빌더 엔진이 "어떤 빈으로부터, 어떤 순서로" 구동되는지를 보는 장이다.

## 핵심 타입

```
@EnableWebSecurity
   │  @Import({...})
   ├─▶ WebSecurityConfiguration       ── WebSecurity 를 돌려 FilterChainProxy 빈 생성
   ├─▶ HttpSecurityConfiguration      ── prototype HttpSecurity 빈 제공
   ├─▶ SpringWebMvcImportSelector     ── MVC 있으면 MVC 연동 설정 추가
   ├─▶ OAuth2ImportSelector           ── oauth2-client 클래스패스 있으면 추가
   └─▶ ObservationImportSelector      ── micrometer 있으면 관측 설정 추가
   +  @EnableGlobalAuthentication     ── AuthenticationConfiguration import (05장)

별도 경로:
   ObjectPostProcessorConfiguration   ── ObjectPostProcessor 빈 (01장의 후처리기)
```

- 소스: `config/src/main/java/org/springframework/security/config/annotation/web/configuration/EnableWebSecurity.java`
- 소스: `.../web/configuration/WebSecurityConfiguration.java`
- 소스: `.../web/configuration/HttpSecurityConfiguration.java`
- 소스: `.../annotation/configuration/ObjectPostProcessorConfiguration.java`

### @EnableWebSecurity 의 정체

```java
@Retention(RUNTIME) @Target(TYPE) @Documented
@Import({ WebSecurityConfiguration.class, SpringWebMvcImportSelector.class, OAuth2ImportSelector.class,
        HttpSecurityConfiguration.class, ObservationImportSelector.class })
@EnableGlobalAuthentication
public @interface EnableWebSecurity {
    boolean debug() default false;
}
```

애너테이션 자체는 비어 있고, 핵심은 `@Import`다. 즉 `@EnableWebSecurity`는 "이 다섯 개의 설정 클래스를 켜라"는 스위치다. `ImportSelector`로 들어오는 셋은 **클래스패스/환경에 따라 조건부**로 설정을 끼워 넣는다(MVC, OAuth2, Observation이 없으면 그 부분을 건너뜀).

## 동작 흐름: 기동 시 필터 체인 빈이 만들어지기까지

```
[1] HttpSecurity 빈 (prototype) 준비  ── HttpSecurityConfiguration.httpSecurity()
       │  매 호출마다 새 HttpSecurity 생성 (@Scope("prototype"))
       │  기본 Configurer 장착: csrf, headers, sessionManagement,
       │    securityContext, requestCache, anonymous, servletApi,
       │    exceptionHandling, logout, DefaultLoginPageConfigurer
       │  + spring.factories의 AbstractHttpConfigurer 들
       │  + ApplicationContext 의 Customizer<HttpSecurity> 빈들
       ▼
[2] 사용자의 @Bean SecurityFilterChain 메서드 실행
       │  파라미터로 위 prototype HttpSecurity 주입받음
       │  http.authorizeHttpRequests{...}.formLogin{...} ...  ← DSL
       │  return http.build();   ← 01장의 doBuild → DefaultSecurityFilterChain
       ▼
[3] WebSecurityConfiguration.springSecurityFilterChain(...)
       │  주입된 List<SecurityFilterChain> 를 WebSecurity 에 등록
       │  WebSecurityCustomizer (ignoring 등) 적용
       │  webSecurity.build()  → FilterChainProxy
       ▼
[4] springSecurityFilterChain 이라는 이름의 Filter 빈 등록
       │  (DEFAULT_FILTER_NAME)
       ▼
[5] DelegatingFilterProxy(서블릿 web.xml/Boot) → 이 빈으로 위임
```

### [1] HttpSecurity 는 왜 prototype 인가

```java
@Bean(HTTPSECURITY_BEAN_NAME)
@Scope("prototype")
HttpSecurity httpSecurity() {
    ...
    AuthenticationManagerBuilder authenticationBuilder =
        new DefaultPasswordEncoderAuthenticationManagerBuilder(this.objectPostProcessor, passwordEncoder);
    authenticationBuilder.parentAuthenticationManager(authenticationManager()); // 전역 AuthManager를 부모로
    HttpSecurity http = new HttpSecurity(this.objectPostProcessor, authenticationBuilder, createSharedObjects());
    http.csrf(withDefaults())
        .addFilter(webAsyncManagerIntegrationFilter)
        .exceptionHandling(withDefaults())
        .headers(withDefaults())
        .sessionManagement(withDefaults())
        .securityContext(withDefaults())
        .requestCache(withDefaults())
        .anonymous(withDefaults())
        .servletApi(withDefaults())
        .with(new DefaultLoginPageConfigurer<>());
    http.logout(withDefaults());
    applyCorsIfAvailable(http);
    applyDefaultConfigurers(http);            // spring.factories 의 AbstractHttpConfigurer
    applyHttpSecurityCustomizers(this.context, http);  // Customizer<HttpSecurity> 빈
    applyTopLevelCustomizers(this.context, http);
    return http;
}
```

`HttpSecurity`는 한 번 `build()`하면 재사용 불가(상태 머신이 BUILT가 됨)이므로, **여러 개의 `SecurityFilterChain` 빈을 정의할 수 있도록 매번 새 인스턴스**를 줘야 한다. 그래서 prototype이다. 또한 여기서 이미 csrf·headers·세션 등 "거의 항상 필요한" 기본 보안이 장착된 상태로 넘어오므로, 사용자는 추가 정책만 얹으면 된다.

`createSharedObjects()`는 `ApplicationContext`, `ContentNegotiationStrategy`, `PathPatternRequestMatcher.Builder`를 공유 객체로 심어, 이후 Configurer들이 매처를 만들거나 빈을 찾을 수 있게 한다.

### [3] WebSecurityConfiguration — FilterChainProxy 조립

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain(ObjectProvider<HttpSecurity> provider) {
    boolean hasFilterChain = !this.securityFilterChains.isEmpty();
    if (!hasFilterChain) {
        // 사용자가 SecurityFilterChain 빈을 하나도 안 만들었으면 기본 체인 자동 생성
        this.webSecurity.addSecurityFilterChainBuilder(() -> {
            HttpSecurity httpSecurity = provider.getObject();
            httpSecurity.authorizeHttpRequests((a) -> a.anyRequest().authenticated());
            httpSecurity.formLogin(withDefaults());
            httpSecurity.httpBasic(withDefaults());
            return httpSecurity.build();
        });
    }
    for (SecurityFilterChain chain : this.securityFilterChains)
        this.webSecurity.addSecurityFilterChainBuilder(() -> chain);
    for (WebSecurityCustomizer c : this.webSecurityCustomizers)
        c.customize(this.webSecurity);
    return this.webSecurity.build();   // → FilterChainProxy (03장)
}
```

여기서 "사용자가 `SecurityFilterChain` 빈을 정의하지 않으면 인증된 사용자만 허용 + 폼/베이직 로그인"이라는 유명한 기본값이 나온다. `setFilterChainProxySecurityConfigurer(...)`는 `@Order`가 붙은 `WebSecurityConfigurer`들을 모아 순서 중복을 검증한 뒤 `WebSecurity`에 `apply`한다.

추가로 이 설정은 `springSecurityPathPatternParserBeanDefinitionRegistryPostProcessor`로 원래 `FilterChainProxy`를 `CompositeFilterChainProxy`로 감싸, 앞단에 `ServletRequestPathFilter`(MVC 경로 매칭 캐시용)를 붙인다.

### [4]~[5] 서블릿까지 연결

만들어진 `Filter` 빈의 이름은 `AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME`, 즉 `"springSecurityFilterChain"`이다. 서블릿 컨테이너에는 `DelegatingFilterProxy`가 같은 이름으로 등록되어, 실제 요청 처리를 이 빈(`FilterChainProxy`)에 위임한다. 이로써 [03장](03-HttpSecurity-와-WebSecurity.md)에서 다룰 요청 처리 경로가 완성된다.

## 핵심 메서드

- **`applyDefaultConfigurers(http)`** — `SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, ...)`로 `spring.factories`에 등록된 기본 Configurer들을 자동 장착. 다른 모듈/스타터가 자기 보안 기능을 끼워 넣는 표준 통로다.
- **`applyTopLevelCustomizers(context, http)`** — 리플렉션으로 `HttpSecurity`의 모든 `public xxx(Customizer<...>)` 메서드를 훑어, 컨테이너에 그 제네릭 타입의 `Customizer` 빈이 있으면 자동 호출. 즉 `Customizer<FormLoginConfigurer<HttpSecurity>>` 빈을 정의하면 `http.formLogin(...)`에 자동 반영된다.
- **`ObjectPostProcessorConfiguration.objectPostProcessor(...)`** — `AutowireBeanFactoryObjectPostProcessor`를 빈으로 등록. 01장의 후처리기가 여기서 나온다. `@Role(ROLE_INFRASTRUCTURE)`로 표시되어 일반 빈 후처리 대상과 구분된다.

## 설계 포인트 / 확장점

- **조건부 import(ImportSelector)**: OAuth2·MVC·Observation 통합을 클래스패스 유무로 켜고 끈다. 불필요한 의존성을 강제하지 않는 설계.
- **prototype HttpSecurity + 다중 SecurityFilterChain**: URL 그룹별로 서로 다른 보안 정책을 `@Order`로 우선순위를 줘 여러 체인으로 나눌 수 있다.
- **확장점**:
  - `SecurityFilterChain` 빈을 직접 정의 → 가장 일반적인 커스터마이징.
  - `WebSecurityCustomizer` 빈 → `web.ignoring()`처럼 `WebSecurity` 레벨 설정.
  - `Customizer<XxxConfigurer>` 빈 → 특정 Configurer를 전역으로 후킹.
  - `spring.factories`의 `AbstractHttpConfigurer` → 라이브러리가 기본 보안 기능을 배포.

## 정리

`@EnableWebSecurity`는 다섯 개의 `@Configuration`/`ImportSelector`를 켜는 스위치다. `HttpSecurityConfiguration`은 기본 보안이 장착된 **prototype `HttpSecurity`**를 공급하고, 사용자(또는 기본 동작)는 그것으로 `SecurityFilterChain`을 만든다. `WebSecurityConfiguration`은 그 체인들을 `WebSecurity`로 묶어 `FilterChainProxy` 필터 빈을 만들고, `DelegatingFilterProxy`가 서블릿 요청을 이 빈으로 흘려보낸다. 두 빌더의 내부 동작은 다음 장에서 본다.
