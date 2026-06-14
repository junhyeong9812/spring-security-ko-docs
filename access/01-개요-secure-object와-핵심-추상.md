# 01. 개요 — secure object 모델과 6개 핵심 추상

## 무엇을 / 왜

`access` 모듈을 관통하는 단 하나의 아이디어는 **"보안 객체(secure object)"** 다. 메서드 호출, HTTP 요청, 메시지 전송처럼 "보호하고 싶은 어떤 작업"을 전부 `Object` 하나로 추상화하고, 그 위에 동일한 인가 절차를 적용한다. 이 챕터는 그 추상화를 떠받치는 6개의 협력자 인터페이스와, 그것들이 어떻게 맞물리는지를 먼저 잡는다. 이후 챕터들은 이 골격의 각 부위를 확대한다.

> 모든 타입이 `@Deprecated`임을 다시 기억하자. 현대 코드는 `AuthorizationManager` 한 인터페이스로 이 6개의 역할 대부분을 흡수했다. 이 챕터는 "왜 그렇게 단순화될 수 있었는가"를 이해하기 위한 출발점이다.

## 핵심 타입 — 인가에 참여하는 6개 협력자

```
                       AbstractSecurityInterceptor (조율자)
                                  │
   ┌───────────────┬─────────────┼──────────────┬───────────────┐
   ▼               ▼             ▼              ▼               ▼
SecurityMetadata  Authentication AccessDecision  RunAsManager   AfterInvocation
Source            Manager        Manager                        Manager
(무엇이 보호?)    (재인증)       (허용/거부)    (권한 승격)     (반환값 후처리)
   │                                │
   ▼                                ▼
ConfigAttribute[]              AccessDecisionVoter[] (투표)
```

| 인터페이스                | 책임                                                                 | 소스 |
| ------------------------- | -------------------------------------------------------------------- | ---- |
| `SecurityMetadataSource`  | 보안 객체 → 필요한 `ConfigAttribute` 목록 조회                       | `access/SecurityMetadataSource.java` |
| `ConfigAttribute`         | "어떤 권한이 필요한가"의 최소 단위 (예: `ROLE_ADMIN`, SpEL 표현식)   | `access/ConfigAttribute.java` |
| `AccessDecisionManager`   | 최종 허용/거부 결정 (`decide()`가 예외 안 던지면 통과)               | `access/AccessDecisionManager.java` |
| `AccessDecisionVoter<S>`  | 개별 투표자. `ACCESS_GRANTED(1)/ABSTAIN(0)/DENIED(-1)` 반환          | `access/AccessDecisionVoter.java` |
| `RunAsManager`            | 호출 동안 임시로 다른 권한을 가진 `Authentication`으로 승격 (선택)   | `access/intercept/RunAsManager.java` |
| `AfterInvocationManager`  | 실행 후 반환값을 검사/필터링/교체 (선택)                            | `access/intercept/AfterInvocationManager.java` |

### ConfigAttribute — 권한 메타데이터의 원자

`ConfigAttribute`는 `Serializable`을 상속한 거의 빈 인터페이스로, 메서드는 `getAttribute()` 하나뿐이다.

```java
// access/ConfigAttribute.java
public interface ConfigAttribute extends Serializable {
    String getAttribute();   // "ROLE_ADMIN" 처럼 문자열로 표현 가능하면 그 문자열, 아니면 null
}
```

문자열로 표현 가능한 단순 권한은 기본 구현 `SecurityConfig`가 담는다(예: `@Secured("ROLE_ADMIN")` → `new SecurityConfig("ROLE_ADMIN")`). 반면 SpEL 표현식 기반 권한(`@PreAuthorize`, `access("hasRole(...)")`)은 `getAttribute()`가 `null`을 돌려주고, 대신 `WebExpressionConfigAttribute`·`PreInvocationExpressionAttribute` 같은 **전용 타입**으로 표현된다. 그래서 투표자는 보통 `attribute instanceof XxxConfigAttribute`로 자기 소관인지 판별한다(03·04·05장 참고).

### supports() — 시작 시 정합성 검증의 기반

세 인터페이스(`AccessDecisionManager`, `AccessDecisionVoter`, `SecurityMetadataSource`)는 모두 `supports(Class<?>)`와 `supports(ConfigAttribute)`를 가진다. 이것은 단순 런타임 분기가 아니라, **부팅 시점에 "이 인터셉터에 꽂힌 협력자들이 서로 호환되는가"를 검증**하기 위한 장치다. `AbstractSecurityInterceptor.afterPropertiesSet()`이 이 메서드들을 호출해, 예컨대 메서드 보안 인터셉터에 `FilterInvocation`만 지원하는 매니저를 꽂으면 부팅을 실패시킨다(02장).

## 동작 흐름 — 한 호출이 6개 협력자를 거치는 큰 그림

메서드 `@Secured("ROLE_ADMIN") void deleteUser()`가 호출됐다고 하자.

```
호출자 ──▶ MethodSecurityInterceptor.invoke(mi)
              │
              ▼ beforeInvocation(mi)  [AbstractSecurityInterceptor]
  ┌───────────────────────────────────────────────────────────┐
  │ 1. SecurityMetadataSource.getAttributes(mi)                 │
  │      → [ SecurityConfig("ROLE_ADMIN") ]                     │
  │    (비어 있으면 "public" 호출로 간주하고 인가 생략)         │
  │ 2. SecurityContext에서 Authentication 확보                  │
  │      (없으면 AuthenticationCredentialsNotFound 예외)        │
  │      (isAuthenticated()==false면 AuthenticationManager 재인증)│
  │ 3. AccessDecisionManager.decide(auth, mi, attrs)           │
  │      → Voter들이 투표 → 거부면 AccessDeniedException        │
  │ 4. RunAsManager.buildRunAs(...) — 승격 토큰 있으면 컨텍스트 교체│
  └───────────────────────────────────────────────────────────┘
              │ InterceptorStatusToken 반환
              ▼
         mi.proceed()      ← 실제 deleteUser() 실행
              │
              ▼ afterInvocation(token, result)
  ┌───────────────────────────────────────────────────────────┐
  │ 5. finallyInvocation — run-as 했으면 원래 컨텍스트로 복귀   │
  │ 6. AfterInvocationManager.decide(...) — 반환값 후처리/필터  │
  └───────────────────────────────────────────────────────────┘
              │
              ▼ 호출자에게 반환
```

웹 요청(`FilterSecurityInterceptor`)이나 메시지(`ChannelSecurityInterceptor`)도 **정확히 같은 1~6단계**를 거친다. 다른 점은 (a) 보안 객체 타입(`FilterInvocation`/`Message`), (b) `proceed()`에 해당하는 "실제 실행"이 무엇인가뿐이다. 이 통일성이 다음 챕터의 주제다.

## 설계 포인트

- **Template Method + Strategy 결합**: `AbstractSecurityInterceptor`가 절차(template)를 고정하고, 6개 협력자를 전략(strategy)으로 주입받는다. 보안 객체 종류만 바뀌는 서브클래스(메서드/필터/채널)는 "실행 한 줄"만 다르게 구현한다.
- **메타데이터와 결정의 분리**: "무엇이 필요한가(`SecurityMetadataSource`)"와 "충족됐는가(`AccessDecisionManager`)"를 분리해, 권한 정의 방식(어노테이션/XML/표현식)과 결정 방식(투표 전략)을 독립적으로 조합한다.
- **현대 API와의 대비**: `AuthorizationManager<T>`는 이 6개 중 메타데이터·결정·투표를 한 인터페이스 `check(supplier, T)`로 통합했다. 메타데이터를 "조회"하던 단계가 사라지고, 각 보안 지점이 자기 표현식/규칙을 직접 들고 있게 된 것이 가장 큰 구조적 차이다.

## 정리

`access` 모듈은 "보안 객체"라는 단일 추상 위에서 — 메타데이터 조회 → 재인증 → 투표 결정 → (선택) 권한 승격 → 실행 → 반환값 후처리 — 라는 6단 절차를 돌리는 1세대 인가 엔진이다. 6개 협력자 인터페이스와 그 `supports()` 정합성 검증을 머리에 넣으면, 이후 모든 챕터는 "이 골격의 어느 부위인가"로 자리매김된다.
