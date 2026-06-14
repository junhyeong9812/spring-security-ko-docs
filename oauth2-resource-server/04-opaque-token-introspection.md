# 04. Opaque Token Introspection

## 무엇을 / 왜

모든 액세스 토큰이 JWT인 것은 아니다. 인가 서버가 **의미 없는 랜덤 문자열(opaque token)** 을 발급하면, 리소스 서버는 토큰만 봐서는 유효성도 속성도 알 수 없다. 이때는 인가 서버의 **introspection 엔드포인트**(RFC 7662)에 "이 토큰 살아 있나? 누구 거고 무슨 scope야?"를 직접 물어야 한다. 이 챕터는 그 질의를 수행하는 `OpaqueTokenIntrospector` 와, 결과를 인증으로 만드는 `OpaqueTokenAuthenticationProvider` 를 다룬다.

JWT 방식과 정반대 트레이드오프다. JWT는 빠르지만(네트워크 없음) 즉시 폐기(revocation)가 어렵고, introspection은 매 요청 네트워크 왕복이 들지만 인가 서버가 토큰을 실시간으로 무효화할 수 있다.

## 핵심 타입

```
OpaqueTokenAuthenticationProvider  (AuthenticationProvider)
   │ uses
   ├─ OpaqueTokenIntrospector            ← SPI: String token → OAuth2AuthenticatedPrincipal
   │    ├ SpringOpaqueTokenIntrospector      (RestTemplate / RestOperations 기반)
   │    └ RestClientOpaqueTokenIntrospector  (RestClient 기반)
   │
   └─ OpaqueTokenAuthenticationConverter  ← introspection 결과 → Authentication
        (기본: OpaqueTokenAuthenticationProvider::convert → BearerTokenAuthentication)
```

`OpaqueTokenIntrospector.introspect(token)` 한 메서드가 핵심 SPI다. "토큰 문자열을 받아, 인가 서버에 물어, 활성 토큰의 속성을 `OAuth2AuthenticatedPrincipal` 로 돌려준다." 비활성/무효 토큰이면 예외를 던진다.

## 동작 흐름 — Introspection 인증

```
authenticate(BearerTokenAuthenticationToken bearer)
   │
   ├─ 1) introspector.introspect(bearer.getToken())
   │        │  [SpringOpaqueTokenIntrospector]
   │        ├ requestEntityConverter: token → POST RequestEntity
   │        │     body: token=<opaque>,  Accept: application/json
   │        │     클라이언트 인증: BasicAuthenticationInterceptor(clientId, secret)
   │        ├ restOperations.exchange(...) → 인가 서버 introspection 엔드포인트
   │        ├ adaptToNimbusResponse: HTTP 200 확인 + "active" 판정
   │        │     · active == false  →  BadOpaqueTokenException
   │        ├ convertClaimsSet: aud/exp/iat/scope 등 타입 정규화
   │        └ authenticationConverter: claims → OAuth2IntrospectionAuthenticatedPrincipal
   │              (scope → SCOPE_* 권한)
   │        │
   │        ├ BadOpaqueTokenException     → InvalidBearerTokenException (401 invalid_token)
   │        └ OAuth2IntrospectionException → AuthenticationServiceException (인프라 오류)
   │
   ├─ 2) authenticationConverter.convert(token, principal)
   │        → BearerTokenAuthentication(principal, OAuth2AccessToken(BEARER, iat, exp), authorities)
   │           + BEARER factor 권한
   │
   ├─ 3) details 비어 있으면 bearer.getDetails() 이식
   └─ return BearerTokenAuthentication
```

JWT 경로와 골격이 똑같다는 데 주목하자. `JwtDecoder.decode` 자리에 `OpaqueTokenIntrospector.introspect` 가, `JwtAuthenticationConverter` 자리에 `OpaqueTokenAuthenticationConverter` 가 들어갔을 뿐이다. 예외 분류(`BadOpaqueTokenException`→401, `OAuth2IntrospectionException`→500)도 3장과 동일한 철학이다.

## 핵심 메서드

### `SpringOpaqueTokenIntrospector.introspect()`

introspection 요청·응답 전 과정을 묶는다.

```java
RequestEntity<?> requestEntity = this.requestEntityConverter.convert(token);
ResponseEntity<Map<String, Object>> responseEntity = makeRequest(requestEntity);
Map<String, Object> claims = adaptToNimbusResponse(responseEntity);   // active 검사
OAuth2TokenIntrospectionClaimAccessor accessor = convertClaimsSet(claims); // 타입 정규화
return this.authenticationConverter.convert(accessor);                // principal 생성
```

세부에서 눈여겨볼 점들.

- **요청은 항상 POST `token=<값>`** 형식이고, 클라이언트 인증은 기본적으로 HTTP Basic(`BasicAuthenticationInterceptor`)으로 붙는다. 빌더(`withIntrospectionUri(...).clientId(...).clientSecret(...)`)는 clientId/secret을 URL 인코딩해 안전하게 처리한다.
- **`active` 판정이 유효성의 전부다**(`adaptToNimbusResponse`). 리소스 서버는 `exp` 등을 스스로 검사하지 않고 "인가 서버가 active=true 라고 했는가"만 믿는다 — introspection의 본질은 검증을 인가 서버에 위임하는 것이기 때문이다. `active` 가 문자열이든 불리언이든 받아 정규화하고, false면 `BadOpaqueTokenException`.
- **`convertClaimsSet` 의 타입 정규화**: `aud` 가 단일 문자열이면 리스트로, `exp`/`iat`/`nbf` 의 epoch 초를 `Instant` 로, `scope` 문자열을 공백 split 한 리스트로 바꾼다. 단, `iss` 는 일부러 문자열 그대로 둔다(URI 정규화로 인한 모호성/부작용을 피하려는 의도적 선택, 코드 주석에 근거).
- **scope → 권한**: 기본 변환기(`defaultAuthenticationConverter`)가 정규화된 scope 리스트의 각 원소에 `SCOPE_` 를 붙여 권한으로 만든다. JWT 쪽 `JwtGrantedAuthoritiesConverter` 와 같은 규칙이다.

### `OpaqueTokenAuthenticationProvider.convert()` (기본 결과 변환)

introspection이 돌려준 principal로 최종 인증 객체를 만든다.

```java
OAuth2AccessToken accessToken = new OAuth2AccessToken(BEARER, introspectedToken, iat, exp);
Collection<GrantedAuthority> authorities = new HashSet<>(principal.getAuthorities());
authorities.add(FactorGrantedAuthority.fromAuthority(BEARER_AUTHORITY));
return new BearerTokenAuthentication(principal, accessToken, authorities);
```

결과 타입이 JWT 경로의 `JwtAuthenticationToken` 이 아니라 `BearerTokenAuthentication` 이라는 점만 다르다. 둘 다 `AbstractOAuth2TokenAuthenticationToken` 을 상속하므로, 인가 단계에서는 `getTokenAttributes()`/권한이라는 동일한 표면으로 다룰 수 있다.

## 설계 포인트 / 확장점

- **introspection 결과 매핑 교체**: `SpringOpaqueTokenIntrospector.setAuthenticationConverter(...)` 로 클레임→principal 변환을, `OpaqueTokenAuthenticationProvider.setAuthenticationConverter(...)` 로 principal→`Authentication` 변환을 각각 바꿀 수 있다. 권한을 토큰이 아닌 로컬 DB에서 끌어오는 식의 커스터마이징이 가능하다.
- **HTTP 클라이언트 선택**: `RestTemplate` 기반(`SpringOpaqueTokenIntrospector`)과 `RestClient` 기반(`RestClientOpaqueTokenIntrospector`) 두 구현이 있다. mTLS·프록시·타임아웃 같은 전송 정책은 주입하는 `RestOperations`/`RestClient` 쪽에서 정한다. 7.1의 빌더 `postProcessor(...)` 로 빌드 직후 추가 설정도 가능하다.
- **리액티브 대응**: WebFlux에서는 `ReactiveOpaqueTokenIntrospector` + `OpaqueTokenReactiveAuthenticationManager` 가 같은 일을 `Mono` 로 한다(`SpringReactiveOpaqueTokenIntrospector`).

## 정리

Opaque 토큰은 "토큰을 인가 서버 introspection 엔드포인트에 POST로 물어 `active` 와 속성을 받아오는" 방식이다. `OpaqueTokenIntrospector` 가 질의를, `OpaqueTokenAuthenticationProvider` 가 결과를 `BearerTokenAuthentication` 으로 빚는다. JWT 경로와 부품 이름만 다를 뿐 흐름·예외 분류·scope 권한 매핑은 동일하며, 차이는 "검증을 누가 하느냐"(로컬 서명 vs 인가 서버 위임)다.
