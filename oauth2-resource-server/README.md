# oauth2-resource-server — OAuth 2.0 리소스 서버

> Spring Security 7.1.1-SNAPSHOT 기준

## 한 줄 정의

요청에 담긴 **OAuth 2.0 Bearer Token**(RFC 6750)을 추출·검증해, 그 토큰을 발급받은 주체를 `Authentication` 으로 만들어 주는 모듈이다. 즉 "API를 토큰으로 지키는" 서버 측 인증 장치다.

## 이 모듈이 푸는 문제

리소스 서버는 사용자 비밀번호나 세션을 직접 다루지 않는다. 대신 인가 서버(Authorization Server)가 발급한 액세스 토큰을 신뢰하고, 그 토큰이 유효한지/누구의 것인지/어떤 권한(scope)을 담고 있는지만 판단하면 된다. 이 모듈은 그 판단을 다음 두 갈래로 구현한다.

- **JWT 검증**: 토큰 자체가 서명된 JWT일 때, 인가 서버에 묻지 않고 서명·만료·발급자만 로컬에서 검증한다. (`JwtAuthenticationProvider`)
- **Opaque Token introspection**: 토큰이 의미 없는 문자열(opaque)일 때, 인가 서버의 introspection 엔드포인트(RFC 7662)에 POST로 물어 활성 여부와 속성을 받아온다. (`OpaqueTokenAuthenticationProvider`)

이 둘은 모두 **하나의 필터**(`BearerTokenAuthenticationFilter`) 뒤에서 동작하며, 실패 시 RFC 6750 규격의 `WWW-Authenticate` 응답을 돌려준다. 추가로 토큰 도난 방지를 위한 **DPoP**(RFC 9449) 바인딩, **리액티브(WebFlux)** 대응, 보호 리소스 메타데이터(RFC 9728) 발행까지 포함한다.

이 모듈은 토큰을 "검증"하는 데까지만 책임진다. JWT 서명 검증의 실제 엔진(`JwtDecoder`, Nimbus 연동, JWK 캐싱 등)과 토큰 도메인 타입(`OAuth2AccessToken`, `Jwt`)은 별도 모듈에 있다.

## 의존·연관 모듈

```
                     ┌──────────────────────────────┐
   클라이언트 ──토큰──▶│  oauth2-resource-server (이 모듈)│
                     └──────────────────────────────┘
                          │ 사용                  │ 사용
                          ▼                       ▼
              ┌────────────────────┐   ┌────────────────────────┐
              │ oauth2-jose        │   │ oauth2-core            │
              │  JwtDecoder        │   │  OAuth2AccessToken     │
              │  Jwt 서명검증 엔진  │   │  OAuth2Authenticated... │
              └────────────────────┘   │  OAuth2Error/Exception │
                          │            └────────────────────────┘
                          ▼
              ┌────────────────────────────────────────────┐
              │ spring-security-web                          │
              │  AuthenticationConverter / EntryPoint /      │
              │  AccessDeniedHandler / OncePerRequestFilter  │
              └────────────────────────────────────────────┘
                          │
                          ▼
              ┌────────────────────────────────────────────┐
              │ spring-security-core                         │
              │  AuthenticationManager / AuthenticationProvider│
              └────────────────────────────────────────────┘
```

- **oauth2-jose**: `JwtDecoder`/`ReactiveJwtDecoder`, JWK 기반 서명 검증, `DPoPProofJwtDecoderFactory` 제공. 이 모듈의 JWT 검증은 전적으로 여기에 위임한다. → [../oauth2-jose/README.md](../oauth2-jose/README.md)
- **oauth2-core**: `OAuth2AccessToken`, `OAuth2AuthenticatedPrincipal`, `OAuth2AuthenticationException`, `OAuth2Error`, introspection 클레임 이름 상수 등 공용 토큰 도메인. → [../oauth2-core/README.md](../oauth2-core/README.md)
- **web**: 서블릿 필터 인프라(`AuthenticationConverter`, `AuthenticationEntryPoint`, `SecurityContextRepository`)와 리액티브 대응 인프라. → [../web/README.md](../web/README.md)
- **core**: `AuthenticationManager`, `AuthenticationProvider`, `GrantedAuthority` 등 인증 추상. → [../core/README.md](../core/README.md)
- **config**: 이 모듈의 필터/프로바이더를 DSL(`oauth2ResourceServer { jwt {} / opaqueToken {} }`)로 조립하는 쪽. 이 모듈 자체에는 Configurer가 없다.

## 내부 목차

1. [01-아키텍처-개요.md](01-아키텍처-개요.md) — 모듈 전체 구조와 Bearer Token 인증의 큰 그림
2. [02-bearer-token-필터와-토큰-추출.md](02-bearer-token-필터와-토큰-추출.md) — `BearerTokenAuthenticationFilter`, `BearerTokenResolver`, `AuthenticationConverter`
3. [03-jwt-토큰-검증.md](03-jwt-토큰-검증.md) — `JwtAuthenticationProvider`, `JwtAuthenticationConverter`, scope→권한 변환, 멀티 테넌트
4. [04-opaque-token-introspection.md](04-opaque-token-introspection.md) — `OpaqueTokenAuthenticationProvider`, `OpaqueTokenIntrospector`, RFC 7662 introspection
5. [05-에러응답-dpop-리액티브.md](05-에러응답-dpop-리액티브.md) — RFC 6750 에러 응답, DPoP 바인딩, WebFlux, 보호 리소스 메타데이터

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.oauth2.server.resource
│
├── (루트)                          공용 도메인/에러
│   ├── BearerTokenError            RFC 6750 에러 (HTTP status + scope 포함)
│   ├── BearerTokenErrorCodes       invalid_request / invalid_token / insufficient_scope
│   ├── BearerTokenErrors           위 코드로 BearerTokenError 만드는 팩토리
│   ├── InvalidBearerTokenException invalid_token 전용 AuthenticationException
│   └── OAuth2ProtectedResourceMetadata*  RFC 9728 메타데이터 도메인
│
├── authentication                  인증 핵심 (Provider · Token · Converter)
│   ├── BearerTokenAuthenticationToken      인증 요청(미검증 토큰 문자열)
│   ├── AbstractOAuth2TokenAuthenticationToken / BearerTokenAuthentication / JwtAuthenticationToken  인증 결과
│   ├── JwtAuthenticationProvider           ── JWT 디코딩·검증 → 인증
│   ├── JwtAuthenticationConverter          Jwt → JwtAuthenticationToken
│   ├── JwtGrantedAuthoritiesConverter      scope/scp 클레임 → SCOPE_* 권한
│   ├── JwtIssuerAuthenticationManagerResolver  멀티 테넌트(iss별 매니저)
│   ├── OpaqueTokenAuthenticationProvider   ── introspection → 인증
│   ├── DPoPAuthenticationProvider          DPoP 증명 검증 (RFC 9449)
│   └── *Reactive* / JwtReactiveAuthenticationManager  WebFlux 대응
│
├── introspection                   Opaque Token introspection (RFC 7662)
│   ├── OpaqueTokenIntrospector             SPI: token → OAuth2AuthenticatedPrincipal
│   ├── SpringOpaqueTokenIntrospector       RestTemplate 기반 구현
│   ├── RestClientOpaqueTokenIntrospector   RestClient 기반 구현
│   └── *Reactive* / Bad/OAuth2IntrospectionException
│
└── web                             서블릿/리액티브 웹 연동
    ├── BearerTokenResolver / DefaultBearerTokenResolver / HeaderBearerTokenResolver  토큰 추출
    ├── BearerTokenAuthenticationEntryPoint          401 + WWW-Authenticate
    ├── OAuth2ProtectedResourceMetadataFilter        /.well-known/oauth-protected-resource
    ├── authentication/
    │   ├── BearerTokenAuthenticationFilter          ★ 진입 필터
    │   ├── BearerTokenAuthenticationConverter       요청 → BearerTokenAuthenticationToken
    │   └── DPoPAuthenticationConverter              요청 → DPoPAuthenticationToken
    ├── access/BearerTokenAccessDeniedHandler        403 insufficient_scope
    ├── server/ ...                                  리액티브 EntryPoint/Converter
    └── reactive/function/client/...                 WebClient/RestClient 토큰 릴레이
```
