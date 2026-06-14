# web — 서블릿 웹 보안의 본체

> Spring Security 7.1.1-SNAPSHOT 기준. 소스: `web/src/main/java/org/springframework/security/web`

## 한 줄 정의

`web` 모듈은 **하나의 서블릿 `Filter`(`FilterChainProxy`) 뒤에 보안 필터들을 줄세워, HTTP 요청이 통과하는 동안 인증·인가·세션·CSRF·헤더 보호를 수행하는 서블릿 보안 엔진**이다.

## 이 모듈이 푸는 문제

웹 애플리케이션은 매 요청마다 "이 사람이 누구인지(인증)", "이 자원에 접근해도 되는지(인가)"를 결정해야 하고, 그 과정에서 세션 고정 공격·CSRF·클릭재킹 같은 위협을 막아야 한다. 이 모든 횡단 관심사를 컨트롤러마다 직접 짜는 것은 비현실적이다.

Spring Security는 이를 **서블릿 필터 체인**으로 푼다. 서블릿 컨테이너 입장에서는 보안 전체가 단 하나의 필터(`DelegatingFilterProxy` → `FilterChainProxy`)로 보이고, 그 안에서 책임별로 잘게 나뉜 보안 필터들이 순서대로 실행된다. 각 필터는 한 가지 일만 한다: `SecurityContextHolderFilter`는 인증 정보를 로드하고, `UsernamePasswordAuthenticationFilter`는 폼 로그인을 처리하고, `AuthorizationFilter`는 접근을 허가/거부하고, `ExceptionTranslationFilter`는 보안 예외를 HTTP 응답으로 번역한다. 이 분업과 순서가 곧 Spring Security 웹 보안의 본질이다.

```
   서블릿 컨테이너
        │
   DelegatingFilterProxy ("springSecurityFilterChain")
        │
   ┌────▼─────────────────────────────────────────────┐
   │  FilterChainProxy                                  │
   │   1) HttpFirewall 로 요청/응답 래핑·검증           │
   │   2) 매칭되는 SecurityFilterChain 한 개 선택       │
   │   3) 그 체인의 Filter 들을 VirtualFilterChain 으로 │
   │      순서대로 실행 → 끝나면 원래 서블릿 체인 복귀  │
   └────────────────────────────────────────────────────┘
```

## 의존 / 연관 관계

- **의존(아래에서 위로 사용)**
  - `core` — `Authentication`, `SecurityContext`, `SecurityContextHolder`, `AuthenticationManager`, `GrantedAuthority` 등 인증 도메인 타입.
  - `authorization`(core 내) — `AuthorizationManager`, `AuthorizationResult` 등 인가 결정 추상화. `AuthorizationFilter`가 이를 호출한다.
  - `crypto` — CSRF 토큰 비교, remember-me 토큰 처리 등.
  - Spring `spring-web`, `jakarta.servlet` — 필터/요청/응답 API, `PathPattern`.
- **연관(이 모듈을 엮어 쓰는 쪽)**
  - `config` — `HttpSecurity` DSL과 각종 `Configurer`가 이 모듈의 필터/핸들러 인스턴스를 조립해 `FilterChainProxy` 빈을 만든다. **즉 이 모듈은 "부품"이고, 조립도는 config가 그린다.**
  - `oauth2`, `saml2`, `web/server`(리액티브) 등은 같은 패턴을 자기 프로토콜/런타임에 맞춰 확장한다.

## 하위 챕터

1. [01-필터체인-아키텍처.md](01-필터체인-아키텍처.md) — `FilterChainProxy`, `SecurityFilterChain`, `VirtualFilterChain`, `HttpFirewall`. 요청이 어떻게 보안 체인에 진입하는가.
2. [02-보안컨텍스트-영속화.md](02-보안컨텍스트-영속화.md) — `SecurityContextHolderFilter`, `SecurityContextRepository`와 그 구현들. 인증 정보가 요청 간에 어떻게 유지되는가.
3. [03-인증-필터.md](03-인증-필터.md) — `AbstractAuthenticationProcessingFilter`, 폼 로그인, HTTP Basic, `AuthenticationConverter`, success/failure 핸들러, remember-me, anonymous.
4. [04-예외변환과-인가.md](04-예외변환과-인가.md) — `ExceptionTranslationFilter`, `AuthenticationEntryPoint`, `AccessDeniedHandler`, `AuthorizationFilter`, `RequestMatcherDelegatingAuthorizationManager`, `RequestCache`.
5. [05-로그아웃과-세션관리.md](05-로그아웃과-세션관리.md) — `LogoutFilter`/`LogoutHandler`, `SessionManagementFilter`, `SessionAuthenticationStrategy`(세션 고정 보호·동시 세션 제어).
6. [06-CSRF와-보안헤더.md](06-CSRF와-보안헤더.md) — `CsrfFilter`/`CsrfTokenRepository`, `HeaderWriterFilter`와 `HeaderWriter` 들.
7. [07-RequestMatcher와-요청매칭.md](07-RequestMatcher와-요청매칭.md) — `RequestMatcher`, `PathPatternRequestMatcher`, 결합자(And/Or/Negated), 매칭이 보안 전반에서 어떻게 재사용되는가.

## 전체 패키지 구조 (ASCII 맵)

```
org.springframework.security.web
├── FilterChainProxy            ← 진입점 필터. 체인 선택 + firewall + 실행
├── SecurityFilterChain         ← (matcher, filters) 묶음 인터페이스
├── DefaultSecurityFilterChain  ← 표준 구현
├── AuthenticationEntryPoint    ← 미인증 시 인증을 "개시"하는 훅
├── RedirectStrategy / PortMapper / WebAttributes
│
├── context/                    ← SecurityContext 영속화 (2장)
│   ├── SecurityContextHolderFilter
│   ├── SecurityContextRepository (인터페이스)
│   ├── HttpSessionSecurityContextRepository
│   ├── RequestAttributeSecurityContextRepository
│   └── DelegatingSecurityContextRepository
│
├── authentication/             ← 인증 필터·핸들러 (3장)
│   ├── AbstractAuthenticationProcessingFilter
│   ├── UsernamePasswordAuthenticationFilter
│   ├── AuthenticationFilter / AuthenticationConverter
│   ├── AnonymousAuthenticationFilter
│   ├── *SuccessHandler / *FailureHandler
│   ├── www/  (BasicAuthenticationFilter, Digest...)
│   ├── rememberme/ (RememberMeServices, Token/Persistent...)
│   ├── logout/ (LogoutFilter, LogoutHandler...)   ← 5장
│   ├── session/ (SessionAuthenticationStrategy...) ← 5장
│   ├── preauth/, switchuser/, ott/, ui/
│
├── access/                     ← 예외 변환 + URL 인가 (4장)
│   ├── ExceptionTranslationFilter
│   ├── AccessDeniedHandler / AccessDeniedHandlerImpl
│   └── intercept/
│       ├── AuthorizationFilter
│       └── RequestMatcherDelegatingAuthorizationManager
│
├── session/                    ← 세션 라이프사이클 필터 (5장)
│   ├── SessionManagementFilter
│   ├── ConcurrentSessionFilter
│   └── HttpSession*Event / Publisher
│
├── csrf/                       ← CSRF 보호 (6장)
│   ├── CsrfFilter
│   └── CsrfTokenRepository (HttpSession/Cookie...)
│
├── header/                     ← 보안 응답 헤더 (6장)
│   ├── HeaderWriterFilter
│   └── writers/ (Hsts, ContentSecurityPolicy, XFrameOptions...)
│
├── savedrequest/               ← 인증 후 원래 요청 복원 (4장)
│   └── RequestCache / HttpSessionRequestCache / SavedRequest
│
├── firewall/                   ← 악성 요청 차단 (1장)
│   ├── HttpFirewall / StrictHttpFirewall
│   └── RequestRejectedHandler
│
├── util/matcher/               ← RequestMatcher 와 결합자 (7장)
├── servlet/util/matcher/       ← PathPatternRequestMatcher (7장)
│
├── server/                     ← 리액티브(WebFlux) 대응판. 본 챕터 범위 밖
├── reactive/, jackson2/, aot/  ← 부가(직렬화, 네이티브 힌트 등)
```

> 본 챕터는 **서블릿(블로킹) 보안**에 집중한다. `server/`·`reactive/` 패키지는 동일한 개념을 WebFlux 런타임에 옮긴 별도 계열이라 여기서는 다루지 않는다.
