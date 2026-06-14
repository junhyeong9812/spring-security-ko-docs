# oauth2-authorization-server — OAuth2 / OIDC 인가 서버

> Spring Security 7.1.1-SNAPSHOT 기준. 소스: `oauth2/oauth2-authorization-server/src/main/java`

## 한 줄 정의

`oauth2-authorization-server`는 **OAuth 2.1 / OpenID Connect 1.0 인가 서버(Authorization Server)** 를 구현한 모듈이다. 클라이언트 등록, 인가 코드 발급, 토큰 발급/검증/폐기, 동의(consent), JWK 공개, 서버 메타데이터 공개까지 "토큰을 발급하는 쪽"의 전 과정을 담당한다.

## 이 모듈이 푸는 문제

리소스 서버(`oauth2-resource-server`)는 토큰을 **검증**하고, 클라이언트(`oauth2-client`)는 토큰을 **소비**한다. 그렇다면 그 토큰은 **누가 발급**하는가? 보통은 Keycloak, Auth0, Okta 같은 외부 제품을 쓴다. 이 모듈은 그 발급 서버를 **Spring 애플리케이션 안에서 직접** 돌릴 수 있게 해준다.

구체적으로 다음을 해결한다.

- **표준 엔드포인트 제공**: `/oauth2/authorize`(인가), `/oauth2/token`(토큰), `/oauth2/introspect`(검사), `/oauth2/revoke`(폐기), `/oauth2/jwks`(공개키), `/userinfo`(OIDC) 등을 RFC 6749 / 8414 / 7662 / OIDC Core 규격대로 노출.
- **인가 그랜트 처리**: authorization_code(+ PKCE), client_credentials, refresh_token, device_code, token_exchange를 각각의 `AuthenticationProvider`로 구현.
- **상태 저장**: 발급한 인가/토큰의 수명·폐기 여부를 `OAuth2Authorization`으로 보관하고, 클라이언트 메타데이터를 `RegisteredClient`로 보관.
- **토큰 생성 추상화**: 액세스 토큰(JWT 또는 불투명), 리프레시 토큰, ID 토큰을 `OAuth2TokenGenerator` 파이프라인으로 생성하고 `OAuth2TokenCustomizer`로 클레임을 커스터마이즈.
- **동의 화면**: 리소스 오너가 클라이언트에 어떤 scope를 허용할지 선택하는 consent 흐름.

## 의존·연관 관계

```
                 oauth2-core  (AuthorizationGrantType, OAuth2AccessToken,
                   ▲           OAuth2Error, ClientAuthenticationMethod ...)
                   │
   oauth2-jose ────┤  (JwtEncoder, JwsAlgorithm, Nimbus JWK)
                   │
   ┌───────────────┴────────────────────────────┐
   │      oauth2-authorization-server            │   ← 이 모듈
   │  (RegisteredClient, OAuth2Authorization,    │
   │   엔드포인트 Filter, AuthenticationProvider, │
   │   OAuth2TokenGenerator)                      │
   └───────────────┬────────────────────────────┘
                   │ 등록·배선
   spring-security-config
   (OAuth2AuthorizationServerConfigurer + *EndpointConfigurer)
                   │ 사용
   spring-security-web  (OncePerRequestFilter, AuthenticationConverter,
                         SecurityFilterChain, RequestMatcher)
```

- **위로**: `oauth2-core`(토큰/그랜트/에러 타입), `oauth2-jose`(JWT 인코딩과 JWK)에 의존한다.
- **옆으로**: 동의 흐름의 `Authentication`은 `spring-security-web`의 `AuthenticationConverter` → `AuthenticationManager` → `AuthenticationProvider` 패턴을 그대로 재사용한다. 즉 **새 인증 메커니즘이 아니라 기존 인증 인프라 위에 OAuth2 도메인을 얹은 것**이다.
- **아래로(배선)**: 실제 `SecurityFilterChain`에 필터를 꽂는 `OAuth2AuthorizationServerConfigurer`와 각 `*EndpointConfigurer`는 이 모듈이 아니라 **`spring-security-config` 모듈**(`config/src/main/java/.../configurers/oauth2/server/authorization/`)에 있다. 이 모듈은 "재료"(Filter, Provider)를 제공하고, config 모듈이 "조립"한다.
- **리소스 서버로서도 동작**: Dynamic Client Registration·UserInfo 엔드포인트는 발급한 액세스 토큰을 받아야 하므로, 인가 서버 스스로가 내부적으로 `oauth2ResourceServer().jwt()`를 켠다(`OAuth2AuthorizationServerConfigurer.init()` 참고).

## 하위 챕터 목차

1. [01-개념과-도메인-모델.md](01-개념과-도메인-모델.md) — `RegisteredClient`, `OAuth2Authorization`, `OAuth2AuthorizationConsent`와 그 저장소/서비스, 그리고 `*Settings`. "무엇을 기억하는가".
2. [02-필터-체인과-요청-처리-골격.md](02-필터-체인과-요청-처리-골격.md) — 엔드포인트 Filter들의 공통 구조, `AuthorizationServerContext`, 클라이언트 인증 Filter. "요청이 들어오면 어떤 순서로".
3. [03-인가-엔드포인트와-동의-흐름.md](03-인가-엔드포인트와-동의-흐름.md) — `/oauth2/authorize`, 인가 코드 발급, consent 게이트.
4. [04-토큰-엔드포인트와-그랜트-처리.md](04-토큰-엔드포인트와-그랜트-처리.md) — `/oauth2/token`, 그랜트별 `AuthenticationProvider`, 액세스/리프레시 토큰 발급.
5. [05-토큰-생성기와-커스터마이징.md](05-토큰-생성기와-커스터마이징.md) — `OAuth2TokenGenerator` 파이프라인, `JwtGenerator`, `OAuth2TokenContext`, `OAuth2TokenCustomizer`.
6. [06-OIDC와-부가-엔드포인트.md](06-OIDC와-부가-엔드포인트.md) — OIDC(UserInfo/Logout/Discovery), 토큰 검사·폐기, JWK Set, 서버 메타데이터, Device/PAR 개요.

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.oauth2.server.authorization
│
├── (root)                      도메인 모델과 저장소 계약
│   ├── OAuth2Authorization               발급된 인가/토큰의 상태 묶음 (집계 루트)
│   ├── OAuth2AuthorizationCode           인가 코드 토큰
│   ├── OAuth2AuthorizationConsent        리소스 오너가 허가한 권한 집합
│   ├── OAuth2AuthorizationService        인가 저장 계약 (save/remove/findByToken)
│   │     ├ InMemoryOAuth2AuthorizationService
│   │     └ JdbcOAuth2AuthorizationService
│   ├── OAuth2AuthorizationConsentService  동의 저장 계약
│   │     ├ InMemoryOAuth2AuthorizationConsentService
│   │     └ JdbcOAuth2AuthorizationConsentService
│   ├── OAuth2TokenType                   토큰 타입 식별자(code/access_token/...)
│   └── OAuth2TokenIntrospection          introspection 응답 클레임
│
├── client/                     클라이언트 등록 정보
│   ├── RegisteredClient                  등록 클라이언트(빌더 기반 불변 객체)
│   ├── RegisteredClientRepository        클라이언트 조회 계약
│   ├── InMemoryRegisteredClientRepository
│   └── JdbcRegisteredClientRepository
│
├── settings/                   런타임 설정
│   ├── AuthorizationServerSettings       엔드포인트 URI, issuer
│   ├── ClientSettings                    PKCE 요구, 동의 요구 등 클라이언트별 설정
│   ├── TokenSettings                     토큰 수명, 포맷, 서명 알고리즘
│   ├── OAuth2TokenFormat                  SELF_CONTAINED / REFERENCE
│   └── ConfigurationSettingNames         설정 키 상수
│
├── context/                    "현재 요청"의 인가 서버 환경
│   ├── AuthorizationServerContext        issuer + settings 접근
│   └── AuthorizationServerContextHolder  ThreadLocal 보관
│
├── authentication/             그랜트/클라이언트 인증 도메인 (가장 큰 패키지)
│   ├── OAuth2AuthorizationCodeRequestAuthenticationProvider  인가요청+동의 판단
│   ├── OAuth2AuthorizationConsentAuthenticationProvider      동의 제출 처리
│   ├── OAuth2AuthorizationCodeAuthenticationProvider         코드→토큰 교환
│   ├── OAuth2ClientCredentialsAuthenticationProvider         client_credentials
│   ├── OAuth2RefreshTokenAuthenticationProvider              refresh_token
│   ├── OAuth2DeviceCode/DeviceAuthorization/DeviceVerification...  Device Flow
│   ├── OAuth2TokenExchangeAuthenticationProvider             RFC 8693
│   ├── OAuth2PushedAuthorizationRequestAuthenticationProvider PAR
│   ├── OAuth2TokenIntrospectionAuthenticationProvider        토큰 검사
│   ├── OAuth2TokenRevocationAuthenticationProvider           토큰 폐기
│   ├── ClientSecret/JwtClientAssertion/PublicClient/X509...Provider  클라이언트 인증
│   ├── DPoPProofVerifier, CodeVerifierAuthenticator          증명 검증
│   └── *AuthenticationToken / *AuthenticationContext         그랜트별 토큰·문맥
│
├── token/                      토큰 생성 파이프라인
│   ├── OAuth2TokenGenerator              토큰 생성 계약(@FunctionalInterface)
│   ├── DelegatingOAuth2TokenGenerator    여러 생성기 체인
│   ├── JwtGenerator                      JWT 액세스/ID 토큰 생성
│   ├── OAuth2AccessTokenGenerator        불투명 액세스 토큰 생성
│   ├── OAuth2RefreshTokenGenerator       리프레시 토큰 생성
│   ├── OAuth2TokenContext / DefaultOAuth2TokenContext  생성 입력 문맥
│   ├── OAuth2TokenCustomizer             클레임 후킹
│   └── JwtEncodingContext / OAuth2TokenClaimsContext   커스터마이저 문맥
│
├── web/                        OAuth2 엔드포인트 Filter
│   ├── OAuth2AuthorizationEndpointFilter        /oauth2/authorize
│   ├── OAuth2TokenEndpointFilter                /oauth2/token
│   ├── OAuth2ClientAuthenticationFilter         클라이언트 인증
│   ├── OAuth2TokenIntrospectionEndpointFilter   /oauth2/introspect
│   ├── OAuth2TokenRevocationEndpointFilter       /oauth2/revoke
│   ├── NimbusJwkSetEndpointFilter               /oauth2/jwks
│   ├── OAuth2AuthorizationServerMetadataEndpointFilter  /.well-known/...
│   ├── OAuth2Device.../PushedAuthorizationRequest.../ClientRegistration... Filter
│   ├── DefaultConsentPage                       기본 동의 화면 HTML
│   └── authentication/                          요청→Authentication 변환기와 핸들러
│       ├── *AuthenticationConverter             HTTP 파라미터 → *AuthenticationToken
│       ├── OAuth2AccessTokenResponseAuthenticationSuccessHandler
│       └── OAuth2ErrorAuthenticationFailureHandler
│
├── oidc/                       OpenID Connect 1.0
│   ├── authentication/  OidcUserInfo / OidcLogout / OidcClientRegistration Provider
│   ├── web/             OidcUserInfo/Logout/ProviderConfiguration/ClientRegistration Filter
│   └── OidcProviderConfiguration / OidcClientRegistration  메타데이터·등록 모델
│
├── http/converter/             HttpMessageConverter (메타데이터, introspection, 등록)
├── converter/                  RegisteredClient ↔ OAuth2ClientRegistration 변환
├── jackson2/                   도메인 객체 JSON 직렬화 모듈
└── aot/hint/                   GraalVM 네이티브 이미지 힌트
```
