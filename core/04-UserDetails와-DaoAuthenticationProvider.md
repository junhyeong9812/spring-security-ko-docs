# 04. UserDetails와 DaoAuthenticationProvider — 사용자 데이터와 비밀번호 인증

## 무엇을 / 왜

03장에서 `ProviderManager`가 `AuthenticationProvider`에게 검증을 위임한다고 했다. 그중 가장 널리 쓰이는, 그리고 "폼 로그인"의 실체인 provider가 `DaoAuthenticationProvider`다. 이 provider는 두 가지 추상화에 기댄다.

1. **`UserDetailsService`** — "username으로 사용자 데이터를 어디서 읽어올 것인가"(DB? 메모리? LDAP?)
2. **`PasswordEncoder`** — "저장된 비밀번호와 입력 비밀번호를 어떻게 비교할 것인가"(crypto 모듈)

이 장은 사용자 데이터 도메인(`UserDetails`/`User`/provisioning)과, 그것을 사용해 실제 비밀번호 검증을 수행하는 `DaoAuthenticationProvider`의 동작을 끝까지 따라간다.

소스: `core/src/main/java/org/springframework/security/core/userdetails/`, `.../authentication/dao/`, `.../provisioning/`

## 핵심 타입

```
  ┌────────────────────────────┐        loadUserByUsername(String): UserDetails
  │ UserDetailsService          │◀──────────────────────────────────────────────┐
  │  (사용자 DAO 전략)            │                                                  │
  └──────────────┬─────────────┘                                                  │
        구현들    │                                                                 │
   ┌─────────────┼──────────────────┬─────────────────┐                          │
   ▼             ▼                  ▼                 ▼                            │
InMemory     JdbcDaoImpl /     커스텀(JPA 등)    CachingUserDetails               │
UserDetails  JdbcUserDetails                     Service(데코레이터)               │
Manager      Manager                                                              │
   │             │                                                                │ 사용
   └──── returns ▼                                                                │
        ┌────────────────────────────┐                                           │
        │ UserDetails (인터페이스)      │   getUsername / getPassword /             │
        │   └ User (참조 구현)          │   getAuthorities / is*NonExpired 등        │
        └────────────────────────────┘                                           │
                                                                                  │
  ┌─────────────────────────────────────────────────────────────────────┐       │
  │ AbstractUserDetailsAuthenticationProvider (템플릿)                      │       │
  │   authenticate(): 캐시조회→retrieveUser→pre/post체크→성공토큰생성         │───────┘
  └──────────────────────────────┬──────────────────────────────────────┘
                                 │ extends
                                 ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ DaoAuthenticationProvider                                             │
  │   retrieveUser(): userDetailsService.loadUserByUsername(username)     │
  │   additionalAuthenticationChecks(): passwordEncoder.matches(...)      │
  └─────────────────────────────────────────────────────────────────────┘
```

### UserDetails — 보안용 사용자 스냅샷

`UserDetails`는 인증·인가에 필요한 사용자 정보를 담는 인터페이스다. 주석이 강조하는 점이 흥미롭다 — "구현은 Spring Security가 **직접** 보안 목적으로 쓰지 않는다. 단지 정보를 저장할 뿐이고, 나중에 `Authentication`으로 캡슐화된다." 즉 `UserDetails`는 인증의 **입력 데이터**이고, 인증 결과의 principal로 들어간다.

핵심 메서드:

```java
Collection<? extends GrantedAuthority> getAuthorities();  // 권한
@Nullable String getPassword();   // 저장된 (인코딩된) 비밀번호. passkey만 쓰면 null 가능
String getUsername();             // 식별자 (never null)
boolean isAccountNonExpired();    // 계정 만료
boolean isAccountNonLocked();     // 계정 잠금
boolean isCredentialsNonExpired();// 비밀번호 만료
boolean isEnabled();              // 활성화
```

뒤의 네 boolean은 계정 상태 플래그로, provider의 pre/post 체크가 이를 검사해 `LockedException`·`DisabledException` 등을 던진다(아래 흐름 참고). 7.1에서는 모두 `default true`인 default 메서드라, 단순 구현은 username/password/authorities만 채우면 된다.

참조 구현 `User`는 이 모든 필드를 담는 불변 클래스이며, `User.withUsername(...).password(...).roles(...).build()` 빌더를 제공한다.

### UserDetailsService — 한 메서드짜리 사용자 DAO

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

읽기 전용 메서드 하나뿐이라, 새 데이터 소스(JPA·Mongo·외부 API)를 붙이기가 쉽다. 사용자를 못 찾으면 `UsernameNotFoundException`을 던진다(권한이 하나도 없을 때도). 구현체:

- **`InMemoryUserDetailsManager`** — 메모리 맵 기반. 테스트·데모용.
- **`JdbcDaoImpl` / `JdbcUserDetailsManager`** — JDBC 기반. 표준 스키마(users/authorities 테이블)를 쿼리.
- **`CachingUserDetailsService`** — 다른 `UserDetailsService`를 감싸 `UserCache`로 캐싱하는 데코레이터.

### provisioning — 읽기를 넘어 쓰기로

`UserDetailsService`는 읽기 전용이다. 사용자 생성/수정/삭제가 필요하면 `provisioning` 패키지의 `UserDetailsManager`를 쓴다.

```java
public interface UserDetailsManager extends UserDetailsService {
    void createUser(UserDetails user);
    void updateUser(UserDetails user);
    void deleteUser(String username);
    void changePassword(String oldPassword, String newPassword);
    boolean userExists(String username);
}
```

`InMemoryUserDetailsManager`와 `JdbcUserDetailsManager`가 구현하며, `GroupManager`는 그룹 단위 권한 관리를 추가한다. 즉 같은 백엔드로 "로그인용 조회"와 "회원 관리"를 함께 처리한다.

## 동작 흐름 — DaoAuthenticationProvider의 인증

`DaoAuthenticationProvider`는 직접 `authenticate()`를 구현하지 않는다. 상위 템플릿 `AbstractUserDetailsAuthenticationProvider.authenticate()`가 골격을 잡고, 두 추상 메서드(`retrieveUser`, `additionalAuthenticationChecks`)만 `DaoAuthenticationProvider`가 채운다. **템플릿 메서드 패턴**이다.

```
ProviderManager → DaoAuthenticationProvider.authenticate(token)   [상속받은 템플릿]
   │
   ├─[1] username 추출 (principal==null → "NONE_PROVIDED")
   │
   ├─[2] UserDetails를 캐시에서 조회 (기본 NullUserCache → 항상 miss)
   │       miss면 → retrieveUser(username, token)   ★Dao가 구현
   │                  = userDetailsService.loadUserByUsername(username)
   │                  (UsernameNotFound → BadCredentials로 숨김, hideUserNotFoundExceptions)
   │
   ├─[3] performPreCheck(user):
   │       preAuthenticationChecks.check(user)        ← 잠김/비활성/계정만료 검사
   │       additionalAuthenticationChecks(user, token) ★Dao가 구현
   │                  = passwordEncoder.matches(입력비번, 저장비번)  ← 핵심 비밀번호 검증!
   │                    불일치 → BadCredentialsException
   │
   ├─[4] postAuthenticationChecks.check(user)         ← 비밀번호 만료 검사
   │
   ├─[5] 캐시에 user 저장 (캐시 미사용이었을 때)
   │
   └─[6] createSuccessAuthentication(principal, token, user)  ★Dao가 오버라이드
            - 비밀번호 업그레이드 인코딩 필요시 재인코딩 후 저장
            - 유출 비밀번호 검사(CompromisedPasswordChecker)
            - 새 UsernamePasswordAuthenticationToken(authenticated) 생성
              권한 = user의 권한 + FACTOR_PASSWORD
            → ProviderManager로 반환 → eraseCredentials → SecurityContext 저장
```

### 비밀번호 검증의 실제 — additionalAuthenticationChecks

`DaoAuthenticationProvider`가 실제 비밀번호를 비교하는 곳은 단 몇 줄이다.

```java
String presentedPassword = authentication.getCredentials().toString();
if (!this.passwordEncoder.get().matches(presentedPassword, userDetails.getPassword())) {
    throw new BadCredentialsException(... "Bad credentials");
}
```

평문을 직접 비교하지 않고 `PasswordEncoder.matches(평문, 인코딩된저장값)`를 쓴다. 기본 인코더는 `PasswordEncoderFactories.createDelegatingPasswordEncoder()` — `{bcrypt}`, `{noop}` 같은 **접두사로 알고리즘을 식별하는** 위임 인코더라, 여러 해시 방식을 동시에 지원하고 점진적 마이그레이션을 가능케 한다.

### 타이밍 공격 방어

`retrieveUser()`에는 보안적으로 미묘한 장치가 있다. 사용자가 존재하지 않아도, **존재할 때와 같은 시간**이 걸리도록 더미 비밀번호 검증을 수행한다.

```java
private void prepareTimingAttackProtection() {     // 최초 1회 더미 해시 준비
    if (userNotFoundEncodedPassword == null)
        userNotFoundEncodedPassword = passwordEncoder.get().encode(USER_NOT_FOUND_PASSWORD);
}
// UsernameNotFound 시:
private void mitigateAgainstTimingAttack(...) {    // 더미와 matches 호출 → 동일 소요시간
    passwordEncoder.get().matches(presentedPassword, userNotFoundEncodedPassword);
}
```

이게 없으면 "응답이 빠르면 없는 계정, 느리면 있는 계정(비번 비교함)"으로 username을 추측당할 수 있다(주석의 SEC-2056).

### 계정 상태 체크 — pre/post

`AbstractUserDetailsAuthenticationProvider`는 두 개의 `UserDetailsChecker`를 내장한다.

- **pre 체크(`DefaultPreAuthenticationChecks`):** `isAccountNonLocked`→`LockedException`, `isEnabled`→`DisabledException`, `isAccountNonExpired`→`AccountExpiredException`. 비밀번호 검증 **전에** 실행.
- **post 체크(`DefaultPostAuthenticationChecks`):** `isCredentialsNonExpired`→`CredentialsExpiredException`. 비밀번호 검증 **후에** 실행.

이 예외들은 `AccountStatusException` 계열이라, 03장에서 본 대로 `ProviderManager`가 **즉시 중단**한다. `alwaysPerformAdditionalChecksOnUser`(기본 true)는 계정 상태가 나빠도 비밀번호 검증 시간을 동일하게 들여 타이밍 누출을 줄이는 옵션이다.

### 부가 보안 기능 (createSuccessAuthentication)

`DaoAuthenticationProvider`는 성공 토큰을 만들기 직전에 두 가지를 더 한다.

- **비밀번호 업그레이드:** `UserDetailsPasswordService`가 설정되어 있고 `passwordEncoder.upgradeEncoding()`이 true면(예: 구식 해시), 입력 비밀번호를 최신 알고리즘으로 재인코딩해 저장소를 갱신한다. 사용자가 모르는 사이 해시가 강화된다.
- **유출 비밀번호 차단:** `CompromisedPasswordChecker`(6.3+)가 설정되면, 알려진 유출 비밀번호인지 검사해 `CompromisedPasswordException`을 던진다.

## 설계 포인트 / 확장점

- **템플릿 메서드:** 인증 골격(캐시·pre/post 체크·성공 토큰 생성)은 추상 클래스가, 데이터 조회와 비밀번호 비교는 서브클래스가 담당. 새 자격 증명 검증 방식은 `retrieveUser`/`additionalAuthenticationChecks`만 바꾸면 된다.
- **`UserDetailsService` 교체가 가장 흔한 커스터마이징:** JPA로 사용자를 읽으려면 `loadUserByUsername`만 구현하면 된다. 나머지(비밀번호 검증·상태 체크)는 `DaoAuthenticationProvider`가 그대로 처리한다.
- **`PasswordEncoder` 주입:** crypto 모듈의 인코더를 갈아끼워 해시 정책을 바꾼다.
- **캐싱:** 무상태(stateless) 앱에서 매 요청 DB 조회가 부담이면 `UserCache`/`CachingUserDetailsService`로 완화. 단 캐시 사용 시에도 비밀번호는 매번 검증된다(pre 체크 실패 시 캐시 무시하고 재조회).

## 정리

`UserDetailsService`는 "username으로 사용자를 읽는" 한 메서드짜리 전략이고, `UserDetails`는 그 결과인 보안용 사용자 스냅샷(권한 + 비밀번호 + 상태 플래그)이다. `DaoAuthenticationProvider`는 `AbstractUserDetailsAuthenticationProvider`의 템플릿 위에서 이 둘을 엮어, `loadUserByUsername`으로 사용자를 가져오고 `PasswordEncoder.matches`로 비밀번호를 검증하며, 타이밍 공격 방어·비밀번호 업그레이드·유출 검사 같은 위생까지 챙긴다. 검증을 통과하면 권한이 채워진 신뢰 토큰을 만들어 `ProviderManager`에 돌려주고, 그 토큰이 `SecurityContext`에 저장되어 다음 단계인 **인가**(05·06장)의 입력이 된다.
