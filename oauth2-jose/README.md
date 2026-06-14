# oauth2-jose — JWT/JOSE 처리 모듈

> Spring Security 7.1.1-SNAPSHOT 기준

## 한 줄 정의

`oauth2-jose`는 **JWT(JSON Web Token)를 생성(인코딩·서명)하고, 검증(디코딩·서명검증·클레임검증)하는 저수준 빌딩 블록**을 제공하는 모듈이다. 내부 암호 연산은 Nimbus JOSE + JWT SDK에 위임하고, Spring Security는 그 위에 도메인 모델(`Jwt`)·검증 정책(`OAuth2TokenValidator<Jwt>`)·설정 빌더를 얹는다.

## 이 모듈이 푸는 문제

Resource Server(리소스 서버)는 들어온 Bearer 토큰이 JWT일 때 "이 문자열이 진짜 신뢰할 수 있는 발급자가 서명한, 아직 유효한 토큰인가?"를 판단해야 한다. Authorization Server(인가 서버)는 반대로 "클레임을 담아 서명된 JWT를 만들어 발급"해야 한다. 이 두 가지 요구를 다음과 같이 분리·추상화한다.

- **디코딩/검증 측**: `JwtDecoder.decode(String)` 한 메서드로 "파싱 → 서명 검증 → 클레임 검증"을 일관되게 수행. 키는 JWK Set URI, 공개키, 시크릿키, `JWKSource` 등 다양한 출처에서 가져온다.
- **인코딩/서명 측**: `JwtEncoder.encode(JwtEncoderParameters)`로 헤더 + 클레임을 받아 서명된 compact JWS 문자열을 생산.
- **검증 정책**: 만료(`exp`)·발급자(`iss`)·대상(`aud`)·타입(`typ`)·인증서 지문(`x5t#S256`) 등 RFC별 규칙을 작은 `OAuth2TokenValidator<Jwt>` 조각으로 만들고 조합.
- **특수 토큰**: RFC 9068 access token(`at+jwt`), RFC 9449 DPoP proof(`dpop+jwt`) 같은 변형을 전용 팩토리/빌더로 지원.

이 모듈 자체는 필터나 인증 흐름을 갖지 않는다. `Jwt`를 만들고 검증하는 **순수 라이브러리 계층**이며, 실제 HTTP 인증 흐름은 상위 모듈(`oauth2-resource-server`, `oauth2-client`, OAuth2 Authorization Server)이 이 디코더/인코더를 주입받아 구동한다.

## 의존 / 연관 관계

```
        ┌────────────────────────────────────────────────┐
        │  상위(이 모듈을 소비)                            │
        │  oauth2-resource-server : JwtDecoder 로 토큰 인증 │
        │  oauth2-client(OIDC)    : ID 토큰 디코딩         │
        │  Authorization Server   : JwtEncoder 로 토큰 발급 │
        └───────────────────────┬────────────────────────┘
                                │ 주입(DI)
                                ▼
        ┌────────────────────────────────────────────────┐
        │  oauth2-jose (이 모듈)                           │
        │   org.springframework.security.oauth2.jwt        │
        │   org.springframework.security.oauth2.jose(.jws) │
        └───────────────────────┬────────────────────────┘
              │ 도메인/검증 토대             │ 암호 연산 위임
              ▼                            ▼
   oauth2-core                    Nimbus JOSE + JWT SDK
   (AbstractOAuth2Token,          (JWTProcessor, JWKSource,
    OAuth2TokenValidator,          JWSSigner, JWSAlgorithm ...)
    OAuth2Error, ClaimAccessor)
```

- **oauth2-core**: `Jwt`는 `AbstractOAuth2Token`을 상속하고 `JwtClaimAccessor`는 `ClaimAccessor`를 상속한다. 검증 결과는 `OAuth2Error`/`OAuth2TokenValidatorResult`로 표현한다.
- **Nimbus JOSE + JWT (com.nimbusds.\*)**: 실제 파싱·서명·검증·JWK 처리는 전부 Nimbus가 담당한다. Spring 클래스는 대부분 Nimbus 객체를 구성·위임하는 어댑터다.
- **spring-web**: JWK Set/OIDC 메타데이터를 가져올 때 `RestOperations`(서블릿)·`WebClient`(리액티브)를 사용한다.

## 내부 목차

1. [01-JWT-도메인-모델과-알고리즘.md](01-JWT-도메인-모델과-알고리즘.md) — `Jwt`, `JwtClaimAccessor`, 헤더/클레임 셋, `JwsAlgorithm` 체계
2. [02-JWT-디코딩과-서명검증.md](02-JWT-디코딩과-서명검증.md) — `JwtDecoder`, `NimbusJwtDecoder`, JWK 소스, decode 흐름
3. [03-클레임-검증기-체계.md](03-클레임-검증기-체계.md) — `OAuth2TokenValidator<Jwt>` 패밀리와 `JwtValidators` 조합
4. [04-JWT-인코딩과-서명.md](04-JWT-인코딩과-서명.md) — `JwtEncoder`, `NimbusJwtEncoder`, JWK 선택·서명
5. [05-리액티브-디코더와-DPoP.md](05-리액티브-디코더와-DPoP.md) — `ReactiveJwtDecoder`, `JwtDecoderFactory`, DPoP proof 검증

## 전체 패키지 구조 ASCII 맵

```
oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/
│
├── jose/                          ← JWA(알고리즘) 추상화
│   ├── JwaAlgorithm                 알고리즘 최상위 인터페이스(getName)
│   └── jws/                        ← JWS 서명 알고리즘
│       ├── JwsAlgorithm             JWS 알고리즘 마커 인터페이스
│       ├── JwsAlgorithms            알고리즘 이름 상수(RS256, HS256 ...)
│       ├── SignatureAlgorithm       비대칭 서명 enum (RS/ES/PS)
│       └── MacAlgorithm             대칭 MAC enum (HS256/384/512)
│
└── jwt/                           ← JWT 핵심 (대부분의 코드)
    │  [도메인 모델]
    ├── Jwt                          토큰 1건 (headers + claims), 1장
    ├── JwtClaimAccessor             표준 클레임 접근(iss/sub/aud/exp...)
    ├── JwtClaimNames / JwtClaimsSet 클레임 이름 상수 / 인코딩용 클레임 셋
    ├── JoseHeader / JwsHeader       JOSE/JWS 헤더와 빌더
    ├── JoseHeaderNames              헤더 이름 상수(alg, typ, kid, jku...)
    │
    │  [디코딩 — 2장]
    ├── JwtDecoder                   decode(String) → Jwt  (서블릿)
    ├── NimbusJwtDecoder             Nimbus 기반 표준 구현 + 빌더 4종
    ├── JwtDecoders                  issuer 메타데이터로 디코더 생성
    ├── JwtDecoderProviderConfigurationUtils  OIDC/AS 메타데이터 조회
    ├── MappedJwtClaimSetConverter   원시 클레임 → 타입 변환
    ├── SupplierJwtDecoder           디코더 초기화 지연(lazy)
    │
    │  [클레임 검증 — 3장]
    ├── JwtValidators                표준 검증기 묶음 + RFC9068 빌더
    ├── JwtClaimValidator            임의 클레임 Predicate 검증
    ├── JwtTimestampValidator        exp/nbf 시각 검증(+clock skew)
    ├── JwtIssuerValidator           iss 검증
    ├── JwtAudienceValidator         aud 검증
    ├── JwtIssuedAtValidator         iat 검증
    ├── JwtTypeValidator             typ 헤더 검증
    ├── X509CertificateThumbprintValidator  x5t#S256(mTLS) 검증
    │
    │  [인코딩 — 4장]
    ├── JwtEncoder / JwtEncoderParameters    encode 계약
    ├── NimbusJwtEncoder             Nimbus 기반 서명 구현 + 키쌍 빌더
    ├── JWKS                         키 → JWK 빌더 헬퍼(package-private)
    │
    │  [리액티브 & DPoP & 팩토리 — 5장]
    ├── ReactiveJwtDecoder           decode(String) → Mono<Jwt>
    ├── NimbusReactiveJwtDecoder     리액티브 표준 구현
    ├── ReactiveJwtDecoders / ReactiveJWKSource / ...  리액티브 보조
    ├── SupplierReactiveJwtDecoder   리액티브 지연 초기화
    ├── JwtDecoderFactory / ReactiveJwtDecoderFactory  컨텍스트별 디코더 생성
    ├── DPoPProofContext             DPoP proof 검증 입력
    ├── DPoPProofJwtDecoderFactory   dpop+jwt 전용 디코더 팩토리
    │
    │  [예외]
    └── JwtException / BadJwtException / JwtValidationException
        JwtEncodingException / JwtDecoderInitializationException
```
