# 04. JWT 인코딩과 서명

## 무엇을 / 왜

Authorization Server나 토큰을 발급하는 쪽은 클레임을 담아 **서명된 JWT 문자열**을 만들어야 한다. `JwtEncoder`가 그 계약이고, 표준 구현 `NimbusJwtEncoder`가 JWK로 키를 골라 Nimbus로 서명한다. 디코딩의 역방향이지만, "여러 키 중 어느 것으로 서명할지 선택"하는 단계가 추가된다.

## 핵심 타입

```
   JwtEncoder (FunctionalInterface)
        Jwt encode(JwtEncoderParameters parameters) throws JwtEncodingException
            │   JwtEncoderParameters = JwsHeader(옵션) + JwtClaimsSet
            ▼
   NimbusJwtEncoder
        ├ jwkSource: JWKSource<SecurityContext>      서명 키 후보 공급
        ├ defaultJwsHeader: JwsHeader                헤더 미지정 시 사용
        ├ jwkSelector: Converter<List<JWK>, JWK>     후보 다수일 때 선택 전략
        └ jwsSigners: Map<JWK, JWSSigner>            서명기 캐시

   생성 경로(7.0+ 정적 빌더):
        withKeyPair(rsaPub, rsaPriv) → RsaKeyPairJwtEncoderBuilder
        withKeyPair(ecPub, ecPriv)   → EcKeyPairJwtEncoderBuilder
        withSecretKey(secretKey)     → SecretKeyJwtEncoderBuilder
        new NimbusJwtEncoder(JWKSource)  ← 직접 JWKSource 주입
```

`JwtEncoder`도 한 줄짜리 함수형 인터페이스다.

```java
@FunctionalInterface
public interface JwtEncoder {
    Jwt encode(JwtEncoderParameters parameters) throws JwtEncodingException;
}
```

입력 `JwtEncoderParameters`는 `JwsHeader`(생략 가능)와 `JwtClaimsSet`을 묶는다. `from(claims)`만 주면 인코더의 기본 헤더가 쓰이고, `from(header, claims)`로 헤더를 명시할 수도 있다.

## 동작 흐름 — encode(parameters)

```java
@Override
public Jwt encode(JwtEncoderParameters parameters) throws JwtEncodingException {
    JwsHeader headers = parameters.getJwsHeader();
    if (headers == null) {
        headers = this.defaultJwsHeader;             // ① 헤더 기본값
    }
    JwtClaimsSet claims = parameters.getClaims();
    JWK jwk = selectJwk(headers);                    // ② 서명 키 선택
    headers = addKeyIdentifierHeadersIfNecessary(headers, jwk); // ③ kid/x5t 보강
    String jws = serialize(headers, claims, jwk);    // ④ 서명·직렬화
    return new Jwt(jws, claims.getIssuedAt(), claims.getExpiresAt(),
                   headers.getHeaders(), claims.getClaims());   // ⑤ Jwt 반환
}
```

```
 JwtEncoderParameters
        │
        ▼
 ① 헤더 없으면 defaultJwsHeader (기본 alg = RS256)
        ▼
 ② selectJwk(headers):
        헤더의 alg/kid/x5t 로 JWKMatcher 생성
        jwkSource.get(selector) → 후보 List<JWK>
          ├ 0개  → JwtEncodingException("Failed to select a JWK signing key")
          ├ 1개  → 그 키 사용
          └ N개  → jwkSelector.convert(jwks)  (기본은 예외!)
        ▼
 ③ JWK에 kid/x5t#S256 있으면 헤더에 자동 주입
        ▼
 ④ serialize:
        JwsHeader  → Nimbus JWSHeader
        JwtClaimsSet → Nimbus JWTClaimsSet
        JWSSigner = jwsSigners.computeIfAbsent(jwk, createSigner)  ← 캐시
        SignedJWT(header, claims).sign(signer).serialize()
        ▼
 ⑤ 원본 문자열 + 클레임/헤더로 Jwt 조립해 반환
```

### ② 서명 키 선택 — selectJwk

인코딩 고유의 단계다. 헤더의 알고리즘을 보고 매칭 조건(`JWKMatcher`)을 만든 뒤 `jwkSource`에서 후보를 가져온다.

```java
private JWK selectJwk(JwsHeader headers) {
    JWKSelector jwkSelector = new JWKSelector(createJwkMatcher(headers));
    List<JWK> jwks = this.jwkSource.get(jwkSelector, null);
    if (jwks.isEmpty()) {
        throw new JwtEncodingException(... "Failed to select a JWK signing key");
    }
    if (jwks.size() == 1) {
        return jwks.get(0);
    }
    return this.jwkSelector.convert(jwks);   // 다수면 사용자 전략에 위임
}
```

`createJwkMatcher`는 알고리즘 패밀리에 따라 다른 매처를 만든다.
- RSA/EC 계열: `keyType` + `kid` + `keyUse=SIGNATURE` + `algorithm` + `x5t#S256` 조건.
- HMAC 계열: `keyType` + `kid` + `privateOnly(true)` + `algorithm` 조건(대칭키는 비공개여야).

후보가 둘 이상이면 기본 `jwkSelector`는 **예외를 던진다**. 키가 여러 개인 환경에서는 `setJwkSelector(List::getFirst)`처럼 선택 전략을 명시해야 한다. 모호한 키 선택을 조용히 넘기지 않고 명시를 강제하는 설계다.

### ③ 키 식별 헤더 자동 보강

`addKeyIdentifierHeadersIfNecessary`는 헤더에 `kid`/`x5t#S256`이 비어 있고 선택된 JWK가 그 값을 가지고 있으면 헤더에 채워 넣는다. 덕분에 발급된 토큰에 검증 측이 키를 찾는 데 쓸 `kid`가 자동으로 들어간다.

### ④ 서명과 직렬화 — serialize

`JwsHeader`→Nimbus `JWSHeader`, `JwtClaimsSet`→Nimbus `JWTClaimsSet`으로 변환한 뒤 서명한다. `JWSSigner`는 JWK별로 캐시한다.

```java
JWSSigner jwsSigner = this.jwsSigners.computeIfAbsent(jwk, NimbusJwtEncoder::createSigner);
SignedJWT signedJwt = new SignedJWT(jwsHeader, jwtClaimsSet);
signedJwt.sign(jwsSigner);
return signedJwt.serialize();
```

`createSigner`는 Nimbus `DefaultJWSSignerFactory`로 키 타입에 맞는 서명기를 만든다(RSA/EC/HMAC 자동 판별). `computeIfAbsent`로 `ConcurrentHashMap`에 캐시해 매 호출마다 서명기를 새로 만들지 않는다. 클레임 변환(`convert(JwtClaimsSet)`)은 표준 클레임은 Nimbus 전용 세터로, 나머지는 커스텀 클레임으로 넣는다.

## 키쌍 빌더 (7.0+)

`JWKSource`를 직접 구성하기 번거로우므로, 7.0부터 키 객체에서 바로 인코더를 만드는 빌더가 추가됐다.

```java
NimbusJwtEncoder encoder = NimbusJwtEncoder.withKeyPair(rsaPublicKey, rsaPrivateKey).build();
```

내부적으로 `private NimbusJwtEncoder(JWK jwk)` 생성자가 단일 JWK를 `ImmutableJWKSet`으로 감싸 `jwkSource`로 쓰고, JWK의 알고리즘으로부터 기본 헤더(`alg`, `kid`, `x5c`/`x5t` 등)를 구성한다. 키 빌드는 package-private 헬퍼 `JWKS`가 담당한다.

```java
static OctetSequenceKey.Builder signing(SecretKey key) throws JOSEException {
    return new OctetSequenceKey.Builder(key)
        .keyOperations(Set.of(KeyOperation.SIGN)).keyUse(KeyUse.SIGNATURE)
        .algorithm(JWSAlgorithm.HS256).keyIDFromThumbprint() ...;
}
```

`JWKS`는 RSA/EC/대칭키 각각에 대해 `use=SIG`, `key_ops=[sign]`, 적절한 기본 알고리즘, 그리고 **thumbprint 기반 `kid` 자동 생성**(`keyIDFromThumbprint`)을 설정한다. `SecretKeyJwtEncoderBuilder`는 키 길이로 지원 가능한 HMAC 알고리즘 집합을 계산해, 너무 짧은 키면 빌드 단계에서 거부한다.

## 핵심 메서드 요약

| 메서드 | 역할 |
|--------|------|
| `NimbusJwtEncoder.encode` | 헤더 기본값→키선택→헤더보강→서명→`Jwt` 반환 |
| `selectJwk` | 알고리즘 기반 `JWKMatcher`로 후보 조회, 0/1/N 분기 |
| `serialize` | Nimbus 타입 변환 + 서명기 캐시 + `SignedJWT.sign` |
| `addKeyIdentifierHeadersIfNecessary` | JWK의 `kid`/`x5t#S256`을 헤더에 자동 주입 |
| `JWKS.signing*` | 키 → 서명용 JWK 빌더(use/key_ops/kid) 구성 |

## 설계 포인트 / 확장점

- **키 선택의 명시성**: 후보가 여럿이면 기본 동작은 예외. `setJwkSelector(...)`로 전략을 강제 주입해야 한다 — 우연한 서명 키 선택을 막는다.
- **서명기 캐시**: `jwsSigners` 맵으로 JWK당 `JWSSigner`를 재사용해 반복 서명 비용을 낮춘다.
- **타입 안전한 입출력**: 입력은 `JwtClaimsSet`/`JwsHeader`, 출력은 `Jwt`. Nimbus 타입은 `serialize` 내부에만 머문다(2장 디코더와 대칭).
- **키 출처 다양화**: `JWKSource` 직접 주입(키 회전·다중 키 환경) vs 키쌍/시크릿키 빌더(단일 키 간편 구성).

## 정리

`NimbusJwtEncoder.encode`는 헤더 기본값을 채우고, 알고리즘 기반 `JWKMatcher`로 서명 키를 고른 뒤(다수면 명시적 selector 필요), JWK의 식별자를 헤더에 보강하고, Nimbus `SignedJWT`로 서명·직렬화해 `Jwt`를 돌려준다. 7.0의 `withKeyPair`/`withSecretKey` 빌더와 `JWKS` 헬퍼는 단일 키를 적절한 메타데이터를 가진 JWK로 감싸 인코더 구성을 단순화한다. 이렇게 만든 토큰은 다른 서비스의 `JwtDecoder`(2·3장)가 검증한다.
