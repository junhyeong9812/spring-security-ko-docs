# 05. 리액티브 디코더와 DPoP

## 무엇을 / 왜

지금까지의 `JwtDecoder`는 서블릿(블로킹) 세계의 것이다. WebFlux 환경에서는 JWK Set을 가져오는 HTTP 호출이 논블로킹이어야 하므로, 결과를 `Mono<Jwt>`로 돌려주는 `ReactiveJwtDecoder`가 따로 있다. 또 토큰 종류에 따라 디코더 구성이 달라지는 경우(예: DPoP proof는 매 요청마다 서로 다른 키로 검증)에는 "컨텍스트를 받아 디코더를 만들어 주는" `JwtDecoderFactory`가 필요하다. 이 장은 그 두 갈래 — **리액티브 디코딩**과 **팩토리/DPoP** — 를 다룬다.

## 핵심 타입

```
   [리액티브 디코딩]
   ReactiveJwtDecoder            Mono<Jwt> decode(String token)
        └ NimbusReactiveJwtDecoder
              jwtProcessor: Converter<JWT, Mono<JWTClaimsSet>>  (논블로킹 서명검증)
              jwtValidator / claimSetConverter  (서블릿판과 동일 개념)
        └ SupplierReactiveJwtDecoder  지연 초기화 래퍼
   ReactiveJwtDecoders            issuer → 리액티브 디코더
   ReactiveRemoteJWKSource / ReactiveJWKSource(Adapter)  논블로킹 JWK 조달

   [팩토리]
   JwtDecoderFactory<C>           JwtDecoder createDecoder(C context)
   ReactiveJwtDecoderFactory<C>   리액티브판

   [DPoP — RFC 9449]
   DPoPProofContext               proof + method + targetUri + accessToken
   DPoPProofJwtDecoderFactory     dpop+jwt 전용 디코더 생성
        implements JwtDecoderFactory<DPoPProofContext>
```

## 리액티브 디코더 — NimbusReactiveJwtDecoder

서블릿판과 책임은 같지만 `jwtProcessor`의 타입이 다르다.

```java
public final class NimbusReactiveJwtDecoder implements ReactiveJwtDecoder {
    private final Converter<JWT, Mono<JWTClaimsSet>> jwtProcessor;
    private OAuth2TokenValidator<Jwt> jwtValidator = JwtValidators.createDefault();
    private Converter<Map<String, Object>, Map<String, Object>> claimSetConverter =
        MappedJwtClaimSetConverter.withDefaults(Collections.emptyMap());
    ...
}
```

서블릿판이 Nimbus의 `JWTProcessor`(동기)를 쓴 자리에, 리액티브판은 `Converter<JWT, Mono<JWTClaimsSet>>`를 쓴다. 즉 서명 검증(특히 원격 JWK Set 조회)이 `Mono` 체인 안에서 논블로킹으로 일어난다. 검증기(`jwtValidator`)와 클레임 변환기(`claimSetConverter`)의 기본값은 서블릿판과 **완전히 동일**하다 — `JwtValidators.createDefault()`, `MappedJwtClaimSetConverter.withDefaults(...)`. 검증 정책(3장)은 동기/비동기 양쪽이 그대로 공유한다.

빌더 진입점도 대칭이다: `withJwkSetUri(...)`, `withPublicKey(...)`, `withSecretKey(...)`, `withJwkSource(...)`. JWK Set은 `WebClient`(spring-web reactive)로 가져오며, `ReactiveRemoteJWKSource`가 논블로킹 조달을 담당한다. `ReactiveJWKSourceAdapter`는 동기 `JWKSource`를 리액티브 인터페이스에 맞춰 주는 어댑터다.

```
 서블릿                          리액티브
 ─────────────────────────────  ─────────────────────────────
 JwtDecoder.decode → Jwt        ReactiveJwtDecoder.decode → Mono<Jwt>
 JWTProcessor(동기)             Converter<JWT, Mono<JWTClaimsSet>>
 RestOperations + Cache         WebClient + ReactiveRemoteJWKSource
 JwtValidators.createDefault()  ← 동일 검증기 공유 →  JwtValidators.createDefault()
```

`SupplierReactiveJwtDecoder`(와 서블릿판 `SupplierJwtDecoder`)는 디코더 생성 비용(특히 issuer 메타데이터·JWK Set 초기 조회)을 **첫 `decode` 호출까지 미루는** 지연 래퍼다. 애플리케이션 기동 시 인가 서버가 아직 안 떠 있어도 부팅이 실패하지 않게 해 준다.

## 팩토리 — JwtDecoderFactory

지금까지는 디코더를 한 번 만들어 재사용했다. 그러나 "토큰/요청마다 검증 파라미터가 달라지는" 경우가 있다. 그럴 때 디코더를 **컨텍스트로부터 생성**하는 것이 `JwtDecoderFactory<C>`다.

```java
public interface JwtDecoderFactory<C> {
    JwtDecoder createDecoder(C context);
}
```

대표 사용처가 OIDC ID 토큰 검증(상위 `oauth2-client` 모듈, `ClientRegistration`을 컨텍스트로 받음)과, 아래의 DPoP다.

## DPoP proof 검증 — RFC 9449

DPoP(Demonstrating Proof of Possession)는 클라이언트가 매 요청마다 **자신의 키로 서명한 짧은 proof JWT**(`typ: dpop+jwt`)를 함께 보내, 토큰이 그 키에 바인딩되었음을 증명하는 방식이다. 검증 키가 토큰 발급자의 것이 아니라 **proof 헤더 안의 `jwk`** 라는 점이 특이하다. `DPoPProofJwtDecoderFactory`가 이를 처리한다.

```java
public JwtDecoder createDecoder(DPoPProofContext dPoPProofContext) {
    NimbusJwtDecoder jwtDecoder = buildDecoder();
    jwtDecoder.setJwtValidator(this.jwtValidatorFactory.apply(dPoPProofContext));
    return jwtDecoder;
}
```

컨텍스트(`DPoPProofContext`: proof 문자열, HTTP method, target URI, 선택적 access token)마다 새 디코더를 만들고, 그 컨텍스트에 맞춘 검증기를 끼운다.

### 헤더의 jwk로 서명 검증

`buildDecoder`의 키 선택자는 JWK Set이 아니라 **proof JWS 헤더의 `jwk`** 에서 공개키를 꺼낸다.

```java
private static JWSKeySelector<SecurityContext> jwsKeySelector() {
    return (header, context) -> {
        JWSAlgorithm algorithm = header.getAlgorithm();
        if (!RSA.contains(algorithm) && !EC.contains(algorithm)) {
            throw new BadJwtException("Unsupported alg parameter in JWS Header: " + algorithm.getName());
        }
        JWK jwk = header.getJWK();
        if (jwk == null)        throw new BadJwtException("Missing jwk parameter in JWS Header.");
        if (jwk.isPrivate())    throw new BadJwtException("Invalid jwk parameter in JWS Header."); // 개인키 금지
        // RSA/EC 공개키로 변환해 검증 키로 사용
        ...
    };
}
```

`typ`은 `dpop+jwt`만 허용(`DPOP_TYPE_VERIFIER`)하고, 헤더 `jwk`가 없거나 개인키를 담고 있으면 거부한다.

### proof 전용 클레임 검증과 재전송 방지

기본 검증기 팩토리는 DPoP proof 고유의 클레임을 검사한다.

```java
private static Function<DPoPProofContext, OAuth2TokenValidator<Jwt>> defaultJwtValidatorFactory() {
    return (context) -> new DelegatingOAuth2TokenValidator<>(
        new JwtClaimValidator<>("htm", context.getMethod()::equals),     // HTTP 메서드 일치
        new JwtClaimValidator<>("htu", context.getTargetUri()::equals),  // 대상 URI 일치
        new JtiClaimValidator(),                                          // jti 단일 사용
        new JwtIssuedAtValidator(true));                                  // iat 존재/유효
}
```

여기서 3장의 `JwtClaimValidator`가 재사용된다 — `htm`/`htu` 클레임이 실제 요청의 메서드·URI와 같은지 검사한다. `JtiClaimValidator`는 **재전송 공격 방지**를 위해 `jti`를 한 번만 허용한다.

```java
String jtiHash = computeSHA256(jti);
if (JTI_CACHE.putIfAbsent(jtiHash, expiry.toEpochMilli()) != null) {
    // 이미 사용된 jti → 거부 (replay)
    return OAuth2TokenValidatorResult.failure(createOAuth2Error("jti claim is invalid."));
}
```

`jti`의 SHA-256 해시를 크기 제한(`MAX_SIZE=1000`)·만료(1시간) 기반 `LinkedHashMap` 캐시에 넣어, 같은 proof가 두 번 쓰이면 두 번째를 거부한다.

```
 요청 + DPoP proof(JWT)
        │
        ▼
 DPoPProofJwtDecoderFactory.createDecoder(DPoPProofContext)
        │  proof별 NimbusJwtDecoder 생성
        ▼
 decoder.decode(proof)
        ├ typ == dpop+jwt ?                 (아니면 거부)
        ├ 헤더 jwk(공개키)로 서명 검증       (jwk 없음/개인키 → 거부)
        └ 검증기:
             htm == 실제 HTTP 메서드?
             htu == 실제 대상 URI?
             jti 처음 보는가?  (replay 방지)
             iat 유효한가?
        ▼
     통과 → 신뢰 가능한 DPoP proof
```

## 핵심 메서드 요약

| 메서드 | 역할 |
|--------|------|
| `NimbusReactiveJwtDecoder.decode` | 논블로킹 서명검증 → `Mono<Jwt>`, 검증기/변환기는 동기판과 공유 |
| `Supplier(Reactive)JwtDecoder` | 첫 호출까지 디코더 초기화를 지연(부팅 내성) |
| `DPoPProofJwtDecoderFactory.createDecoder` | 컨텍스트별 `dpop+jwt` 디코더 + 맞춤 검증기 생성 |
| `jwsKeySelector`(DPoP) | JWK Set이 아닌 헤더의 `jwk` 공개키로 서명 검증 |
| `JtiClaimValidator.validate` | `jti` 해시 캐시로 proof 재전송 차단 |

## 설계 포인트 / 확장점

- **동기/비동기 대칭**: 같은 빌더 API(`withJwkSetUri`/`withPublicKey`...)와 같은 검증기·변환기를 양쪽이 공유한다. 차이는 HTTP 클라이언트(`RestOperations` vs `WebClient`)와 반환 타입(`Jwt` vs `Mono<Jwt>`)뿐이다.
- **팩토리 패턴으로 컨텍스트 의존 검증 흡수**: 토큰/요청마다 키·검증이 달라지는 경우(OIDC ID 토큰, DPoP)를 `JwtDecoderFactory<C>`가 일반화한다.
- **부품 재사용**: DPoP 검증도 결국 3장의 `JwtClaimValidator`·`DelegatingOAuth2TokenValidator`로 조립된다. 새 토큰 프로파일이 생겨도 같은 검증기 체계를 재사용한다.
- **지연 초기화**: `Supplier*JwtDecoder`는 외부 인가 서버 가용성에 부팅이 결합되지 않게 하는 운영상의 안전장치다.

## 정리

리액티브 세계에서는 `ReactiveJwtDecoder`/`NimbusReactiveJwtDecoder`가 서명 검증을 `Mono` 체인으로 논블로킹 수행하되, 검증 정책과 클레임 변환은 서블릿판과 동일한 부품을 공유한다. 토큰·요청별로 디코더 구성이 달라지는 경우는 `JwtDecoderFactory<C>`가 추상화하며, 그 대표 구현인 `DPoPProofJwtDecoderFactory`는 RFC 9449 proof를 헤더의 `jwk`로 검증하고 `htm`/`htu`/`jti`/`iat` 클레임 검사(특히 `jti` 단일 사용으로 재전송 방지)를 수행한다. 이로써 oauth2-jose는 동기·비동기·특수 토큰까지 일관된 디코딩/검증 토대를 제공한다.
