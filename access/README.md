# access 모듈 — 레거시 접근 제어 인프라

> Spring Security 7.1.1-SNAPSHOT 기준. 소스: `access/src/main/java`

## 한 줄 정의

`access` 모듈은 Spring Security의 **1세대 인가(authorization) 엔진**이다. "보안 객체(secure object)"를 가로채 `SecurityMetadataSource`로 권한 메타데이터를 조회하고, `AccessDecisionManager` + `AccessDecisionVoter`의 **투표(voting)** 로 접근 허용/거부를 결정하는 구조 전체를 담는다.

## 가장 먼저 알아야 할 사실 — 이 모듈은 통째로 deprecated 다

이 모듈의 거의 모든 공개 타입에는 `@Deprecated`가 붙어 있다. Spring Security는 6.x~7.x를 거치며 인가 엔진을 **`AuthorizationManager` 기반**으로 전면 교체했고, `access` 모듈은 그 이전 세대(Acegi Security 계보)의 코드다. 새 코드를 짤 때 쓰는 게 아니라, **기존 애플리케이션과 프레임워크 내부 동작을 이해하기 위해 읽는** 모듈이다.

| 레거시 (이 모듈)                                   | 현대 대체재 (다른 모듈)                                                  |
| -------------------------------------------------- | ------------------------------------------------------------------------ |
| `AccessDecisionManager` / `AccessDecisionVoter`    | `AuthorizationManager`                                                    |
| `FilterSecurityInterceptor`                        | `AuthorizationFilter`                                                     |
| `MethodSecurityInterceptor` (+ Pre/PostAdvice)     | `AuthorizationManagerBeforeMethodInterceptor` / `...AfterMethodInterceptor` |
| `ChannelSecurityInterceptor` (messaging)           | `AuthorizationChannelInterceptor`                                        |
| `ChannelProcessingFilter` (HTTP/HTTPS)             | `HttpsRedirectFilter`                                                    |
| `SecurityMetadataSource` / `ConfigAttribute`       | (각 API가 자체 설정 컨텍스트를 관리 — 직접 대응 타입 없음)               |

> 그럼에도 이 모듈을 읽을 가치가 있는 이유: `secure object` 추상화, "before/after invocation" 가로채기 패턴, 투표 기반 결정, run-as 권한 승격 같은 **개념의 원형**이 여기에 있고, 현대 API의 설계 동기를 역으로 이해하게 해준다.

## 이 모듈이 푸는 문제

"메서드 호출이든, HTTP 요청이든, WebSocket 메시지든 — **무엇이 호출되는지**와 **누가 호출하는지**를 알아내고, **그 호출을 허용할지** 결정한다." 이 한 문장을 일반화한 것이 `secure object`(보안 객체) 모델이다.

```
   보안 객체            메타데이터                  결정
   (무엇)               (어떤 권한 필요)            (허용/거부)
 ┌──────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
 │ MethodInvocation │ │ SecurityMetadataSource │  │ AccessDecisionManager  │
 │ FilterInvocation │─▶│  → ConfigAttribute[]  │─▶│  → Voter들이 투표      │
 │ Message<?>       │ │                        │  │  → GRANTED/DENIED      │
 └──────────────┘    └──────────────────────┘    └──────────────────────┘
        세 종류 모두 AbstractSecurityInterceptor 가 동일한 흐름으로 처리
```

세 가지 보안 객체 타입을 **하나의 추상 엔진**(`AbstractSecurityInterceptor`)으로 처리하는 것이 이 모듈의 설계 핵심이다.

## 의존 / 연관 모듈

- **`core`** — `Authentication`, `GrantedAuthority`, `SecurityContextHolder`, `AuthenticationManager`, `RoleHierarchy`, 그리고 현대 대체재인 `org.springframework.security.authorization.AuthorizationManager`. access의 모든 타입이 core 위에 선다.
- **`web`** — `FilterInvocation`, 서블릿 필터 인프라. `web/access/**` 패키지가 여기에 의존한다.
- **`messaging`** — `Message`, `ChannelInterceptor`. `messaging/access/**` 패키지가 사용.
- **`acl`** — ACL 기반 도메인 객체 권한. `acls/**` 패키지가 `AclService`, `Permission`을 사용.
- **AOP Alliance / Spring AOP** — `MethodInterceptor`, `MethodInvocation` (메서드 보안 가로채기).

## 하위 챕터 목차

1. [01-개요-secure-object와-핵심-추상.md](01-개요-secure-object와-핵심-추상.md) — secure object 모델, 6개 핵심 협력자 인터페이스, 모듈 전체 지형도
2. [02-AbstractSecurityInterceptor-가로채기-엔진.md](02-AbstractSecurityInterceptor-가로채기-엔진.md) — before/after/finally invocation 흐름, InterceptorStatusToken, run-as, 이벤트
3. [03-투표-기반-인가-AccessDecisionManager와-Voter.md](03-투표-기반-인가-AccessDecisionManager와-Voter.md) — Affirmative/Consensus/Unanimous, RoleVoter·AuthenticatedVoter·RoleHierarchyVoter
4. [04-메서드-보안-메타데이터와-어노테이션.md](04-메서드-보안-메타데이터와-어노테이션.md) — MethodSecurityMetadataSource, @Secured/@JSR250, @PreAuthorize/@PostAuthorize 표현식, 리액티브
5. [05-웹-채널-메시징-보안.md](05-웹-채널-메시징-보안.md) — FilterSecurityInterceptor, WebExpressionVoter, ChannelProcessingFilter, ChannelSecurityInterceptor
6. [06-run-as와-after-invocation-acl.md](06-run-as와-after-invocation-acl.md) — RunAsManager 권한 승격, AfterInvocationProviderManager, ACL 결과 필터링

## 전체 패키지 구조 ASCII 맵

```
access/src/main/java/org/springframework/security/
│
├── access/                         ── 핵심 추상 + 레거시 인가 엔진
│   ├── AccessDecisionManager        최종 인가 결정자 (deprecated → AuthorizationManager)
│   ├── AccessDecisionVoter          투표자 (GRANTED/DENIED/ABSTAIN)
│   ├── SecurityMetadataSource       보안 객체 → ConfigAttribute[] 조회
│   ├── ConfigAttribute              "ROLE_ADMIN" 같은 권한 메타데이터 단위
│   ├── SecurityConfig               ConfigAttribute의 기본 문자열 구현
│   ├── AfterInvocationProvider      반환값 후처리/필터링 제공자
│   │
│   ├── intercept/                   ── 가로채기 엔진의 심장
│   │   ├── AbstractSecurityInterceptor    before/after invocation 골격
│   │   ├── InterceptorStatusToken         호출 전후를 잇는 토큰
│   │   ├── AfterInvocationManager/...Manager  반환값 후처리 조율
│   │   ├── RunAsManager / RunAsManagerImpl    임시 권한 승격
│   │   ├── RunAsUserToken / RunAsImplAuthenticationProvider
│   │   ├── MethodInvocationPrivilegeEvaluator  사전 권한 검사 도우미
│   │   ├── aopalliance/             MethodSecurityInterceptor (+ Advisor)
│   │   └── aspectj/                 AspectJMethodSecurityInterceptor
│   │
│   ├── vote/                        ── 투표 집계 전략 + 투표자들
│   │   ├── AbstractAccessDecisionManager
│   │   ├── AffirmativeBased / ConsensusBased / UnanimousBased
│   │   ├── RoleVoter / RoleHierarchyVoter / AuthenticatedVoter
│   │   └── AbstractAclVoter
│   │
│   ├── method/                     ── 메서드 메타데이터 소스
│   │   ├── MethodSecurityMetadataSource (+ Abstract/AbstractFallback)
│   │   ├── DelegatingMethodSecurityMetadataSource
│   │   └── MapBasedMethodSecurityMetadataSource, P (파라미터 이름)
│   │
│   ├── annotation/                 ── @Secured, JSR-250(@RolesAllowed 등)
│   │   ├── SecuredAnnotationSecurityMetadataSource
│   │   └── Jsr250MethodSecurityMetadataSource / Jsr250Voter
│   │
│   ├── prepost/                    ── @PreAuthorize/@PostAuthorize/@PreFilter/@PostFilter
│   │   ├── Pre/PostInvocationAuthorizationAdvice (+ Voter, Provider)
│   │   ├── PrePostAnnotationSecurityMetadataSource
│   │   └── PrePostAdviceReactiveMethodInterceptor (Mono/Flux/코루틴)
│   │
│   ├── expression/method/          ── SpEL 기반 메서드 인가 표현식
│   │   ├── ExpressionBasedPre/PostInvocationAdvice
│   │   └── Pre/PostInvocationExpressionAttribute
│   │
│   └── event/                      ── 인가 결과 ApplicationEvent + LoggerListener
│
├── acls/                           ── ACL 기반 도메인 객체 인가 (레거시)
│   ├── AclEntryVoter
│   └── afterinvocation/            반환 객체/컬렉션을 ACL로 필터링
│       ├── AclEntryAfterInvocationProvider
│       ├── AclEntryAfterInvocationCollectionFilteringProvider
│       └── Filterer / CollectionFilterer / ArrayFilterer
│
├── messaging/access/              ── WebSocket/STOMP 메시지 인가 (레거시)
│   ├── intercept/ChannelSecurityInterceptor
│   └── expression/MessageExpressionVoter 등
│
└── web/access/                    ── 서블릿 HTTP 인가 (레거시)
    ├── intercept/FilterSecurityInterceptor (+ MetadataSource)
    ├── expression/WebExpressionVoter, DefaultWebSecurityExpressionHandler
    ├── channel/ChannelProcessingFilter, ChannelDecisionManager, *ChannelProcessor
    └── DefaultWebInvocationPrivilegeEvaluator
```

각 챕터를 순서대로 읽으면, "보안 객체를 가로채 → 메타데이터 조회 → 투표로 결정 → (선택) 권한 승격 → 실행 → 반환값 후처리"라는 하나의 큰 흐름이 메서드/웹/메시징/ACL 각 영역에서 어떻게 변주되는지 보이게 된다.
