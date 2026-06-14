# 03. 인증 처리 — AuthenticationManager와 ProviderManager

## 무엇을 / 왜

01장에서 `Authentication`이 "요청 토큰 → 결과 토큰"으로 변신한다고 했다. 그 변신을 **실제로 수행하는 엔진**이 `AuthenticationManager`다. 자격 증명을 검증하고, 맞으면 권한이 채워진 신뢰 토큰을 돌려주며, 틀리면 예외를 던진다.

문제는 인증 방식이 한둘이 아니라는 점이다 — username/password, remember-me 쿠키, 익명, OAuth2, LDAP, JWT…. 이 모두를 한 클래스에 넣을 수는 없다. 그래서 Spring Security는 **요청을 여러 `AuthenticationProvider`에게 차례로 넘기는** 위임 구조를 택한다. 이 장은 그 핵심인 `ProviderManager`를 해부한다.

소스: `core/src/main/java/org/springframework/security/authentication/`

## 핵심 타입

```
   ┌──────────────────────────────────┐
   │ AuthenticationManager (인터페이스)  │   authenticate(Authentication): Authentication
   └─────────────────┬────────────────┘
                     │ 대표 구현
                     ▼
   ┌──────────────────────────────────┐        ┌────────────────────────────┐
   │ ProviderManager                   │───────▶│ AuthenticationEventPublisher│ (성공/실패 이벤트)
   │  - List<AuthenticationProvider>   │        └────────────────────────────┘
   │  - parent: AuthenticationManager  │
   └─────────────────┬────────────────┘
                     │ 순서대로 시도
        ┌────────────┼─────────────┬───────────────┐
        ▼            ▼             ▼               ▼
  DaoAuthentication  Anonymous   RememberMe    (커스텀)
  Provider           Authentication            Provider
  (→ 04장)           Provider     Provider
        each: supports(Class) + authenticate(Authentication)
```

### AuthenticationManager — 한 메서드짜리 계약

`AuthenticationManager`는 `@FunctionalInterface`로, 메서드가 하나다.

```java
Authentication authenticate(Authentication authentication) throws AuthenticationException;
```

인터페이스 주석은 **예외 계약**을 엄격히 규정한다 — 계정이 비활성/잠금이면 `DisabledException`/`LockedException`을, 자격 증명이 틀리면 `BadCredentialsException`을 던지고, 이 순서를 지켜야 한다(잠긴 계정은 비밀번호를 검증하기도 전에 거부). 반환값은 "권한까지 채워진 완전한 인증 객체"다.

### AuthenticationProvider — 방식별 검증 단위

`AuthenticationProvider`도 두 메서드뿐이다.

```java
@Nullable Authentication authenticate(Authentication authentication);  // 검증 (못 하면 null)
boolean supports(Class<?> authentication);                              // 이 토큰 타입을 다룰 수 있나
```

`supports()`가 핵심 분기점이다. `ProviderManager`는 이 메서드로 "이 토큰을 처리할 자격이 있는 provider"만 고른다. `authenticate()`가 `null`을 반환하면 "내가 다룰 수 없으니 다음 provider를 시도하라"는 뜻이고, 예외를 던지면 "다뤘지만 실패"라는 뜻이다. 이 둘의 구분이 체인 동작을 지배한다.

대표 provider: `DaoAuthenticationProvider`(04장), `AnonymousAuthenticationProvider`(익명 토큰 키 검증), `RememberMeAuthenticationProvider`(remember-me 키 검증).

## 동작 흐름 — ProviderManager.authenticate()의 내부

`ProviderManager.authenticate()`는 이 모듈에서 가장 중요한 메서드 중 하나다. 단계별로 보자.

```
authenticate(authentication)
  │  toTest = authentication.getClass()
  │
  ├─[1] for each provider:
  │       if (!provider.supports(toTest)) continue;    ← 자격 없는 provider 건너뜀
  │       try {
  │           result = provider.authenticate(authentication);
  │           if (result != null) { copyDetails(...); break; }  ← 성공! 루프 탈출
  │       }
  │       catch (AccountStatusException | InternalAuthenticationServiceException ex) {
  │           prepareException(ex, ...); throw ex;       ← 즉시 중단 (SEC-546)
  │       }
  │       catch (AuthenticationException ex) {
  │           lastException = ex;                        ← 기록만 하고 다음 provider 시도
  │       }
  │
  ├─[2] if (result == null && parent != null):
  │       parentResult = parent.authenticate(...)        ← 부모 매니저에 위임
  │
  ├─[3] if (result != null):
  │       if (eraseCredentials && result instanceof CredentialsContainer)
  │           result.eraseCredentials();                 ← 비밀번호 메모리에서 소거
  │       if (parentResult == null)
  │           eventPublisher.publishAuthenticationSuccess(result);
  │       return result;                                 ← 성공 반환
  │
  └─[4] // 아무도 성공 못 함
        if (lastException == null) lastException = new ProviderNotFoundException(...);
        if (parentException == null) prepareException(lastException, ...);  ← 실패 이벤트
        throw lastException;
```

### 체인 동작의 규칙 4가지

**1) 첫 성공이 이긴다.** 한 provider가 non-null을 반환하면 `break`로 즉시 멈춘다. 앞선 provider들이 던진 예외(`lastException`)는 무시된다. 즉 "한 명이라도 인증에 성공하면 통과".

**2) 계정 상태 예외는 즉시 중단.** `AccountStatusException`(계정 잠김/만료 등)과 `InternalAuthenticationServiceException`(내부 오류)은 catch 후 **바로 throw**한다(주석의 SEC-546). 비활성 계정인데 다른 provider가 우연히 통과시키는 일을 막는다.

**3) 일반 인증 실패는 기록 후 계속.** `BadCredentialsException` 같은 일반 실패는 `lastException`에 담고 다음 provider를 시도한다. 모두 실패하면 마지막 예외를 던진다.

**4) parent 폴백.** 자식 provider 중 누구도 처리 못 하면 부모 `AuthenticationManager`에 위임한다. 이는 전역 매니저 ↔ 로컬 매니저 계층(네임스페이스 설정)을 위한 장치다. 부모가 `ProviderNotFoundException`을 던지면 무시하고 자식의 결과/예외를 우선한다.

### 인증 직후 처리 두 가지

성공 시 `ProviderManager`는 두 가지 부가 작업을 한다.

- **자격 증명 소거:** `eraseCredentialsAfterAuthentication`(기본 true)이면 결과 토큰의 `eraseCredentials()`를 호출해 비밀번호를 메모리에서 지운다(01장의 `CredentialsContainer`). 보안 위생.
- **details 복사:** `copyDetails()`로 요청 토큰의 details(IP·세션 등)를 결과 토큰에 옮긴다(결과에 details가 없을 때만).

## 이벤트 발행 — DefaultAuthenticationEventPublisher

`ProviderManager`는 인증 성공/실패를 `AuthenticationEventPublisher`로 알린다. 기본 주입은 아무것도 안 하는 `NullEventPublisher`라서, 이벤트를 받으려면 명시적으로 `DefaultAuthenticationEventPublisher`를 주입해야 한다(웹 네임스페이스 설정에서는 자동 적용).

`DefaultAuthenticationEventPublisher`의 영리한 점은 **예외 타입을 이벤트 타입으로 매핑**한다는 것이다.

```
BadCredentialsException     → AuthenticationFailureBadCredentialsEvent
LockedException             → AuthenticationFailureLockedEvent
DisabledException           → AuthenticationFailureDisabledEvent
CredentialsExpiredException → AuthenticationFailureCredentialsExpiredEvent
... (매핑에 없으면) GenericAuthenticationFailureEvent
(성공)                       → AuthenticationSuccessEvent
```

이 이벤트들은 Spring의 `ApplicationEventPublisher`로 발행되어, 로그인 시도 감사·계정 잠금(brute-force 방어) 같은 기능을 `@EventListener`로 구현할 수 있게 한다. parent와 자식이 둘 다 이벤트를 내면 중복되므로, `ProviderManager`는 `parentResult`/`parentException` 체크로 중복 발행을 막는다.

## 리액티브·관측 변형

- **`ReactiveAuthenticationManager`:** `Mono<Authentication> authenticate(...)`를 반환하는 리액티브 대응. `DelegatingReactiveAuthenticationManager`가 여러 매니저를 순차 시도하는 `ProviderManager`의 리액티브 짝이다.
- **`AuthenticationManagerResolver`:** 요청(컨텍스트)에 따라 **어떤 `AuthenticationManager`를 쓸지** 동적으로 고르는 전략. 멀티 테넌시(테넌트별 다른 인증 매니저)에 쓰인다.
- **`ObservationAuthenticationManager`:** 다른 매니저를 감싸 Micrometer 관측 스팬을 남기는 데코레이터.

## 설계 포인트 / 확장점

- **Chain of Responsibility:** `ProviderManager`는 책임 연쇄 패턴의 교과서적 적용이다. `supports()`로 자격을 묻고, 첫 성공에서 멈춘다. 새 인증 방식은 `AuthenticationProvider`를 하나 추가하면 끝.
- **함수형 인터페이스:** `AuthenticationManager`가 단일 메서드라, 람다나 간단한 위임 구현으로 대체하기 쉽다.
- **이벤트 훅:** `AuthenticationEventPublisher`를 통해 인증 결과에 횡단 관심사(감사·잠금·메트릭)를 붙인다.
- **parent 계층:** 글로벌 `AuthenticationManager` 아래에 모듈별 자식 매니저를 두는 구성을 지원.

## 정리

`AuthenticationManager.authenticate()`는 "요청 토큰을 신뢰 토큰으로 바꾸거나 예외를 던진다"는 단일 계약이고, 대표 구현 `ProviderManager`는 이를 **여러 `AuthenticationProvider`에 위임하는 책임 연쇄**로 실현한다. `supports()`로 처리 자격을 가리고, 첫 성공에서 멈추며, 계정 상태 예외는 즉시 중단하고, 성공 시 자격 증명을 소거하고 이벤트를 발행한다. 가장 흔한 provider인 `DaoAuthenticationProvider`가 username/password를 어떻게 검증하는지는 다음 04장에서 `UserDetailsService`와 함께 본다.
