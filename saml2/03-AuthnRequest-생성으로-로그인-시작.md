# 03. AuthnRequest 생성으로 로그인 시작

## 무엇을 / 왜

SAML 로그인의 첫 왕복은 SP가 IdP에게 "이 사용자를 인증해 달라"고 보내는 `<AuthnRequest>` 다. 이 장은 그 요청을 **언제(어떤 URL에서) 만들고, 어떻게 XML로 빚어 서명하고, 어떤 바인딩으로 브라우저를 IdP에 보내는가**를 다룬다. 담당자는 필터 `Saml2WebSsoAuthenticationRequestFilter` 와 전략 `Saml2AuthenticationRequestResolver`(구현 `BaseOpenSamlAuthenticationRequestResolver`)다.

## 핵심 타입

```
요청 ──▶ Saml2WebSsoAuthenticationRequestFilter (OncePerRequestFilter)
            │  resolve()
            ▼
         Saml2AuthenticationRequestResolver           (전략 인터페이스)
            └ BaseOpenSamlAuthenticationRequestResolver (OpenSAML로 AuthnRequest 빌드)
                 │ 반환
                 ▼
         AbstractSaml2AuthenticationRequest
            ├ Saml2RedirectAuthenticationRequest  (HTTP-Redirect: samlRequest+relayState+sigAlg+signature)
            └ Saml2PostAuthenticationRequest      (HTTP-POST: samlRequest+relayState)
            │ 저장
            ▼
         Saml2AuthenticationRequestRepository      (보낸 요청을 보관 → 나중에 InResponseTo 대조)
            ├ HttpSessionSaml2AuthenticationRequestRepository (기본)
            └ CacheSaml2AuthenticationRequestRepository
```

## 동작 흐름 — `/saml2/authenticate/{registrationId}`

```
브라우저 ── GET /saml2/authenticate/{id} ──▶ Saml2WebSsoAuthenticationRequestFilter
                                               │ resolver.resolve(request)
   ┌───────────────────────────────────────────┘
   ▼
 BaseOpenSamlAuthenticationRequestResolver.resolve():
   1. requestMatcher 로 registrationId 추출
   2. relyingPartyRegistrationResolver.resolve(req, id)  → 확정 등록본
   3. uriResolver 로 entityId / acsLocation 플레이스홀더 치환
   4. AuthnRequest 객체 빌드 (Issuer, Destination=IdP SSO, ACS URL, NameIDPolicy)
   5. 바인딩 분기:
        REDIRECT → deflate+base64, (서명필요시) 쿼리파라미터 서명 → Saml2RedirectAuthenticationRequest
        POST     → (서명필요시) XML 자체 서명 → base64 → Saml2PostAuthenticationRequest
   ▼
 필터로 반환
   6. authenticationRequestRepository.saveAuthenticationRequest(...)   세션에 보관
   7. 바인딩에 맞게 응답:
        REDIRECT → 302 Location: {IdP SSO}?SAMLRequest=...&RelayState=...&SigAlg=...&Signature=...
        POST     → 자동제출 폼(FormPostRedirectStrategy)
   ▼
 브라우저 ── AuthnRequest ──▶ IdP
```

### 1) 어디서 트리거되나 — 필터

`saml2/.../web/Saml2WebSsoAuthenticationRequestFilter.java`

`OncePerRequestFilter` 로, 리졸버가 `null` 을 주면(이 요청은 인증요청이 아님) 그냥 체인을 흘려보낸다.

```java
protected void doFilterInternal(req, res, chain) {
    AbstractSaml2AuthenticationRequest authenticationRequest = resolver.resolve(req);
    if (authenticationRequest == null) { chain.doFilter(req, res); return; }
    if (authenticationRequest instanceof Saml2RedirectAuthenticationRequest)
        sendRedirect(req, res, ...);   // 302 + 쿼리 파라미터
    else
        sendPost(req, res, ...);       // 폼 자동 제출
}
```

`sendRedirect` 는 `SAMLRequest`/`RelayState`/`SigAlg`/`Signature` 를 `Saml2ParameterNames` 상수로 쿼리에 붙이고 ISO-8859-1로 인코딩한다. `sendPost` 는 `FormPostRedirectStrategy` 로 `<form method=post>` 자동 제출 HTML을 내려준다(POST 바인딩은 쿼리 길이 제한을 피한다).

### 2) AuthnRequest를 빚는다 — 리졸버

`saml2/.../web/authentication/BaseOpenSamlAuthenticationRequestResolver.java`

기본 매칭 URL은 `/saml2/authenticate/{registrationId}`(그리고 `/saml2/authenticate?registrationId=...`). 핵심 빌드 부분:

```java
AuthnRequest authnRequest = this.authnRequestBuilder.buildObject();
authnRequest.setForceAuthn(Boolean.FALSE);
authnRequest.setIsPassive(Boolean.FALSE);
authnRequest.setProtocolBinding(registration.getAssertionConsumerServiceBinding().getUrn()); // 응답 받을 바인딩
iss.setValue(entityId); authnRequest.setIssuer(iss);                                           // 우리가 누구인지
authnRequest.setDestination(registration.getAssertingPartyMetadata().getSingleSignOnServiceLocation());
authnRequest.setAssertionConsumerServiceURL(assertionConsumerServiceLocation);                // 응답 받을 곳
authnRequest.setIssueInstant(Instant.now(this.clock));
```

ID는 `parametersConsumer`(커스터마이저)가 직접 채우지 않았으면 `"ARQ" + UUID` 로 생성한다. 이 **ID가 곧 나중에 IdP 응답의 `InResponseTo` 와 대조**되는 값이다.

### 3) 바인딩별 인코딩과 서명

서명 여부는 두 설정의 OR로 결정된다 — RP가 항상 서명하기로 했거나(`isAuthnRequestsSigned`), IdP가 서명을 원하면(`getWantAuthnRequestsSigned`, 기본 true).

```java
Saml2MessageBinding binding = registration.getAssertingPartyMetadata().getSingleSignOnServiceBinding();
if (binding == Saml2MessageBinding.POST) {
    if (wantSigned || rpSigned)
        saml.withSigningKeys(signingCreds).algorithms(algs).sign(authnRequest);  // XML 자체에 <Signature>
    String encoded = Saml2Utils.withDecoded(serialize(authnRequest)).encode();   // base64
    return Saml2PostAuthenticationRequest...samlRequest(encoded)...build();
} else { // REDIRECT
    String deflated = Saml2Utils.withDecoded(xml).deflate(true).encode();        // deflate + base64
    if (wantSigned || rpSigned) {
        Map<String,String> query = saml.withSigningKeys(signingCreds).algorithms(algs)
            .sign(Map.of(SAML_REQUEST, deflated, RELAY_STATE, relayState));       // 쿼리스트링 서명
        builder.sigAlg(query.get(SIG_ALG)).signature(query.get(SIGNATURE));
    }
    return builder.build();
}
```

POST 바인딩은 **XML 내부에 `<ds:Signature>`** 를 심고, Redirect 바인딩은 **쿼리스트링 전체를 서명**해 `SigAlg`/`Signature` 파라미터로 분리한다 — 이는 SAML 바인딩 명세가 두 경우에 서로 다른 서명 방식을 요구하기 때문이다.

### 4) 보낸 요청을 저장하는 이유 — `Saml2AuthenticationRequestRepository`

필터는 응답을 보내기 직전에 `saveAuthenticationRequest(...)` 로 방금 만든 요청(특히 그 ID)을 세션에 저장한다. §04에서 IdP 응답이 돌아오면, 응답의 `InResponseTo` 가 이 저장된 요청 ID와 일치하는지 검사해 **응답 재생(replay)·교차 공격** 을 막는다. 기본 구현은 `HttpSessionSaml2AuthenticationRequestRepository`(세션 attribute), 무상태 환경에선 `CacheSaml2AuthenticationRequestRepository` 로 교체한다.

## 핵심 메서드 정리

- `Saml2AuthenticationRequestResolver#resolve(HttpServletRequest)` → 요청이 인증요청 URL이면 `AbstractSaml2AuthenticationRequest`, 아니면 `null`.
- `BaseOpenSamlAuthenticationRequestResolver#setParametersConsumer(...)` → 빌드된 `AuthnRequest` 를 보내기 직전에 손볼 수 있는 훅(예: `ForceAuthn=true`, 커스텀 ID, `Scoping`).
- `BaseOpenSamlAuthenticationRequestResolver#setRelayStateResolver(...)` → 기본은 랜덤 UUID. 로그인 후 돌아갈 위치 등 앱 상태를 RelayState에 실어 보낼 때 교체.

## 설계 포인트 / 확장점

- **전략 패턴**: 필터는 "언제/어떻게 보낼지"만 알고, "무엇을 보낼지"는 `Saml2AuthenticationRequestResolver` 에 위임. OpenSAML 의존은 리졸버 안에 갇힌다.
- **커스터마이징 훅**: `parametersConsumer` / `relayStateResolver` / `clock`(테스트용) / `requestMatcher` 를 통해 XML·URL·시간을 모두 바꿀 수 있다.
- **바인딩 추상화**: 같은 `AuthnRequest` 가 `Saml2MessageBinding` 에 따라 Redirect/POST 두 산출물로 갈린다. 호출부는 분기를 몰라도 된다.

## 정리

로그인 시작은 `Saml2WebSsoAuthenticationRequestFilter` 가 `/saml2/authenticate/{id}` 를 가로채면서 비롯된다. `BaseOpenSamlAuthenticationRequestResolver` 가 확정 등록본으로 `<AuthnRequest>` 를 빚고, RP/IdP 서명 선호에 따라 서명한 뒤, 바인딩에 맞춰 Redirect 쿼리 또는 POST 폼으로 변환한다. 필터는 그 요청을 세션에 저장(InResponseTo 대조용)하고 브라우저를 IdP로 보낸다. 이제 공은 IdP에 넘어갔고, 돌아온 응답을 받는 §04로 이어진다.
