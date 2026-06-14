# 04. 리액티브 테스트 지원 — WebTestClient Mutator 와 Reactor Context 전파

## 무엇을 / 왜

WebFlux 애플리케이션은 서블릿 `MockMvc` 가 아니라 `WebTestClient` 로 테스트하며, 보안 모델도 다르다. 인증은 `ThreadLocal` 이 아니라 **Reactor `Context`**(`ReactiveSecurityContextHolder`)를 타고 흐른다. 따라서 03 챕터의 "필터에서 Repository 를 꺼내 바꿔치기" 전략을 그대로 쓸 수 없고, 리액티브 파이프라인의 `Context` 에 `SecurityContext` 를 주입하는 별도 메커니즘이 필요하다. 이 챕터는 두 갈래 — (A) 애너테이션(`@WithMockUser`)을 Reactor Context 로 옮기는 `ReactorContextTestExecutionListener`, (B) `WebTestClient` 요청별로 사용자를 지정하는 `SecurityMockServerConfigurers` — 를 다룬다.

## 핵심 타입

```
 (A) 애너테이션 → Reactor Context
     ReactorContextTestExecutionListener   (order 11000, WithSecurityContext 리스너 다음)
       └ DelegatingTestExecutionListener 를 상속, Reactor 유무에 따라 위임 대상 결정

 (B) WebTestClient 요청별 지정
     SecurityMockServerConfigurers
       springSecurity()                   ─ MutatorFilter 를 필터 0번에 추가
       mockUser(...) / mockAuthentication(...) / mockJwt() / mockOpaqueToken()
       mockOAuth2Login() / mockOidcLogin() / mockOAuth2Client() / csrf()
         └ 각각 WebTestClientConfigurer + MockServerConfigurer 인 Mutator
            (SetupMutatorFilter 로 exchange attribute 에 SecurityContext 공급자를 심음)
```

## 동작 흐름 A: @WithMockUser 를 Reactor Context 로 옮기기

`ReactorContextTestExecutionListener`(`.../context/support/ReactorContextTestExecutionListener.java`)는 01 챕터에서 본 두 번째 자동 등록 리스너다. `order = 11000` 이라 `WithSecurityContextTestExecutionListener`(10000)가 `TestSecurityContextHolder` 를 **이미 채운 뒤**에 실행된다.

```
 beforeTestMethod (order 11000, WithSecurityContext 리스너 다음)
   │
   ├ SecurityContext sc = TestSecurityContextHolder.getContext();   // 10000 이 채워둔 값
   │
   └ Hooks.onLastOperator(KEY, Operators.lift((s, sub) ->
            new SecuritySubContext<>(sub, sc)));
        → 이후 모든 Reactor 연산자 체인에 SecurityContext 를 끼워 넣는 전역 훅 설치

 afterTestMethod
   └ Hooks.resetOnLastOperator(KEY);   // 훅 제거 (테스트 간 누수 방지)
```

`SecuritySubContext.currentContext()` 가 실제 주입 지점이다. 한 번만(중복 방지 플래그) `Authentication` 이 있을 때 `ReactiveSecurityContextHolder.withSecurityContext(...)` 로 Reactor `Context` 에 병합한다.

```java
public Context currentContext() {
    Context context = this.delegate.currentContext();
    if (context.hasKey(CONTEXT_DEFAULTED_ATTR_NAME)) return context;   // 이미 처리됨
    context = context.put(CONTEXT_DEFAULTED_ATTR_NAME, Boolean.TRUE);
    Authentication authentication = this.securityContext.getAuthentication();
    if (authentication == null) return context;                        // 인증 없으면 그대로
    Context toMerge = ReactiveSecurityContextHolder.withSecurityContext(Mono.just(this.securityContext));
    return toMerge.putAll(context.readOnly());
}
```

덕분에 `@WithMockUser` 하나로 **블로킹 메서드 테스트와 리액티브 메서드 테스트가 동일하게** 동작한다. 단, 이 모든 것은 Reactor 가 클래스패스에 있을 때만이다 — 생성자에서 `reactor.core.publisher.Hooks` 존재를 확인하고, 없으면 아무 일도 하지 않는 빈 `AbstractTestExecutionListener` 로 위임한다.

```java
private static TestExecutionListener createDelegate() {
    if (!ClassUtils.isPresent(HOOKS_CLASS_NAME, ...))
        return new AbstractTestExecutionListener() { };   // no-op
    return new DelegateTestExecutionListener();
}
```

`DelegatingTestExecutionListener` 는 모든 콜백(`beforeTestMethod`, `afterTestMethod` 등)을 내부 delegate 로 그대로 넘기는 베이스 클래스다. 이 위임 구조가 "Reactor 유무에 따라 동작/무동작을 바꾸는" 분기를 깔끔하게 만든다.

## 동작 흐름 B: WebTestClient 의 두 필터 협업

`WebTestClient` 요청별로 사용자를 지정하는 메커니즘은 **두 개의 `WebFilter` 가 짝을 이뤄** 돌아간다. 하나는 컨텍스트를 exchange 에 심고(setup), 다른 하나는 그걸 Reactor Context 로 쓴다(apply).

```
 WebTestClient.Builder
   .apply(springSecurity())          → MutatorFilter 를 필터 0번에 추가          (apply 측)
   .apply(mockUser("u"))  또는
   client.mutateWith(mockUser("u"))  → SetupMutatorFilter 를 필터 0번에 추가     (setup 측)

  요청이 들어오면 (필터 체인):
   ┌──────────────────────────────────────────────────────────┐
   │ SetupMutatorFilter.filter(exchange, chain)                 │
   │   exchange attribute["context"] = Supplier<Mono<SC>>       │  ← 사용자 공급자만 심음
   │   → chain.filter(exchange)                                  │
   └──────────────────────────────┬───────────────────────────┘
                                   ▼
   ┌──────────────────────────────────────────────────────────┐
   │ MutatorFilter.filter(exchange, chain)                      │
   │   Supplier ctx = exchange.getAttribute("context")          │
   │   if (ctx != null) {                                        │
   │     remove attribute;                                       │
   │     return chain.filter(exchange)                           │
   │        .contextWrite(                                       │
   │          ReactiveSecurityContextHolder.withSecurityContext( │
   │             ctx.get()));   ← 여기서 Reactor Context 에 주입   │
   │   }                                                         │
   └──────────────────────────────────────────────────────────┘
```

코드로 보면:

```java
// springSecurity() : MutatorFilter 를 0번에 추가
public static MockServerConfigurer springSecurity() {
    return new MockServerConfigurer() {
        public void beforeServerCreated(WebHttpHandlerBuilder builder) {
            builder.filters((filters) -> filters.add(0, new MutatorFilter()));
        }
    };
}

// mockAuthentication(...) : SetupMutatorFilter 로 공급자만 심음
public static <T extends ...> T mockAuthentication(Authentication authentication) {
    return (T) new MutatorWebTestClientConfigurer(
        () -> Mono.just(authentication).map(SecurityContextImpl::new));
}
```

왜 굳이 둘로 나눌까? `SetupMutatorFilter` 는 **요청 단위**(어떤 사용자로 보낼지)로 바뀌고, `MutatorFilter` 는 **서버 단위**(컨텍스트를 실제 Reactor Context 로 흘리는 공통 로직)다. 책임을 분리해, 사용자가 `springSecurity()` 를 한 번만 적용해 두면 요청마다 `mockUser(...)` 만 바꿔 끼울 수 있다.

`mockUser(UserDetails)` 는 `mockAuthentication(UsernamePasswordAuthenticationToken...)` 로, `mockJwt()`/`mockOpaqueToken()`/`mockOAuth2Login()` 등은 각자 적절한 `Authentication` 을 만들어 같은 setup 경로로 수렴한다. `csrf()`(`CsrfMutator`)는 `CsrfWebFilter` 의 보호 매처를 "항상 불일치"로 바꿔 CSRF 검증을 건너뛰게 한다.

## 핵심 메서드

- `ReactorContextTestExecutionListener.DelegateTestExecutionListener.beforeTestMethod(...)` — `TestSecurityContextHolder` 의 컨텍스트를 읽어 `Hooks.onLastOperator(...)` 전역 훅으로 모든 Reactor 체인에 주입한다. `afterTestMethod` 에서 훅을 해제한다.
- `SecuritySubContext.currentContext()` — 실제로 `ReactiveSecurityContextHolder.withSecurityContext(...)` 를 Reactor `Context` 에 병합하는 지점. 중복 적용 방지 플래그를 둔다.
- `SecurityMockServerConfigurers.springSecurity()` / `mockXxx(...)` — 각각 `MutatorFilter`(apply)와 `SetupMutatorFilter`(setup)를 필터 0번에 끼워, exchange attribute 를 매개로 협업하게 만든다.

## 설계 포인트 / 확장점

- **두 갈래의 일관성**: 선언적 경로(`@WithMockUser` → Reactor Context)와 요청별 경로(`WebTestClient.mutateWith(mockUser(...))`)가 결국 모두 `ReactiveSecurityContextHolder.withSecurityContext(...)` 로 귀결된다. 서블릿 쪽에서 모든 후처리기가 `save(...)` 로 수렴하던 것과 같은 설계 철학이다.
- **선택적 의존(Reactor)**: `ClassUtils.isPresent` 로 Reactor 유무를 감지해, 리액티브 의존이 없는 프로젝트에서도 모듈이 안전하게 로드된다(no-op 위임).
- **order 로 보장하는 인과관계**: 10000(컨텍스트 채움) → 11000(Reactor 전파) 순서가 빈 컨텍스트 전파를 막는다.
- **필터 0번 삽입**: 두 Mutator 모두 `filters.add(0, ...)` 로 체인 맨 앞에 들어가, 다른 보안 필터보다 먼저 Reactor Context 를 세팅한다.

## 정리

리액티브 테스트는 `ThreadLocal` 대신 Reactor `Context` 를 채워야 한다. 애너테이션 경로에서는 `ReactorContextTestExecutionListener` 가 `TestSecurityContextHolder` 의 컨텍스트를 `Hooks.onLastOperator` 전역 훅으로 모든 체인에 주입하고, `WebTestClient` 경로에서는 `springSecurity()` 의 `MutatorFilter` 와 `mockUser(...)` 등의 `SetupMutatorFilter` 가 exchange attribute 를 매개로 협업해 `contextWrite` 로 `SecurityContext` 를 흘려보낸다. 두 경로 모두 종착지는 `ReactiveSecurityContextHolder` 다.
