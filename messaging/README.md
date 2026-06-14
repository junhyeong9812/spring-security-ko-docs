# messaging — 메시징(WebSocket/STOMP) 보안

> Spring Security 7.1.1-SNAPSHOT 기준 해설

## 한 줄 정의

`spring-security-messaging`은 Spring Messaging의 **`ChannelInterceptor`** 확장 지점에 올라타서, WebSocket/STOMP 메시지가 메시지 채널을 통과하는 순간 **인증 컨텍스트 전파 → 인가 → CSRF 검증**을 수행하는 모듈이다.

## 이 모듈이 푸는 문제

HTTP는 요청-응답이 한 번에 끝나지만, WebSocket은 한 번 연결(handshake)되면 그 위로 STOMP 프레임(CONNECT, SUBSCRIBE, SEND, DISCONNECT...)이 양방향으로 흐른다. 이때 두 가지 빈틈이 생긴다.

1. **인증의 단절** — 핸드셰이크 시점의 HTTP 세션 인증 정보가, 이후 비동기로 처리되는 개별 메시지 핸들러 스레드에는 자동으로 따라오지 않는다.
2. **인가의 부재** — `web` 모듈의 `AuthorizationFilter`는 HTTP 요청만 본다. "이 사용자가 `/topic/admin`을 SUBSCRIBE 할 수 있는가" 같은 **목적지(destination) 단위 인가**는 메시지 레벨에서 다시 판단해야 한다.
3. **CSRF** — WebSocket 핸드셰이크는 브라우저의 same-origin 정책 밖에 있어, CONNECT 프레임에 대해 별도의 CSRF 토큰 검증이 필요하다.

이 모듈은 Spring Messaging이 제공하는 `ChannelInterceptor`라는 단일 확장 지점에 보안 인터셉터들을 끼워 넣어, 이 세 문제를 메시지 파이프라인 안에서 해결한다. HTTP 필터 체인과 달리 **필터가 아니라 인터셉터**가 주역이라는 점이 핵심이다.

## 의존·연관 관계

- **Spring Messaging** (`spring-messaging`) — `Message`, `MessageChannel`, `ChannelInterceptor`, `SimpMessageHeaderAccessor`, `SimpMessageType` 등 메시징 추상화. 이 모듈은 그 위의 보안 레이어다.
- [`core`](../core/README.md) — `Authentication`, `SecurityContext`, `SecurityContextHolderStrategy`, `AuthorizationManager`, `AuthenticatedAuthorizationManager`, `AuthorityAuthorizationManager` 등 인증/인가 기본 타입을 그대로 재사용한다.
- [`web`](../web/README.md) — CSRF 토큰 타입(`CsrfToken`, `DeferredCsrfToken`, `MissingCsrfTokenException`)을 빌려 쓴다. WebSocket 핸드셰이크가 결국 HTTP 위에서 일어나기 때문이다.
- **Spring WebSocket** (`spring-websocket`) — `HandshakeInterceptor`. 핸드셰이크 단계에서 CSRF 토큰을 WebSocket 세션 속성으로 옮긴다.
- **config 모듈** — 실제 사용자는 `@EnableWebSocketSecurity` / `MessageSecurityMetadataSourceRegistry`(DSL)를 통해 이 모듈의 인터셉터들을 채널에 등록한다. 이 모듈 자체는 DSL을 포함하지 않고 "재료"만 제공한다.

## 내부 목차

1. [01-메시지-보안-아키텍처.md](01-메시지-보안-아키텍처.md) — 왜 필터가 아니라 `ChannelInterceptor`인가, 메시지 파이프라인 전체 그림과 인터셉터 실행 순서.
2. [02-인증-컨텍스트-전파.md](02-인증-컨텍스트-전파.md) — `SecurityContextChannelInterceptor`가 메시지 헤더의 인증 정보를 핸들러 스레드의 `SecurityContextHolder`로 옮기는 방법, `@AuthenticationPrincipal` 인자 해석.
3. [03-MessageMatcher.md](03-MessageMatcher.md) — 메시지를 무엇으로 식별하는가. `PathPatternMessageMatcher`, `SimpMessageTypeMatcher`, 조합 매처.
4. [04-메시지-인가.md](04-메시지-인가.md) — `AuthorizationChannelInterceptor` → `MessageMatcherDelegatingAuthorizationManager` → `AuthorizationManager` 인가 결정 흐름과 표현식 기반 인가.
5. [05-WebSocket-CSRF.md](05-WebSocket-CSRF.md) — 핸드셰이크 시 토큰 적재(`CsrfTokenHandshakeInterceptor`)와 CONNECT 프레임 검증(`CsrfChannelInterceptor`, `XorCsrfChannelInterceptor`).

## 전체 패키지 구조 ASCII 맵

```
messaging/src/main/java/org/springframework/security/messaging/
│
├── context/                          [02장] 인증 컨텍스트 전파
│   ├── SecurityContextChannelInterceptor          헤더의 user → SecurityContextHolder
│   ├── SecurityContextPropagationChannelInterceptor  현재 컨텍스트 → 헤더 → 다른 스레드
│   └── AuthenticationPrincipalArgumentResolver    @AuthenticationPrincipal 해석(서블릿)
│
├── util/matcher/                     [03장] 메시지 식별
│   ├── MessageMatcher (interface)                 매칭 API + MatchResult
│   ├── PathPatternMessageMatcher                  destination 패턴 매칭(+변수 추출)
│   ├── SimpMessageTypeMatcher                     SimpMessageType(CONNECT/SUBSCRIBE...) 매칭
│   ├── AbstractMessageMatcherComposite
│   ├── AndMessageMatcher / OrMessageMatcher       조합 매처
│
├── access/                           [04장] 인가
│   ├── intercept/
│   │   ├── AuthorizationChannelInterceptor        preSend 훅에서 인가 수행(default deny)
│   │   ├── MessageMatcherDelegatingAuthorizationManager  매처→매니저 매핑 + Builder DSL
│   │   └── MessageAuthorizationContext            매칭된 메시지 + 추출 변수
│   └── expression/
│       ├── MessageExpressionAuthorizationManager  SpEL 표현식 기반 인가(7.1)
│       ├── MessageAuthorizationContextSecurityExpressionHandler
│       ├── DefaultMessageSecurityExpressionHandler
│       └── MessageSecurityExpressionRoot          표현식 루트 객체
│
├── web/                              [05장] CSRF
│   ├── csrf/
│   │   ├── CsrfChannelInterceptor                 CONNECT 프레임 토큰 검증
│   │   ├── XorCsrfChannelInterceptor              마스킹된 토큰 검증(상수시간 비교)
│   │   └── XorCsrfTokenUtils                       XOR 토큰 디코딩
│   └── socket/server/
│       └── CsrfTokenHandshakeInterceptor          핸드셰이크 시 토큰을 세션 속성으로 적재
│
└── handler/invocation/reactive/      [02장 보론] 리액티브 인자 해석
    ├── AuthenticationPrincipalArgumentResolver    @AuthenticationPrincipal (RSocket/리액티브)
    └── CurrentSecurityContextArgumentResolver     @CurrentSecurityContext
```

위에서 아래로 읽으면 "한 메시지가 들어왔을 때"의 시간 순서와 대략 일치한다: 핸드셰이크에서 CSRF 토큰이 적재되고(web), 인증이 컨텍스트로 전파되며(context), 매처로 식별되어(matcher) 인가되고(access), 핸들러에서 `@AuthenticationPrincipal`이 풀린다.
