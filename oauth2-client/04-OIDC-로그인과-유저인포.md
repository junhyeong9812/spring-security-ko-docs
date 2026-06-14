# 04. OIDC 로그인과 UserInfo

## 무엇을 / 왜

scope에 `openid`가 있으면 그것은 단순 OAuth2가 아니라 **OpenID Connect**다. OIDC는 OAuth2 위에 "사용자 인증"을 표준화한 계층으로, 토큰 응답에 **id_token**(서명된 JWT)이 포함된다. id_token은 그 자체로 "이 사용자가 인증되었다"는 검증 가능한 증거이므로, 클라이언트는 그것을 **반드시 검증**해야 한다.

이 장은 OIDC 전용 Provider(`OidcAuthorizationCodeAuthenticationProvider`)가 어떻게 id_token을 검증하고, `OidcUserService`가 어떻게 UserInfo와 합쳐 `OidcUser`를 만드는지, 그리고 OIDC 로그아웃(RP-Initiated / Back-Channel)을 다룬다.

소스 위치: `oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/oidc/`, `.../userinfo/`

---

## 핵심 타입

```
OidcAuthorizationCodeAuthenticationProvider (AuthenticationProvider)
   ├─ OAuth2AccessTokenResponseClient   code→token (+ id_token)
   ├─ JwtDecoderFactory<ClientRegistration>  ── OidcIdTokenDecoderFactory
   │     → jwkSetUri로 서명 검증, OidcIdTokenValidator로 클레임 검증
   └─ OAuth2UserService<OidcUserRequest, OidcUser> → OidcUserService

OidcUserService (OAuth2UserService<OidcUserRequest, OidcUser>)
   └─ DefaultOAuth2UserService 로 UserInfo 호출 위임
   └─ id_token 클레임 + UserInfo 클레임 병합 → DefaultOidcUser

DefaultOAuth2UserService (OAuth2UserService<OAuth2UserRequest, OAuth2User>)
   └─ UserInfo Endpoint를 RestOperations로 호출 → DefaultOAuth2User

OidcClientInitiatedLogoutSuccessHandler   RP-Initiated Logout (end_session_endpoint로 리다이렉트)
OidcSessionRegistry / OidcLogoutToken     Back-Channel Logout
```

---

## 동작 흐름: id_token 검증 + UserInfo 병합

```
OidcAuthorizationCodeAuthenticationProvider.authenticate()
   │
   1. scope에 openid 없으면 → return null (03장 일반 Provider에 양보)
   2. authorizationResponse.statusError() → 오류면 예외
   3. response.state == request.state 검증 (CSRF)
   4. accessTokenResponseClient.getTokenResponse() → access/refresh/id_token
   5. id_token 없으면 → invalid_id_token 예외 (OIDC 필수)
   6. createOidcToken(): JwtDecoder로 id_token 서명·클레임 검증
   7. validateNonce(): SHA-256(요청 nonce) == id_token.nonce
   8. userService.loadUser(OidcUserRequest) → OidcUser
   9. OAuth2LoginAuthenticationToken(주체=OidcUser) 반환
```

### id_token 검증 — JwtDecoderFactory

핵심은 `createOidcToken`이 `JwtDecoderFactory`로 등록별 디코더를 얻어 id_token을 디코딩하는 부분이다.

```java
private OidcIdToken createOidcToken(ClientRegistration clientRegistration,
        OAuth2AccessTokenResponse accessTokenResponse) {
    JwtDecoder jwtDecoder = this.jwtDecoderFactory.createDecoder(clientRegistration);
    Jwt jwt = getJwt(accessTokenResponse, jwtDecoder);   // 서명 검증 + 파싱
    return new OidcIdToken(jwt.getTokenValue(), jwt.getIssuedAt(), jwt.getExpiresAt(), jwt.getClaims());
}
```

기본 팩토리 `OidcIdTokenDecoderFactory`는 `jwkSetUri`(또는 discovery 메타데이터)로부터 공개키를 받아 **서명을 검증**하고, `OidcIdTokenValidator`(`DefaultOidcIdTokenValidatorFactory`)로 **클레임을 검증**한다 — `iss`(issuer), `aud`(우리 clientId 포함), `exp`(만료), `iat` 등. 검증 실패 시 `invalid_id_token` 오류로 인증을 막는다.

### nonce 검증 — replay 방어

02장 resolver가 인가 요청에 심은 nonce를 여기서 대조한다. 인가 요청엔 `nonce`의 SHA-256 해시를 보냈고, 그 원본 nonce는 세션에 보관되어 있다.

```java
private void validateNonce(OAuth2AuthorizationRequest authorizationRequest, OidcIdToken idToken) {
    String requestNonce = authorizationRequest.getAttribute(OidcParameterNames.NONCE);
    if (requestNonce == null) return;
    String nonceHash = getNonceHash(requestNonce);     // SHA-256(requestNonce)
    if (!nonceHash.equals(idToken.getNonce())) {       // id_token의 nonce 클레임과 일치?
        throw new OAuth2AuthenticationException(new OAuth2Error("invalid_nonce"));
    }
}
```

이로써 가로챈 id_token을 다른 세션에 재사용하는 공격을 막는다.

### OidcUserService — id_token + UserInfo 병합

```java
public OidcUser loadUser(OidcUserRequest userRequest) {
    OidcUserInfo userInfo = null;
    if (this.retrieveUserInfo.test(userRequest)) {          // UserInfo 호출이 필요한가?
        OAuth2User oauth2User = this.oauth2UserService.loadUser(userRequest); // DefaultOAuth2UserService
        Map<String, Object> claims = getClaims(userRequest, oauth2User);
        userInfo = new OidcUserInfo(claims);
        // 보안: UserInfo의 sub == id_token의 sub (토큰 치환 공격 방어)
        if (!userInfo.getSubject().equals(userRequest.getIdToken().getSubject())) {
            throw new OAuth2AuthenticationException(new OAuth2Error("invalid_user_info_response"));
        }
    }
    OidcUserSource source = new OidcUserSource(userRequest, userInfo, oauth2User);
    return this.oidcUserConverter.convert(source);  // DefaultOidcUser
}
```

두 가지 보안 포인트:

- **UserInfo는 조건부 호출**: `shouldRetrieveUserInfo`가 판단. UserInfo URI가 설정되어 있고, access token의 scope에 `profile`/`email` 등 사용자 정보 scope가 있을 때만 부른다. id_token만으로 충분하면 호출을 생략한다.
- **sub 일치 검증**: UserInfo 응답의 `sub`가 id_token의 `sub`와 다르면 거부한다. OIDC 명세가 명시한 토큰 치환(token substitution) 공격 방어.

### DefaultOAuth2UserService — UserInfo Endpoint 호출

OIDC가 아닌 일반 OAuth2 Login도, OIDC의 UserInfo 단계도 결국 이 클래스를 거친다.

```java
public OAuth2User loadUser(OAuth2UserRequest userRequest) {
    String userNameAttributeName = getUserNameAttributeName(userRequest); // 미설정 시 명확한 오류
    RequestEntity<?> request = this.requestEntityConverter.convert(userRequest);
    ResponseEntity<Map<String, Object>> response = getResponse(userRequest, request); // GET/POST + Bearer
    ...
    return new DefaultOAuth2User(authorities, attributes, userNameAttributeName);
}
```

`getAuthorities`는 `OAuth2UserAuthority`와 함께 access token의 각 scope를 `SCOPE_xxx` 권한으로 부여한다. 그래서 `@PreAuthorize("hasAuthority('SCOPE_read')")` 같은 인가가 자연스럽게 동작한다. `userNameAttributeName`이 등록에 없으면 친절한 오류를 던지는데, 이는 01장에서 본 제공자별 이름 키 차이 때문이다.

---

## OIDC 로그아웃

```
RP-Initiated Logout (사용자 주도)
   OidcClientInitiatedLogoutSuccessHandler (SimpleUrlLogoutSuccessHandler 확장)
     로그아웃 성공 시 → 인가 서버의 end_session_endpoint 로 리다이렉트
     id_token_hint, post_logout_redirect_uri 첨부 → IdP 세션까지 종료

Back-Channel Logout (IdP 주도)
   인가 서버가 우리 앱 백채널로 Logout Token(JWT) 전송
     OidcLogoutToken 검증 → OidcSessionRegistry 에서 해당 세션 무효화
     InMemoryOidcSessionRegistry: (sub, sid) → 활성 세션 추적
```

`OidcClientInitiatedLogoutSuccessHandler.determineTargetUrl`은 현재 인증이 `OAuth2AuthenticationToken` + `OidcUser`인지 확인하고, 등록의 `end_session_endpoint`를 찾아 리다이렉트 URL을 만든다. 우리 앱 로그아웃만으로는 IdP에 세션이 남으므로, OIDC에서는 IdP 세션까지 끊는 이 핸들러가 필요하다.

---

## 핵심 메서드

| 메서드 | 하는 일 |
|---|---|
| `OidcAuthorizationCodeAuthenticationProvider.authenticate` | state·id_token·nonce 검증 후 OidcUser 조립 |
| `OidcIdTokenDecoderFactory.createDecoder` | jwkSetUri로 서명 검증 디코더 + 클레임 validator 구성 |
| `OidcUserService.loadUser` | UserInfo 조건부 호출, sub 일치 검증, id_token+UserInfo 병합 |
| `DefaultOAuth2UserService.loadUser` | UserInfo Endpoint 호출, scope→`SCOPE_*` 권한 부여 |
| `OidcClientInitiatedLogoutSuccessHandler.determineTargetUrl` | end_session_endpoint로 IdP 로그아웃 리다이렉트 |

---

## 설계 포인트 / 확장점

- **검증의 다층 방어**: state(CSRF) → id_token 서명/클레임 → nonce(replay) → UserInfo sub 일치(치환). 각 단계가 서로 다른 공격을 막는다.
- **디코더 팩토리 교체**: `setJwtDecoderFactory`로 클레임 검증 규칙(허용 issuer, clock skew 등)을 커스터마이즈한다.
- **UserInfo 매핑 훅**: `OidcUserService.setOidcUserConverter` / `DefaultOAuth2UserService.setAttributesConverter`로 중첩 속성을 평탄화하거나 커스텀 `OidcUser`를 만든다.
- **권한 매핑**: `setAuthoritiesMapper`로 OIDC scope/claim을 도메인 ROLE로 변환.

---

## 정리

OIDC 경로는 id_token이라는 검증 가능한 인증 증거를 다룬다. `OidcAuthorizationCodeAuthenticationProvider`가 서명·클레임·nonce를 검증하고, `OidcUserService`가 (필요할 때만) UserInfo를 불러 sub 일치까지 확인한 뒤 `OidcUser`를 만든다. 일반 OAuth2의 UserInfo 호출도 `DefaultOAuth2UserService`를 공유하며 scope를 `SCOPE_*` 권한으로 승격시킨다. 로그아웃은 우리 앱을 넘어 IdP 세션까지 끊는 RP-Initiated/Back-Channel 메커니즘을 제공한다.
