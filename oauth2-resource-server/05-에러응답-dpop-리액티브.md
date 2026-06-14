# 05. 에러 응답 · DPoP · 리액티브

## 무엇을 / 왜

인증이 실패했을 때 리소스 서버는 클라이언트에게 "왜 거절했는지"를 RFC 6750이 정한 형식으로 알려줘야 한다. 또한 토큰 탈취에 더 강한 **DPoP**(RFC 9449) 바인딩, **WebFlux** 환경 대응, 보호 리소스 메타데이터(RFC 9728) 발행처럼 앞 장에서 미뤄둔 주변 장치들이 있다. 이 챕터는 이 "가장자리"들을 한데 모은다.

## 핵심 타입

```
에러 응답
  BearerTokenError / BearerTokenErrorCodes / BearerTokenErrors   RFC 6750 에러 도메인
  BearerTokenAuthenticationEntryPoint   인증 실패 → 401 + WWW-Authenticate
  BearerTokenAccessDeniedHandler        인가 실패 → 403 insufficient_scope

DPoP (RFC 9449)
  DPoPAuthenticationConverter   요청(Authorization: DPoP ...) → DPoPAuthenticationToken
  DPoPAuthenticationProvider    DPoP 증명(proof) 검증 + 토큰 바인딩 확인

리액티브 (WebFlux)
  JwtReactiveAuthenticationManager / OpaqueTokenReactiveAuthenticationManager
  ServerBearerTokenAuthenticationConverter
  BearerTokenServerAuthenticationEntryPoint / BearerTokenServerAccessDeniedHandler

메타데이터 / 토큰 릴레이
  OAuth2ProtectedResourceMetadataFilter        /.well-known/oauth-protected-resource
  Servlet/ServerBearerExchangeFilterFunction   하위 호출에 토큰 전파
```

## 1) 에러 응답 — 인증 실패 vs 인가 실패

두 실패 지점을 다른 핸들러가 처리하며, 그 차이가 그대로 HTTP 의미가 된다.

```
토큰 자체가 없거나 무효       → 401 Unauthorized
  BearerTokenAuthenticationEntryPoint.commence()
  WWW-Authenticate: Bearer error="invalid_token",
                    error_description="...", error_uri="...",
                    resource_metadata="https://.../.well-known/oauth-protected-resource"

토큰은 유효하나 권한 부족     → 403 Forbidden
  BearerTokenAccessDeniedHandler.handle()
  WWW-Authenticate: Bearer error="insufficient_scope", ...
```

### `BearerTokenAuthenticationEntryPoint.commence()`

실패 예외에서 RFC 6750 파라미터를 뽑아 `WWW-Authenticate` 헤더를 조립한다.

- 예외가 `OAuth2AuthenticationException` 이면 `error`/`error_description`/`error_uri` 를 채운다.
- 그 `OAuth2Error` 가 `BearerTokenError` 면 거기 담긴 `scope` 와 **HTTP status** 를 쓴다. 즉 상태 코드가 에러 종류에 따라 결정된다 — `invalid_request`→400, `invalid_token`→401, `insufficient_scope`→403 (`BearerTokenErrors` 팩토리가 코드별 status를 박아둔다).
- 7.1부터 `resource_metadata` 파라미터를 자동으로 붙여, 클라이언트가 보호 리소스 메타데이터 문서를 찾아갈 수 있게 한다.

### `BearerTokenAccessDeniedHandler.handle()`

인가(권한) 단계에서 거부됐을 때 호출된다. **요청 주체가 OAuth2 토큰 인증(`AbstractOAuth2TokenAuthenticationToken`)으로 판명된 경우에만** `insufficient_scope` 에러를 싣는다. 토큰이 아예 없는 익명 요청이었다면 scheme(`Bearer`)과 realm만 알리고 끝낸다 — "토큰은 맞는데 권한이 모자란다"와 "토큰이 없다"를 구분하는 것이다.

## 2) DPoP — 토큰을 키에 묶기 (RFC 9449)

일반 Bearer 토큰은 "들고 있으면 누구나 쓸 수 있는" 무기명 토큰이라, 탈취되면 그대로 악용된다. DPoP는 토큰을 **클라이언트의 키쌍에 바인딩**해, 매 요청마다 그 개인키로 서명한 **증명(proof) JWT** 를 함께 보내게 한다. 키가 없으면 토큰만 훔쳐도 못 쓴다.

```
요청 헤더
  Authorization: DPoP <access-token>
  DPoP: <proof-jwt>              (이 요청에 대해 클라이언트 개인키로 서명)
        │
   DPoPAuthenticationConverter.convert()
        ├ "Authorization: DPoP ..." 패턴 매칭 (헤더 1개만 허용)
        ├ DPoP 헤더에서 proof 추출 (1개만 허용)
        └ DPoPAuthenticationToken(accessToken, proof, method, requestUrl)
        │
   DPoPAuthenticationProvider.authenticate()
        ├ 1) 내부 tokenAuthenticationManager 로 access-token 먼저 인증
        │       (JWT or introspection — 즉 앞 장들의 흐름 재사용)
        ├ 2) DPoPProofContext 구성 (proof + accessToken + method + targetUri)
        ├ 3) DPoPProofJwtDecoderFactory 로 proof 검증 디코더 생성, proof.decode()
        │       검증 항목:
        │       · ath  : proof의 ath == SHA-256(access-token)   [AthClaimValidator]
        │       · jkt  : access-token의 cnf.jkt == proof jwk thumbprint [JwkThumbprintValidator]
        │       · htm/htu/iat 등 기본 검증 (DPoPProofJwtDecoderFactory 기본 validator)
        │       실패 → OAuth2Error(invalid_dpop_proof)
        └ return (1단계의 access-token 인증 결과)
```

핵심 검증 두 가지가 DPoP의 안전성을 떠받친다.

- **`ath` 클레임**: proof 안에 담긴 access-token 해시가 실제 access-token의 SHA-256과 일치해야 한다. proof가 "바로 이 토큰"에 대한 것임을 보장한다.
- **`jkt` 바인딩**: access-token의 `cnf.jkt`(확인 키 thumbprint)가 proof를 서명한 공개키(JWK)의 thumbprint와 일치해야 한다. 토큰이 "바로 이 키"에 묶여 있음을 보장한다.

이 `jkt` 바인딩 때문에, 02장에서 본 `BearerTokenAuthenticationFilter` 의 **다운그레이드 차단**이 의미를 갖는다. DPoP 바인딩 토큰(`cnf.jkt` 보유)을 `Authorization: Bearer` 로 들고 오면 — proof 검증을 우회하려는 시도이므로 — 필터가 `invalid_token` 으로 막는다.

## 3) 리액티브(WebFlux) 대응

서블릿 스택의 부품마다 리액티브 쌍이 존재한다. 책임은 동일하고 반환 타입만 `Mono`/`Flux` 다.

| 서블릿 | 리액티브 |
|--------|----------|
| `JwtAuthenticationProvider` | `JwtReactiveAuthenticationManager` |
| `OpaqueTokenAuthenticationProvider` | `OpaqueTokenReactiveAuthenticationManager` |
| `BearerTokenAuthenticationConverter` | `ServerBearerTokenAuthenticationConverter` |
| `BearerTokenAuthenticationEntryPoint` | `BearerTokenServerAuthenticationEntryPoint` |
| `BearerTokenAccessDeniedHandler` | `BearerTokenServerAccessDeniedHandler` |
| `JwtIssuerAuthenticationManagerResolver` | `JwtIssuerReactiveAuthenticationManagerResolver` |

예로 `JwtReactiveAuthenticationManager.authenticate()` 는 같은 일을 논블로킹 파이프라인으로 표현한다.

```java
return Mono.justOrEmpty(authentication)
    .filter((a) -> a instanceof BearerTokenAuthenticationToken)
    .cast(BearerTokenAuthenticationToken.class)
    .map(BearerTokenAuthenticationToken::getToken)
    .flatMap(this.jwtDecoder::decode)                 // ReactiveJwtDecoder
    .flatMap(this.jwtAuthenticationConverter::convert)
    .cast(Authentication.class)
    .onErrorMap(JwtException.class, this::onError);   // Bad→Invalid, 그 외→ServiceException
```

`ServerBearerTokenAuthenticationConverter` 의 토큰 추출 규칙(헤더/쿼리/바디 세 자리, 중복 토큰 거부, 기본 비활성)도 서블릿판 `DefaultBearerTokenResolver` 와 동일하다.

## 4) 부가 장치 — 메타데이터와 토큰 릴레이

- **`OAuth2ProtectedResourceMetadataFilter`** (RFC 9728): `/.well-known/oauth-protected-resource` GET 요청에 리소스 서버 자신의 메타데이터(JSON)를 응답한다. resource 식별자를 요청에서 동적으로 도출하고, `bearer_methods_supported`, mTLS 바인딩 지원 여부 등을 싣는다. 위 EntryPoint의 `resource_metadata` 파라미터가 가리키는 문서가 이것이다.
- **`Servlet/ServerBearerExchangeFilterFunction`**: 리소스 서버가 또 다른 다운스트림 API를 호출할 때, 현재 인증의 토큰을 `Authorization: Bearer` 로 자동 전파(token relay)하는 `WebClient`/`RestClient` 필터다.

## 설계 포인트 / 확장점

- **에러 응답 커스터마이징**: `BearerTokenAuthenticationEntryPoint.setRealmName(...)`/`setResourceMetadataParameterResolver(...)`, `BearerTokenAccessDeniedHandler.setRealmName(...)` 로 realm·메타데이터 링크를 조정한다.
- **DPoP 검증 정책 교체**: `DPoPAuthenticationProvider.setDPoPProofVerifierFactory(...)` 로 proof 검증 디코더 팩토리를 교체할 수 있다. 내부적으로 토큰 인증은 주입된 `AuthenticationManager` 에 위임하므로, DPoP는 JWT/Opaque 어느 검증 방식과도 조합된다.
- **서블릿/리액티브 동형성**: 두 스택이 같은 개념(Converter·Provider/Manager·EntryPoint·AccessDeniedHandler)을 데칼코마니로 갖는다. 한쪽을 이해하면 다른 쪽도 바로 읽힌다.

## 정리

인증 실패는 401 `invalid_token`, 인가 실패는 403 `insufficient_scope` 로 RFC 6750 `WWW-Authenticate` 헤더에 실린다. DPoP는 토큰을 클라이언트 키에 묶고(`ath`·`jkt` 검증) 다운그레이드를 차단해 탈취 내성을 높이며, WebFlux에서는 모든 부품이 `Mono`/`Flux` 쌍으로 제공된다. 이로써 토큰 추출(02) → JWT/Opaque 검증(03·04) → 에러·DPoP·리액티브(05)까지, 리소스 서버가 요청 하나를 받아 인증을 끝내고 응답을 빚는 전 과정이 닫힌다.
