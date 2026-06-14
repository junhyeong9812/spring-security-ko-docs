# 06. OIDC와 부가 엔드포인트 — 검사·폐기·공개키·메타데이터·OpenID Connect

## 무엇을 / 왜

토큰을 발급하는 핵심 경로(03~05장) 외에도 인가 서버는 여러 보조 엔드포인트를 노출한다. 토큰을 **검사**(introspection)하고 **폐기**(revocation)하며, JWT 검증에 필요한 **공개키**(JWK Set)와 **서버 메타데이터**(discovery)를 공개하고, OpenID Connect 1.0의 **UserInfo·Logout·Discovery·Dynamic Client Registration**을 제공한다. 이 챕터는 이들의 역할과, 02장의 골격이 여기에도 그대로 적용된다는 점을 정리한다.

## 핵심 타입

```
web/                                         설정 키 (기본 URI)
  OAuth2TokenIntrospectionEndpointFilter      /oauth2/introspect
  OAuth2TokenRevocationEndpointFilter         /oauth2/revoke
  NimbusJwkSetEndpointFilter                  /oauth2/jwks
  OAuth2AuthorizationServerMetadataEndpointFilter  /.well-known/oauth-authorization-server
  OAuth2DeviceAuthorizationEndpointFilter     /oauth2/device_authorization
  OAuth2DeviceVerificationEndpointFilter      /oauth2/device_verification
  OAuth2PushedAuthorizationRequestEndpointFilter   /oauth2/par
  OAuth2ClientRegistrationEndpointFilter      /oauth2/register
oidc/web/
  OidcUserInfoEndpointFilter                  /userinfo
  OidcLogoutEndpointFilter                    /connect/logout
  OidcProviderConfigurationEndpointFilter     /.well-known/openid-configuration
  OidcClientRegistrationEndpointFilter        /connect/register
```

검사·폐기·UserInfo·Logout·Registration은 02장의 **Filter → Converter → Provider → Handler** 골격을 따른다. JWK Set과 두 메타데이터 엔드포인트는 인증할 게 없어 Provider 없이 직접 JSON을 쓴다.

## 토큰 검사 — `/oauth2/introspect` (RFC 7662)

리소스 서버가 불투명 토큰을 받았을 때 "이 토큰 유효해?"를 인가 서버에 물어보는 엔드포인트다. `OAuth2TokenIntrospectionAuthenticationProvider`가 처리한다. 클라이언트 인증(02장)을 통과한 뒤:

```java
OAuth2Authorization authorization = this.authorizationService
    .findByToken(tokenIntrospectionAuthentication.getToken(), null);   // 타입 모름 → null 힌트
if (authorization == null)
    return tokenIntrospectionAuthentication;            // 못 찾으면 그대로 (= active:false 응답)

OAuth2Authorization.Token<OAuth2Token> authorizedToken =
    authorization.getToken(tokenIntrospectionAuthentication.getToken());
if (!authorizedToken.isActive())
    return new OAuth2TokenIntrospectionAuthenticationToken(token, clientPrincipal,
            OAuth2TokenIntrospection.builder().build());    // active:false
...
OAuth2TokenIntrospection tokenClaims = withActiveTokenClaims(authorizedToken, authorizedClient);
```

핵심은 **저장된 인가에서 토큰을 찾아 `isActive()`로 판단**한다는 것이다(01장의 `Token.isActive()` = !invalidated && !expired && !beforeUse). 토큰을 찾지 못하거나 비활성이면 `{"active":false}`만 응답해 정보 누출을 막는다. 활성이면 `withActiveTokenClaims`가 저장된 클레임(05장에서 발급 시 메타데이터로 적재해둔)을 꺼내 `client_id`·`iat`·`exp`·`token_type` 등을 채운다. 토큰을 **다시 디코딩·서명검증하지 않고** 저장된 클레임을 그대로 쓰는 점이 설계 포인트다 — 인가 서버는 자기가 발급한 토큰의 상태를 이미 알고 있기 때문이다.

## 토큰 폐기 — `/oauth2/revoke` (RFC 7009)

`OAuth2TokenRevocationAuthenticationProvider`가 처리한다. 제출된 토큰으로 인가를 찾아 해당 토큰을 무효화(`OAuth2Authorization.from(auth).invalidate(token).build()` 후 `save`)한다. 리프레시 토큰을 폐기하면 01장의 연쇄 무효화로 액세스 토큰·코드까지 함께 죽는다. RFC 7009 규약상 토큰을 못 찾아도 성공(200)으로 응답한다.

## JWK Set — `/oauth2/jwks`

`NimbusJwkSetEndpointFilter`는 인증 없이 **공개키만** 노출한다. 05장에서 `JwtEncoder`가 JWT를 서명할 때 쓴 키쌍의 공개 부분을 JSON Web Key Set으로 직렬화한다. 리소스 서버·클라이언트가 이 엔드포인트로 키를 받아 JWT 서명을 검증한다. 키 소스는 이 모듈이 아니라 애플리케이션이 제공하는 `JWKSource<SecurityContext>` 빈(Nimbus)이며, configurer가 그 빈이 있을 때만 이 필터를 추가한다(`OAuth2AuthorizationServerConfigurer.configure()`).

```java
JWKSource<...> jwkSource = OAuth2ConfigurerUtils.getJwkSource(httpSecurity);
if (jwkSource != null) {
    NimbusJwkSetEndpointFilter jwkSetEndpointFilter = new NimbusJwkSetEndpointFilter(jwkSource, jwkSetEndpointUri);
    httpSecurity.addFilterBefore(postProcess(jwkSetEndpointFilter), AbstractPreAuthenticatedProcessingFilter.class);
}
```

즉 **개인키는 인가 서버 안에만, 공개키만 이 엔드포인트로** 나간다. 키 관리·회전은 `JWKSource` 구현(`oauth2-jose`)의 책임이다.

## 서버 메타데이터 — Discovery

`OAuth2AuthorizationServerMetadataEndpointFilter`(`/.well-known/oauth-authorization-server`, RFC 8414)와 OIDC의 `OidcProviderConfigurationEndpointFilter`(`/.well-known/openid-configuration`)는 issuer, 각 엔드포인트 URL, 지원 그랜트·scope·서명 알고리즘 등을 JSON으로 공개한다. 클라이언트가 이 문서 하나만 읽으면 인가 서버의 모든 엔드포인트를 자동 발견할 수 있다. configurer는 활성화된 엔드포인트(Device·PAR·Client Registration 등)에 따라 메타데이터에 해당 항목을 동적으로 추가한다(`addDefaultAuthorizationServerMetadataCustomizer`, `OAuth2AuthorizationServerConfigurer.configure()`). issuer 값은 02장의 `AuthorizationServerContextHolder`에서 가져오므로 멀티테넌트에서 요청마다 달라질 수 있다.

## OpenID Connect 1.0 (옵션, 기본 비활성)

OIDC는 `OAuth2AuthorizationServerConfigurer.oidc(Customizer)`로 명시적으로 켜야 활성화된다(`OidcConfigurer`). 켜지면:

- **UserInfo** (`/userinfo`): `OidcUserInfoEndpointFilter` + `OidcUserInfoAuthenticationProvider`. 액세스 토큰을 받아(인가 서버가 스스로 리소스 서버로 동작) 사용자 클레임을 응답한다.
- **Logout** (`/connect/logout`): `OidcLogoutEndpointFilter` + `OidcLogoutAuthenticationProvider`. RP-Initiated Logout. ID 토큰 힌트와 세션(02·04장에서 `SessionRegistry`에 등록된 `sid`)을 검증하고 post-logout redirect로 보낸다.
- **Client Registration** (`/connect/register`): `OidcClientRegistrationEndpointFilter` + `OidcClientRegistrationAuthenticationProvider`. 동적으로 `RegisteredClient`를 등록/조회한다.

OIDC 활성화는 03장에서 본 것처럼 인가 엔드포인트의 동작도 바꾼다. 켜지면 `openid` scope 요청이 허용되고 세션이 추적되며, 꺼져 있으면 `openid` 요청을 `invalid_scope`로 거부하는 validator가 주입된다(`OAuth2AuthorizationServerConfigurer.init()`).

```java
if (isOidcEnabled()) {
    initSessionRegistry(httpSecurity);                  // 세션 추적 (sid/auth_time 의 원천)
    authorizationEndpointConfigurer.setSessionAuthenticationStrategy(...registerNewSession...);
} else {
    // openid scope 요청을 거부
    authorizationEndpointConfigurer.addAuthorizationCodeRequestAuthenticationValidator(rejectOidc);
}
```

UserInfo·Client Registration이 액세스 토큰을 요구하므로, configurer는 이때 인가 서버 자신에 `oauth2ResourceServer().jwt()`를 켠다 — 발급자가 곧 검증자가 되는 셈이다.

## 그 외 — Device Flow / PAR / Dynamic Client Registration

- **Device Authorization Grant**(RFC 8628): `OAuth2DeviceAuthorizationEndpointFilter`(코드 발급) + `OAuth2DeviceVerificationEndpointFilter`(사용자 승인). 두 엔드포인트는 한쪽을 켜면 다른 쪽도 자동으로 켜진다(configurer의 상호 활성화).
- **Pushed Authorization Requests**(RFC 9126): `OAuth2PushedAuthorizationRequestEndpointFilter`. 인가 요청 파라미터를 백채널로 미리 밀어넣고 `request_uri`만 받아 가는 방식. 03장의 인가 Provider가 `request_uri`를 해석해 저장된 요청을 복원한다.
- **OAuth2 Dynamic Client Registration**(RFC 7591): `OAuth2ClientRegistrationEndpointFilter`. OIDC 등록(`/connect/register`)과 별개로 OAuth2 표준 등록(`/oauth2/register`)을 제공.

이들은 모두 옵션이며, 활성화하면 02장의 동일 골격으로 동작하고 메타데이터에도 자동 반영된다.

## 설계 포인트 / 확장점

- **발급자=검증자의 이점**: introspection·revocation이 저장된 `OAuth2Authorization` 상태를 직접 보므로, 토큰을 다시 파싱·검증하지 않고 즉답한다. 불투명 토큰의 폐기가 "저장소에서 무효화 표시" 한 번으로 즉시 반영되는 것도 같은 이유다.
- **엔드포인트의 모듈식 on/off**: 거의 모든 부가 엔드포인트가 `*EndpointConfigurer`로 켜고 끌 수 있고, 메타데이터가 활성 엔드포인트만 광고한다. 필요한 표면만 노출하는 보안 기본값.
- **OIDC의 침투성**: OIDC를 켜면 인가 엔드포인트·세션·ID 토큰 클레임까지 연쇄적으로 동작이 바뀐다. "OIDC 활성 여부"는 단일 플래그가 아니라 여러 컴포넌트에 validator·strategy를 주입하는 스위치다.

## 정리

핵심 발급 경로 바깥에서, 인가 서버는 토큰을 검사·폐기하고(저장된 인가 상태를 직접 조회), 서명 검증용 공개키와 서버 메타데이터를 공개하며, OIDC를 켜면 UserInfo·Logout·동적 등록과 세션 추적까지 제공한다. 이 모든 부가 엔드포인트가 02장의 균일한 Filter 골격과 01장의 도메인 모델 위에서 동작한다는 점이, 이 모듈 전체를 관통하는 일관성이다. 실제 `SecurityFilterChain` 배선과 각 엔드포인트의 세부 커스터마이징은 `spring-security-config`의 `OAuth2AuthorizationServerConfigurer` 및 `*EndpointConfigurer`에서 마무리된다.
