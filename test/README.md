# test 모듈 — Spring Security 테스트 지원

> 버전: Spring Security 7.1.1-SNAPSHOT
> 소스 루트: `test/src/main/java/org/springframework/security/test`

## 한 줄 정의

`spring-security-test` 는 **실제 로그인·필터·DB 없이 "인증된 사용자"를 흉내 내어, 그 보안 컨텍스트를 테스트가 실행되는 동안 `SecurityContextHolder`(또는 Reactor `Context`)에 심어 주는** 테스트 전용 지원 모듈이다. JUnit 단위 테스트(`@WithMockUser`), Servlet MockMvc, Reactive WebTestClient 세 영역을 모두 커버한다.

## 이 모듈이 푸는 문제

운영 코드에서 인증은 로그인 필터 → `AuthenticationManager` → `SecurityContextRepository` 라는 긴 경로를 거쳐 `SecurityContextHolder` 에 자리잡는다. 테스트에서 이 과정을 매번 진짜로 수행하기는 번거롭고 느리다. 그렇다고 `SecurityContextHolder.setContext(...)` 를 직접 호출하면 다음 문제가 생긴다.

- **웹 요청 경로의 충돌**: MockMvc 요청 처리 중 `SecurityContextPersistenceFilter`(또는 `SecurityContextHolderFilter`)가 `SecurityContextRepository` 의 값으로 `SecurityContextHolder` 를 덮어쓰고, 필터 체인이 끝나면 비워 버린다. 직접 심은 컨텍스트가 사라진다.
- **정리(clean-up) 누락**: 테스트마다 컨텍스트를 비우지 않으면 `ThreadLocal` 이 다음 테스트로 새어 나간다.
- **메서드 보안 vs 웹 보안의 차이**: 메서드 인가 테스트는 `SecurityContextHolder` 만 있으면 되지만, 웹 요청 테스트는 컨텍스트가 `HttpServletRequest` 를 거쳐 흘러야 한다.
- **리액티브의 다른 모델**: WebFlux 는 `ThreadLocal` 이 아니라 Reactor `Context` 로 인증을 흘려보낸다.

이 모듈은 그 간극을 메운다. 핵심은 **선언적 보안 컨텍스트 주입**(애너테이션) + **테스트 생명주기 훅**(`TestExecutionListener`) + **웹 통합 어댑터**(MockMvc / WebTestClient) 의 조합이다.

## 의존 / 연관 모듈

- `../core/README.md` — `SecurityContext`, `Authentication`, `UsernamePasswordAuthenticationToken`, `AnonymousAuthenticationToken`, `UserDetails`, `SecurityContextHolderStrategy`, `ReactiveSecurityContextHolder` 등 도메인 타입을 그대로 사용한다.
- `../web/README.md` — `SecurityContextRepository`, `SecurityContextPersistenceFilter`, `SecurityContextHolderFilter`, `CsrfFilter`, `CsrfTokenRepository` 등 Servlet 웹 보안 인프라를 리플렉션으로 들여다보고 교체한다.
- `../oauth2-client/README.md`, `../oauth2-resource-server/README.md` — `jwt()`, `opaqueToken()`, `oauth2Login()`, `oidcLogin()`, `oauth2Client()` 같은 OAuth2/OIDC 테스트 헬퍼가 그쪽 토큰/클라이언트 타입에 의존한다.
- Spring Framework `spring-test` — `TestExecutionListener`, `TestContext`, `MockMvc`, `WebTestClient`, `RequestPostProcessor` 같은 테스트 인프라 위에 얹혀 있다.
- Project Reactor — 리액티브 지원은 `reactor.core` 가 클래스패스에 있을 때만 동작하는 선택적 의존이다.

## 내부 목차 (읽는 순서)

1. [01-핵심-개념과-구조.md](01-핵심-개념과-구조.md) — `TestSecurityContextHolder` 와 `WithSecurityContext` 메타 애너테이션, `TestExecutionListener` 자동 등록, 보안 컨텍스트가 흐르는 두 갈래 경로.
2. [02-어노테이션-기반-인증.md](02-어노테이션-기반-인증.md) — `@WithMockUser` / `@WithUserDetails` / `@WithAnonymousUser` 와 그 뒤의 `WithSecurityContextFactory`, `setupBefore` 타이밍, 동작 시퀀스.
3. [03-MockMvc-테스트-지원.md](03-MockMvc-테스트-지원.md) — `SecurityMockMvcRequestPostProcessors` / `RequestBuilders` / `ResultMatchers` / `Configurers` 와 `WebTestUtils` 의 필터 가로채기.
4. [04-리액티브-WebTestClient-지원.md](04-리액티브-WebTestClient-지원.md) — `ReactorContextTestExecutionListener` 와 `SecurityMockServerConfigurers`.

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.test
│
├── context                                   # 보안 컨텍스트의 "테스트용" 홀더
│   ├── TestSecurityContextHolder             # SecurityContextHolder 와 짝을 이루는 ThreadLocal
│   ├── TestSecurityContextHolderStrategyAdapter   # 위를 Strategy 인터페이스로 감싼 어댑터
│   │
│   ├── annotation
│   │   └── SecurityTestExecutionListeners    # 보안 리스너만 켜는 메타 애너테이션
│   │
│   └── support                               # 선언적 보안 컨텍스트 주입의 핵심
│       ├── WithSecurityContext               #  ├ 모든 @With* 의 메타 애너테이션
│       ├── WithSecurityContextFactory        #  ├ 애너테이션 → SecurityContext 변환 전략 인터페이스
│       ├── WithMockUser / ...Factory         #  ├ 가짜 username/role 사용자
│       ├── WithUserDetails / ...Factory      #  ├ UserDetailsService 로 실제 조회
│       ├── WithAnonymousUser / ...Factory    #  ├ 익명 사용자
│       ├── TestExecutionEvent                #  ├ 주입 타이밍(TEST_METHOD / TEST_EXECUTION)
│       ├── WithSecurityContextTestExecutionListener  # ★ 애너테이션을 읽어 컨텍스트를 심는 리스너
│       ├── ReactorContextTestExecutionListener       # ★ Reactor Context 로 전파
│       └── DelegatingTestExecutionListener   #  └ 위 리액티브 리스너의 위임 베이스
│
├── web
│   ├── servlet                               # MockMvc(Servlet) 통합
│   │   ├── request
│   │   │   ├── SecurityMockMvcRequestPostProcessors   # user(), csrf(), jwt(), oauth2Login()...
│   │   │   └── SecurityMockMvcRequestBuilders          # formLogin(), logout()
│   │   ├── response
│   │   │   ├── SecurityMockMvcResultMatchers           # authenticated(), unauthenticated()
│   │   │   └── SecurityMockMvcResultHandlers
│   │   └── setup
│   │       ├── SecurityMockMvcConfigurers             # springSecurity() 진입점
│   │       └── SecurityMockMvcConfigurer              # 필터 등록 + testSecurityContext() 적용
│   │
│   ├── reactive.server
│   │   └── SecurityMockServerConfigurers     # WebTestClient: springSecurity(), mockUser()...
│   │
│   └── support
│       └── WebTestUtils                      # 필터 체인에서 Repository/CsrfFilter 를 꺼내고 바꿔치기
│
└── aot.hint                                  # 네이티브 이미지(AOT) 리플렉션 힌트
    ├── WithSecurityContextTestRuntimeHints   # @With* factory 생성자 힌트 등록
    └── WebTestUtilsRuntimeHints
```

## 큰 그림: 두 갈래의 주입 경로

이 모듈을 이해하는 가장 빠른 길은 "보안 컨텍스트가 어디로 흘러야 하는가"를 갈래로 나누어 보는 것이다.

```
                         ┌──────────────────────────────────────────┐
   @WithMockUser 등      │  WithSecurityContextTestExecutionListener │
   (선언적)         ────▶│  beforeTestMethod() 에서 SecurityContext   │
                         │  를 만들어 TestSecurityContextHolder 에 set │
                         └───────────────┬──────────────────────────┘
                                         │
                  ┌──────────────────────┼───────────────────────┐
                  ▼                      ▼                       ▼
        (A) 메서드/단위 테스트   (B) MockMvc 요청          (C) 리액티브 테스트
        SecurityContextHolder    testSecurityContext()      ReactorContext-
        .getContext() 로 바로     RequestPostProcessor 가    TestExecutionListener
        읽힌다                    HttpServletRequest 에       가 Reactor Context 로
                                  컨텍스트를 실어 보냄         전파
```

(A) 는 01·02 챕터, (B) 는 03 챕터, (C) 는 04 챕터에서 다룬다. 한편 MockMvc/WebTestClient 에는 애너테이션을 거치지 않고 요청 단위로 직접 사용자를 지정하는 별도 경로(`user("...")`, `mockUser(...)`)도 있는데, 이는 03·04 챕터에서 설명한다.
