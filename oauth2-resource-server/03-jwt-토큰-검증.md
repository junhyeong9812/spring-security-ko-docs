# 03. JWT 토큰 검증

## 무엇을 / 왜

가장 흔한 리소스 서버 구성은 **자체 완결형(self-contained) JWT 액세스 토큰**이다. 토큰 안에 발급자·만료·subject·scope가 서명과 함께 들어 있어, 리소스 서버는 인가 서버에 매번 묻지 않고 **서명만 검증하면** 토큰을 신뢰할 수 있다(stateless). 이 챕터는 그 검증을 수행하는 `JwtAuthenticationProvider` 와, 검증된 `Jwt` 를 `Authentication` 으로 빚는 `JwtAuthenticationConverter`, 그리고 멀티테넌트 처리를 다룬다.

## 핵심 타입

```
JwtAuthenticationProvider  (AuthenticationProvider)
   │ uses
   ├─ JwtDecoder                  ← oauth2-jose. 서명/exp/iss 실제 검증 엔진
   └─ Converter<Jwt, AbstractAuthenticationToken>
          └ JwtAuthenticationConverter         ← 기본 구현
                ├ jwtGrantedAuthoritiesConverter : Jwt → Collection<GrantedAuthority>
                │     └ JwtGrantedAuthoritiesConverter   (scope/scp → SCOPE_*)
                └ jwtPrincipalConverter          : Jwt → OAuth2AuthenticatedPrincipal

멀티테넌트
   JwtIssuerAuthenticationManagerResolver  (AuthenticationManagerResolver<HttpServletRequest>)
```

`JwtDecoder` 가 "이 토큰이 진짜인가"를, `JwtAuthenticationConverter` 가 "그 토큰이 누구이고 무슨 권한인가"를 책임진다. Provider는 이 둘을 잇는 얇은 접착제다.

## 동작 흐름 — JWT 인증

```
authenticate(BearerTokenAuthenticationToken bearer)
   │
   ├─ 1) jwtDecoder.decode(bearer.getToken())          [oauth2-jose]
   │        · JWS 서명 검증 (JWK Set 으로)
   │        · exp / nbf 시간 검증
   │        · iss / aud 등 클레임 검증
   │        ├ BadJwtException  → InvalidBearerTokenException  (401 invalid_token)
   │        └ JwtException     → AuthenticationServiceException (500, 인프라 오류)
   │        → Jwt (검증된 토큰)
   │
   ├─ 2) jwtAuthenticationConverter.convert(jwt)
   │        ├ jwtGrantedAuthoritiesConverter.convert(jwt)
   │        │     "scope":"read write" → [SCOPE_read, SCOPE_write]
   │        ├ FactorGrantedAuthority(BEARER) 권한 추가
   │        ├ jwtPrincipalConverter.convert(jwt) → principal (sub 클레임이 이름)
   │        └ new JwtAuthenticationToken(jwt, principal, authorities)
   │
   ├─ 3) details 비어 있으면 bearer.getDetails() 이식
   └─ return JwtAuthenticationToken  (isAuthenticated = true)
```

### `getJwt()` — 예외 분류의 의미

검증 실패를 두 부류로 나누는 것이 핵심이다.

```java
try {
    return this.jwtDecoder.decode(bearer.getToken());
}
catch (BadJwtException failed) {            // 토큰이 잘못됨 (서명 불일치, 만료 등)
    throw new InvalidBearerTokenException(...);   // → 401, WWW-Authenticate: invalid_token
}
catch (JwtException failed) {               // 디코더 자체 문제 (JWK 엔드포인트 장애 등)
    throw new AuthenticationServiceException(...); // → 500 계열, 재시도 대상
}
```

`BadJwtException` 은 "클라이언트 토큰이 틀렸다"(고치라고 401), `JwtException` 은 "우리 쪽 검증 인프라가 일시적으로 안 됐다"(서버 오류)로 의미가 다르다. 이 구분이 그대로 HTTP 응답 의미로 연결된다.

## 핵심 메서드

### `JwtAuthenticationConverter.convert(Jwt)`

검증된 `Jwt` 를 인증 객체로 바꾸는 곳. 본문이 짧고 분명하다.

```java
Collection<GrantedAuthority> authorities = new HashSet<>(jwtGrantedAuthoritiesConverter.convert(jwt));
authorities.add(FactorGrantedAuthority.fromAuthority(AUTHORITY));   // BEARER factor
OAuth2AuthenticatedPrincipal principal = jwtPrincipalConverter.convert(jwt);
authorities.addAll(principal.getAuthorities());
return new JwtAuthenticationToken(jwt, principal, authorities);
```

- 권한은 **scope 기반 권한** + **`BEARER` factor 권한**의 합집합이다. factor 권한은 "이 인증이 Bearer 토큰으로 이뤄졌다"는 사실을 인가 단계에서 활용할 수 있게 하는 표식이다.
- principal은 `Jwt` 를 그대로 감싼 내부 타입(`JwtAuthenticatedPrincipal`)이며, `getName()` 은 기본적으로 `sub` 클레임을 쓴다. `setPrincipalClaimName("user_name")` 처럼 이름으로 쓸 클레임을 바꿀 수 있다(7.1부터는 `setJwtPrincipalConverter` 로 principal 변환 전체를 교체 가능).

### `JwtGrantedAuthoritiesConverter` — scope → 권한 변환 규칙

```java
WELL_KNOWN_AUTHORITIES_CLAIM_NAMES = ["scope", "scp"]   // 이 순서로 먼저 발견된 것 사용
DEFAULT_AUTHORITY_PREFIX           = "SCOPE_"
DEFAULT_DELIMITER                  = " "
```

변환 규칙은 다음과 같다.

- `scope`(또는 `scp`) 클레임을 찾는다. 둘 다 없으면 권한 없음.
- 값이 `String` 이면 공백으로 split(`"read write"` → `read`, `write`), `Collection` 이면 그대로 순회.
- 각 원소 앞에 `SCOPE_` 를 붙여 `SimpleGrantedAuthority` 를 만든다 → `SCOPE_read`, `SCOPE_write`.

접두사·구분자·클레임 이름은 모두 세터로 바꿀 수 있어, 인가 서버가 `authorities` 나 `roles` 같은 다른 클레임에 권한을 싣는 경우에도 맞출 수 있다. 여러 클레임을 합치려면 `DelegatingJwtGrantedAuthoritiesConverter` 로 여러 변환기를 묶는다.

## 멀티테넌트 — `JwtIssuerAuthenticationManagerResolver`

하나의 리소스 서버가 **여러 인가 서버**의 토큰을 받아야 할 때 쓴다. 토큰의 `iss` 클레임을 보고 그에 맞는 `AuthenticationManager`(= 그 발급자용 `JwtDecoder` 를 가진 매니저)를 고른다.

```
요청 토큰의 iss = "https://issuerOne.example.org"
        │
   ResolvingAuthenticationManager.authenticate(token)
        │
   ├ JwtClaimIssuerConverter: 토큰 파싱해 iss 추출 (서명 검증 전, JWTParser)
   │    · iss 없음/파싱 실패 → InvalidBearerTokenException
   │
   ├ issuerAuthenticationManagerResolver.resolve(iss)
   │    · 신뢰 목록에 없으면 → null → "Invalid issuer" 401
   │    · 신뢰되면 → issuer별 JwtDecoder 로 JwtAuthenticationProvider 생성 (캐시)
   │
   └ 선택된 manager.authenticate(token)  → 일반 JWT 검증 진행
```

보안상 가장 중요한 점: **`iss` 는 누구나 위조할 수 있으므로, 반드시 "신뢰하는 발급자 목록"으로 걸러야 한다.** 그래서 생성 팩토리가 `fromTrustedIssuers(...)` 형태로, 신뢰 issuer 집합/Predicate를 강제로 받는다. 신뢰된 issuer는 `JwtDecoders.fromIssuerLocation(issuer)` 로 OIDC 메타데이터에서 JWK Set URI를 자동 발견해 디코더를 만들고, 이를 `ConcurrentHashMap` 에 캐싱한다. `iss` 추출 자체는 서명 검증 전에 하지만, 실제 신뢰는 "신뢰 목록 통과 + 그 발급자 키로 서명 검증 성공"의 이중 관문을 거친다.

## 설계 포인트 / 확장점

- **검증 엔진과 매핑의 분리**: 서명/시간/발급자 검증은 `JwtDecoder`(oauth2-jose)에, 클레임→권한/principal 매핑은 `JwtAuthenticationConverter` 에 있다. 검증 정책(허용 clock skew, 추가 `OAuth2TokenValidator`)은 디코더 쪽에서, 권한 매핑은 변환기 쪽에서 손본다.
- **`JwtBearerTokenAuthenticationConverter`**: `JwtAuthenticationToken` 대신 Opaque 경로와 동일한 `BearerTokenAuthentication` 결과 타입을 쓰고 싶을 때 사용하는 변환기. JWT/Opaque 결과 타입을 통일하고 싶은 경우의 선택지다.
- **리액티브 대응**: WebFlux에서는 `JwtReactiveAuthenticationManager` 가 같은 일을 `Mono` 파이프라인으로 한다(`ReactiveJwtDecoder` + `ReactiveJwtAuthenticationConverterAdapter`). 멀티테넌트도 `JwtIssuerReactiveAuthenticationManagerResolver` 로 대응된다(05장).

## 정리

JWT 검증은 "디코더가 서명을 확인 → 변환기가 scope를 `SCOPE_*` 권한으로, sub를 이름으로 매핑 → `JwtAuthenticationToken` 산출"의 3단이다. `BadJwtException`(401)과 `JwtException`(500)을 구분해 클라이언트 오류와 인프라 오류를 다르게 처리하고, 멀티테넌트는 `iss` 를 신뢰 목록으로 거른 뒤 발급자별 디코더로 분기한다. 다음 장은 토큰 안에 정보가 없는 Opaque 토큰을 다룬다.
