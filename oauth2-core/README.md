# oauth2-core — OAuth 2.0 / OIDC 공통 도메인

> Spring Security 7.1.1-SNAPSHOT 기준. 소스: `oauth2/oauth2-core/src/main/java`

## 한 줄 정의

`oauth2-core`는 OAuth 2.0과 OpenID Connect 1.0을 다루는 모든 상위 모듈(클라이언트·리소스 서버·인가 서버)이 **공유하는 값 객체·상수·예외·변환기**를 모아 둔, 프로토콜의 "어휘 사전"이자 도메인 모델 계층이다.

## 이 모듈이 푸는 문제

OAuth 2.0은 여러 역할(클라이언트, 리소스 서버, 인가 서버)이 같은 규약 위에서 대화하는 프로토콜이다. 이때 `access_token`, `grant_type`, `invalid_request` 같은 **표준 문자열**, 토큰·에러·인가 요청 같은 **메시지 구조**, 클레임을 안전하게 꺼내는 **타입 변환 규칙**은 모든 역할이 똑같이 알아야 한다.

이 공통 어휘를 각 모듈이 따로 정의하면 철자 불일치·중복·상호 비호환이 생긴다. `oauth2-core`는 이 모든 것을 한곳에 정의해서:

- RFC 6749/6750/7662/8628, OIDC Core 1.0의 **표준 파라미터/엔드포인트/에러 코드를 상수**로 박제한다.
- `OAuth2AccessToken`, `OAuth2Error`, `OidcIdToken` 등 **불변 값 객체**로 메시지를 모델링한다.
- `ClaimAccessor`/`ClaimConversionService`로 **JSON에서 읽은 느슨한 타입(Map)을 안전한 자바 타입**으로 바꾼다.
- `OAuth2TokenValidator`로 토큰 검증을 조립 가능한 형태로 추상화한다.
- `OAuth2AuthorizationManagers`로 **스코프 기반 인가**를 표준화한다.

이 모듈 자체에는 필터도, 인증 엔드포인트도 없다. 순수 도메인이다.

## 의존 / 연관 관계

```
        ┌──────────────────────────────────────────────────┐
        │  oauth2-client · oauth2-resource-server ·         │
        │  oauth2-jose · (인가 서버: spring-authz-server)   │  ← 이 모듈을 소비
        └───────────────────────┬──────────────────────────┘
                                 │ uses
                                 ▼
                       ┌───────────────────┐
                       │   oauth2-core     │   (값 객체 / 상수 / 변환 / 인가)
                       └─────────┬─────────┘
                                 │ depends on
            ┌────────────────────┼─────────────────────┐
            ▼                    ▼                     ▼
   spring-security-core   spring-core (convert)   spring-web (http,
   (Authentication,       ClaimConversionService   MessageConverter,
    GrantedAuthority,     가 사용                   UriBuilder)
    AuthorizationManager)
```

- **상위 의존**: `spring-security-core` — `AuthenticatedPrincipal`, `GrantedAuthority`, `AuthenticationException`, `AuthorizationManager`를 확장/구현한다.
- **인프라 의존**: `spring-core`의 `ConversionService`/`Converter`, `spring-web`의 `HttpMessageConverter`·`UriBuilder`·`BodyExtractor`.
- **소비자**: [`../oauth2-client`](../oauth2-client/README.md), `oauth2-resource-server`, `oauth2-jose` 등 모든 OAuth2 상위 모듈이 이 모듈의 타입을 그대로 사용한다.

## 하위 챕터 목차

| # | 파일 | 다루는 내용 |
|---|------|------------|
| 01 | [01-토큰-도메인과-검증.md](01-토큰-도메인과-검증.md) | `OAuth2Token` 계층, AccessToken/RefreshToken/DeviceCode, `OAuth2TokenValidator` 조립 |
| 02 | [02-클레임-접근과-타입변환.md](02-클레임-접근과-타입변환.md) | `ClaimAccessor`, `ClaimConversionService`, `ClaimTypeConverter`, 인트로스펙션 클레임 |
| 03 | [03-에러와-예외.md](03-에러와-예외.md) | `OAuth2Error`/`OAuth2ErrorCodes`, 인증/인가 예외 두 갈래 |
| 04 | [04-프로토콜-상수와-엔드포인트-메시지.md](04-프로토콜-상수와-엔드포인트-메시지.md) | grant type·파라미터 상수, 인가 요청/응답·토큰 응답 메시지, HTTP 변환기 |
| 05 | [05-사용자-프린시펄과-OIDC.md](05-사용자-프린시펄과-OIDC.md) | `OAuth2AuthenticatedPrincipal`→`OAuth2User`→`OidcUser`, ID 토큰/UserInfo |
| 06 | [06-스코프-기반-인가.md](06-스코프-기반-인가.md) | `OAuth2AuthorizationManagers`, `hasScope`/`hasAnyScope`/`hasAllScopes` |

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.oauth2.core
│
├── (루트 패키지) ─── 토큰·상수·예외·검증의 핵심
│     OAuth2Token, AbstractOAuth2Token
│     OAuth2AccessToken (+ TokenType), OAuth2RefreshToken
│     OAuth2DeviceCode, OAuth2UserCode
│     OAuth2TokenValidator, OAuth2TokenValidatorResult, DelegatingOAuth2TokenValidator
│     ClaimAccessor, OAuth2TokenIntrospectionClaimAccessor (+ ClaimNames)
│     OAuth2Error, OAuth2ErrorCodes
│     OAuth2AuthenticationException, OAuth2AuthorizationException
│     AuthorizationGrantType, ClientAuthenticationMethod, AuthenticationMethod
│     OAuth2AuthenticatedPrincipal, DefaultOAuth2AuthenticatedPrincipal
│
├── authorization/ ─ 스코프 기반 인가
│     OAuth2AuthorizationManagers, OAuth2ReactiveAuthorizationManagers
│     OAuth2AuthorizationManagerFactory, DefaultOAuth2AuthorizationManagerFactory
│
├── converter/ ───── 클레임 타입 변환
│     ClaimConversionService, ClaimTypeConverter
│     ObjectTo{Boolean,Instant,ListString,MapStringObject,String,URL}Converter
│
├── endpoint/ ────── 인가/토큰 엔드포인트 메시지 + 파라미터 상수
│     OAuth2AuthorizationRequest, OAuth2AuthorizationResponse, OAuth2AuthorizationExchange
│     OAuth2AccessTokenResponse, OAuth2DeviceAuthorizationResponse
│     OAuth2ParameterNames, PkceParameterNames, OAuth2AuthorizationResponseType
│     Default{Map↔Response} 변환기
│
├── http/converter/ ─ Spring MVC용 HttpMessageConverter
│     OAuth2AccessTokenResponseHttpMessageConverter
│     OAuth2ErrorHttpMessageConverter, OAuth2DeviceAuthorizationResponseHttpMessageConverter
│
├── web/reactive/function/ ─ WebClient용 BodyExtractor
│     OAuth2BodyExtractors, OAuth2AccessTokenResponseBodyExtractor
│
├── user/ ────────── OAuth2 사용자 프린시펄
│     OAuth2User, DefaultOAuth2User, OAuth2UserAuthority
│
└── oidc/ ────────── OpenID Connect 도메인
      OidcIdToken (+ Builder), OidcUserInfo, OidcScopes
      IdTokenClaimAccessor, StandardClaimAccessor (+ ClaimNames)
      AddressStandardClaim, DefaultAddressStandardClaim
      ├── endpoint/  OidcParameterNames
      └── user/      OidcUser, DefaultOidcUser, OidcUserAuthority
```

각 챕터는 "이 타입이 왜 존재하고, 요청/응답이 흐를 때 무슨 일을 하는가"를 중심으로 읽으면 된다. 상위 모듈(`oauth2-client` 등)의 필터·프로바이더는 결국 여기서 정의한 값 객체를 만들고, 검증하고, 직렬화한다.
