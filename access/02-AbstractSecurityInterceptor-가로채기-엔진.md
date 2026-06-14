# 02. AbstractSecurityInterceptor — 가로채기 엔진

## 무엇을 / 왜

`AbstractSecurityInterceptor`는 access 모듈의 **심장**이다. 메서드·필터·메시지 어느 보안 객체든, 인가의 실제 절차는 전부 이 추상 클래스에 들어 있다. 서브클래스(`MethodSecurityInterceptor`, `FilterSecurityInterceptor`, `ChannelSecurityInterceptor`)는 "보안 객체 타입이 무엇인지"와 "실행을 어떻게 진행시키는지"만 채운다. 이 챕터는 그 절차 — `beforeInvocation` → 실행 → `finallyInvocation` → `afterInvocation` — 를 코드 레벨로 따라간다.

소스: `access/src/main/java/org/springframework/security/access/intercept/AbstractSecurityInterceptor.java`

## 핵심 타입 — 추상 골격과 3개 구현

```
              AbstractSecurityInterceptor (abstract)
              ├─ getSecureObjectClass() : Class<?>       ← 서브클래스가 지정
              ├─ obtainSecurityMetadataSource()          ← 서브클래스가 제공
              ├─ beforeInvocation(Object)  → InterceptorStatusToken
              ├─ finallyInvocation(token)
              └─ afterInvocation(token, returned) → Object
                        △
        ┌───────────────┼─────────────────────────┐
        │               │                          │
MethodSecurityInterceptor  FilterSecurityInterceptor  ChannelSecurityInterceptor
 (MethodInvocation)         (FilterInvocation)         (Message<?>)
 implements                 implements Filter          implements
 MethodInterceptor          (doFilter)                 ChannelInterceptor
```

세 서브클래스가 추상 엔진을 감싸는 한 줄짜리 패턴은 거의 동일하다. `MethodSecurityInterceptor.invoke()`가 가장 명료하다.

```java
// access/intercept/aopalliance/MethodSecurityInterceptor.java
public Object invoke(MethodInvocation mi) throws Throwable {
    InterceptorStatusToken token = super.beforeInvocation(mi);  // 인가 + 준비
    Object result;
    try {
        result = mi.proceed();                                  // 실제 실행
    } finally {
        super.finallyInvocation(token);                         // 컨텍스트 복구
    }
    return super.afterInvocation(token, result);                // 반환값 후처리
}
```

`FilterSecurityInterceptor`는 `mi.proceed()` 자리에 `chain.doFilter(...)`를, `ChannelSecurityInterceptor`는 `preSend`/`postSend`/`afterSendCompletion` 콜백에 각 단계를 나눠 배치한다는 점만 다르다.

### InterceptorStatusToken — 전과 후를 잇는 끈

`beforeInvocation`이 만든 토큰은 실행 후 `finally`/`afterInvocation`으로 전달된다. 토큰은 (1) 호출 시점의 `SecurityContext`, (2) run-as로 컨텍스트를 바꿨는지 여부(`contextHolderRefreshRequired`), (3) `ConfigAttribute` 목록, (4) 보안 객체를 담는다. **public 호출이면 `beforeInvocation`이 `null`을 반환**하고, 이후 단계는 토큰이 `null`인지로 "후처리 불필요"를 판단한다.

## 동작 흐름 — beforeInvocation 한 단계씩

`beforeInvocation(object)`(소스 196~251행)의 절차:

```
beforeInvocation(object)
  │
  ├─ getSecureObjectClass().isAssignableFrom(object)  검사 (아니면 예외)
  │
  ├─ attributes = metadataSource.getAttributes(object)
  │     │
  │     └─ 비어 있음? ──▶ [public 호출]
  │            ├─ rejectPublicInvocations==true 면 예외
  │            ├─ PublicInvocationEvent 발행
  │            └─ return null   ← 인가 없이 통과
  │
  ├─ [secured 호출]
  │   ├─ Authentication 없음? ─▶ credentialsNotFound() → 예외 + 이벤트
  │   ├─ authenticateIfRequired()
  │   │     isAuthenticated()==true && !alwaysReauthenticate ? 그대로 사용
  │   │     아니면 AuthenticationManager.authenticate() 로 재인증 후 컨텍스트 교체
  │   ├─ attemptAuthorization(object, attributes, auth)
  │   │     accessDecisionManager.decide(...)
  │   │       └─ AccessDeniedException? → AuthorizationFailureEvent + 재throw
  │   │   (성공 시 publishAuthorizationSuccess면 AuthorizedEvent 발행)
  │   │
  │   └─ runAs = runAsManager.buildRunAs(auth, object, attributes)
  │         runAs != null ?
  │           ├─ 새 SecurityContext에 runAs 세팅 후 Holder 교체
  │           └─ return token(origCtx, refresh=true, ...)
  │         아니면
  │           └─ return token(currentCtx, refresh=false, ...)
```

핵심 포인트 셋:

- **public vs secured 분기**는 "메타데이터가 비어 있는가"로만 결정된다. 즉 `SecurityMetadataSource`가 빈 컬렉션을 주면 그 호출은 인가 없이 통과한다. 이것을 막고 싶으면 `rejectPublicInvocations=true`로 "모든 보안 객체는 명시적 설정을 가져야 한다"는 fail-safe 모드를 켠다.
- **재인증(`authenticateIfRequired`)**: 평소엔 이미 인증된 토큰을 신뢰하지만, `alwaysReauthenticate=true`면 매 호출마다 `AuthenticationManager`로 다시 인증한다. 기본 `AuthenticationManager`는 `NoOpAuthenticationManager`(예외만 던짐)이므로, 재인증이 필요한 구성에서만 실제 매니저를 주입한다.
- **run-as 권한 승격**: 인가를 통과한 뒤, `RunAsManager`가 `RUN_AS_*` 속성을 보고 임시 권한이 추가된 `RunAsUserToken`을 만들면, **호출 동안만** `SecurityContext`를 그 토큰으로 바꾼다(06장에서 상세). 바꿨다는 사실을 토큰의 `contextHolderRefreshRequired=true`로 기록한다.

### finallyInvocation / afterInvocation

```java
// finally 블록에서: run-as로 바꿨으면 원래 컨텍스트로 복구
protected void finallyInvocation(InterceptorStatusToken token) {
    if (token != null && token.isContextHolderRefreshRequired()) {
        this.securityContextHolderStrategy.setContext(token.getSecurityContext());
    }
}
```

`afterInvocation(token, returnedObject)`은 (1) public 호출이면(`token==null`) 반환값 그대로 통과, (2) `finallyInvocation`을 한 번 더 호출(멱등하게 정리), (3) `AfterInvocationManager`가 있으면 반환값을 넘겨 검사/필터링/교체하게 한다. ACL 결과 필터링(06장)이 바로 여기서 일어난다.

## 핵심 메서드 — afterPropertiesSet 의 부팅 검증

엔진이 잘못 조립되는 것을 **부팅 시점에** 잡는 것이 `afterPropertiesSet()`의 역할이다(149~176행). `AuthenticationManager`/`AccessDecisionManager`/`RunAsManager`/`SecurityMetadataSource`가 모두 존재하는지 확인하고, 각각의 `supports(getSecureObjectClass())`로 **보안 객체 타입 호환성**을 검사한다. 나아가 `validateConfigAttributes=true`(기본)면, 메타데이터가 내놓는 모든 `ConfigAttribute`가 매니저·run-as·after-invocation 중 적어도 하나에 의해 `supports()`되는지 확인해, 아무도 처리하지 못하는 "고아 속성"이 있으면 `IllegalArgumentException`을 던진다. 즉 오타 난 권한 이름이 런타임이 아니라 부팅에서 드러난다.

## 설계 포인트 / 확장점

- **Template Method 패턴의 교과서적 사례**: 절차(template)는 추상 클래스에, 가변점(보안 객체 타입·실행 방법)은 서브클래스에. 새로운 보안 객체 종류를 추가하려면 `getSecureObjectClass()`와 실행 진입점만 구현하면 된다.
- **이벤트 훅**: `PublicInvocationEvent`, `AuthorizedEvent`, `AuthorizationFailureEvent`, `AuthenticationCredentialsNotFoundEvent`를 `ApplicationEventPublisher`로 발행한다. `event` 패키지의 `LoggerListener`가 이를 받아 로깅하는 식으로 감사(audit) 훅을 붙일 수 있다.
- **`SecurityContextHolderStrategy` 주입**(5.8+): 컨텍스트 저장 전략을 교체 가능. 멀티스레드/리액티브 환경 대응의 확장점.
- **현대 대체**: 이 엔진 전체가 `AuthorizationFilter`(웹)·`AuthorizationManagerBeforeMethodInterceptor`(메서드)·`AuthorizationChannelInterceptor`(메시징)로 대체됐다. 새 구조는 `SecurityMetadataSource`·`AccessDecisionManager`·run-as 단계를 없애고, 보안 지점마다 `AuthorizationManager.check()` 한 번 호출로 단순화했다.

## 정리

`AbstractSecurityInterceptor`는 "가로채 → 메타데이터 조회 → (재)인증 → 투표 인가 → run-as 승격 → 실행 → 컨텍스트 복구 → 반환값 후처리"를 한 절차로 고정한 Template Method 엔진이다. `InterceptorStatusToken`이 실행 전후를 잇고, public/secured 분기는 메타데이터 유무로 결정되며, 부팅 시 `supports()` 검증으로 잘못된 조립을 조기에 차단한다. 세 서브클래스는 이 엔진을 메서드·HTTP·메시지에 끼워 넣는 얇은 어댑터일 뿐이다.
