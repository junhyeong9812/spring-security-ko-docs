# core 모듈 — Spring Security 인증·인가의 핵심 도메인

> 버전 기준: Spring Security 7.1.1-SNAPSHOT
> 소스 위치: `core/src/main/java/org/springframework/security/...`

## 한 줄 정의

`core`는 "**누가**(Authentication) **무엇을 할 수 있는가**(Authorization)"라는 보안의 두 축을, 웹/서블릿/리액티브 같은 특정 환경에 의존하지 않는 **순수 도메인 모델과 처리 엔진**으로 정의하는 모듈이다.

## 이 모듈이 푸는 문제

Spring Security의 다른 모듈(`web`, `oauth2`, `saml2`, `config` 등)은 전부 "필터", "엔드포인트", "프로토콜" 같은 **환경에 묶인 코드**다. 하지만 그 모든 환경이 공통으로 필요로 하는 것이 있다.

- 인증된 사용자를 표현하는 **공통 타입**(`Authentication`, `GrantedAuthority`)
- 그 사용자를 "현재 실행 흐름"에 묶어두는 **저장소**(`SecurityContextHolder`)
- 자격 증명을 검증해 인증 객체를 만들어내는 **처리 엔진**(`AuthenticationManager` / `ProviderManager`)
- 사용자 데이터를 어디서 읽어올지에 대한 **추상화**(`UserDetailsService`)
- 접근 허용 여부를 판단하는 **결정 엔진**(`AuthorizationManager`)

`core`는 이것들을 **인터페이스 우선(interface-first)**으로 정의하고, 기본 구현을 제공한다. 폼 로그인이든 JWT든 LDAP이든, 결국 모두 이 모듈의 `Authentication`을 만들어 `SecurityContextHolder`에 넣고, 이 모듈의 `AuthorizationManager`로 접근을 판단한다. 즉 **core는 Spring Security 전체가 공유하는 어휘(vocabulary)이자 심장**이다.

## 의존 / 연관 관계

```
        ┌────────────────────────────────────────────────┐
        │  config / web / oauth2 / saml2 / ...            │  (상위 모듈: core를 사용)
        └───────────────────────┬────────────────────────┘
                                │ 사용
                                ▼
        ┌────────────────────────────────────────────────┐
        │                    core (본 모듈)                 │
        │  Authentication · SecurityContext ·             │
        │  AuthenticationManager · AuthorizationManager   │
        └───────────────────────┬────────────────────────┘
                                │ 사용
                                ▼
        ┌──────────────┬──────────────────┬───────────────┐
        │  crypto       │  spring-core/    │  reactor      │
        │ (PasswordEnc) │  spring-context  │ (리액티브 전파) │
        └──────────────┴──────────────────┴───────────────┘
```

- **상위(이 모듈에 의존):** `config`(설정 DSL), `web`(필터 체인·서블릿), `oauth2`/`saml2`(프로토콜) 등 거의 모든 모듈.
- **하위(이 모듈이 의존):** `spring-core`/`spring-context`(빈·이벤트·메시지), `spring-security-crypto`(`PasswordEncoder` — `DaoAuthenticationProvider`가 사용), Reactor(리액티브 컨텍스트 전파, 선택적).

## 내부 목차 (읽는 순서)

1. [01-인증-도메인-모델.md](01-인증-도메인-모델.md) — `Authentication`, `GrantedAuthority`, principal/credentials의 의미와 토큰 계층.
2. [02-SecurityContext와-컨텍스트-전파.md](02-SecurityContext와-컨텍스트-전파.md) — `SecurityContextHolder`/Strategy, ThreadLocal vs 전역 vs 리액티브 전파.
3. [03-인증-처리-AuthenticationManager.md](03-인증-처리-AuthenticationManager.md) — `AuthenticationManager`/`ProviderManager`/`AuthenticationProvider` 체인과 이벤트.
4. [04-UserDetails와-DaoAuthenticationProvider.md](04-UserDetails와-DaoAuthenticationProvider.md) — 사용자 데이터 추상화와 폼 로그인 인증의 실제 동작.
5. [05-인가-AuthorizationManager.md](05-인가-AuthorizationManager.md) — `AuthorizationManager` 결정 모델, 권한·역할·역할 계층.
6. [06-메서드-보안.md](06-메서드-보안.md) — `@PreAuthorize` 등 AOP 기반 메서드 인가.

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security
│
├── core                         ── 인증 도메인의 최상위 어휘
│   ├── Authentication            (인증 요청/결과 토큰의 최상위 인터페이스)
│   ├── GrantedAuthority          (부여된 권한)
│   ├── CredentialsContainer      (자격 증명 소거 계약)
│   ├── AuthenticationException   (인증 예외 루트)
│   │
│   ├── context                   ── 컨텍스트 저장/전파  → 02장
│   │   ├── SecurityContext / SecurityContextImpl
│   │   ├── SecurityContextHolder / *HolderStrategy (ThreadLocal/Global/...)
│   │   └── ReactiveSecurityContextHolder
│   ├── authority                 ── GrantedAuthority 구현
│   │   ├── SimpleGrantedAuthority / AuthorityUtils
│   │   └── mapping/ (GrantedAuthoritiesMapper)
│   └── userdetails               ── 사용자 데이터 추상화  → 04장
│       ├── UserDetails / UserDetailsService / User
│       ├── jdbc/JdbcDaoImpl, memory/, cache/
│       └── (Reactive 변형들)
│
├── authentication               ── 인증 처리 엔진  → 03장
│   ├── AuthenticationManager / ProviderManager
│   ├── AuthenticationProvider (+ Anonymous/RememberMe/Testing)
│   ├── *AuthenticationToken (UsernamePassword/Anonymous/RememberMe)
│   ├── AbstractAuthenticationToken
│   ├── dao/ (AbstractUserDetailsAuthenticationProvider, DaoAuthenticationProvider) → 04장
│   ├── event/ (인증 성공·실패 이벤트)
│   ├── password/ (CompromisedPasswordChecker)
│   └── ott/, jaas/ (원타임토큰, JAAS)
│
├── authorization                ── 인가 결정 엔진  → 05장
│   ├── AuthorizationManager / ReactiveAuthorizationManager
│   ├── AuthorityAuthorizationManager / AuthoritiesAuthorizationManager
│   ├── AuthenticatedAuthorizationManager
│   ├── AuthorizationManagers (anyOf/allOf/not 조합)
│   ├── AuthorizationDecision / AuthorizationResult
│   └── method/ (AOP 메서드 보안)  → 06장
│       ├── AuthorizationManager{Before,After}MethodInterceptor
│       ├── PreAuthorize/PostAuthorize/Secured/Jsr250 AuthorizationManager
│       └── PreFilter/PostFilter 인터셉터
│
├── access                       ── 권한 표현식·역할 계층 지원
│   ├── AccessDeniedException / PermissionEvaluator
│   ├── hierarchicalroles/ (RoleHierarchy, RoleHierarchyImpl)
│   └── expression/ (SpEL 보안 표현식 루트)
│
├── provisioning                 ── 사용자 CRUD  → 04장
│   ├── UserDetailsManager (extends UserDetailsService)
│   ├── InMemoryUserDetailsManager / JdbcUserDetailsManager
│   └── GroupManager
│
├── concurrent / scheduling / task ── SecurityContext를 보존하는 실행 래퍼
├── converter / util / aot / jackson ── 직렬화·매처·AOT 지원 유틸
└── context                      ── DelegatingApplicationListener (이벤트 위임)
```

각 장에서 "이 타입이 **왜** 존재하고, 요청이 들어오면 **무슨 일**이 일어나는지"를 코드 근거와 함께 설명한다.
