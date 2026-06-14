# 02. 인증 프로바이더와 Authenticator

## 무엇을 / 왜

인증의 본체는 "이 비밀번호가 맞는가"를 디렉터리에 묻는 일이다. LDAP에는 이를 묻는 방법이 두 가지 있고, 둘은 보안·권한 특성이 다르다. 이 챕터는 `LdapAuthenticationProvider`가 어떻게 `AuthenticationManager` 체인에 끼어 동작하는지, 그리고 그 안의 `LdapAuthenticator` 두 구현(`BindAuthenticator`, `PasswordComparisonAuthenticator`)이 무엇을 어떻게 다르게 하는지를 본다.

## 핵심 타입

```
AuthenticationProvider (core)
   ▲
   │ implements
AbstractLdapAuthenticationProvider   ── 템플릿 (01챕터 참고)
   ▲
   │ extends
LdapAuthenticationProvider
   │  - authenticator : LdapAuthenticator        ← 비번 검증 전략
   │  - authoritiesPopulator : LdapAuthoritiesPopulator
   │  - hideUserNotFoundExceptions (기본 true)

LdapAuthenticator (interface)
   ▲
   │ implements
AbstractLdapAuthenticator   ── DN 패턴(userDnPatterns) + userSearch 공통 보유
   ▲
   ├─ BindAuthenticator               사용자 DN으로 직접 접속(bind) 시도
   └─ PasswordComparisonAuthenticator 디렉터리 저장 비번과 compare
```

### 두 authenticator의 차이

| | BindAuthenticator | PasswordComparisonAuthenticator |
|---|---|---|
| 검증 방식 | 사용자 자격증명으로 LDAP **bind**(로그인) 시도 | 엔트리를 읽고 비번 속성을 **compare**(또는 로컬 비교) |
| 누구 권한으로 엔트리를 읽나 | 바인딩한 **사용자 본인** 권한 | 관리자(manager) 컨텍스트 |
| 비번이 디렉터리에 평문/해시로 있어야? | 아니오 (서버가 검증) | 예 (`userPassword` 속성 필요) |
| salted-SHA(SSHA) | 무관 | compare 불가 (salt를 모름) |

## 동작 흐름 ① — BindAuthenticator

`BindAuthenticator.authenticate()`의 단계 (`.../authentication/BindAuthenticator.java`):

```
authenticate("ben"/"pw")
  │
  ├─ 비번 비어있으면 BadCredentialsException
  │
  ├─ (a) userDnPatterns 가 있으면: getUserDns("ben") = ["uid=ben,ou=people", ...]
  │       각 DN 마다 bindWithDn(dn, ...) 시도 → 성공하면 break
  │
  ├─ (b) 아직 못 찾았고 userSearch 가 있으면:
  │       userSearch.searchForUser("ben") 로 엔트리 검색 → 그 DN 으로 bind
  │
  └─ 둘 다 실패 → BadCredentialsException

bindWithDn(dn, user, pw):
  fullDn = dn + baseLdapName
  ctx = contextSource.getContext(fullDn, pw)   ← 여기서 실제 LDAP bind = 비번 검증
  ppolicy = PasswordPolicyControlExtractor.extractControl(ctx)  ← 정책 컨트롤 수신
  attrs  = ctx.getAttributes(userDn, userAttributes)           ← 본인 권한으로 속성 읽기
  return DirContextAdapter(attrs, userDn, base)  (+ ppolicy 속성)
```

핵심은 `contextSource.getContext(fullDn, password)` 한 줄이다. **접속(bind) 자체가 비밀번호 검증**이다. 접속이 성공하면 그 컨텍스트로 사용자 속성을 읽으므로, "사용자 본인이 읽을 수 있는 속성"만 가져온다는 보안 특성이 생긴다(주석에서 명시: 속성 권한이 인증 방식에 의존하기 때문에 속성 획득을 authenticator에 위임).

bind가 실패하면 `NamingException`이 던져지는데, DN 패턴을 여러 개 시도하는 도중일 수 있으므로 `handleIfBindException`이 인증/미지원 예외만 삼키고 나머지는 다시 던진다.

## 동작 흐름 ② — PasswordComparisonAuthenticator

`PasswordComparisonAuthenticator.authenticate()` (`.../authentication/PasswordComparisonAuthenticator.java`):

```
authenticate("ben"/"pw")
  │
  ├─ 관리자 컨텍스트의 SpringSecurityLdapTemplate 으로 엔트리 먼저 확보:
  │     getUserDns → retrieveEntry(dn)  또는  userSearch.searchForUser
  │     못 찾으면 UsernameNotFoundException
  │
  ├─ (옵션) usePasswordAttrCompare: 엔트리의 userPassword 속성을 로컬에서
  │          passwordEncoder.matches(pw, 저장값) 로 비교 → 맞으면 성공
  │
  └─ LDAP compare 연산:
        encoded = passwordEncoder.encode(pw)
        ldapTemplate.compare(userDn, "userPassword", encoded바이트)
        서버가 true/false → 틀리면 BadCredentialsException
```

여기서는 bind를 하지 않는다. 대신 관리자 권한으로 엔트리를 읽고, 비번 속성을 비교한다. compare는 서버가 "이 속성 값이 이것과 같냐"만 답하므로 비번 값이 노출되진 않지만, 디렉터리에 비교 가능한 형태(평문 또는 알려진 해시)로 비번이 있어야 한다. `setPasswordEncoder`를 호출하면 `usePasswordAttrCompare`가 자동으로 켜진다.

## 동작 흐름 ③ — 사용자 검색 (FilterBasedLdapUserSearch)

DN 패턴만으로 사용자를 못 찾을 때 쓰는 검색 전략. `FilterBasedLdapUserSearch.searchForUser()` (`.../search/FilterBasedLdapUserSearch.java`):

```
searchForUser("ben")
  └ SpringSecurityLdapTemplate.searchForSingleEntry(
        searchBase="ou=people", filter="(uid={0})", params=["ben"])
     ├ 결과 1건 → DirContextOperations 반환
     ├ 결과 0건 → UsernameNotFoundException
     └ 결과 2건+ → IncorrectResultSizeDataAccessException (설정 오류)
```

`{0}`에 username이 치환되며, `SpringSecurityLdapTemplate`이 RFC2254에 따라 username을 필터 인코딩한다(05챕터). 기본 검색 범위는 서브트리(`SUBTREE_SCOPE`).

DN 패턴은 `AbstractLdapAuthenticator.getUserDns()`가 `MessageFormat`으로 만든다. 이때 username은 `LdapEncoder.nameEncode`로 DN 인코딩되어 인젝션을 막는다.

## 동작 흐름 ④ — Provider 단의 예외 변환

`LdapAuthenticationProvider.doAuthentication()`은 authenticator의 저수준 예외를 인증 도메인 예외로 번역한다 (`.../authentication/LdapAuthenticationProvider.java`):

```
authenticator.authenticate(...)
  ├ PasswordPolicyException  → LockedException (bind 중 ppolicy = 계정 잠김)
  ├ UsernameNotFoundException → hideUserNotFoundExceptions(기본 true)면
  │                              BadCredentialsException 으로 숨김 (계정 존재 노출 방지)
  └ NamingException          → InternalAuthenticationServiceException
```

`hideUserNotFoundExceptions`가 기본 true인 이유는 "그런 사용자 없음"과 "비번 틀림"을 외부에서 구분하지 못하게 해 사용자 열거 공격을 막기 위해서다.

## 설계 포인트 / 확장점

- **bind vs compare는 `LdapAuthenticator` 교체로 결정** — 같은 provider, 다른 전략.
- **DN 패턴과 검색은 병행 가능** — `AbstractLdapAuthenticator`는 `userDnFormat`와 `userSearch`를 모두 가질 수 있고, `afterPropertiesSet`에서 둘 중 하나는 반드시 있어야 한다고 검증한다.
- **빈 비밀번호 방어** — provider와 authenticator 양쪽에서 빈 비번을 막는다. LDAP의 익명 bind가 빈 비번을 통과시킬 수 있어, 이를 거르지 않으면 아무 계정이나 인증될 위험이 있기 때문이다(`LdapAuthenticationProvider` Javadoc 경고).
- **`handleBindException` 훅** — `BindAuthenticator` 서브클래스가 bind 실패를 가로채 추가 처리(예: ppolicy 해석)를 할 수 있다.

## 정리

`LdapAuthenticationProvider`는 `AbstractLdapAuthenticationProvider` 템플릿 위에서 `LdapAuthenticator`로 비번을 검증하고 `LdapAuthoritiesPopulator`로 권한을 채운다. `BindAuthenticator`는 사용자로 접속해 검증하고 본인 권한으로 속성을 읽으며, `PasswordComparisonAuthenticator`는 관리자 권한으로 엔트리를 읽어 비번 속성을 compare한다. 사용자를 DN 패턴으로 못 찾으면 `FilterBasedLdapUserSearch`가 필터로 찾아주고, provider는 저수준 예외를 인증 예외로 안전하게 번역한다.
