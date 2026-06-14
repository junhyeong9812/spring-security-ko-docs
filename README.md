# Spring Security 한글 해설서

> **Spring Security 7.1.1-SNAPSHOT 소스 코드를 "책처럼 읽으며" 인증·인가의 내부 구조와 동작 원리를 이해하기 위한 한글 해설서.**

이 문서는 Spring Security의 22개 모듈을 하나씩 분석해, 각 모듈이 **무엇을 풀고(why)**, **어떤 핵심 타입으로 구성되며(what)**, **요청이 들어오면 내부에서 무슨 일이 일어나는가(how)** 를 코드 레벨에서 풀어쓴 책이다. 모든 구조도·흐름도는 ASCII 아트로 그려 GitHub/에디터에서 바로 읽을 수 있다.

> 자매편: [Spring Framework 한글 해설서](../spring-framework-ko-docs/README.md)

---

## 이 책을 읽는 법

- **대상 독자**: Spring Security를 설정해 써봤지만, "필터 체인 안에서 실제로 무슨 일이 일어나는지", "인증/인가가 어떻게 흐르는지"를 코드 레벨로 이해하고 싶은 중급 개발자.
- **고도(추상화 수준)**: 아키텍처 + 핵심 흐름 중심. 모든 메서드를 나열하는 API 레퍼런스가 아니라, **모듈의 책임 → 핵심 타입 → 대표 인증/인가 시나리오의 흐름**을 따라간다.
- **구성**: 각 모듈은 디렉토리 하나다. 그 안의 `README.md`(모듈 표지)부터 읽고, 번호가 붙은 하위 챕터(`01-…`, `02-…`)를 순서대로 읽으면 된다.
- **표기 규약**: 집필 규약은 [STYLE-GUIDE.md](./STYLE-GUIDE.md) 참고. 클래스/메서드 이름은 영어 원문(`SecurityFilterChain`, `authenticate()`), 설명은 한글이다.

---

## 전체 아키텍처 개요

Spring Security는 **서블릿 필터 체인**을 보안의 진입점으로 삼아, 그 안에서 **인증(authentication)** 과 **인가(authorization)** 를 처리한다. `core`가 도메인 모델의 토대를, `web`이 필터들을, `config`가 그것들을 조립하는 DSL을, 나머지 모듈이 다양한 인증 방식과 통합을 제공한다.

```
   HTTP 요청
      │
      ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  [web]  FilterChainProxy ─▶ SecurityFilterChain (필터 체인)         │ ◀── [config] DSL이 구성
   │   SecurityContextHolderFilter · 인증필터 · CsrfFilter ·            │      (HttpSecurity / @EnableWebSecurity)
   │   ExceptionTranslationFilter · AuthorizationFilter                 │
   └───────────────┬─────────────────────────────┬─────────────────────┘
       인증(authn) │                             │ 인가(authz)
                   ▼                             ▼
        AuthenticationManager            AuthorizationManager ── [access] 메서드/웹 보안
        └▶ ProviderManager                       │              [acl] 도메인객체 ACL
            └▶ AuthenticationProvider 들          │              [aspects] AspectJ 위빙
                ├─ DaoAuthenticationProvider ─▶ UserDetailsService   ┐
                ├─ OAuth2 / OIDC      [oauth2-*]                     │ 비밀번호 검증
                ├─ SAML2              [saml2]                        │   ▲
                ├─ LDAP / AD          [ldap]                        │   │
                ├─ CAS · Kerberos     [cas] [kerberos]              │  [crypto]
                └─ WebAuthn(패스키)   [webauthn]                     │  PasswordEncoder
                        │                                            ┘
                        ▼
              ┌───────────────────────────────────────────┐
              │ [core] SecurityContext / SecurityContextHolder │  ← 인증 결과 저장·전파
              └───────────────────────────────────────────┘

   다른 프로토콜에도 같은 모델 적용:  [messaging] WebSocket/STOMP · [rsocket] RSocket
   뷰 통합: [taglibs] JSP 태그   |   테스트: [test]   |   Spring Data 연동: [data]
```

- **토대(core)**: `Authentication`, `SecurityContext`, `GrantedAuthority`, `UserDetailsService`, `AuthenticationManager`, `AuthorizationManager` 등 모든 모듈이 공유하는 도메인 모델.
- **웹(web) + 설정(config)**: 필터 체인이 실제 방어선이고, `config`의 DSL이 이를 선언적으로 조립한다.
- **인증 방식들**: OAuth2/OIDC, SAML2, LDAP, CAS, Kerberos, WebAuthn은 각자 `AuthenticationProvider`/필터로 core의 계약에 꽂힌다.
- **인가**: `access`(메서드/웹 보안), `acl`(도메인 객체 단위), `aspects`(AspectJ 위빙)가 인가 결정을 담당한다.

---

## 추천 학습 순서

1. **도메인 토대** — [core](./core/README.md). `Authentication`/`SecurityContext`/`AuthenticationManager`/`AuthorizationManager`. 이후 모든 챕터의 어휘가 여기서 나온다.
2. **필터 체인** — [web](./web/README.md). 요청이 보안에 진입해 인증·인가·CSRF·세션을 거치는 실제 길.
3. **설정 DSL** — [config](./config/README.md). `@EnableWebSecurity`와 `HttpSecurity`가 필터 체인을 어떻게 조립하는지.
4. **비밀번호와 인가** — [crypto](./crypto/README.md) → [access](./access/README.md). 비밀번호 검증과 메서드 보안.
5. **인증 방식 선택** — 필요에 따라: [oauth2-client](./oauth2-client/README.md)/[oauth2-resource-server](./oauth2-resource-server/README.md) · [saml2](./saml2/README.md) · [ldap](./ldap/README.md) · [webauthn](./webauthn/README.md) 등.
6. **확장 영역** — [acl](./acl/README.md), [messaging](./messaging/README.md), [rsocket](./rsocket/README.md), [oauth2-authorization-server](./oauth2-authorization-server/README.md).
7. **테스트** — [test](./test/README.md). 보안 설정을 어떻게 검증하는지.

---

## 모듈 목차

### 1. 핵심 도메인

- **[core](./core/README.md)** — 인증·인가의 토대. Authentication, SecurityContext, AuthenticationManager, AuthorizationManager.
  - [01. 인증 도메인 모델 — Authentication과 GrantedAuthority](./core/01-인증-도메인-모델.md)
  - [02. SecurityContext와 컨텍스트 전파](./core/02-SecurityContext와-컨텍스트-전파.md)
  - [03. 인증 처리 — AuthenticationManager와 ProviderManager](./core/03-인증-처리-AuthenticationManager.md)
  - [04. UserDetails와 DaoAuthenticationProvider](./core/04-UserDetails와-DaoAuthenticationProvider.md)
  - [05. 인가 — AuthorizationManager 결정 모델](./core/05-인가-AuthorizationManager.md)
  - [06. 메서드 보안 — AOP로 인가를 메서드에 거는 법](./core/06-메서드-보안.md)
- **[crypto](./crypto/README.md)** — 비밀번호 인코딩과 암호화. PasswordEncoder 계층, Encryptors, 키 생성.
  - [01. 개요와 PasswordEncoder 계약](./crypto/01-개요와-PasswordEncoder-계약.md)
  - [02. 단방향 해시 인코더 (BCrypt · Argon2 · SCrypt · PBKDF2)](./crypto/02-단방향-해시-인코더.md)
  - [03. DelegatingPasswordEncoder와 비밀번호 업그레이드](./crypto/03-Delegating과-비밀번호-업그레이드.md)
  - [04. Password4j 통합 (7.0 신규)](./crypto/04-Password4j-통합.md)
  - [05. 데이터 암호화 · 키 생성 · 인코딩](./crypto/05-데이터암호화-키생성-인코딩.md)

### 2. 웹 보안과 설정

- **[web](./web/README.md)** — 서블릿 필터 체인. FilterChainProxy, 인증/인가 필터, CSRF, 세션, 예외 변환.
  - [01. 필터 체인 아키텍처 — 요청이 보안에 진입하는 길](./web/01-필터체인-아키텍처.md)
  - [02. 보안 컨텍스트 영속화](./web/02-보안컨텍스트-영속화.md)
  - [03. 인증 필터 — 자격 증명을 Authentication으로](./web/03-인증-필터.md)
  - [04. 예외 변환과 인가](./web/04-예외변환과-인가.md)
  - [05. 로그아웃과 세션 관리](./web/05-로그아웃과-세션관리.md)
  - [06. CSRF 보호와 보안 응답 헤더](./web/06-CSRF와-보안헤더.md)
  - [07. RequestMatcher와 요청 매칭](./web/07-RequestMatcher와-요청매칭.md)
- **[config](./config/README.md)** — 보안 설정 DSL. @EnableWebSecurity, HttpSecurity/WebSecurity, Configurer, Kotlin DSL.
  - [01. 빌더와 Configurer 기반구조](./config/01-빌더와-Configurer-기반구조.md)
  - [02. @EnableWebSecurity와 빈 구성](./config/02-EnableWebSecurity-와-빈-구성.md)
  - [03. HttpSecurity와 WebSecurity](./config/03-HttpSecurity-와-WebSecurity.md)
  - [04. HTTP Configurer 카탈로그](./config/04-HTTP-Configurer-카탈로그.md)
  - [05. 인증 구성](./config/05-인증-구성.md)
  - [06. 메서드 보안](./config/06-메서드-보안.md)
  - [07. Kotlin DSL과 XML 네임스페이스](./config/07-Kotlin-DSL-과-XML-네임스페이스.md)

### 3. 인가 · 메서드 보안

- **[access](./access/README.md)** — 접근 제어 인프라. SecurityInterceptor, AccessDecisionManager/Voter, 메서드 보안 메타데이터.
  - [01. 개요 — secure object 모델과 6개 핵심 추상](./access/01-개요-secure-object와-핵심-추상.md)
  - [02. AbstractSecurityInterceptor — 가로채기 엔진](./access/02-AbstractSecurityInterceptor-가로채기-엔진.md)
  - [03. 투표 기반 인가 — AccessDecisionManager와 Voter](./access/03-투표-기반-인가-AccessDecisionManager와-Voter.md)
  - [04. 메서드 보안 — 메타데이터 소스와 어노테이션](./access/04-메서드-보안-메타데이터와-어노테이션.md)
  - [05. 웹 · 채널 · 메시징 보안](./access/05-웹-채널-메시징-보안.md)
  - [06. run-as 권한 승격과 after-invocation / ACL](./access/06-run-as와-after-invocation-acl.md)
- **[acl](./acl/README.md)** — 도메인 객체 수준 ACL. Acl/AclService, ObjectIdentity, Sid, Permission 마스크.
  - [01. 핵심 개념과 도메인 모델](./acl/01-핵심-개념과-도메인-모델.md)
  - [02. 권한 마스크와 Permission](./acl/02-권한-마스크와-Permission.md)
  - [03. 인가 결정 흐름](./acl/03-인가-결정-흐름.md)
  - [04. JDBC 영속화와 조회 전략](./acl/04-JDBC-영속화와-조회-전략.md)
  - [05. Spring Security 통합 — hasPermission으로 가는 길](./acl/05-Spring-Security-통합.md)
- **[aspects](./aspects/README.md)** — AspectJ 메서드 보안 위빙. @PreAuthorize 등을 프록시 대신 .aj 애스펙트로.
  - [01. 왜 AspectJ 위빙인가 — 프록시 모드와의 차이](./aspects/01-왜-AspectJ-위빙인가.md)
  - [02. 애스펙트 구조와 위빙 흐름](./aspects/02-애스펙트-구조와-위빙-흐름.md)
  - [03. 설정 연결과 레거시 애스펙트](./aspects/03-설정-연결과-레거시-애스펙트.md)

### 4. OAuth2 · OIDC

- **[oauth2-core](./oauth2-core/README.md)** — OAuth2 공통 도메인. 토큰 타입, 에러, 표준 파라미터, OIDC 프린시펄.
  - [01. 토큰 도메인과 검증](./oauth2-core/01-토큰-도메인과-검증.md)
  - [02. 클레임 접근과 타입 변환](./oauth2-core/02-클레임-접근과-타입변환.md)
  - [03. 에러와 예외](./oauth2-core/03-에러와-예외.md)
  - [04. 프로토콜 상수와 엔드포인트 메시지](./oauth2-core/04-프로토콜-상수와-엔드포인트-메시지.md)
  - [05. 사용자 프린시펄과 OIDC](./oauth2-core/05-사용자-프린시펄과-OIDC.md)
  - [06. 스코프 기반 인가](./oauth2-core/06-스코프-기반-인가.md)
- **[oauth2-client](./oauth2-client/README.md)** — OAuth2/OIDC 클라이언트. ClientRegistration, 인가 코드 그랜트, OAuth2 로그인.
  - [01. 클라이언트 등록과 인가된 클라이언트 모델](./oauth2-client/01-클라이언트-등록과-인가된-클라이언트-모델.md)
  - [02. 인가 코드 그랜트 흐름과 서블릿 필터](./oauth2-client/02-인가-코드-그랜트-흐름과-필터.md)
  - [03. 인증 프로바이더와 Token Endpoint 클라이언트](./oauth2-client/03-인증-프로바이더와-토큰-엔드포인트.md)
  - [04. OIDC 로그인과 UserInfo](./oauth2-client/04-OIDC-로그인과-유저인포.md)
  - [05. 인가된 클라이언트 매니저와 프로바이더](./oauth2-client/05-인가된-클라이언트-매니저와-프로바이더.md)
  - [06. 리액티브 스택(WebFlux) 대응](./oauth2-client/06-리액티브-스택-대응.md)
- **[oauth2-jose](./oauth2-jose/README.md)** — JWT/JOSE. Jwt, JwtDecoder/Encoder(Nimbus), 서명·검증, 클레임 검증.
  - [01. JWT 도메인 모델과 알고리즘](./oauth2-jose/01-JWT-도메인-모델과-알고리즘.md)
  - [02. JWT 디코딩과 서명 검증](./oauth2-jose/02-JWT-디코딩과-서명검증.md)
  - [03. 클레임 검증기 체계](./oauth2-jose/03-클레임-검증기-체계.md)
  - [04. JWT 인코딩과 서명](./oauth2-jose/04-JWT-인코딩과-서명.md)
  - [05. 리액티브 디코더와 DPoP](./oauth2-jose/05-리액티브-디코더와-DPoP.md)
- **[oauth2-resource-server](./oauth2-resource-server/README.md)** — 리소스 서버. Bearer 토큰 인증, JWT/opaque 토큰 검증.
  - [01. 아키텍처 개요 — Bearer Token 인증의 큰 그림](./oauth2-resource-server/01-아키텍처-개요.md)
  - [02. Bearer Token 필터와 토큰 추출](./oauth2-resource-server/02-bearer-token-필터와-토큰-추출.md)
  - [03. JWT 토큰 검증](./oauth2-resource-server/03-jwt-토큰-검증.md)
  - [04. Opaque Token Introspection](./oauth2-resource-server/04-opaque-token-introspection.md)
  - [05. 에러 응답 · DPoP · 리액티브](./oauth2-resource-server/05-에러응답-dpop-리액티브.md)
- **[oauth2-authorization-server](./oauth2-authorization-server/README.md)** — OAuth2 인가 서버. 인가/토큰 엔드포인트, RegisteredClient, 토큰 생성.
  - [01. 개념과 도메인 모델 — 인가 서버가 "기억하는 것"](./oauth2-authorization-server/01-개념과-도메인-모델.md)
  - [02. 필터 체인과 요청 처리 골격](./oauth2-authorization-server/02-필터-체인과-요청-처리-골격.md)
  - [03. 인가 엔드포인트와 동의 흐름 — /oauth2/authorize](./oauth2-authorization-server/03-인가-엔드포인트와-동의-흐름.md)
  - [04. 토큰 엔드포인트와 그랜트 처리 — /oauth2/token](./oauth2-authorization-server/04-토큰-엔드포인트와-그랜트-처리.md)
  - [05. 토큰 생성기와 커스터마이징](./oauth2-authorization-server/05-토큰-생성기와-커스터마이징.md)
  - [06. OIDC와 부가 엔드포인트](./oauth2-authorization-server/06-OIDC와-부가-엔드포인트.md)

### 5. 외부 인증 연동 (페더레이션)

- **[saml2](./saml2/README.md)** — SAML2 서비스 제공자(SP). RelyingPartyRegistration, AuthnRequest, assertion 검증.
  - [01. 핵심 개념과 OpenSAML 통합](./saml2/01-핵심-개념과-OpenSAML-통합.md)
  - [02. RelyingPartyRegistration과 레지스트리](./saml2/02-RelyingPartyRegistration과-레지스트리.md)
  - [03. AuthnRequest 생성으로 로그인 시작](./saml2/03-AuthnRequest-생성으로-로그인-시작.md)
  - [04. 응답 검증과 인증 프로바이더](./saml2/04-응답검증과-인증프로바이더.md)
  - [05. 메타데이터와 싱글 로그아웃(SLO)](./saml2/05-메타데이터와-싱글로그아웃.md)
- **[webauthn](./webauthn/README.md)** — WebAuthn/패스키. 등록·인증 흐름, RelyingParty, credential 저장.
  - [01. 개념과 전체 구조](./webauthn/01-개념과-전체-구조.md)
  - [02. api 도메인 모델](./webauthn/02-api-도메인-모델.md)
  - [03. 등록 흐름 (Registration)](./webauthn/03-등록-흐름.md)
  - [04. 인증 흐름 (Authentication)](./webauthn/04-인증-흐름.md)
  - [05. Relying Party 연산과 저장소 (management)](./webauthn/05-relyingparty-연산과-저장소.md)
  - [06. 직렬화 (jackson)와 AOT 힌트](./webauthn/06-직렬화-jackson.md)
- **[ldap](./ldap/README.md)** — LDAP/AD 인증. LdapAuthenticationProvider, Authenticator, 권한 매핑.
  - [01. 개요와 핵심 추상화](./ldap/01-개요와-핵심-추상화.md)
  - [02. 인증 프로바이더와 Authenticator](./ldap/02-인증-프로바이더와-Authenticator.md)
  - [03. 사용자 조회와 권한 매핑](./ldap/03-사용자-조회와-권한-매핑.md)
  - [04. Active Directory 전용 인증](./ldap/04-Active-Directory.md)
  - [05. 인프라 — ContextSource, Template, 내장 서버](./ldap/05-인프라-ContextSource-Template-내장서버.md)
- **[cas](./cas/README.md)** — CAS SSO 통합. CasAuthenticationProvider/Filter, 티켓 검증, 프록시 티켓.
  - [01. 아키텍처 개요 — CAS SSO의 전체 그림](./cas/01-아키텍처-개요.md)
  - [02. 인증 진입 · 수신 필터 · 게이트웨이](./cas/02-인증-진입-필터-게이트웨이.md)
  - [03. 티켓 검증 — Provider · 토큰 3종 · UserDetailsService](./cas/03-티켓검증-프로바이더-토큰.md)
  - [04. 프록시 티켓 · 무상태 캐시 · 직렬화](./cas/04-프록시티켓-무상태캐시-직렬화.md)
- **[kerberos](./kerberos/README.md)** — Kerberos/SPNEGO 인증. SPNEGO 필터, 티켓 검증, JAAS 통합.
  - [01. 핵심 개념과 전체 구조](./kerberos/01-핵심개념과-전체구조.md)
  - [02. SPNEGO 웹 인증 흐름](./kerberos/02-SPNEGO-웹-인증-흐름.md)
  - [03. 인증 프로바이더와 토큰](./kerberos/03-인증-프로바이더와-토큰.md)
  - [04. 티켓 검증과 JAAS 통합](./kerberos/04-티켓검증과-JAAS-통합.md)
  - [05. 클라이언트 측 통합과 테스트 지원](./kerberos/05-클라이언트측-통합과-테스트.md)

### 6. 프로토콜 · 뷰 통합

- **[messaging](./messaging/README.md)** — WebSocket/STOMP 메시지 보안. ChannelInterceptor 기반 인증/인가, WebSocket CSRF.
  - [01. 메시지 보안 아키텍처 — 필터가 아니라 ChannelInterceptor](./messaging/01-메시지-보안-아키텍처.md)
  - [02. 인증 컨텍스트 전파](./messaging/02-인증-컨텍스트-전파.md)
  - [03. MessageMatcher — 메시지를 무엇으로 식별하는가](./messaging/03-MessageMatcher.md)
  - [04. 메시지 인가 — AuthorizationChannelInterceptor](./messaging/04-메시지-인가.md)
  - [05. WebSocket CSRF](./messaging/05-WebSocket-CSRF.md)
- **[rsocket](./rsocket/README.md)** — RSocket 보안. PayloadInterceptor 체인, 인증/인가, 메타데이터 인코딩.
  - [01. 개요와 인터셉터 체인](./rsocket/01-개요와-인터셉터-체인.md)
  - [02. 인증 (Authentication)](./rsocket/02-인증.md)
  - [03. 인가와 페이로드 매칭](./rsocket/03-인가와-페이로드-매칭.md)
  - [04. 메타데이터 인코딩](./rsocket/04-메타데이터-인코딩.md)
- **[taglibs](./taglibs/README.md)** — JSP 보안 태그. authorize/authentication/accesscontrollist/csrf 태그.
  - [01. `<authorize>` 인가 태그](./taglibs/01-authorize-인가-태그.md)
  - [02. `<authentication>` 출력 태그와 `<accesscontrollist>` ACL 태그](./taglibs/02-authentication-과-acl-태그.md)
  - [03. CSRF 토큰 태그](./taglibs/03-csrf-태그.md)

### 7. 테스트 · 데이터

- **[test](./test/README.md)** — 보안 테스트 지원. @WithMockUser 계열, TestSecurityContextHolder, MockMvc/WebTestClient 통합.
  - [01. 핵심 개념과 구조 — 가짜 인증을 어디에, 어떻게 꽂는가](./test/01-핵심-개념과-구조.md)
  - [02. 어노테이션 기반 인증 — @WithMockUser / @WithUserDetails / @WithAnonymousUser](./test/02-어노테이션-기반-인증.md)
  - [03. MockMvc 테스트 지원 — RequestPostProcessor · Configurer · ResultMatcher](./test/03-MockMvc-테스트-지원.md)
  - [04. 리액티브 테스트 지원 — WebTestClient Mutator와 Reactor Context 전파](./test/04-리액티브-WebTestClient-지원.md)
- **[data](./data/README.md)** — Spring Data 통합. SecurityEvaluationContextExtension으로 쿼리에서 principal 참조.
  - [01. SpEL 쿼리에 현재 사용자 노출 — SecurityEvaluationContextExtension](./data/01-SpEL-쿼리에-현재-사용자-노출.md)
  - [02. AOT 반환객체 프록시 힌트 — AuthorizeReturnObjectDataHintsRegistrar](./data/02-AOT-반환객체-프록시-힌트.md)

---

## 표기 규약

집필 규약(고도, 문체, ASCII 다이어그램 규칙, 챕터 골격)은 [STYLE-GUIDE.md](./STYLE-GUIDE.md)에 정리되어 있다.

> 총 22개 모듈 · 약 110개 챕터. Spring Security 7.1.1-SNAPSHOT 소스 기준으로 작성되었다.

---

## 출처 및 라이선스

- 이 해설서는 [Spring Security](https://github.com/spring-projects/spring-security) **7.1.1-SNAPSHOT** 소스 코드를 분석해 작성한 **학습용 2차 저작물(한글 해설)** 이다.
- 원본 Spring Security는 **Apache License 2.0** 하에 배포되며, 저작권은 원저작자(VMware/Broadcom 및 기여자들)에게 있다.
- 본 문서(텍스트·다이어그램)는 **[Creative Commons BY 4.0](https://creativecommons.org/licenses/by/4.0/)** 으로 제공된다. 인용된 코드 조각의 저작권·라이선스는 원본 Apache-2.0을 따른다. 자세한 내용은 [LICENSE](./LICENSE) 참고.
- 본 저장소는 **비공식 학습 자료**이며 Spring 프로젝트/Broadcom과 제휴·보증 관계가 없다.
