# 03. HttpSecurity 와 WebSecurity

## 무엇을 / 왜

`config` 모듈에는 웹 보안을 만드는 두 개의 빌더가 있다. 이름이 비슷해 헷갈리지만 책임이 완전히 다르다.

- **`HttpSecurity`** — **하나의 보안 정책(=하나의 `SecurityFilterChain`)** 을 만든다. "이 URL 그룹은 폼 로그인 + CSRF + 인증 필요" 같은 한 묶음의 필터 리스트를 조립한다.
- **`WebSecurity`** — 여러 개의 `SecurityFilterChain`을 모아 **하나의 `FilterChainProxy`** 로 묶고, 방화벽·무시 경로(ignoring)·권한 평가기 같은 애플리케이션 전역 웹 보안 설정을 얹는다.

이 장은 두 빌더가 `performBuild()`에서 각각 무엇을 산출하고, 그 결과가 요청 처리 시 어떻게 연결되는지를 본다.

## 핵심 타입

```
   HttpSecurity                                WebSecurity
   extends AbstractConfiguredSecurityBuilder    extends AbstractConfiguredSecurityBuilder
        <DefaultSecurityFilterChain,...>             <Filter,...>
   implements HttpSecurityBuilder               implements ... ApplicationContextAware
        │                                            │
        │ performBuild()                             │ performBuild()
        ▼                                            ▼
   DefaultSecurityFilterChain                   FilterChainProxy (Filter)
   = (RequestMatcher, List<Filter>)             = List<SecurityFilterChain> + 방화벽/디코레이터
```

- 소스: `config/src/main/java/org/springframework/security/config/annotation/web/builders/HttpSecurity.java`
- 소스: `.../web/builders/WebSecurity.java`
- 소스: `.../web/builders/FilterOrderRegistration.java` (필터 순서표)
- 소스: `.../annotation/web/HttpSecurityBuilder.java` (인터페이스)

## HttpSecurity — 한 체인 만들기

`HttpSecurity`의 메서드는 거의 전부 `xxx(Customizer<XxxConfigurer> c)` 모양이고, 내부적으로 `getOrApply(new XxxConfigurer())` 후 사용자 람다를 적용한다. 예:

```java
public HttpSecurity formLogin(Customizer<FormLoginConfigurer<HttpSecurity>> c) {
    c.customize(getOrApply(new FormLoginConfigurer<>()));
    return this;
}
```

즉 `http.formLogin(...)`은 "필터를 지금 만드는" 게 아니라 **Configurer를 등록만** 한다. 실제 필터 생성·삽입은 `build()` 때 각 Configurer의 `configure()`에서 일어난다([04장](04-HTTP-Configurer-카탈로그.md)).

### 필터 순서는 어디서 정해지나 — FilterOrderRegistration

필터 체인에서 순서는 보안의 핵심이다(`SecurityContextHolderFilter`가 `AuthorizationFilter`보다 앞이어야 한다). `HttpSecurity`는 필터를 `OrderedFilter`로 감싸 정수 순서값과 함께 보관한다.

```java
private List<OrderedFilter> filters = new ArrayList<>();
private FilterOrderRegistration filterOrders = new FilterOrderRegistration();

public HttpSecurity addFilter(Filter filter) {
    Integer order = this.filterOrders.getOrder(filter.getClass());  // 알려진 필터면 등록된 순서
    if (order == null) throw new IllegalArgumentException(... "does not have a registered order" ...);
    this.filters.add(new OrderedFilter(filter, order));
    return this;
}
```

`FilterOrderRegistration`은 `SecurityContextHolderFilter`, `CsrfFilter`, `UsernamePasswordAuthenticationFilter`, `AuthorizationFilter` 등 **표준 필터 클래스 → 순서 정수** 의 사전을 들고 있다. `addFilterBefore/After/At`는 기준 필터의 순서값에 `±offset`을 적용해 새 순서를 부여한다. 이 덕분에 사용자가 커스텀 필터를 "어떤 필터 앞/뒤"로만 지정하면 절대 위치를 몰라도 올바른 자리에 들어간다.

### performBuild — 정렬 후 DefaultSecurityFilterChain

```java
@Override
protected void beforeConfigure() {          // configure() 직전에 실행
    if (this.authenticationManager != null)
        setSharedObject(AuthenticationManager.class, this.authenticationManager);
    else {
        AuthenticationManager manager = getAuthenticationRegistry().build();  // AuthManagerBuilder
        if (manager != null)
            setSharedObject(AuthenticationManager.class, postProcessor.postProcess(manager));
    }
}

@Override
protected DefaultSecurityFilterChain performBuild() {
    this.filters.sort(OrderComparator.INSTANCE);               // 순서값으로 정렬
    List<Filter> sortedFilters = new ArrayList<>(this.filters.size());
    for (Filter filter : this.filters) sortedFilters.add(((OrderedFilter) filter).filter);
    return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
}
```

두 단계가 핵심이다. (1) `beforeConfigure()`에서 `AuthenticationManager`를 미리 빌드해 공유 객체로 심는다 — 그래야 `configure()` 단계의 인증 필터들이 그걸 주입받는다([05장](05-인증-구성.md)). (2) `performBuild()`에서 등록 필터들을 순서값으로 정렬해 `DefaultSecurityFilterChain(매처, 필터목록)`을 만든다. 이 매처(`requestMatcher`)가 "이 체인이 어떤 요청에 적용되는지"를 결정한다(`securityMatcher(...)`로 지정, 기본은 `AnyRequestMatcher`).

## WebSecurity — 체인들을 FilterChainProxy 로

`WebSecurity`는 `addSecurityFilterChainBuilder(...)`로 받은 체인 빌더들과 `ignoring()`으로 등록된 무시 경로를 모아 하나의 `FilterChainProxy`를 만든다.

```java
@Override
protected Filter performBuild() {
    Assert.state(!this.securityFilterChainBuilders.isEmpty(), () -> "At least one ...");
    List<SecurityFilterChain> securityFilterChains = new ArrayList<>(...);
    RequestMatcherDelegatingAuthorizationManager.Builder builder =
        RequestMatcherDelegatingAuthorizationManager.builder();

    for (RequestMatcher ignoredRequest : this.ignoredRequests) {        // ignoring() 경로
        logger.warn("You are asking Spring Security to ignore " + ignoredRequest + " ...");
        securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));  // 필터 0개 체인
        builder.add(ignoredRequest, SingleResultAuthorizationManager.permitAll());
    }
    for (SecurityBuilder<? extends SecurityFilterChain> b : this.securityFilterChainBuilders) {
        SecurityFilterChain chain = b.build();        // ← 각 HttpSecurity.build() 호출
        securityFilterChains.add(chain);
        addAuthorizationManager(chain, builder);      // 권한 평가기용 매핑 수집
    }
    FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
    if (this.httpFirewall != null) filterChainProxy.setFirewall(this.httpFirewall);
    if (this.requestRejectedHandler != null) filterChainProxy.setRequestRejectedHandler(...);
    filterChainProxy.setFilterChainValidator(new WebSecurityFilterChainValidator());
    filterChainProxy.setFilterChainDecorator(getFilterChainDecorator());
    filterChainProxy.afterPropertiesSet();
    Filter result = filterChainProxy;
    if (this.debugEnabled) result = new DebugFilter(filterChainProxy);
    return result;
}
```

핵심 포인트:

- **무시 경로(`ignoring`)** 는 "필터가 0개인 체인"으로 표현된다. 매칭되면 보안 필터를 아예 통과하지 않는다. 소스는 이를 권장하지 않는다고 경고 로그를 남기고, 대신 `permitAll`을 쓰라고 안내한다.
- 각 `securityFilterChainBuilder.build()` 호출이 곧 [02장](02-EnableWebSecurity-와-빈-구성.md)에서 준비된 `HttpSecurity`/`SecurityFilterChain`을 구체화한다.
- `WebInvocationPrivilegeEvaluator`(JSP/뷰에서 "이 URL 접근 가능한가" 판단용)를 각 체인의 `AuthorizationFilter`에서 권한 매니저를 긁어 모아 자동 구성한다.
- 디버그 모드면 `DebugFilter`로 감싼다.

## 동작 흐름: 요청이 들어오면

```
   HTTP 요청
     │
     ▼
   DelegatingFilterProxy ("springSecurityFilterChain")
     │  위임
     ▼
   FilterChainProxy            ◀── WebSecurity.performBuild() 산출물
     │  요청 URL 로 매칭되는 첫 SecurityFilterChain 선택
     │  (DefaultSecurityFilterChain.matches(request))
     ▼
   매칭된 SecurityFilterChain  ◀── HttpSecurity.performBuild() 산출물
     │  그 안의 정렬된 필터들을 순서대로 실행
     ▼
   SecurityContextHolderFilter → CsrfFilter → ... →
   UsernamePasswordAuthenticationFilter → ... → AuthorizationFilter → (대상 자원)
```

`FilterChainProxy`는 여러 체인 중 요청에 **처음으로 매칭되는 하나**만 고른다. 그래서 `@Order`로 더 구체적인 URL 패턴 체인을 앞에 둬야 한다. 선택된 체인 안에서는 `HttpSecurity`가 정렬해 둔 필터 순서대로 실행된다. 각 필터의 실제 동작(인증·인가 로직)은 `web`/`core` 모듈의 책임이다 — [../web/README.md](../web/README.md) 참고.

## 핵심 메서드

- **`HttpSecurity.securityMatcher(...)`** — 이 체인이 적용될 요청 범위를 지정. 미지정 시 모든 요청(`AnyRequestMatcher`). 다중 체인을 나눌 때 필수.
- **`HttpSecurity.addFilterBefore/After/At(filter, refClass)`** — 커스텀 필터를 표준 필터 기준 상대 위치로 삽입. `FilterOrderRegistration`이 순서를 계산.
- **`HttpSecurity.authenticationProvider(...)` / `userDetailsService(...)`** — 공유 `AuthenticationManagerBuilder`(getAuthenticationRegistry)에 위임해 이 체인 로컬 인증 설정을 추가([05장](05-인증-구성.md)).
- **`WebSecurity.ignoring()` / `httpFirewall(...)` / `requestRejectedHandler(...)`** — 전역 웹 보안 설정. `setApplicationContext`에서 `HttpFirewall`·`RequestRejectedHandler`·`ObservationRegistry` 빈을 자동 탐지한다.

## 설계 포인트 / 확장점

- **두 빌더의 책임 분리**: `HttpSecurity`(한 정책) vs `WebSecurity`(전역 묶음 + 방화벽). 이 분리 덕에 다중 정책을 깔끔히 표현한다.
- **순서의 데이터화**: 필터 순서를 코드 분기 대신 `FilterOrderRegistration` 사전으로 관리해, 새 필터 추가/삽입이 선언적이다.
- **확장점**: 커스텀 필터는 `addFilterBefore/After`로, 전역 방화벽/거부 핸들러는 빈 등록으로, 권한 평가기는 `privilegeEvaluator(...)`로 교체 가능. 체인 자체를 여러 개 만들어 영역별 보안을 분리할 수 있다.

## 정리

`HttpSecurity`는 Configurer들을 모아 `build()` 때 필터를 만들고 순서대로 정렬해 **하나의 `DefaultSecurityFilterChain`** 을 낳는다. `WebSecurity`는 그런 체인들과 무시 경로를 모아 **하나의 `FilterChainProxy`** 로 묶는다. 런타임에는 `FilterChainProxy`가 요청에 맞는 체인을 골라 그 필터들을 순서대로 실행한다. 그 필터들을 누가 만들고 어디에 끼우는지는 다음 장의 Configurer가 답한다.
