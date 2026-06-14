# 01. JWT 도메인 모델과 알고리즘

## 무엇을 / 왜

디코더든 인코더든 결국 다루는 대상은 **JWT 한 건**이다. 이 모듈은 그 토큰을 표현하기 위해 불변(immutable) 값 객체 `Jwt`와, 그 안의 표준 클레임을 타입 안전하게 꺼내는 `JwtClaimAccessor`를 둔다. 또 토큰의 헤더·클레임을 "만들 때" 쓰는 `JwsHeader`/`JwtClaimsSet`, 그리고 어떤 알고리즘으로 서명할지를 나타내는 `JwsAlgorithm` 체계가 있다.

이 장은 모듈 전체에서 흘러다니는 **데이터 구조**를 먼저 정리한다. 2~4장의 디코딩/검증/인코딩은 모두 이 타입들을 입력·출력으로 삼는다.

## 핵심 타입

```
                    ClaimAccessor (oauth2-core)
                          ▲
                          │ extends
                  JwtClaimAccessor ───── getIssuer()/getSubject()/
                          ▲               getAudience()/getExpiresAt()...
                          │ implements
   AbstractOAuth2Token ──▶ Jwt           ← 검증·인증의 결과물(읽기 전용)
   (tokenValue,             - headers: Map
    issuedAt, expiresAt)    - claims:  Map

   [인코딩 입력 측]
   JwsHeader  (alg, typ, kid, jku, x5c, x5t#S256 ...)   JoseHeader 상속
   JwtClaimsSet (iss, sub, aud, exp, nbf, iat, jti, 커스텀)

   [알고리즘]
   JwaAlgorithm (getName)
      └ JwsAlgorithm
           ├ SignatureAlgorithm  enum: RS256/384/512, ES256/384/512, PS256/384/512
           └ MacAlgorithm        enum: HS256/384/512
```

### Jwt — 검증된 토큰의 표현

`Jwt`는 `oauth2/oauth2-jose/src/main/java/org/springframework/security/oauth2/jwt/Jwt.java`에 있다. 핵심은 **불변성**이다. 생성자에서 헤더와 클레임을 각각 `LinkedHashMap`으로 복사한 뒤 `Collections.unmodifiableMap`으로 감싼다.

```java
public Jwt(String tokenValue, @Nullable Instant issuedAt, @Nullable Instant expiresAt,
        Map<String, Object> headers, Map<String, Object> claims) {
    super(tokenValue, issuedAt, expiresAt);
    Assert.notEmpty(headers, "headers cannot be empty");
    Assert.notEmpty(claims, "claims cannot be empty");
    this.headers = Collections.unmodifiableMap(new LinkedHashMap<>(headers));
    this.claims = Collections.unmodifiableMap(new LinkedHashMap<>(claims));
}
```

- `tokenValue`는 원본 compact 문자열(`header.payload.signature`)이고, `headers`/`claims`는 그것을 파싱한 결과다.
- `AbstractOAuth2Token`을 상속하므로 `getTokenValue()`, `getIssuedAt()`, `getExpiresAt()`를 그대로 갖는다. 즉 `Jwt`는 OAuth2 토큰 추상화의 한 구현이다.
- 헤더/클레임이 비면 안 된다(`notEmpty`). 빈 JWT는 의미가 없기 때문이다.

`Jwt.Builder`(정적 내부 클래스)는 `withTokenValue(...)`로 시작해 `claim()`, `header()`, 그리고 `issuer()`/`subject()`/`expiresAt()` 같은 편의 메서드를 제공한다. `build()`는 클레임 맵에서 `iat`/`exp`를 꺼내 `Instant`로 변환해 상위 생성자에 넘긴다. 디코더가 토큰을 만들 때 바로 이 빌더를 쓴다.

### JwtClaimAccessor — 표준 클레임 읽기

`Jwt`가 구현하는 `JwtClaimAccessor`는 RFC 7519의 등록된 클레임을 타입 변환해 꺼내는 default 메서드 모음이다. 모두 `ClaimAccessor`의 `getClaimAs*` 헬퍼에 위임한다.

```java
default @Nullable URL getIssuer()       { return getClaimAsURL(JwtClaimNames.ISS); }
default @Nullable String getSubject()   { return getClaimAsString(JwtClaimNames.SUB); }
default @Nullable List<String> getAudience() { return getClaimAsStringList(JwtClaimNames.AUD); }
default @Nullable Instant getExpiresAt(){ return getClaimAsInstant(JwtClaimNames.EXP); }
```

덕분에 검증기들은 `jwt.getExpiresAt()`, `jwt.getId()`처럼 의미 있는 이름으로 클레임에 접근한다. 클레임이 없으면 `null`을 돌려준다(예외 아님).

### 헤더와 클레임 셋 — 인코딩 입력

읽기 모델인 `Jwt`와 달리, **만들 때**는 `JwsHeader`(JOSE 헤더: `alg`, `typ`, `kid`, `jku`, `x5c`, `x5t#S256` 등)와 `JwtClaimsSet`(페이로드 클레임)을 따로 둔다. 이 둘이 `JwtEncoderParameters`로 묶여 4장의 `JwtEncoder.encode(...)`로 들어간다. `JoseHeaderNames`/`JwtClaimNames`는 각 필드의 문자열 키 상수를 모아 둔 곳이다.

## 알고리즘 체계

서명 알고리즘은 한 갈래의 인터페이스 계층으로 정리된다.

- `JwaAlgorithm`: `getName()` 하나만 가진 최상위. JWA(RFC 7518) 사양의 알고리즘 이름을 표현.
- `JwsAlgorithm`: JWS(서명/MAC)용 마커 인터페이스. `JwaAlgorithm`을 상속만 한다.
- `SignatureAlgorithm`(enum): 비대칭 서명. `RS256/384/512`(RSASSA-PKCS1), `ES256/384/512`(ECDSA), `PS256/384/512`(RSASSA-PSS).
- `MacAlgorithm`(enum): 대칭 MAC. `HS256/384/512`.

`SignatureAlgorithm`은 이름→enum 역변환 정적 메서드도 갖는다.

```java
public static @Nullable SignatureAlgorithm from(String name) {
    for (SignatureAlgorithm value : values()) {
        if (value.getName().equals(name)) {
            return value;
        }
    }
    return null;
}
```

이 `getName()` 값("RS256" 등)은 Nimbus의 `JWSAlgorithm.parse(name)`으로 그대로 넘어간다. 즉 Spring의 enum은 "타입 안전한 이름 보관함"이고, 실제 알고리즘 식별은 문자열을 매개로 Nimbus와 맞춘다. 이것이 모듈 전반의 패턴이다 — **Spring 타입으로 API를 만들고, 경계에서 Nimbus 타입으로 변환**한다.

## 설계 포인트

- **읽기 모델과 쓰기 모델의 분리**: 검증 결과는 불변 `Jwt`, 인코딩 입력은 빌더가 있는 `JwsHeader`/`JwtClaimsSet`. 같은 "JWT"라도 방향에 따라 다른 타입을 써서 불변성과 편의성을 동시에 얻는다.
- **oauth2-core와의 통합**: `Jwt extends AbstractOAuth2Token`, `JwtClaimAccessor extends ClaimAccessor`. 덕분에 OAuth2 토큰을 다루는 상위 코드가 JWT를 특수 케이스로 자연스럽게 흡수한다.
- **enum 기반 알고리즘 화이트리스트**: 임의 문자열이 아니라 enum으로 알고리즘을 받음으로써, 디코더/인코더 빌더가 받아들이는 알고리즘 집합을 컴파일 타임에 제한한다.

## 정리

`Jwt`는 헤더+클레임을 담은 불변 토큰 객체로, `AbstractOAuth2Token`을 상속하고 `JwtClaimAccessor`로 표준 클레임을 타입 안전하게 노출한다. 인코딩 쪽은 가변 빌더를 가진 `JwsHeader`/`JwtClaimsSet`을 따로 둔다. 알고리즘은 `JwaAlgorithm → JwsAlgorithm → SignatureAlgorithm/MacAlgorithm`의 계층과 문자열 이름을 통해 Nimbus와 연결된다. 이 데이터 토대 위에서 2장의 디코딩이 시작된다.
