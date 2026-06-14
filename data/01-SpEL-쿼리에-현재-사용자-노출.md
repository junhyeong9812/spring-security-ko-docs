# 01. SpEL 쿼리에 현재 사용자 노출 — SecurityEvaluationContextExtension

## 무엇을 / 왜

데이터 접근 계층의 흔한 요구는 "지금 로그인한 사용자에 종속된 데이터만 조회"다. 예를 들어 메시지 목록에서 **내게 온 메시지만** 가져오려면 `WHERE m.to.id = :현재사용자id` 가 필요하다. 이 "현재 사용자 id"를 서비스에서 매번 꺼내 파라미터로 넘기는 대신, 쿼리 정의에 직접 박을 수 있으면 코드가 깔끔해진다.

Spring Data는 쿼리에 쓰는 SpEL 표현식 컨텍스트에 임의 객체를 끼워 넣을 수 있는 SPI를 제공한다. 바로 `org.springframework.data.spel.spi.EvaluationContextExtension`이다. 이 챕터의 주인공 `SecurityEvaluationContextExtension`은 이 SPI를 구현해, Spring Security의 표현식 루트 객체를 `security`라는 id로 쿼리 SpEL에 노출한다. 그 결과 개발자는 다음처럼 쓸 수 있다.

```java
@Query("select m from Message m where m.to.id = ?#{ principal?.id }")
List<Message> findAll();
```

여기서 `principal`은 곧 현재 `Authentication.getPrincipal()`이며, 그것이 id 필드를 가진 사용자 엔티티이기 때문에 `principal?.id`가 동작한다.

## 핵심 타입

```
  ┌───────────────────────────────────────────────────────────┐
  │ EvaluationContextExtension  (spring-data-commons SPI)      │
  │   String getExtensionId()                                  │
  │   Object getRootObject()                                   │
  └───────────────▲───────────────────────────────────────────┘
                  │ implements
  ┌───────────────┴───────────────────────────────────────────┐
  │ SecurityEvaluationContextExtension                         │
  │   getExtensionId() → "security"                            │
  │   getRootObject()  → SecurityExpressionRoot<Object>        │
  │                                                            │
  │   - securityContextHolderStrategy                          │
  │   - authentication (선택: 고정 인증 객체)                    │
  │   - authorizationManagerFactory                            │
  │   - permissionEvaluator                                    │
  │   - defaultRolePrefix                                      │
  └───────────────┬───────────────────────────────────────────┘
                  │ getRootObject() 가 생성
  ┌───────────────▼───────────────────────────────────────────┐
  │ SecurityExpressionRoot<Object>  (spring-security-core)     │
  │   principal / authentication                               │
  │   hasRole(..) / hasAnyRole(..) / hasAuthority(..)          │
  │   isAuthenticated() / isAnonymous() / permitAll / denyAll  │
  │   hasPermission(target, perm)                              │
  └────────────────────────────────────────────────────────────┘
```

- **`EvaluationContextExtension`**: Spring Data가 쿼리 SpEL을 만들 때 등록된 모든 확장을 모아 컨텍스트를 구성한다. 각 확장은 고유 id와 "루트 객체"를 제공한다. 루트 객체의 public 프로퍼티/메서드가 SpEL에서 이름으로 바로 접근 가능해진다.
- **`SecurityEvaluationContextExtension`**: 확장 id로 `"security"`를 돌려주고(`getExtensionId()`), 루트 객체로 `SecurityExpressionRoot`를 만들어 준다.
- **`SecurityExpressionRoot`**: Spring Security의 표현식 평가 루트. `@PreAuthorize("hasRole('ADMIN')")` 같은 메서드 보안 표현식에서 쓰는 바로 그 객체다. 즉 **메서드 보안에서 쓰던 표현식 어휘를 쿼리 SpEL에서 그대로 재사용**하는 셈이다.

## 동작 흐름

### 시나리오: `?#{ principal?.id }`가 평가되는 순간

```
 리포지토리 메서드 호출
   findAll()  (@Query "... where m.to.id = ?#{ principal?.id }")
        │
        ▼
 Spring Data 쿼리 처리기
   - 등록된 EvaluationContextExtension 빈들을 수집
   - 그중 id="security" 확장의 루트 객체를 컨텍스트에 추가
        │
        │  getRootObject() 호출
        ▼
 SecurityEvaluationContextExtension.getRootObject()
   1) getAuthentication()
        - 고정 authentication 이 주입돼 있으면 그걸 사용
        - 아니면 securityContextHolderStrategy.getContext().getAuthentication()
   2) new SecurityExpressionRoot<>(() -> authentication, new Object())
   3) root.setAuthorizationManagerFactory(...)
      root.setPermissionEvaluator(...)
      (커스텀 rolePrefix 면) root.setDefaultRolePrefix(...)
        │
        ▼
 SpEL 평가: principal?.id
   = root.getPrincipal()?.id
   = authentication.getPrincipal().getId()
        │
        ▼
 쿼리 파라미터로 바인딩 → JPQL/SQL 실행
```

핵심은 **루트 객체가 쿼리마다 새로 만들어지고, 그 시점의 `SecurityContextHolder`에서 현재 인증을 읽는다**는 점이다. 따라서 요청별로 다른 사용자가 같은 리포지토리 메서드를 호출해도 각자 자기 principal로 평가된다.

## 핵심 메서드

### `getExtensionId()`

```java
@Override
public String getExtensionId() {
    return "security";
}
```

이 확장의 네임스페이스 이름이다. 쿼리에서 `?#{ security.principal }`처럼 명시적으로 접두를 붙여 충돌을 피할 수도 있고, 다른 확장과 이름 충돌이 없으면 `?#{ principal }`처럼 생략해도 된다.

### `getRootObject()`

```java
@Override
public SecurityExpressionRoot<Object> getRootObject() {
    Authentication authentication = getAuthentication();
    SecurityExpressionRoot<Object> root =
        new SecurityExpressionRoot<>(() -> authentication, new Object()) {};
    root.setAuthorizationManagerFactory(this.authorizationManagerFactory);
    root.setPermissionEvaluator(this.permissionEvaluator);
    if (!DEFAULT_ROLE_PREFIX.equals(this.defaultRolePrefix)) {
        root.setDefaultRolePrefix(this.defaultRolePrefix);
    }
    return root;
}
```

- `SecurityExpressionRoot`는 추상 클래스라 익명 서브클래스(`{}`)로 인스턴스화한다.
- 생성자에 `Authentication`을 **`Supplier`로** 넘긴다(`() -> authentication`). 두 번째 인자 `new Object()`는 "평가 대상 객체"(`hasPermission` 등에서 쓰는 target) 자리로, 쿼리 SpEL 맥락에는 특별한 대상이 없으므로 빈 객체를 둔다.
- `authorizationManagerFactory`/`permissionEvaluator`를 루트에 주입해 `hasRole(...)`, `hasPermission(...)` 같은 표현식이 동작하도록 한다.
- `defaultRolePrefix`가 기본값(`"ROLE_"`)과 다를 때만 별도로 설정한다 — 커스텀 접두사를 쓸 때 `hasRole`이 접두사를 올바르게 처리하게 하기 위해서다.

### `getAuthentication()`

```java
private @Nullable Authentication getAuthentication() {
    if (this.authentication != null) {
        return this.authentication;
    }
    SecurityContext context = this.securityContextHolderStrategy.getContext();
    return context.getAuthentication();
}
```

두 가지 모드를 지원한다. (1) 생성자로 고정 `Authentication`을 받은 경우 항상 그것을 사용 — 테스트나 특수 상황용. (2) 기본 모드에서는 `SecurityContextHolderStrategy`(기본적으로 `SecurityContextHolder`의 전략)에서 **호출 시점의** 현재 인증을 가져온다.

## 설계 포인트 / 확장점

- **빈으로만 등록하면 끝**: 사용법은 단순히 `SecurityEvaluationContextExtension`을 `@Bean`으로 등록하는 것이다. Spring Data가 `EvaluationContextExtension` 타입 빈을 자동 수집하므로, 별도 배선 코드가 없다.

  ```java
  @Bean
  public SecurityEvaluationContextExtension securityEvaluationContextExtension() {
      return new SecurityEvaluationContextExtension();
  }
  ```

- **메서드 보안과 어휘 공유**: 루트가 `SecurityExpressionRoot`이므로 `@PreAuthorize`/`@PostAuthorize`에서 쓰던 표현식(`hasRole`, `principal`, `authentication`, `hasPermission` 등)을 쿼리에서 그대로 쓸 수 있다. 학습 비용이 낮다.

- **커스터마이징 훅**: `setSecurityContextHolderStrategy`, `setAuthorizationManagerFactory`, `setPermissionEvaluator`로 동작을 바꾼다. 7.0부터 권한/역할 계층/신뢰 해석은 `AuthorizationManagerFactory`로 통합되었고, `setTrustResolver`/`setRoleHierarchy`/`setDefaultRolePrefix`는 `@Deprecated(since = "7.0")`로 표시되어 내부적으로 `DefaultAuthorizationManagerFactory`에 위임한다(8.0에서 제거 예정).

- **`null` 인증 주의**: 익명/미인증 상태에선 `getAuthentication()`이 `null`을 돌려줄 수 있다. 그래서 예제 쿼리도 `principal?.id`처럼 SpEL의 null-safe 내비게이션(`?.`)을 쓴다. 인증이 없을 때 안전하게 처리하는 것은 쿼리 작성자의 몫이다.

## 정리

`SecurityEvaluationContextExtension`은 Spring Data의 `EvaluationContextExtension` SPI를 구현해, Spring Security의 `SecurityExpressionRoot`를 `"security"` 네임스페이스로 쿼리 SpEL에 노출하는 어댑터다. 빈 하나만 등록하면 `@Query` 안에서 `principal`, `authentication`, `hasRole(...)` 등을 쓸 수 있고, 루트 객체는 쿼리 평가 시점마다 `SecurityContextHolder`에서 현재 인증을 읽어 요청별로 올바른 사용자로 동작한다.
