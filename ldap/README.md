# ldap 모듈 — LDAP/Active Directory 인증

> Spring Security 7.1.1-SNAPSHOT 기준. 소스 위치: `ldap/src/main/java/org/springframework/security/ldap`

## 한 줄 정의

LDAP 디렉터리 서버(또는 Active Directory)를 **자격 증명 저장소**로 사용해 사용자를 인증하고, 디렉터리의 그룹/속성으로부터 권한(`GrantedAuthority`)을 채워 넣는 인증 모듈이다.

## 이 모듈이 푸는 문제

기업 환경에서는 사용자 계정과 비밀번호가 애플리케이션 DB가 아니라 **LDAP 디렉터리**(OpenLDAP, Active Directory 등)에 들어 있는 경우가 많다. 이때 다음과 같은 문제를 풀어야 한다.

- **인증을 어디서 하나** — 비밀번호 검증을 디렉터리에 위임해야 한다. 그런데 LDAP에는 인증 방식이 둘 있다. 사용자 DN으로 직접 "bind"(접속)해 보는 방식과, 디렉터리에 저장된 비밀번호 값을 "compare"(비교)하는 방식이다. 둘은 권한·보안 특성이 달라 전략으로 분리해야 한다.
- **사용자를 어떻게 찾나** — 로그인 이름(uid)만으로는 bind에 필요한 전체 DN을 모른다. DN 패턴(`uid={0},ou=people`)으로 조합하거나, 필터 검색(`(uid={0})`)으로 엔트리를 찾아야 한다.
- **권한을 어디서 가져오나** — 인증과 별개로, 사용자가 속한 그룹(`groupOfNames`의 `member` 속성 등)을 검색해 역할로 변환해야 한다.
- **Active Directory의 관례** — AD는 `userPrincipalName`(user@domain) bind, `memberOf` 속성, 49 에러의 서브코드 등 고유 관례가 있어 별도 프로바이더가 필요하다.
- **테스트/개발용 서버** — 실 디렉터리 없이 돌릴 수 있는 내장(in-memory) LDAP 서버가 필요하다.

이 모듈은 위 책임들을 각각 독립된 전략 인터페이스로 쪼개고, `LdapAuthenticationProvider`가 이를 조립한다.

## 두 가지 동작 모드

이 모듈은 두 갈래로 쓰인다. 헷갈리기 쉬우니 먼저 구분한다.

```
 [모드 A] 인증 (AuthenticationManager 체인에 참여)
   AuthenticationManager
        │ authenticate(UsernamePasswordAuthenticationToken)
        ▼
   LdapAuthenticationProvider ──▶ LdapAuthenticator (bind / compare 로 비번 검증)
                              └──▶ LdapAuthoritiesPopulator (그룹 검색 → 권한)

 [모드 B] 사용자 조회 (UserDetailsService 로만 사용 — 비번 검증 안 함)
   LdapUserDetailsService.loadUserByUsername(name)
        │
        ├──▶ LdapUserSearch (디렉터리에서 엔트리 검색)
        └──▶ LdapAuthoritiesPopulator (권한)
```

모드 A는 "이 비밀번호가 맞는가"를 디렉터리에 묻는다. 모드 B는 디렉터리를 단순 조회 백엔드로 쓰며 비밀번호 검증은 하지 않는다(예: 폼 로그인이 아니라 다른 인증 후 LDAP에서 사용자 정보만 가져올 때).

## 의존 / 연관 관계

- **Spring LDAP** (`org.springframework.ldap`) — `ContextSource`, `LdapTemplate`, `DirContextOperations` 등 LDAP 접근 기반. 이 모듈은 그 위의 보안 어댑터 층이다.
- **spring-security-core** — `AuthenticationProvider`, `UserDetails`, `GrantedAuthority`, `UserDetailsService`, `PasswordEncoder` 등을 구현/사용한다. ([../core/README.md](../core/README.md))
- **UnboundID LDAP SDK** — 내장 서버(`UnboundIdContainer`) 구현에 사용.
- **상위 설정 모듈** — `config` 모듈의 `LdapAuthenticationProviderConfigurer` / 표준 설정이 여기 빈들을 조립한다(이 모듈 자체에는 Configurer가 없다).

## 하위 챕터 목차

1. [01-개요와-핵심-추상화.md](01-개요와-핵심-추상화.md) — 모듈의 책임 분해, 핵심 전략 인터페이스 4종, 패키지 지도.
2. [02-인증-프로바이더와-Authenticator.md](02-인증-프로바이더와-Authenticator.md) — `LdapAuthenticationProvider`의 인증 흐름, Bind vs PasswordComparison, 사용자 검색.
3. [03-사용자-조회와-권한-매핑.md](03-사용자-조회와-권한-매핑.md) — `UserDetailsContextMapper`, `LdapAuthoritiesPopulator`/그룹 검색, `LdapUserDetailsService`/`Manager`.
4. [04-Active-Directory.md](04-Active-Directory.md) — `ActiveDirectoryLdapAuthenticationProvider`의 AD 전용 관례와 에러 코드 처리.
5. [05-인프라-ContextSource-Template-내장서버.md](05-인프라-ContextSource-Template-내장서버.md) — `ContextSource`/풀링, `SpringSecurityLdapTemplate`, 비밀번호 정책, 인코딩, 내장 LDAP 서버.

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.ldap
│
├─ (root)                              LDAP 접근 인프라 (보안 어댑터 층)
│   ├─ DefaultSpringSecurityContextSource   접속 소스(ContextSource) + 풀링 제어
│   ├─ SpringSecurityLdapTemplate           LdapTemplate 확장: compare/검색/속성수집
│   ├─ LdapUtils / LdapEncoder              DN/필터 인코딩, URL 파싱
│   ├─ LdapUsernameToDnMapper(+Default)     username → DN 변환 전략
│   └─ aot.hint / jackson(2)                AOT 힌트, 직렬화 Mixin
│
├─ authentication                      인증(모드 A)의 핵심
│   ├─ AbstractLdapAuthenticationProvider   템플릿: 인증→매핑→권한→토큰
│   ├─ LdapAuthenticationProvider           표준 구현 (authenticator + populator 조립)
│   ├─ LdapAuthenticator (interface)        "비번 검증+엔트리 획득" 전략
│   ├─ AbstractLdapAuthenticator            DN 패턴/검색 공통 기반
│   ├─ BindAuthenticator                    bind 방식 인증
│   ├─ PasswordComparisonAuthenticator      compare 방식 인증
│   ├─ (Null/UserDetailsService)Populator   권한 전략 보조 구현
│   └─ ad                                   Active Directory 전용
│       ├─ ActiveDirectoryLdapAuthenticationProvider
│       └─ DefaultActiveDirectoryAuthoritiesPopulator (memberOf)
│
├─ search                              사용자 엔트리 검색
│   ├─ LdapUserSearch (interface)
│   └─ FilterBasedLdapUserSearch            필터(uid={0})로 단일 엔트리 검색
│
├─ userdetails                         UserDetails 매핑 + 권한 + 조회/CRUD
│   ├─ UserDetailsContextMapper (interface) 컨텍스트 ↔ UserDetails
│   ├─ LdapUserDetailsMapper                기본 매퍼 (LdapUserDetailsImpl 생성)
│   ├─ LdapAuthoritiesPopulator (interface) 권한 채우기 전략
│   ├─ DefaultLdapAuthoritiesPopulator      그룹 검색 → 역할
│   ├─ NestedLdapAuthoritiesPopulator       중첩 그룹 재귀 검색
│   ├─ LdapUserDetailsService               조회 전용 UserDetailsService (모드 B)
│   ├─ LdapUserDetailsManager               CRUD + 비밀번호 변경 (UserDetailsManager)
│   ├─ LdapUserDetails / Impl, Person, InetOrgPerson  UserDetails 표현
│   └─ *ContextMapper (Person/InetOrgPerson)
│
├─ ppolicy                             비밀번호 정책(RFC draft) 제어
│   ├─ PasswordPolicyAwareContextSource     bind 시 ppolicy 컨트롤 수신
│   └─ PasswordPolicyControl/ResponseControl/Exception ...
│
└─ server                              내장 LDAP 서버 (테스트/개발용)
    ├─ EmbeddedLdapServerContainer (interface)
    └─ UnboundIdContainer                   UnboundID In-Memory 서버 래퍼
```
