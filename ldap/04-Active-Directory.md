# 04. Active Directory 전용 인증

## 무엇을 / 왜

Active Directory(AD)도 LDAP이지만, 인증 관례가 표준 OpenLDAP과 충분히 달라서 표준 `LdapAuthenticationProvider`를 그대로 쓰기 번거롭다. AD는 사용자를 `userPrincipalName`(= `user@domain`)으로 bind하고, 권한을 그룹 검색이 아니라 사용자 엔트리의 `memberOf` 속성에서 읽으며, 인증 실패(LDAP 49) 시 메시지 안에 잠김/만료 같은 **서브 에러 코드**를 숨겨 보낸다. 이 챕터는 이런 AD 관례를 흡수한 `ActiveDirectoryLdapAuthenticationProvider`를 본다.

## 핵심 타입

```
AbstractLdapAuthenticationProvider   (01챕터 템플릿)
   ▲
   │ extends
ActiveDirectoryLdapAuthenticationProvider (final)
   │  - domain   : "example.com"  (소문자화)
   │  - url      : "ldap://ad.example.com/"
   │  - rootDn   : "dc=example,dc=com" (domain 에서 유도 가능)
   │  - searchFilter : "(&(objectClass=user)(userPrincipalName={0}))"
   │  - convertSubErrorCodesToExceptions (기본 false)
   │  - authoritiesPopulator : DefaultActiveDirectoryAuthoritiesPopulator (memberOf)

DefaultActiveDirectoryAuthoritiesPopulator
   └ 사용자 엔트리의 memberOf 값(그룹 DN)에서 CN 을 뽑아 ROLE_* 생성
```

표준 provider와 달리 `LdapAuthenticator`/`LdapUserSearch`/`ContextSource`를 외부에서 조립하지 않는다. AD provider는 인증부터 검색까지 자체 JNDI 컨텍스트로 직접 수행한다.

## 동작 흐름 — bindAsUser → searchForUser

`doAuthentication()`은 01챕터의 템플릿이 호출하는 추상 메서드 구현이다 (`.../authentication/ad/ActiveDirectoryLdapAuthenticationProvider.java`):

```
doAuthentication(auth: "jdoe"/"pw")
  │
  ├ bindAsUser("jdoe","pw"):
  │     bindPrincipal = createBindPrincipal("jdoe")  → "jdoe@example.com"
  │     env: SECURITY_AUTHENTICATION=simple,
  │          SECURITY_PRINCIPAL=bindPrincipal,
  │          PROVIDER_URL=url, CREDENTIALS=pw
  │     ctx = new InitialLdapContext(env)   ← 이 접속이 비번 검증
  │       └ 실패(AuthenticationException) → handleBindException → BadCredentials
  │
  └ searchForUser(ctx, "jdoe"):
        searchRoot = rootDn  또는  principal 의 @domain 에서 유도
        LdapClient(ctx).search(
            base=searchRoot, scope=SUBTREE,
            filter="(&(objectClass=user)(userPrincipalName={0}))",
            args=[bindPrincipal, "jdoe"])
        → DirContextOperations (사용자 엔트리, memberOf 포함)
           0건 → BadCredentials (사용자 못 찾음 = 자격증명 불일치로 처리)
```

`createBindPrincipal`은 username이 이미 도메인으로 끝나면 그대로, 아니면 `@domain`을 붙인다. domain이 설정 안 됐으면 username이 도메인을 포함한다고 가정한다. 검색 베이스(`rootDn`)는 명시하지 않으면 `user@example.com`의 `example.com`을 `dc=example,dc=com`으로 변환해 유도한다(`rootDnFromDomain`).

권한은 `loadUserAuthorities()` → `DefaultActiveDirectoryAuthoritiesPopulator`가 엔트리의 `memberOf`(그룹 DN 목록)에서 역할을 만든다. 표준 provider처럼 별도 그룹 검색을 돌리지 않는다 — AD는 멤버십을 사용자 쪽에 비정규화해 두기 때문이다.

## 동작 흐름 — AD 서브 에러 코드 처리

AD는 bind 실패 시 LDAP 49 에러 메시지 안에 `data 52e` 같은 16진 서브코드를 넣는다. `handleBindException`이 이를 정규식으로 뽑아 로그를 남기고, 옵션에 따라 구체적 예외로 승격한다:

```
AuthenticationException(메시지: "...data 533...")
  │ parseSubErrorCode → 0x533
  │ 로그: "Account is disabled"
  └ convertSubErrorCodesToExceptions == true 이면:
        532 PASSWORD_EXPIRED → CredentialsExpiredException
        533 ACCOUNT_DISABLED → DisabledException
        701 ACCOUNT_EXPIRED  → AccountExpiredException
        775 ACCOUNT_LOCKED   → LockedException
        그 외(525,52e,530,773...) → BadCredentialsException
```

기본값(`convertSubErrorCodesToExceptions == false`)에서는 모든 실패가 `BadCredentialsException`이다. 즉 외부로는 "자격증명 실패"만 보이고, 잠김/만료 같은 세부는 로그에만 남는다 — 계정 상태 노출을 막는 안전한 기본값이다. 세분화된 UX가 필요하면 옵션을 켠다.

`searchForUser`에서 결과 0건은 `BadCredentialsException`으로 바뀐다(주석: bind는 됐는데 엔트리를 못 찾는 비정상 상황도 자격증명 실패로 취급). 통신 자체가 안 되면 `InternalAuthenticationServiceException`(`badLdapConnection`)이다.

## 핵심 메서드 — `searchForUser`의 자체 검색

표준 provider가 `FilterBasedLdapUserSearch` + `ContextSource`를 쓰는 것과 달리, AD provider는 bind로 얻은 `DirContext`를 `SingleContextSource`로 감싸 `LdapClient`로 직접 검색한다. 이로써 **사용자 본인의 인증된 컨텍스트**로 엔트리를 읽고, `ignorePartialResultException(true)`로 AD가 흔히 던지는 부분 결과 예외를 무시한다. 검색 필터의 `{0}`은 `user@domain`, `{1}`은 username으로 치환되어 커스텀 필터에서 둘 다 쓸 수 있다.

## 설계 포인트 / 확장점

- **AbstractLdapAuthenticationProvider 재사용** — 인증·매핑·토큰 생성의 골격은 표준과 공유하고, `doAuthentication`/`loadUserAuthorities`만 AD식으로 채웠다. `UserDetailsContextMapper`/`GrantedAuthoritiesMapper` 같은 공통 훅은 그대로 쓸 수 있다.
- **안전한 기본 + 선택적 세분화** — 실패를 기본은 뭉뚱그리고(`BadCredentials`), 필요 시 `convertSubErrorCodesToExceptions`로 구체화.
- **교체 가능한 권한 전략** — `setAuthoritiesPopulator`로 `memberOf` 기반 기본 populator를 다른 전략으로 바꿀 수 있다(6.3+).
- **컨텍스트 환경 커스터마이즈** — `setContextEnvironmentProperties`로 JNDI 환경(예: SSL, 타임아웃)을 주입한다.

## 정리

`ActiveDirectoryLdapAuthenticationProvider`는 표준 템플릿을 상속하되, AD 관례인 `userPrincipalName` bind·`memberOf` 권한·LDAP 49 서브 에러 코드를 흡수한다. bind로 비번을 검증하고 같은 컨텍스트로 엔트리를 찾으며, 실패는 기본적으로 `BadCredentialsException`으로 일원화하고 옵션을 켜면 잠김/만료 등으로 세분화한다. 외부 `Authenticator`/`ContextSource` 조립 없이 도메인·URL·rootDn만으로 동작하는 점이 표준 경로와의 큰 차이다.
