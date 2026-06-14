# kerberos — Kerberos / SPNEGO 인증 통합

> Spring Security 7.1.1-SNAPSHOT 기준 해설

## 한 줄 정의

`spring-security-kerberos` 는 **Kerberos 티켓**과 그것을 HTTP 위에 실어 나르는 **SPNEGO(Negotiate)** 프로토콜을 Spring Security 의 인증 추상화(`AuthenticationProvider`, `AuthenticationEntryPoint`, 서블릿 필터)에 끼워 넣어, **윈도우 도메인 통합 SSO(Single Sign-On)** 를 가능하게 하는 모듈이다.

## 이 모듈이 푸는 문제

기업 환경에서는 사용자가 윈도우 도메인(Active Directory)에 한 번 로그인하면, 웹 애플리케이션에 다시 ID/비밀번호를 입력하지 않고 접근하기를 기대한다. 이를 위한 표준이 Kerberos 이고, 브라우저는 Kerberos 서비스 티켓을 HTTP `Authorization: Negotiate ...` 헤더(SPNEGO)에 담아 보낸다. 그러나 이 흐름에는 두 가지 어려운 부분이 있다.

1. **JAAS / GSS-API 의 복잡성** — JRE 의 Kerberos 구현(`Krb5LoginModule`)을 쓰려면 keytab 로딩, `LoginContext`, `Subject.doAs`, `GSSContext` 같은 저수준 JAAS/GSS API 를 직접 다뤄야 한다.
2. **Spring Security 인증 모델과의 접합** — 받은 티켓을 검증하고, 거기서 얻은 사용자 이름으로 `UserDetails`(권한)를 채워 `SecurityContext` 에 인증을 심어야 한다.

이 모듈은 (1)을 `KerberosTicketValidator` / `KerberosClient` 구현 뒤에 감추고, (2)를 표준 필터 + 프로바이더 체인으로 풀어, 설정만으로 SPNEGO SSO 를 붙일 수 있게 한다.

## 지원하는 두 가지 시나리오

```
 [A] 서버측 SPNEGO 검증 (가장 흔함, 브라우저 SSO)
     브라우저 ──Negotiate 티켓──▶ SpnegoAuthenticationProcessingFilter
                                  └▶ KerberosServiceAuthenticationProvider
                                       └▶ KerberosTicketValidator.validateTicket()

 [B] 서버측 폼 로그인을 Kerberos KDC 로 검증 (ID/PW 를 받아 KDC 에 로그인)
     폼 ──username/password──▶ KerberosAuthenticationProvider
                               └▶ KerberosClient.login()  (KDC 에 직접 로그인)
```

추가로 **클라이언트 측**(우리 앱이 다른 Kerberos 보호 서비스를 호출)을 위한 `KerberosRestTemplate`, `KerberosLdapContextSource` 도 제공한다.

## 의존 · 연관 모듈

- `spring-security-core` — `AuthenticationProvider`, `Authentication`, `UserDetailsService`, `UserDetails` 등 인증 핵심 추상.
- `spring-security-web` — `AuthenticationEntryPoint`, `AuthenticationSuccessHandler`, `SecurityContextRepository`, `OncePerRequestFilter`.
- `spring-security-ldap` — `KerberosLdapContextSource` 가 `DefaultSpringSecurityContextSource` 를 확장 (GSSAPI 로 AD 조회).
- JRE 표준 — `javax.security.auth.*`(JAAS), `org.ietf.jgss.*`(GSS-API), `com.sun.security.jgss.*`(SUN 전용).
- Apache HttpClient 5 — `KerberosRestTemplate` 의 SPNEGO 스킴.

> 참고: 관련 모듈로 [../web/README.md](../web/README.md) (필터 체인·EntryPoint 일반), [../ldap/README.md](../ldap/README.md) 를 함께 보면 좋다.

## 내부 목차

1. [01-핵심개념과-전체구조.md](01-핵심개념과-전체구조.md) — Kerberos/SPNEGO/GSS-API 배경과 모듈 전체 지도, 두 인증 경로.
2. [02-SPNEGO-웹-인증-흐름.md](02-SPNEGO-웹-인증-흐름.md) — `SpnegoEntryPoint` → `SpnegoAuthenticationProcessingFilter` 의 협상/필터 흐름.
3. [03-인증-프로바이더와-토큰.md](03-인증-프로바이더와-토큰.md) — `KerberosServiceAuthenticationProvider`, `KerberosAuthenticationProvider`, 토큰 타입들.
4. [04-티켓검증과-JAAS-통합.md](04-티켓검증과-JAAS-통합.md) — `SunJaasKerberosTicketValidator`/`SunJaasKerberosClient`, JAAS `LoginConfig`, 멀티티어, AOT 힌트.
5. [05-클라이언트측-통합과-테스트.md](05-클라이언트측-통합과-테스트.md) — `KerberosRestTemplate`, `KerberosLdapContextSource`, `MiniKdc` 테스트 지원.

## 전체 패키지 구조 ASCII 맵

```
kerberos/                          (Gradle 멀티모듈)
├── kerberos-core/                 핵심 인증 추상 + SUN JAAS 구현
│   └── org.springframework.security.kerberos
│       ├── authentication/
│       │   ├── KerberosTicketValidator           (인터페이스) 티켓 검증 추상
│       │   ├── KerberosTicketValidation           검증 결과(username, Subject, GSSContext)
│       │   ├── KerberosServiceAuthenticationProvider  [A] 티켓 검증 프로바이더
│       │   ├── KerberosServiceRequestToken        [A] SPNEGO 토큰 보관 Authentication
│       │   ├── KerberosClient                     (인터페이스) KDC 로그인 추상
│       │   ├── KerberosAuthenticationProvider     [B] ID/PW → KDC 로그인 프로바이더
│       │   ├── KerberosUsernamePasswordAuthenticationToken [B] 결과 토큰
│       │   ├── KerberosAuthentication             JaasSubjectHolder 노출 인터페이스
│       │   ├── JaasSubjectHolder                  Subject + 발급 티켓 보관
│       │   ├── KerberosMultiTier                  위임(delegation) 멀티티어 헬퍼
│       │   └── sun/
│       │       ├── SunJaasKerberosTicketValidator [A] GSS 로 티켓 검증
│       │       ├── SunJaasKerberosClient          [B] Krb5LoginModule 로 로그인
│       │       ├── GlobalSunJaasKerberosConfig    krb5.conf/디버그 전역 설정
│       │       └── JaasUtil                        Subject 복제 유틸
│       └── aot/hint/KerberosRuntimeHints          GraalVM 리플렉션 힌트
│
├── kerberos-web/                  서블릿 SPNEGO 통합
│   └── ...web.authentication
│       ├── SpnegoEntryPoint                       401 + WWW-Authenticate: Negotiate
│       ├── SpnegoAuthenticationProcessingFilter   Negotiate 헤더 파싱 → 인증
│       └── ResponseHeaderSettingKerberosAuthenticationSuccessHandler 응답 토큰 헤더
│
├── kerberos-client/               우리가 클라이언트로서 호출할 때
│   └── ...client
│       ├── KerberosRestTemplate                   SPNEGO 인증 REST 호출
│       ├── ldap/KerberosLdapContextSource         GSSAPI 로 LDAP(AD) 조회
│       └── config/SunJaasKrb5LoginConfig          JAAS Configuration 빈
│
└── kerberos-test/                 테스트 지원
    └── ...test
        ├── MiniKdc                                내장 테스트용 KDC
        └── KerberosSecurityTestcase               KDC 기동 베이스 클래스
```
