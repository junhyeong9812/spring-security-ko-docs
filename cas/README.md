# cas 모듈 — CAS 기반 SSO(Single Sign-On) 통합

## 한 줄 정의

Apereo CAS(Central Authentication Service) 서버와 Spring Security를 연결해, **티켓(ticket) 기반 엔터프라이즈 SSO**를 구현하는 모듈이다.

## 이 모듈이 푸는 문제

여러 웹 애플리케이션이 하나의 중앙 인증 서버(CAS 서버)에 로그인을 위임하고 싶을 때를 생각해보자. 사용자는 CAS 서버에서 **한 번만** 로그인하고, 각 애플리케이션은 비밀번호를 직접 보지 않은 채 CAS가 발급한 **티켓**만으로 사용자를 인증한다. 이것이 CAS가 제공하는 SSO(Single Sign-On)다.

Spring Security 입장에서 CAS 통합은 두 가지 일을 해야 한다.

1. **인증되지 않은 사용자를 CAS 서버로 보내기** — 보호된 자원에 접근하면 사용자의 브라우저를 CAS 로그인 페이지로 리다이렉트하고, 로그인 후 우리 애플리케이션의 콜백 URL(`service`)로 다시 돌아오게 한다.
2. **돌아온 티켓을 검증하기** — 콜백 URL에 `ticket` 파라미터로 실려 온 불투명한(opaque) 티켓 문자열을 CAS 서버에 되물어(`TicketValidator`) 검증하고, 검증 결과인 `Assertion`으로부터 `UserDetails`와 권한을 만들어 `SecurityContext`에 채운다.

이 모듈은 위 두 일을 각각 `CasAuthenticationEntryPoint`(보내기)와 `CasAuthenticationFilter` + `CasAuthenticationProvider`(받아 검증하기)로 나누어 구현한다. 여기에 더해 한 서비스가 다른 백엔드 서비스를 사용자 신원으로 호출하는 **프록시 티켓(proxy ticket)**, 세션이 없는 호출자를 위한 **무상태(stateless) 티켓 캐시**, 그리고 로그인 강제 없이 SSO 세션 유무만 확인하는 **gateway 모드**까지 다룬다.

## 의존 / 연관 관계

- **외부 라이브러리**: `org.apereo.cas.client`(JA-SIG/Apereo CAS Client) — `TicketValidator`, `Assertion`, `AttributePrincipal`, `WebUtils`, `CommonUtils`, `ProxyGrantingTicketStorage` 등 CAS 프로토콜 처리의 실제 작업을 담당한다. 이 모듈은 그 위에 Spring Security의 인증 추상화를 얹는 얇은 통합 계층이다.
- **spring-security-web**: `AbstractAuthenticationProcessingFilter`, `AuthenticationEntryPoint`, `SecurityContextRepository`, `RequestCache`, `RedirectStrategy`, `WebAuthenticationDetails` 등 필터/웹 인프라를 그대로 활용한다. (→ [../web/README.md](../web/README.md))
- **spring-security-core**: `AuthenticationProvider`, `AuthenticationManager`, `UserDetails`, `UserDetailsService`, `GrantedAuthority` 등 인증의 핵심 계약. (→ [../core/README.md](../core/README.md))
- **연관**: 실제 보안 설정(필터 체인 등록, EntryPoint 연결)은 보통 `spring-security-config`의 DSL/XML로 이뤄진다.

## 내부 목차

1. [01-아키텍처-개요.md](01-아키텍처-개요.md) — CAS SSO의 전체 그림, 패키지 구조, "로그인 한 바퀴" 흐름
2. [02-인증-진입-필터-게이트웨이.md](02-인증-진입-필터-게이트웨이.md) — `CasAuthenticationEntryPoint`, `CasAuthenticationFilter`, gateway 리다이렉트
3. [03-티켓검증-프로바이더-토큰.md](03-티켓검증-프로바이더-토큰.md) — `CasAuthenticationProvider`, 티켓 검증, 토큰 3종, `UserDetailsService`
4. [04-프록시티켓-무상태캐시-직렬화.md](04-프록시티켓-무상태캐시-직렬화.md) — 프록시 티켓, `ServiceAuthenticationDetails`, `StatelessTicketCache`, Jackson 직렬화

## 전체 패키지 구조 (ASCII 맵)

```
cas/src/main/java/org/springframework/security/cas/
│
├── ServiceProperties.java            # 이 서비스(우리 앱)의 CAS 관련 설정값 보관
│                                     #  - service(콜백 URL), artifactParameter("ticket"),
│                                     #    serviceParameter("service"), sendRenew,
│                                     #    authenticateAllArtifacts
├── SamlServiceProperties.java        # CAS의 SAML 모드용 파라미터 기본값(SAMLart/TARGET)
│
├── authentication/                   # ── 인증 핵심(프로바이더 + 토큰 + 캐시) ──
│   ├── CasAuthenticationProvider     #   티켓을 검증해 Authentication을 만드는 Provider
│   ├── CasServiceTicketAuthenticationToken  # 검증 전: 티켓을 담은 인증 요청 토큰
│   ├── CasAssertionAuthenticationToken      # UserDetailsService 호출용 임시 토큰
│   ├── CasAuthenticationToken         #   검증 후: 인증 완료 토큰(UserDetails+Assertion)
│   ├── ServiceAuthenticationDetails   #   동적 service URL을 제공하는 details 계약
│   ├── StatelessTicketCache           #   무상태 호출자용 티켓 캐시 인터페이스
│   ├── NullStatelessTicketCache       #     기본 구현(캐시 안 함)
│   └── SpringCacheBasedTicketCache    #     Spring Cache 기반 구현
│
├── web/                              # ── 서블릿 필터 계층 ──
│   ├── CasAuthenticationEntryPoint    #   미인증 시 CAS 로그인 페이지로 리다이렉트
│   ├── CasAuthenticationFilter        #   콜백 URL에서 ticket을 받아 인증 처리
│   ├── CasGatewayAuthenticationRedirectFilter  # gateway=true로 조용히 SSO 확인
│   ├── CasGatewayResolverRequestMatcher        #   gateway 무한루프 방지 매처
│   └── authentication/
│       ├── ServiceAuthenticationDetailsSource  # 요청에서 동적 service URL 추출
│       └── DefaultServiceAuthenticationDetails #   그 기본 구현
│
├── userdetails/                     # ── Assertion → UserDetails 변환 ──
│   ├── AbstractCasAssertionUserDetailsService  # Assertion 기반 UserDetails 템플릿
│   └── GrantedAuthorityFromAssertionAttributesUserDetailsService # 속성→권한 매핑
│
├── jackson/                         # Jackson 3 기반 토큰 직렬화 Mixin/Module
│   └── CasJacksonModule
└── jackson2/                        # Jackson 2 기반(7.0부터 deprecated)
    └── CasJackson2Module
```
