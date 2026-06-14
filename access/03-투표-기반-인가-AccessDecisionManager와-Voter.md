# 03. 투표 기반 인가 — AccessDecisionManager와 Voter

## 무엇을 / 왜

02장의 4단계 `accessDecisionManager.decide(...)` 안에서 실제로 "허용/거부"가 어떻게 정해지는지를 이 챕터가 다룬다. access 모듈의 결정 방식은 **투표(voting)** 다. 여러 `AccessDecisionVoter`가 각자 찬성/반대/기권을 던지고, `AccessDecisionManager`가 그 표를 **집계 전략**(과반·만장일치·단 한 표면 OK)에 따라 최종 결정한다. 이것이 1세대 인가의 가장 특징적인 부분이며, 현대 `AuthorizationManager`가 통합해버린 바로 그 구조다.

소스: `access/src/main/java/org/springframework/security/access/vote/`

## 핵심 타입 — 집계 전략과 투표자

```
        AccessDecisionManager.decide(auth, object, attrs)
                       │  (예외 안 던지면 = 허용)
                       ▼
        AbstractAccessDecisionManager  (List<AccessDecisionVoter> 보유)
            △        △        △
   ┌────────┘        │        └────────┐
AffirmativeBased  ConsensusBased   UnanimousBased
(한 표면 통과)    (과반)           (반대 0이면 통과)
                       │
                       │ 각자 voter.vote(...) 호출
                       ▼
        AccessDecisionVoter.vote() → GRANTED(1) / ABSTAIN(0) / DENIED(-1)
            △        △        △           △
       RoleVoter  Authenticated  WebExpression  PreInvocationAdvice
                  Voter          Voter(05장)    Voter(04장)
```

`AccessDecisionVoter`는 세 상수만 알면 된다(`access/AccessDecisionVoter.java`).

```java
int ACCESS_GRANTED = 1;   // 찬성
int ACCESS_ABSTAIN = 0;   // 기권 (내 소관 아님)
int ACCESS_DENIED  = -1;  // 반대
```

**기권의 의미가 핵심**이다. 투표자는 "이 `ConfigAttribute`가 내가 다룰 종류인가"를 먼저 보고, 아니면 반드시 `ABSTAIN`을 반환해야 한다. 그래야 집계 매니저가 무관한 투표자의 표를 세지 않는다.

## 동작 흐름 — 세 가지 집계 전략

같은 투표 결과라도 매니저에 따라 결론이 달라진다. `[grant, deny, abstain]` 표 분포로 비교한다.

### AffirmativeBased — "한 명이라도 찬성하면 통과"

```java
// vote/AffirmativeBased.java (요약)
int deny = 0;
for (voter : voters) {
    switch (voter.vote(...)) {
        case GRANTED: return;        // 단 한 표라도 찬성 → 즉시 통과
        case DENIED:  deny++; break;
    }
}
if (deny > 0) throw new AccessDeniedException(...);  // 찬성 0 + 반대 ≥1 → 거부
checkAllowIfAllAbstainDecisions();                   // 전원 기권 → 정책 따름
```

```
GRANTED 한 표라도 있으면      → 허용 (반대표 무시)
GRANTED 0, DENIED ≥ 1        → 거부
전원 ABSTAIN                  → allowIfAllAbstain (기본 false → 거부)
```

Spring Security 설정에서 가장 흔히 쓰였던 기본 전략이다.

### ConsensusBased — "다수결"

```
grant > deny  → 허용
deny > grant  → 거부
grant == deny (둘 다 ≠0) → allowIfEqualGrantedDeniedDecisions (기본 true → 허용)
전원 ABSTAIN  → allowIfAllAbstain (기본 false → 거부)
```

### UnanimousBased — "만장일치(반대가 한 표도 없을 것)"

다른 전략은 전체 속성 목록을 한 번에 투표자에게 넘기지만, `UnanimousBased`는 **속성 하나씩** 모든 투표자에게 돌린다(소스 62~91행). 어느 속성에서든 `DENIED`가 나오면 즉시 거부한다.

```
어떤 속성에서든 DENIED 발생 → 즉시 거부
DENIED 0, GRANTED ≥ 1        → 허용
전원 ABSTAIN                  → allowIfAllAbstain (기본 false → 거부)
```

세 전략 모두 `AbstractAccessDecisionManager`를 상속하며, "전원 기권" 처리(`checkAllowIfAllAbstainDecisions`)와 투표자 목록 관리, `supports()` 위임을 공유한다. `AbstractAccessDecisionManager.supports(clazz)`는 "**모든** 투표자가 지원해야 true", `supports(attribute)`는 "**하나라도** 지원하면 true"라는 비대칭에 주의.

## 핵심 메서드 — 대표 투표자들

### RoleVoter — 권한 이름 정확 일치

`ConfigAttribute`가 `ROLE_` 접두사로 시작하면 자기 소관으로 보고, 사용자의 `GrantedAuthority`와 **정확히 문자열 일치**하는 것이 있으면 찬성한다(`vote()`, 92~111행).

```java
for (ConfigAttribute attribute : attributes) {
    if (this.supports(attribute)) {       // "ROLE_"로 시작?
        result = ACCESS_DENIED;           // 일단 거부로 두고
        for (GrantedAuthority authority : authorities) {
            if (attribute.getAttribute().equals(authority.getAuthority()))
                return ACCESS_GRANTED;    // 정확히 일치하면 찬성
        }
    }
}
return result;  // ROLE_ 속성이 하나도 없었으면 ABSTAIN, 있었는데 못 맞췄으면 DENIED
```

접두사는 `setRolePrefix`로 바꿀 수 있고, 빈 문자열로 두면 모든 속성에 투표하게 되지만 권장되지 않는다.

### AuthenticatedVoter — 인증 "강도" 검사

`IS_AUTHENTICATED_FULLY` / `IS_AUTHENTICATED_REMEMBERED` / `IS_AUTHENTICATED_ANONYMOUSLY` 세 의사(pseudo) 권한을 처리한다. `AuthenticationTrustResolver`로 익명·remember-me 여부를 판별해, 요구 강도를 만족하면 찬성한다. 강도는 FULLY > REMEMBERED > ANONYMOUSLY 순이며, 상위 강도이면 하위 요구도 만족시킨다(예: 완전 인증이면 REMEMBERED·ANONYMOUSLY 요구도 통과).

### RoleHierarchyVoter — 역할 계층 반영

`RoleVoter`를 상속하되, 투표 전에 사용자의 권한 집합을 `RoleHierarchy.getReachableGrantedAuthorities()`로 확장한다(소스 54~57행). `ROLE_ADMIN > ROLE_USER` 계층이 정의돼 있으면, `ROLE_ADMIN`만 가진 사용자도 `ROLE_USER` 요구를 통과한다. 나머지 일치 로직은 부모 그대로다.

> SpEL 표현식을 투표로 처리하는 `WebExpressionVoter`(웹)·`PreInvocationAuthorizationAdviceVoter`(메서드)·`MessageExpressionVoter`(메시징)도 같은 투표 인터페이스를 따른다. 이들은 각각 05·04장에서 다룬다.

## 설계 포인트 / 확장점

- **전략 패턴 × 책임 분리**: "집계 방식"(매니저)과 "개별 판단"(투표자)을 분리해, 같은 투표자를 다른 집계 전략에 재사용한다. 새 권한 종류는 새 투표자 하나 추가로 끝난다.
- **기권 규약**: 투표자가 자기 소관이 아닐 때 `ABSTAIN`을 지키는 것이 시스템 정확성의 전제다. `DENIED`를 남발하면 다른 투표자의 찬성을 무력화한다(특히 Unanimous).
- **현대 대체 — 왜 사라졌나**: 실무에서는 대부분 투표자 하나(`RoleVoter` 또는 `WebExpressionVoter`)만 쓰는데도 매니저+투표자 목록을 조립해야 했다. `AuthorityAuthorizationManager`·`WebExpressionAuthorizationManager`는 이 투표 인프라 전체를 단일 `check()`로 접었다. `AuthorizationManager`들을 `AND`/`OR`로 합성하는 방식이, 사실상 Unanimous/Affirmative 전략을 더 명시적으로 표현한 것이다.

## 정리

이 모듈의 인가 결정은 투표다. `AccessDecisionVoter`가 GRANTED/ABSTAIN/DENIED를 던지고, `AffirmativeBased`·`ConsensusBased`·`UnanimousBased`가 각각 "한 표면 OK / 과반 / 반대 없을 것"으로 집계한다. `RoleVoter`(권한 일치), `AuthenticatedVoter`(인증 강도), `RoleHierarchyVoter`(계층 확장)가 대표 투표자이며, 표현식 기반 권한도 같은 투표 인터페이스로 끼어든다. 현대 `AuthorizationManager`는 이 다단 구조를 한 호출로 통합했다.
