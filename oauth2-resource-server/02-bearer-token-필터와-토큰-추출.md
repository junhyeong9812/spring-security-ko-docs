# 02. Bearer Token 필터와 토큰 추출

## 무엇을 / 왜

검증 이전에 먼저 할 일은 "요청 어디에 토큰이 있는가"를 정하고 꺼내는 것이다. RFC 6750은 Bearer Token을 보내는 자리를 세 곳으로 정의한다 — `Authorization` 헤더, form-encoded 바디 파라미터, URI 쿼리 파라미터. 이 챕터는 그 추출 책임을 맡은 `BearerTokenResolver`/`BearerTokenAuthenticationConverter` 와, 추출 결과를 인증으로 넘기는 `BearerTokenAuthenticationFilter` 를 다룬다.

## 핵심 타입

```
BearerTokenAuthenticationFilter (OncePerRequestFilter)
        │ has-a
        ▼
AuthenticationConverter  (web 모듈의 표준 인터페이스)
        │
        └ BearerTokenAuthenticationConverter   ← 기본 구현 (since 7.0)
              │ has-a
              ▼
          BearerTokenResolver  (이 모듈의 SPI)
              ├ DefaultBearerTokenResolver   ← RFC 6750 3곳에서 추출
              └ HeaderBearerTokenResolver    ← 지정 헤더에서만 추출

      DPoPAuthenticationConverter   ← DPoP 경로용 (05장)
```

`AuthenticationConverter.convert(request)` 의 계약은 단순하다. **토큰이 있으면 미인증 `Authentication` 을, 없으면 `null` 을, 잘못됐으면 예외를** 돌려준다. 이 세 갈래가 필터의 분기와 정확히 대응한다.

## 동작 흐름 — 토큰 추출

### 1) 필터 → Converter → Resolver

```
doFilterInternal(request, ...)
   │
   ├─ authenticationConverter.convert(request)
   │      │
   │      ├ bearerTokenResolver.resolve(request)  →  "eyJhbGci..."  (String)
   │      │
   │      ├ token 있음 → new BearerTokenAuthenticationToken(token)
   │      │              + setDetails(authenticationDetailsSource.buildDetails)
   │      │              return (미인증 Authentication)
   │      │
   │      └ token 없음 → return null
   │
   ├─ 결과 null  → filterChain.doFilter()  (토큰 없는 요청은 그대로 통과)
   ├─ 예외       → entryPoint.commence()   (형식 오류 → 401)
   └─ 결과 있음  → authenticationManager.authenticate(...)  (다음 단계, 03·04장)
```

`BearerTokenAuthenticationConverter.convert()` 의 본문은 사실상 다음 한 토막이다.

```java
String token = this.bearerTokenResolver.resolve(request);
if (StringUtils.hasText(token)) {
    BearerTokenAuthenticationToken authenticationToken = new BearerTokenAuthenticationToken(token);
    authenticationToken.setDetails(this.authenticationDetailsSource.buildDetails(request));
    return authenticationToken;
}
return null;
```

`Converter` 는 "추출 → 토큰 객체 포장 → details 부착"만 하고 검증은 전혀 하지 않는다. 검증은 오로지 `AuthenticationProvider` 의 몫이라는 책임 분리가 여기서도 지켜진다.

### 2) `DefaultBearerTokenResolver` — RFC 6750 세 자리

```
resolve(request)
   │
   ├ resolveFromAuthorizationHeader   "Authorization: Bearer <token>"
   ├ resolveAccessTokenFromQueryString ?access_token=...  (allowUriQueryParameter && GET)
   └ resolveAccessTokenFromBody        body access_token=... (allowFormEncoded && form && !GET)
        │
        ▼
   resolveToken(...)  ── 여러 자리에서 동시에 발견되면
                          invalid_request "Found multiple bearer tokens"
```

세부 규칙이 보안상 중요하다.

- **헤더 매칭은 정규식**으로 한다: `^Bearer (?<token>[a-zA-Z0-9-._~+/]+=*)$` (대소문자 무시). `Bearer` 로 시작하지만 이 패턴에 안 맞으면 `invalid_token "Bearer token is malformed"` 예외를 던진다. 빈 토큰 문자열도 `invalid_request` 로 거른다.
- **쿼리/바디 추출은 기본 비활성**(`allowUriQueryParameter=false`, `allowFormEncodedBodyParameter=false`). RFC 자체가 URI 쿼리 전송을 "완전성을 위해서만 포함했다"며 권장하지 않기 때문이다. 명시적으로 켜야 동작한다.
- **여러 자리에 동시에 토큰이 있으면 거부**한다. 헤더와 쿼리에 둘 다 토큰이 있으면 `invalid_request`. 토큰 출처를 하나로 강제해 혼선/우회를 막는다.
- 헤더 이름은 `setBearerTokenHeaderName` 으로 바꿀 수 있다(예: `Proxy-Authorization`).

`HeaderBearerTokenResolver` 는 더 단순히 "지정한 헤더 값 자체를 토큰으로" 취급하는 변형으로, 게이트웨이가 검증 후 별도 헤더로 토큰을 실어 보내는 구성 등에 쓴다.

## 핵심 메서드

### `BearerTokenAuthenticationFilter` 의 성공 경로 후처리

추출·검증이 끝난 뒤 필터는 두 가지 후처리를 한다. 둘 다 코드 근거가 분명하다.

- **DPoP 다운그레이드 차단**: 인증 결과가 `cnf.jkt`(JWK thumbprint) 클레임을 담고 있으면 — 즉 DPoP에 바인딩된 토큰인데 평범한 `Bearer` 로 들어왔으면 — `invalid_token` 으로 거부한다(`isDPoPBoundAccessToken`). DPoP 바인딩 토큰을 탈취해 Bearer처럼 재사용하는 것을 막는 방어다.
- **기존 권한 병합(7.0)**: 이미 인증된 컨텍스트가 있고 결과 토큰이 `toBuilder()` 를 제공하면, 기존 `Authentication` 의 권한 중 새 결과에 없는 것을 합쳐 넣는다. 다단계 인증(앞 필터가 부여한 factor 권한)을 토큰 인증이 덮어쓰지 않게 하는 처리다.

### details 의 전파

`Converter` 가 붙인 `details`(`WebAuthenticationDetails` — 원격 IP, 세션 ID 등)는 그대로 버려지지 않는다. `JwtAuthenticationProvider`/`OpaqueTokenAuthenticationProvider` 는 검증 결과 토큰의 `getDetails()` 가 비어 있으면 입력 토큰의 details를 옮겨 담는다. 그래서 최종 `Authentication` 에도 요청 메타데이터가 남는다.

## 설계 포인트 / 확장점

- **추출 지점 교체**: 표준이 아닌 곳(쿠키 등)에서 토큰을 읽으려면 `BearerTokenResolver` 를 직접 구현해 `BearerTokenAuthenticationConverter.setBearerTokenResolver()` 로 꽂는다.
- **7.0의 책임 이동**: 과거 필터의 `setBearerTokenResolver`/`setAuthenticationDetailsSource` 는 `@Deprecated` 가 됐다. 이제 생성자에 `AuthenticationConverter` 를 직접 넘기고, 토큰 추출/details 설정은 그 Converter 안에서 한다. 필터는 "추출 방식"을 모르게 더 얇아졌다.
- **`Converter` 추상 공유**: `BearerTokenAuthenticationConverter` 는 web 모듈의 일반 `AuthenticationConverter` 를 구현한다. 덕분에 같은 필터가 `DPoPAuthenticationConverter` 같은 다른 변환기도 그대로 받아들인다.

## 정리

토큰 추출은 "RFC 6750이 정한 세 자리 중 활성화된 곳에서, 정확히 하나의 토큰만, 형식 검사를 거쳐 꺼내는" 작업이다. `BearerTokenResolver` 가 추출을, `BearerTokenAuthenticationConverter` 가 미인증 토큰 포장을, 필터가 흐름 지휘를 맡는다. 추출 결과 `BearerTokenAuthenticationToken` 은 검증되지 않은 토큰일 뿐이며, 다음 두 장에서 이 토큰이 어떻게 진짜로 판명되는지를 본다.
