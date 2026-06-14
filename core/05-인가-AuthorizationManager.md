# 05. 인가 — AuthorizationManager 결정 모델

## 무엇을 / 왜

인증(Authentication)이 "당신은 누구인가"를 답한다면, 인가(Authorization)는 "당신이 **이것을** 해도 되는가"를 답한다. 03·04장에서 만든 `Authentication`이 `SecurityContext`에 저장된 뒤, 보호된 자원(URL, 메서드, 도메인 객체)에 접근할 때마다 "허용/거부"를 판단하는 엔진이 필요하다. 그 엔진이 `AuthorizationManager`다.

Spring Security 5.5부터 인가 모델은 과거의 `AccessDecisionManager`/`Voter` 구조에서 **단일 함수형 인터페이스 `AuthorizationManager`** 로 재편되었다. 이 장은 그 결정 모델의 구조와 대표 구현, 권한·역할·역할 계층을 다룬다.

소스: `core/src/main/java/org/springframework/security/authorization/`, `.../access/hierarchicalroles/`

## 핵심 타입

```
 ┌──────────────────────────────────────────────────────────┐
 │ AuthorizationManager<T>  (@FunctionalInterface)            │
 │   AuthorizationResult authorize(Supplier<Authentication>, T)│  ← 핵심 결정 메서드
 │   default void verify(...) { 거부면 AuthorizationDeniedException }│
 └──────────────────────────┬───────────────────────────────┘
                            │ 반환
                            ▼
 ┌────────────────────────────────────┐
 │ AuthorizationResult  isGranted():boolean│
 │   └ AuthorizationDecision (granted)     │
 │       ├ AuthorityAuthorizationDecision  │
 │       └ ExpressionAuthorizationDecision │
 └────────────────────────────────────┘

 구현체들:
   AuthenticatedAuthorizationManager   ── 로그인 여부 (authenticated/fullyAuth/rememberMe/anonymous)
   AuthorityAuthorizationManager       ── 특정 권한/역할 보유 (hasRole/hasAuthority)
   AuthoritiesAuthorizationManager     ── 권한 집합 중 하나라도 보유
   AuthorizationManagers.{anyOf,allOf,not} ── 조합기
   (web/method 모듈의 표현식·요청 매처 기반 매니저들)
```

### AuthorizationManager — 함수형 결정자

핵심 메서드 하나(`authorize`)와 편의 메서드 하나(`verify`)로 이루어진 함수형 인터페이스다.

```java
@Nullable AuthorizationResult authorize(Supplier<? extends Authentication> authentication, T object);

default void verify(Supplier<? extends Authentication> authentication, T object) {
    AuthorizationResult result = authorize(authentication, object);
    if (result != null && !result.isGranted())
        throw new AuthorizationDeniedException("Access Denied", result);
}
```

두 가지 설계 결정이 눈에 띈다.

**1) `Supplier<Authentication>`로 받는다.** `Authentication` 자체가 아니라 공급자다. 인가 판단이 실제로 인증 객체를 필요로 할 때까지(예: `permitAll`은 인증을 보지도 않음) `SecurityContext` 조회를 **미룰 수 있다**. 02장의 지연 컨텍스트(`getDeferredContext`)와 같은 철학이다.

**2) `authorize`는 `@Nullable`을 반환한다.** `null`은 "**기권(abstain)**"을 의미한다 — "나는 이 건에 대해 판단하지 않겠다". `false`(거부)와 `null`(기권)은 다르며, 이 구분이 조합기(`anyOf`/`allOf`)의 동작을 좌우한다.

`verify()`는 결과가 거부일 때 `AuthorizationDeniedException`을 던지는 편의 래퍼다. 필터·인터셉터는 보통 `authorize()`를 직접 호출해 결과를 이벤트로 발행한 뒤 거부를 처리한다(06장).

### AuthorizationResult / AuthorizationDecision

`AuthorizationResult`(6.3+)는 `isGranted()` 하나짜리 인터페이스이고, 기본 구현 `AuthorizationDecision`은 boolean 하나를 감싼다. 서브타입이 추가 맥락을 싣는다.

- **`AuthorityAuthorizationDecision`** — 어떤 권한을 요구했는지(`Collection<GrantedAuthority>`)를 함께 담아, 거부 사유를 설명 가능하게 한다.
- **`ExpressionAuthorizationDecision`** — 평가된 SpEL 표현식을 담는다.
- **`AuthorizationManagers`의 `CompositeAuthorizationDecision`/`NotAuthorizationDecision`** — 조합 결과의 구성 요소를 담는다.

과거 `AccessDecisionManager`가 단순 boolean/예외였던 것에 비해, "왜 거부됐는가"를 결과 객체에 실어 관측·디버깅을 돕는 방향으로 진화했다.

## 동작 흐름 — 권한 검사의 실제

### AuthorityAuthorizationManager.hasRole("ADMIN")

가장 흔한 시나리오. `hasRole("ADMIN")`은 내부적으로 `"ROLE_"` 접두사를 붙여 `hasAuthority("ROLE_ADMIN")`로 위임한다.

```java
public static <T> AuthorityAuthorizationManager<T> hasRole(String role) {
    Assert.isTrue(!role.startsWith(ROLE_PREFIX), ...);   // "ROLE_" 직접 붙이면 거부
    return hasAuthority(ROLE_PREFIX + role);             // "ROLE_" 자동 prepend
}
```

`AuthorityAuthorizationManager`는 실제 판단을 `AuthoritiesAuthorizationManager`에 위임하고, 그것이 권한 집합 비교를 수행한다.

```
authorize(authSupplier, object)
   │
   ▼  AuthorityAuthorizationManager.authorize
   delegate.authorize(authSupplier, {"ROLE_ADMIN"})    ← 요구 권한 집합 전달
   │
   ▼  AuthoritiesAuthorizationManager.isAuthorized
   for ga in roleHierarchy.getReachableGrantedAuthorities(auth.getAuthorities()):
       if requiredAuthorities.contains(ga.getAuthority()):
           return GRANTED                                ← 하나라도 일치하면 허용
   return DENIED
   │
   ▼
   new AuthorityAuthorizationDecision(granted, [ROLE_ADMIN])
```

여기서 중요한 점: 사용자의 권한을 그냥 쓰지 않고 **`roleHierarchy.getReachableGrantedAuthorities()`로 확장**한 뒤 비교한다. 역할 계층이 설정돼 있으면 상위 역할이 하위 역할 권한을 자동으로 포함한다(아래 참고).

### AuthenticatedAuthorizationManager — 로그인 상태 판단

"로그인했는가"를 묻는 매니저다. 단순히 `authenticated` 플래그만 보는 게 아니라 `AuthenticationTrustResolver`로 **인증의 질**을 구분한다.

```
authenticated()      → trustResolver.isAuthenticated(auth)      (익명이 아니면 OK)
fullyAuthenticated() → trustResolver.isFullyAuthenticated(auth) (remember-me 제외, 진짜 로그인)
rememberMe()         → trustResolver.isRememberMe(auth)
anonymous()          → trustResolver.isAnonymous(auth)
```

전략 패턴으로 네 가지를 구현한다. 예를 들어 "비밀번호 변경 페이지는 remember-me로는 안 되고 방금 로그인한 사용자만"이라는 정책을 `fullyAuthenticated()`로 표현한다.

## 조합기 — AuthorizationManagers

여러 판단을 묶을 때 `AuthorizationManagers`의 정적 메서드를 쓴다. 기권(`null`) 처리가 핵심이다.

```
anyOf(m1, m2, ...)  → 하나라도 GRANTED면 즉시 허용
                       모두 기권이면 기본값(allAbstainDefaultDecision, 기본 거부)
                       그 외엔 CompositeAuthorizationDecision(false)

allOf(m1, m2, ...)  → 하나라도 DENIED면 즉시 거부
                       모두 기권이면 기본값(기본 허용)
                       그 외엔 CompositeAuthorizationDecision(true)

not(m)              → m의 결정을 뒤집음 (기권은 그대로 기권)
```

`anyOf`의 코드를 보면, `result == null`(기권)이면 `continue`로 건너뛰고, `isGranted()`면 즉시 그 결과를 반환한다. 이 "기권은 무시, 첫 명시적 허용에서 멈춤" 규칙이 조합 의미론의 핵심이다.

## 역할 계층 — RoleHierarchy

`ADMIN`이 `USER`의 모든 권한을 자동으로 갖게 하려면, 매번 두 역할을 다 부여하는 대신 **계층**을 선언한다.

```java
RoleHierarchy h = RoleHierarchyImpl.fromHierarchy("ROLE_ADMIN > ROLE_USER");
```

`RoleHierarchy.getReachableGrantedAuthorities(authorities)`는 주어진 권한 집합에서 **계층상 도달 가능한 모든 권한**을 펼쳐 반환한다. 위 선언이면 `ROLE_ADMIN`을 가진 사용자에게 `getReachableGrantedAuthorities`는 `{ROLE_ADMIN, ROLE_USER}`를 돌려준다. 그래서 `AuthoritiesAuthorizationManager`가 `ROLE_USER`를 요구해도 `ADMIN` 사용자가 통과한다.

기본값은 `NullRoleHierarchy`(계층 없음 — 권한을 그대로 반환)다. 즉 역할 계층은 **옵트인** 기능이고, 인가 매니저에 `setRoleHierarchy()`로 주입한다.

## 설계 포인트 / 확장점

- **단일 함수형 인터페이스로 통합:** 과거 `AccessDecisionManager` + 여러 `AccessDecisionVoter`의 복잡한 합산 로직을, 하나의 `AuthorizationManager`와 명시적 조합기(`anyOf`/`allOf`/`not`)로 단순화했다. 커스텀 인가 규칙은 람다 한 줄로 만들 수 있다.
- **기권(`null`) 시맨틱:** "판단 안 함"을 1급 개념으로 만들어, 여러 매니저를 안전하게 합성한다.
- **지연 인증 공급:** `Supplier<Authentication>`로 불필요한 컨텍스트 조회를 피한다.
- **결과에 맥락 부여:** `AuthorizationResult` 서브타입으로 거부 사유를 설명 가능하게 해 관측성을 높인다.
- **리액티브 짝:** `ReactiveAuthorizationManager`가 `Mono<AuthorizationResult>`를 반환하는 대응을 제공하며, `Authenticated`/`Authority` 등 동일 라인업이 리액티브 버전으로 존재한다.
- **AuthorizationManagerFactory (7.x):** `AuthorizationManagerFactory`/`DefaultAuthorizationManagerFactory`는 `hasRole`/`authenticated` 같은 표준 매니저를 일관된 정책(예: 공통 역할 접두사·trust resolver)으로 찍어내는 팩토리다. `RequiredFactor`/`AllRequiredFactorsAuthorizationManager` 등은 다중 인증 요소(MFA) 기반 인가를 표현한다.

## 정리

`AuthorizationManager<T>`는 "(공급된) 인증과 보호 대상 객체를 받아 허용/거부/기권을 결정한다"는 단일 함수형 계약이다. `AuthorityAuthorizationManager`(권한·역할), `AuthenticatedAuthorizationManager`(로그인 상태)가 대표 구현이고, `AuthorizationManagers.anyOf/allOf/not`이 이들을 합성하며, `RoleHierarchy`가 역할 간 포함 관계를 펼친다. 판단 결과는 `AuthorizationResult`로 표현되어 거부 사유까지 싣는다. 이 결정 모델이 URL 보안(web 모듈)과 메서드 보안(다음 06장)의 공통 기반이다.
