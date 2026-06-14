# 02. JWT 디코딩과 서명 검증

## 무엇을 / 왜

Resource Server가 받은 토큰 문자열을 신뢰하려면 세 가지를 통과시켜야 한다. (1) 형식이 올바른 JWT인가(파싱), (2) 신뢰하는 키로 서명되었는가(서명 검증), (3) 만료·발급자 등 내용이 유효한가(클레임 검증). `JwtDecoder`는 이 셋을 `decode(String)` 한 호출로 묶는다. 이 장은 (1)·(2)와 그 전제인 **키 조달(JWK 소스)**을, (3)은 3장에서 다룬다.

## 핵심 타입

```
        JwtDecoder (FunctionalInterface)
            │  Jwt decode(String token) throws JwtException
            ▼
        NimbusJwtDecoder  ──────────────┐ 위임
            ├ jwtProcessor: JWTProcessor<SecurityContext>   (Nimbus: 파싱+서명검증)
            ├ claimSetConverter: Converter<Map,Map>          (원시 클레임 → 타입화)
            └ jwtValidator: OAuth2TokenValidator<Jwt>         (클레임 검증 — 3장)

   생성 경로(정적 빌더 4종):
     withJwkSetUri(uri)      → JwkSetUriJwtDecoderBuilder   (원격 JWK Set + 캐시)
     withPublicKey(rsaKey)   → PublicKeyJwtDecoderBuilder   (단일 RSA 공개키)
     withSecretKey(secret)   → SecretKeyJwtDecoderBuilder   (HMAC 대칭키)
     withJwkSource(source)   → JwkSourceJwtDecoderBuilder   (임의 JWKSource)
     withIssuerLocation(iss) → (OIDC/AS 메타데이터에서 jwks_uri 유도)

   JwtDecoders.fromIssuerLocation(iss) : 위 빌더를 감싸 issuer 검증까지 끼워줌
```

`JwtDecoder` 인터페이스 자체는 한 줄이다.

```java
@FunctionalInterface
public interface JwtDecoder {
    Jwt decode(String token) throws JwtException;
}
```

표준 구현 `NimbusJwtDecoder`는 "raw Nimbus 설정을 받는 저수준 구현"이라고 스스로를 소개한다. 핵심 협력자는 Nimbus의 `JWTProcessor`(파싱·키선택·서명검증을 캡슐화)다.

## 동작 흐름 — decode(token)

`NimbusJwtDecoder.decode`는 짧지만 전체 파이프라인을 보여준다.

```java
@Override
public Jwt decode(String token) throws JwtException {
    JWT jwt = parse(token);                       // ① Nimbus 파싱
    if (jwt instanceof PlainJWT) {                // ② 서명 없는 토큰 거부
        throw new BadJwtException("Unsupported algorithm of " + jwt.getHeader().getAlgorithm());
    }
    Jwt createdJwt = createJwt(token, jwt);       // ③ 서명검증 + Jwt 조립
    return validateJwt(createdJwt);               // ④ 클레임 검증
}
```

```
 토큰 문자열 ─▶ ① JWTParser.parse
                    │  실패 → BadJwtException("Malformed token")
                    ▼
              ② PlainJWT 인가?
                    │  맞으면 → BadJwtException (alg=none 거부)
                    ▼
              ③ createJwt:
                    jwtProcessor.process(parsedJwt, null)   ◀── 서명 검증(핵심)
                       │     ├ JWS 헤더의 alg/kid로 키 선택(JWSKeySelector)
                       │     ├ JWKSource 에서 검증 키 조달
                       │     └ 서명 불일치/키없음 → JOSEException → JwtException
                       ▼
                    claimSetConverter.convert(claims)        ◀── 타입 변환
                       ▼
                    Jwt.withTokenValue(token).headers(...).claims(...).build()
                    ▼
              ④ validateJwt: jwtValidator.validate(jwt)      ◀── 3장
                    │  errors 있으면 → JwtValidationException
                    ▼
                  검증된 Jwt 반환
```

### ① 파싱과 ② alg=none 거부

`parse`는 `JWTParser.parse(token)`을 호출하고, `ParseException`이면 "Malformed token", 그 외 예외는 메시지를 담아 `BadJwtException`으로 바꾼다. 파싱 결과가 `PlainJWT`(서명 없는 토큰)면 즉시 거부한다 — `alg: none` 공격을 막는 지점이다.

### ③ 서명 검증과 Jwt 조립 (createJwt)

여기가 서명 검증의 실제 위치다.

```java
private Jwt createJwt(String token, JWT parsedJwt) {
    try {
        JWTClaimsSet jwtClaimsSet = this.jwtProcessor.process(parsedJwt, null); // 서명 검증
        Map<String, Object> headers = new LinkedHashMap<>(parsedJwt.getHeader().toJSONObject());
        Map<String, Object> claims = this.claimSetConverter.convert(jwtClaimsSet.getClaims());
        return Jwt.withTokenValue(token).headers(h -> h.putAll(headers))
                  .claims(c -> c.putAll(claims)).build();
    }
    catch (RemoteKeySourceException ex) { /* JWK Set 조회 실패 → JwtException */ }
    catch (JOSEException ex)            { /* 서명/처리 실패 → JwtException */ }
    catch (Exception ex)               { /* 페이로드 malformed → BadJwtException */ }
}
```

- `jwtProcessor.process(...)`가 Nimbus 내부에서 헤더의 `alg`/`kid`를 보고 `JWSKeySelector`로 키를 골라 서명을 검증한다. 실패하면 `JOSEException` 계열이 던져지고, Spring은 그것을 `JwtException`/`BadJwtException`으로 번역한다.
- 검증을 통과한 Nimbus `JWTClaimsSet`을 `claimSetConverter`로 통과시킨다. 기본값은 `MappedJwtClaimSetConverter.withDefaults(...)`로, `iss→URL`, `exp/iat→Instant`, `aud→List<String>` 같은 표준 변환을 적용한다(없는 클레임은 그대로 둠).
- **중요한 설계 결정**: 각 빌더의 `processor()`는 Nimbus의 클레임 검증을 일부러 비활성화한다 — `jwtProcessor.setJWTClaimsSetVerifier((claims, context) -> {})`. 주석대로 "Spring Security validates the claim set independent from Nimbus". 즉 Nimbus는 **서명만** 책임지고, 만료·발급자 같은 내용 검증은 4단계의 Spring `jwtValidator`가 전담한다. 검증 책임을 한 곳(3장의 검증기)으로 모으기 위함이다.

### ④ 클레임 검증

`validateJwt`는 `jwtValidator.validate(jwt)`를 호출하고 오류가 있으면 `JwtValidationException`을 던진다(상세는 3장).

## 키는 어디서 오는가 — JWK 소스

`decode`가 동작하려면 `jwtProcessor`에 키를 공급하는 `JWKSource`가 박혀 있어야 한다. 빌더별로 다르다.

### 원격 JWK Set URI

`withJwkSetUri(...)`의 `jwkSource()`는 Spring 전용 `SpringJWKSource`를 Nimbus의 `JWKSourceBuilder`로 감싼다.

```java
JWKSource<SecurityContext> jwkSource() {
    String jwkSetUri = this.jwkSetUri.apply(this.restOperations);
    return JWKSourceBuilder.create(new SpringJWKSource<>(this.restOperations, this.cache, jwkSetUri))
        .refreshAheadCache(false).rateLimited(false)
        .cache(this.cache instanceof NoOpCache)   // Spring Cache를 직접 쓰면 Nimbus 캐시 끔
        .build();
}
```

`SpringJWKSource.getJWKSet(...)`은 `ReentrantLock`으로 보호된 상태에서 Spring `Cache`를 통해 JWK Set을 가져온다. 캐시 미스 시 `fetchJwks()`가 `RestOperations`로 `jwkSetUri`에 GET 요청(Accept: `application/json`, `application/jwk-set+json`)을 보내 응답 본문을 `JWKSet.parse`한다. 키 회전(rotation)에 대비해, refresh가 필요하면 캐시를 무효화하고 다시 받아온다. 별도 `RestOperations`를 주입하지 않으면 Nimbus 기본 타임아웃이 설정된 `RestTemplateWithNimbusDefaultTimeouts`를 써서 락을 쥔 채 무한 대기하는 것을 막는다.

키 선택은 `jwsKeySelector(...)`가 만든다. 알고리즘을 지정하지 않으면 기본 `RS256`만 허용하고(`Set.of(JWSAlgorithm.RS256)`), `jwsAlgorithm(...)`/`discoverJwsAlgorithms()`로 넓힐 수 있다.

### 단일 키 (공개키 / 시크릿키)

`withPublicKey(rsaKey)`는 `SingleKeyJWSKeySelector`로 하나의 RSA 공개키만 쓰고, 알고리즘은 RSA 계열(RS256 기본)이어야 한다고 `Assert.state`로 강제한다. `withSecretKey(secret)`는 HMAC(HS256 기본)용 단일 시크릿키를 쓴다. 원격 호출이 없으므로 캐시·RestOperations 개념이 없다.

### typ 헤더 검증 정책

모든 빌더는 `validateType(boolean)`를 가진다. 기본은 `NO_TYPE_VERIFIER`(Nimbus가 `typ`을 검사하지 않음)이고, 대신 Spring의 `JwtTypeValidator`가 클레임 검증 단계에서 `typ`을 본다. `validateType(true)`로 켜면 Nimbus의 `DefaultJOSEObjectTypeVerifier`에 위임한다. `at+jwt` 같은 커스텀 타입을 Spring 검증기로 다루고 싶을 때 `false`로 두는 것이 권장 경로다.

## issuer 한 줄 부트스트랩 — JwtDecoders

가장 흔한 설정은 "issuer만 알고 나머지는 자동"이다. `JwtDecoders.fromIssuerLocation(issuer)`가 그 길을 만든다.

```java
public static <T extends JwtDecoder> T fromIssuerLocation(String issuer) {
    NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder.withIssuerLocation(issuer).build();
    jwtDecoder.setJwtValidator(JwtValidators.createDefaultWithIssuer(issuer));
    return (T) jwtDecoder;
}
```

`withIssuerLocation`은 OIDC Provider Configuration / OAuth2 Authorization Server Metadata 엔드포인트를 조회(`JwtDecoderProviderConfigurationUtils`)해 `jwks_uri`를 알아내고, 그 URI로 `JwkSetUriJwtDecoderBuilder`를 구성한다. 동시에 검증기를 `createDefaultWithIssuer(issuer)`로 세팅해 발급자 검증까지 자동으로 끼운다. `fromOidcIssuerLocation`은 OIDC 엔드포인트만, `fromIssuerLocation`은 세 가지 well-known 엔드포인트를 순차 시도한다.

## 핵심 메서드 요약

| 메서드 | 역할 |
|--------|------|
| `NimbusJwtDecoder.decode` | 파싱→서명검증→클레임검증 전체 파이프라인 |
| `createJwt` | `jwtProcessor.process`로 서명 검증 후 `Jwt` 조립, 예외 번역 |
| `JwkSetUriJwtDecoderBuilder.processor` | Nimbus `DefaultJWTProcessor` 구성(키선택자 + 클레임검증 비활성) |
| `SpringJWKSource.getJWKSet` | 캐시·락·HTTP로 JWK Set 조달, 키 회전 대응 |
| `JwtDecoders.fromIssuerLocation` | issuer → 메타데이터 → 디코더+검증기 자동 구성 |

## 설계 포인트 / 확장점

- **서명과 내용 검증의 책임 분리**: Nimbus=서명, Spring=클레임. 빌더가 일부러 Nimbus 클레임 검증기를 비운다. 덕분에 검증 정책을 3장의 조합형 검증기로 일원화한다.
- **빌더 4종 = 키 출처 4종**: JWK Set URI / 공개키 / 시크릿키 / 임의 `JWKSource`(7.0+). 같은 `NimbusJwtDecoder`를 만들되 키 조달 전략만 바꾼다.
- **확장 훅**: `jwtProcessorCustomizer(Consumer<ConfigurableJWTProcessor>)`로 Nimbus 프로세서를 직접 손볼 수 있고, `setJwtValidator(...)`/`setClaimSetConverter(...)`로 검증·변환 정책을 교체할 수 있다.
- **캐시·동시성**: `cache(Cache)`로 Spring `Cache`를 주입하면 JWK Set을 그 캐시에 저장하고 Nimbus 자체 캐시는 끈다. `ReentrantLock`으로 동시 갱신을 직렬화한다.

## 정리

`NimbusJwtDecoder.decode`는 파싱(`alg:none` 거부 포함) → Nimbus `JWTProcessor`로 서명 검증 → `claimSetConverter`로 타입 변환 → Spring 검증기로 클레임 검증의 4단계를 수행한다. 키는 JWK Set URI(캐시·락·HTTP), 단일 공개키/시크릿키, 임의 `JWKSource` 중 하나로 조달된다. 빌더가 Nimbus 클레임 검증을 비활성화함으로써 내용 검증은 전적으로 다음 장의 `OAuth2TokenValidator<Jwt>`가 맡는다.
