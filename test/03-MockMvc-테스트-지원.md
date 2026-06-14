# 03. MockMvc 테스트 지원 — RequestPostProcessor · Configurer · ResultMatcher

## 무엇을 / 왜

컨트롤러를 보안 필터 체인과 함께 통합 테스트하려면 `MockMvc` 로 실제 HTTP 요청을 흉내 내야 한다. 그런데 01 챕터에서 봤듯, 요청을 처리하는 `SecurityContextPersistenceFilter`/`SecurityContextHolderFilter` 가 `SecurityContextRepository` 값으로 `SecurityContextHolder` 를 덮어쓴다. 따라서 단순히 컨텍스트를 ThreadLocal 에 심어 두는 것으로는 부족하고, **컨텍스트를 `HttpServletRequest` 에 실어 Repository 경로로 흘려보내야** 한다. 이 챕터는 그 세 축 — 요청을 꾸미는 `RequestPostProcessors`, 환경을 세팅하는 `Configurers`/`RequestBuilders`, 결과를 검증하는 `ResultMatchers` — 를 다룬다.

## 핵심 타입

```
  MockMvc 셋업
   └ SecurityMockMvcConfigurers.springSecurity()  ─ 필터 등록 + 매 요청에 testSecurityContext() 적용
   
  요청 꾸미기 (RequestPostProcessor)
   └ SecurityMockMvcRequestPostProcessors
        user("u").roles("ADMIN")   csrf()    httpBasic(u,p)   x509(...)   digest()
        authentication(auth)        anonymous()  securityContext(ctx)
        jwt()  opaqueToken()  oauth2Login()  oidcLogin()  oauth2Client()
        testSecurityContext()      ← @WithMockUser 가 심어둔 컨텍스트를 요청에 연결
   
  요청 빌더 (RequestBuilder)
   └ SecurityMockMvcRequestBuilders.formLogin() / logout()   ← CSRF 자동 포함
   
  결과 검증 (ResultMatcher)
   └ SecurityMockMvcResultMatchers.authenticated() / unauthenticated()
   
  필터 가로채기 유틸
   └ WebTestUtils   ← 필터 체인에서 Repository/CsrfFilter 를 꺼내고 테스트용으로 바꿔치기
```

## 동작 흐름 1: springSecurity() 가 하는 일

`MockMvc` 빌더에 `apply(springSecurity())` 를 부르면 `SecurityMockMvcConfigurer` 가 두 시점에 개입한다.

```
 MockMvcBuilder.apply(springSecurity())
   │
   ├─ afterConfigurerAdded(builder)
   │     builder.addFilters(delegateFilter)     // 필터 순서를 보존하려 자리만 먼저 잡음
   │
   └─ beforeMockMvcCreated(builder, context)
         ├ springSecurityFilterChain 빈("springSecurityFilterChain")을 찾아 delegate 로 주입
         ├ servletContext 에 SPRING_SECURITY_FILTER_CHAIN 속성으로 등록
         │   (→ WebTestUtils 가 나중에 이 필터 체인을 꺼낼 수 있게)
         └ return testSecurityContext()         // 모든 요청에 이 RequestPostProcessor 자동 적용
```

`DelegateFilter` 라는 빈 껍데기 필터를 먼저 추가해 **필터 순서를 확정**한 뒤, 빈을 찾을 수 있는 `beforeMockMvcCreated` 단계에서 실제 `FilterChainProxy` 를 지연 주입하는 점이 영리하다(주석에 "`DelegatingFilterProxy` 는 delegate 를 지연 설정/조회하기 어려워 쓰지 않는다"고 명시). 마지막에 `testSecurityContext()` 를 반환하므로, 사용자가 매 요청에 따로 붙이지 않아도 `@WithMockUser` 가 심은 컨텍스트가 자동으로 요청에 연결된다.

## 동작 흐름 2: user("...") 또는 testSecurityContext() 가 컨텍스트를 요청에 싣는 법

핵심은 `SecurityContextRequestPostProcessorSupport.save(...)` 다. 컨텍스트를 그냥 두는 게 아니라 **요청의 `SecurityContextRepository` 를 테스트용으로 감싼 뒤 그 경로로 저장**한다.

```
 .with(user("admin").roles("ADMIN"))  또는  자동 적용된 testSecurityContext()
   │
   ▼ postProcessRequest(request)
   │
   ├ WebTestUtils.getSecurityContextRepository(request)
   │     필터 체인에서 SecurityContextPersistenceFilter→repo / SecurityContextHolderFilter→
   │     securityContextRepository 를 리플렉션으로 꺼냄 (없으면 HttpSessionSecurityContextRepository)
   │
   ├ 그 repo 를 TestSecurityContextRepository 로 감싸 WebTestUtils.setSecurityContextRepository 로 되꽂음
   │     (요청 attribute 로 컨텍스트를 들고 다녀, stateless 모드에서도 동작)
   │
   └ repo.loadContext(holder) → repo.saveContext(securityContext, req, resp)
         이제 실제 필터가 돌 때 이 컨텍스트가 SecurityContextHolder 로 복원됨
```

`testSecurityContext()` 의 후처리기는 한 발 더 신중하다. 이미 요청에 컨텍스트가 저장돼 있으면 건드리지 않고, `TestSecurityContextHolder` 의 컨텍스트가 **빈 컨텍스트가 아닐 때만** 저장한다.

```java
SecurityContext existingContext = TestSecurityContextRepository.getContext(request);
if (existingContext != null) return request;          // 이미 user(...) 등으로 지정됨
SecurityContext empty = strategy.createEmptyContext();
SecurityContext context = strategy.getContext();
if (!empty.equals(context)) save(context, request);   // @WithMockUser 가 심은 게 있을 때만
```

즉 `@WithMockUser`(컨텍스트를 `TestSecurityContextHolder` 에 심음) + `springSecurity()`(매 요청에 `testSecurityContext()` 적용)의 조합이, 메서드 보안 테스트와 웹 통합 테스트를 하나의 사용자로 자연스럽게 잇는다.

### user("...") 의 권한 규칙

`UserRequestPostProcessor` 는 `@WithMockUser` 와 거의 같은 규칙을 따른다. 기본 권한 `ROLE_USER`, `roles(...)` 는 `ROLE_` 접두를 자동으로 붙이며(이미 붙어 있으면 예외), `authorities(...)` 는 접두 없이 그대로 쓴다. 내부적으로 `User` 를 만들어 `UserDetailsRequestPostProcessor` → `AuthenticationRequestPostProcessor` 로 위임한다.

### csrf() 와 토큰 후처리기

`csrf()` 는 `WebTestUtils` 로 요청의 `CsrfTokenRepository`/`CsrfTokenRequestHandler` 를 꺼내, 유효한 토큰을 만들어 파라미터(기본) 또는 헤더(`.asHeader()`)로 넣는다. `.useInvalidToken()` 으로 일부러 깨진 토큰을 넣어 CSRF 거부도 테스트할 수 있다. 한편 `jwt()`/`opaqueToken()` 같은 토큰 기반 후처리기는 `CsrfFilter.skipRequest(request)` 를 호출해 — 베어러 토큰 요청에는 CSRF 가 적용되지 않는 운영 동작을 그대로 반영한다.

`jwt()`, `opaqueToken()`, `oauth2Login()`, `oidcLogin()`, `oauth2Client()` 는 각각 `JwtAuthenticationToken`, `BearerTokenAuthentication`, `OAuth2AuthenticationToken` 등을 **declarative 하게**(실제 토큰 검증 없이) 만들어 같은 `save(...)` 경로로 요청에 싣는다. 빌더형 API(`jwt(j -> j.claim(...))`, `authorities(...)`)로 클레임·권한을 조정한다.

## 동작 흐름 3: formLogin() / logout()

`SecurityMockMvcRequestBuilders` 는 후처리기가 아니라 **요청 자체를 만드는 빌더**다. 폼 로그인 흐름을 실제로 통과시키고 싶을 때 쓴다.

```java
public static FormLoginRequestBuilder formLogin() { return new FormLoginRequestBuilder(); }
```

`FormLoginRequestBuilder.buildRequest` 는 `POST /login` 에 `username`/`password` 파라미터를 싣고, **CSRF 토큰을 자동 포함**한다(`postProcessor = csrf()`). 파라미터 이름·값·URL·Accept 미디어타입을 모두 커스터마이즈할 수 있다.

```java
MockHttpServletRequestBuilder loginRequest = post(this.loginProcessingUrl)
    .accept(this.acceptMediaType)
    .param(this.usernameParam, this.username)
    .param(this.passwordParam, this.password);
...
return this.postProcessor.postProcessRequest(request);   // csrf() 적용
```

`logout()` 도 동일하게 `POST /logout` + CSRF 자동 포함이다. 둘 다 `Mergeable` 이라 `defaultRequest(...)` 와 병합된다.

## 동작 흐름 4: authenticated() / unauthenticated() 검증

`SecurityMockMvcResultMatchers` 는 응답이 아니라 **요청에 연결된 `SecurityContext`** 를 다시 로드해 검증한다.

```java
protected SecurityContext load(MvcResult result) {
    HttpRequestResponseHolder holder = new HttpRequestResponseHolder(result.getRequest(), result.getResponse());
    SecurityContextRepository repository = WebTestUtils.getSecurityContextRepository(result.getRequest());
    return repository.loadContext(holder);
}
```

- `authenticated()` — `Authentication` 이 null 이 아닌지 확인하고, 빌더 메서드로 추가 단언을 건다: `withUsername(...)`, `withAuthenticationName(...)`, `withAuthorities(...)`, `withRoles(prefix, roles)`(roles 만 비교, 나머지 권한은 무시), `withAuthenticationPrincipal(...)`, `withAuthentication(Consumer)` 등.
- `unauthenticated()` — `Authentication` 이 null 이거나 `AuthenticationTrustResolver.isAnonymous(...)` 면 통과. 즉 "익명도 미인증으로 간주"한다.

이렇게 검증이 응답 본문이 아닌 보안 컨텍스트를 직접 보기 때문에, 폼 로그인 후 실제로 누가 인증됐는지를 정확히 단언할 수 있다.

## 핵심 메서드

- `SecurityMockMvcConfigurer.beforeMockMvcCreated(...)` — `springSecurityFilterChain` 빈을 찾아 지연 주입하고, ServletContext 에 등록한 뒤 `testSecurityContext()` 를 반환한다. MockMvc 보안 통합의 진입점.
- `SecurityContextRequestPostProcessorSupport.save(SecurityContext, request)` — Repository 를 `TestSecurityContextRepository` 로 감싸 컨텍스트를 요청 경로에 저장한다. 거의 모든 후처리기가 결국 이 메서드로 수렴한다.
- `WebTestUtils.getSecurityContextRepository / setSecurityContextRepository / getCsrfTokenRepository / findFilter` — 필터 체인 내부 필드를 `ReflectionTestUtils` 로 읽고 바꿔치기하는, 모듈 전체가 의존하는 저수준 유틸.

## 설계 포인트 / 확장점

- **Repository 가로채기**: `WebTestUtils` 가 실제 필터의 `repo`/`tokenRepository` 필드를 리플렉션으로 꺼내고 테스트용 래퍼로 교체하는 것이 MockMvc 통합의 비밀이다. 운영 필터 구성을 바꾸지 않고도 컨텍스트/토큰을 주입한다.
- **stateless 대응**: `TestSecurityContextRepository` 는 컨텍스트를 요청 attribute 로도 들고 다녀, 세션을 쓰지 않는 stateless 설정에서도 후속 단언이 컨텍스트를 찾을 수 있게 한다.
- **선언 경로와 요청 경로의 통합**: `@WithMockUser`(선언)와 `user(...)`(요청별)가 동일한 `save(...)`/Repository 메커니즘으로 수렴해, 두 스타일을 섞어 쓸 수 있다.

## 정리

MockMvc 보안 테스트는 세 박자다. (1) `springSecurity()` 로 필터 체인을 붙이고 매 요청에 `testSecurityContext()` 를 자동 적용하며, (2) `user(...)`·`csrf()`·`jwt()` 같은 `RequestPostProcessor` 또는 `formLogin()` 빌더로 요청을 꾸미고, (3) `authenticated()`/`unauthenticated()` 로 결과의 보안 컨텍스트를 검증한다. 이 모든 것의 토대는 `WebTestUtils` 가 실제 필터에서 `SecurityContextRepository` 를 꺼내 테스트용으로 바꿔치기하는 리플렉션 기법이다.
