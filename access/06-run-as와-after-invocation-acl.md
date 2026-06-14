# 06. run-as 권한 승격과 after-invocation / ACL

## 무엇을 / 왜

02장 엔진의 두 선택적 단계 — **실행 전의 run-as 권한 승격**(`RunAsManager`)과 **실행 후의 반환값 후처리**(`AfterInvocationManager`) — 를 확대한다. 후자의 대표 용례가 ACL 기반 결과 필터링이다. `@PostFilter`로 컬렉션을 깎는 일(04장)도 이 after-invocation 슬롯에서 일어난다. 패키지로는 `access/intercept`(run-as)와 `access/intercept` + `acls/afterinvocation`(after-invocation).

## 핵심 타입

```
실행 전(선택):  RunAsManager.buildRunAs(auth, object, attrs) → Authentication?
   ├─ NullRunAsManager         항상 null (기본 = 승격 안 함)
   └─ RunAsManagerImpl         RUN_AS_* 속성 → RunAsUserToken 생성
                                   ↑ 검증
                               RunAsImplAuthenticationProvider (key 대조)

실행 후(선택):  AfterInvocationManager.decide(auth, object, attrs, returned) → Object
   └─ AfterInvocationProviderManager
         └─ List<AfterInvocationProvider> 순차 적용
              ├─ PostInvocationAdviceProvider          @PostAuthorize/@PostFilter
              └─ acls/afterinvocation/*                ACL 기반 필터링
                   ├─ AclEntryAfterInvocationProvider               단일 객체 권한 검사
                   └─ AclEntryAfterInvocationCollectionFilteringProvider  컬렉션 필터링
                        └─ Filterer (CollectionFilterer / ArrayFilterer)
```

## 동작 흐름 — run-as 권한 승격

서비스 A가 더 높은 권한이 필요한 서비스 B를 호출해야 할 때, "이 메서드 실행 동안만 임시로 권한을 빌려준다"는 패턴이다.

```
beforeInvocation 끝부분 (02장)
   runAs = runAsManager.buildRunAs(authenticated, object, attributes)
              │
              ▼ RunAsManagerImpl.buildRunAs(...)
   for attr in attributes:
       if attr.getAttribute().startsWith("RUN_AS_"):
           newAuthorities += new SimpleGrantedAuthority("ROLE_" + attr)   // RUN_AS_FOO → ROLE_RUN_AS_FOO
   newAuthorities.isEmpty() ? return null                                 // 승격 없음
   newAuthorities += 기존 권한
   return new RunAsUserToken(key, principal, credentials, newAuthorities, ...)
              │
   runAs != null →
       원래 컨텍스트를 token에 보관(refresh=true)
       SecurityContextHolder에 runAs 컨텍스트 설정      ← 이후 호출은 승격된 권한으로
   ...
   finallyInvocation(token) →
       refresh==true 면 원래 컨텍스트로 복귀            ← 메서드 끝나면 원상복구
```

`RunAsUserToken`은 `RunAsManagerImpl.key`(부팅 시 필수, 소스 72~75행)의 해시를 품고, 짝이 되는 `RunAsImplAuthenticationProvider`가 같은 key를 가진 토큰만 인증해 위조를 막는다. `RUN_AS_FOO` 속성은 `ROLE_RUN_AS_FOO` 권한으로 변환되므로, 승격 후 인가는 그 역할을 요구하는 보안 객체만 통과시킨다. 기본값인 `NullRunAsManager`는 항상 `null`을 반환해 이 기능을 끈다.

## 동작 흐름 — after-invocation 반환값 후처리

`afterInvocation`(02장)이 `AfterInvocationManager.decide(...)`를 호출하면, `AfterInvocationProviderManager`가 등록된 모든 `AfterInvocationProvider`를 **체인처럼** 통과시킨다.

```java
// access/intercept/AfterInvocationProviderManager.java
public Object decide(auth, object, config, returnedObject) {
    Object result = returnedObject;
    for (AfterInvocationProvider provider : this.providers) {
        result = provider.decide(auth, object, config, result);  // 이전 결과를 다음으로
    }
    return result;
}
```

각 provider는 **자기 소관이 아니어도 받은 객체를 반드시 그대로 반환**해야 한다(소스 44~50행 주석). 그래야 반환값이 체인 끝까지 보존된다. 이 체인이 "거부면 예외, 통과면 (필터링된) 객체"를 만들어 최종적으로 호출자에게 돌아간다.

## 동작 흐름 — ACL 기반 결과 필터링

도메인 객체별 권한(누가 어떤 객체를 읽을 수 있는가)을 ACL로 관리할 때, 메서드가 반환한 객체/컬렉션을 사후 검사·필터링한다.

```
단일 객체:  AclEntryAfterInvocationProvider.decide(...)
   처리 속성(예: AFTER_ACL_READ) 매칭 시
   → AclService로 returnedObject의 ACL 조회
   → Acl.isGranted(requirePermission, sids, ...) 검사
   → 권한 없으면 AccessDeniedException, 있으면 객체 그대로 반환
   (returnedObject == null 이면 항상 허용)

컬렉션:  AclEntryAfterInvocationCollectionFilteringProvider.decide(...)
   → Filterer(CollectionFilterer/ArrayFilterer)로 컬렉션을 감싸고
   → 각 원소마다 ACL 검사, 권한 없는 원소를 Filterer.remove(...)
   → 걸러진 컬렉션/배열 반환
```

`Filterer`는 컬렉션과 배열의 "원소 제거"를 추상화한 인터페이스이고, `CollectionFilterer`/`ArrayFilterer`가 각 자료구조용 구현이다. 이렇게 "반환값을 통째로 거부"가 아니라 "원소 단위로 깎기"가 가능한 것이 after-invocation 단계의 고유 가치다. `@PostFilter`의 SpEL 필터링도 같은 슬롯의 `PostInvocationAdviceProvider`가 수행한다(04장).

> 현대 권장: ACL은 `AclPermissionEvaluator`를 통해 `@PostAuthorize("hasPermission(returnObject, 'read')")` / `@PostFilter("hasPermission(filterObject, 'read')")`로 표현한다. provider 체인을 직접 조립하던 방식을 표현식으로 대체한 것이다.

## 핵심 메서드 / 부품

- **`RunAsManagerImpl.buildRunAs`** — `RUN_AS_*` 속성을 `ROLE_RUN_AS_*` 권한으로 변환해 새 토큰 생성. `supports(attribute)`는 `RUN_AS_` 접두사로 판별.
- **`AfterInvocationProviderManager.decide`** — provider 순차 적용 + 결과 전파. `supports(clazz)`는 "모든 provider가 지원"해야 true.
- **`AbstractAclProvider` / `AclEntryAfterInvocationProvider`** — `AclService`로 도메인 객체의 ACL을 조회하고 `requirePermission`(기본 `READ`)을 검사. `processConfigAttribute`(기본 `AFTER_ACL_READ`)로 활성화 여부 결정.
- **`event` 패키지 + `LoggerListener`** — 02장 엔진이 발행하는 인가 이벤트(`AuthorizedEvent`, `AuthorizationFailureEvent` 등)를 받아 로깅하는 감사 훅.

## 설계 포인트 / 확장점

- **before/after 대칭**: 인가 전(run-as)과 인가 후(after-invocation)가 02장 엔진의 두 슬롯에 깔끔히 대응한다. 전자는 "권한을 더해 실행", 후자는 "결과를 검사/축소".
- **provider 체인의 불변식**: after-invocation provider가 객체를 항상 반환해야 한다는 규약이 깨지면 반환값이 유실된다. 합성 가능성을 위해 의도된 제약이다.
- **현대 대체**: run-as는 직접 대응 대체재가 없다(인증/인가가 분리되며 거의 쓰이지 않게 됨). after-invocation/ACL 필터링은 `@PostAuthorize`/`@PostFilter` + `AuthorizationManagerAfterMethodInterceptor` + `AclPermissionEvaluator`로 표현식화됐다.

## 정리

run-as는 인가 통과 후 `RUN_AS_*` 속성을 임시 역할로 승격해 실행 동안만 `SecurityContext`를 바꾸고, 끝나면 복구한다(`RunAsManagerImpl` + `RunAsUserToken` + key 검증 provider). after-invocation은 `AfterInvocationProviderManager`가 provider 체인으로 반환값을 후처리하며, ACL provider들이 도메인 객체 권한으로 단일 객체를 거부하거나 컬렉션을 원소 단위로 필터링한다. 두 단계 모두 deprecated이고, 각각 표현식 기반 메서드 보안과 `AclPermissionEvaluator`로 흡수됐다.
