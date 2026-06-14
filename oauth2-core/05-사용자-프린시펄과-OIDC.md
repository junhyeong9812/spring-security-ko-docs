# 05. 사용자 프린시펄과 OIDC

## 무엇을 / 왜

OAuth2 로그인이 끝나면 Spring Security의 `Authentication` 안에는 "이 사람이 누구인가"를 담은 **프린시펄**이 들어가야 한다. 그런데 OAuth2 사용자 정보는 제공자마다 키가 다르고(`sub`/`id`/`login`...), OIDC는 그 위에 표준 클레임과 ID 토큰을 더한다.

이 챕터는 프린시펄 계층을 **추상 → OAuth2 → OIDC**로 좁혀 가며 본다. 핵심 질문은 "왜 `getName()` 하나를 정하는 데 nameAttributeKey가 필요한가", "OIDC 사용자의 클레임은 어디서 합쳐지는가"다.

## 핵심 타입 — 좁혀지는 상속 계층

```
   spring-security-core
   «interface» AuthenticatedPrincipal   getName()
        ▲
        │ extends
   «interface» OAuth2AuthenticatedPrincipal      getAttributes(), getAuthorities(), getAttribute(name)
        ▲                          ▲
        │ extends                  │ implements
   «interface» OAuth2User      DefaultOAuth2AuthenticatedPrincipal  (리소스 서버 introspection 프린시펄)
        ▲
        │ implements
   DefaultOAuth2User  ──── nameAttributeKey 로 getName() 결정
        ▲
        │ extends, + IdTokenClaimAccessor
   DefaultOidcUser  implements OidcUser
        └─ OidcIdToken + (OidcUserInfo)  →  getClaims() 통합

   [권한]  GrantedAuthority
            └ OAuth2UserAuthority (attributes 보유)
                 └ OidcUserAuthority (idToken + userInfo 보유)
```

## OAuth2AuthenticatedPrincipal — 공통 계약

`AuthenticatedPrincipal`(이름만 있음)에 OAuth2다운 두 가지를 더한다: **attributes(클레임/속성 Map)** 와 **authorities(권한)**.

```java
// OAuth2AuthenticatedPrincipal.java
default <A> @Nullable A getAttribute(String name) { return (A) getAttributes().get(name); }
Map<String, Object> getAttributes();
Collection<? extends GrantedAuthority> getAuthorities();
```

리소스 서버의 토큰 인트로스펙션이 만드는 `DefaultOAuth2AuthenticatedPrincipal`이 이 인터페이스의 직접 구현이다. OAuth2 로그인 쪽은 한 단계 더 좁힌 `OAuth2User`를 쓴다.

## OAuth2User / DefaultOAuth2User — 이름 키의 문제

`OAuth2User`는 마커에 가깝다(`OAuth2AuthenticatedPrincipal`을 그대로 상속). 진짜 일은 `DefaultOAuth2User`가 한다. 여기서 OAuth2 사용자 모델링의 핵심 난점이 드러난다: **attributes Map에는 표준화된 "이름" 키가 없다.** 구글은 `sub`, 깃허브는 `id` 또는 `login`을 쓴다.

그래서 생성자에 **nameAttributeKey**를 받아, 어떤 속성을 `getName()`으로 쓸지 지정하게 한다.

```java
// DefaultOAuth2User.java
public DefaultOAuth2User(@Nullable Collection<? extends GrantedAuthority> authorities,
        Map<String, Object> attributes, String nameAttributeKey) {
    Assert.notEmpty(attributes, "attributes cannot be empty");
    Assert.hasText(nameAttributeKey, "nameAttributeKey cannot be empty");
    Assert.notNull(attributes.get(nameAttributeKey), "Attribute value for '" + nameAttributeKey + "' cannot be null");
    ...
}

@Override
public String getName() {
    Object nameAttributeValue = this.getAttribute(this.nameAttributeKey);
    Assert.notNull(nameAttributeValue, "name attribute value cannot be null");
    return nameAttributeValue.toString();
}
```

생성자가 "그 키의 값이 실제로 존재"함을 강제하므로, `getName()`이 `null`을 돌려줄 일이 없다. attributes는 `unmodifiableMap`, authorities는 권한명으로 정렬 후 `unmodifiableSet` — 전형적인 불변 프린시펄이다. `oauth2-client`의 `OAuth2UserService`가 UserInfo 응답을 이 객체로 빚는다.

### OAuth2UserAuthority — 속성을 품은 권한

`GrantedAuthority`(기본값 `"OAUTH2_USER"`)이면서 사용자 attributes를 함께 들고 있다. `equals`가 흥미로운데, attribute 값 중 `URL`은 문자열로 환산해 비교한다 — `URL.equals()`가 DNS 조회를 유발하는 함정을 피하기 위해서다.

```java
// OAuth2UserAuthority.convertURLIfNecessary
return (value instanceof URL) ? ((URL) value).toExternalForm() : value;
```

## OIDC: 클레임이 합쳐지는 지점

OpenID Connect는 OAuth2 위에 **신원(identity)** 계층을 얹는다. 핵심 차이: 인증 결과로 **ID 토큰**(서명된 JWT, 사용자에 대한 클레임)을 받고, 선택적으로 **UserInfo 엔드포인트**에서 추가 클레임을 받는다.

### 클레임 접근자 계층

```
ClaimAccessor (02장)
   └ StandardClaimAccessor   ── OIDC 표준 프로필 클레임 (name, email, picture, ...)
        └ IdTokenClaimAccessor  ── ID 토큰 전용 (iss, sub, aud, exp, nonce, auth_time, at_hash, ...)
```

`StandardClaimAccessor`는 `getEmail()`, `getFullName()` 등을, `IdTokenClaimAccessor`는 `getIssuer()`, `getNonce()` 등을 전부 `default` 메서드로 제공한다. 구현체는 `getClaims()` Map만 내놓으면 된다(02장의 패턴).

### OidcIdToken — 토큰이자 클레임 접근자

ID 토큰은 두 정체성을 동시에 가진다: `AbstractOAuth2Token`(토큰 값/만료)이면서 `IdTokenClaimAccessor`(클레임).

```java
// OidcIdToken.java
public class OidcIdToken extends AbstractOAuth2Token implements IdTokenClaimAccessor {
    @Override public Map<String, Object> getClaims() { return this.claims; }
    public static Builder withTokenValue(String tokenValue) { ... }
}
```

빌더는 `subject()`, `audience()`, `nonce()`, `authTime()` 같은 의미 있는 메서드로 클레임을 쌓고, `build()` 시 `iat`/`exp` 클레임에서 토큰의 발급/만료 시각을 끌어온다(01장의 시각 불변식과 연결).

### OidcUserInfo — UserInfo 응답

`StandardClaimAccessor`를 구현한 불변 클레임 컨테이너. 빌더(`email()`, `address()`, ...)로 만든다. ID 토큰이 "인증 사실"이라면 UserInfo는 "프로필 상세"다.

### OidcUser / DefaultOidcUser — 통합 프린시펄

`OidcUser`는 `OAuth2User`와 `IdTokenClaimAccessor`를 **둘 다** 상속해, "OAuth2 사용자이면서 ID 토큰 클레임 접근자"가 된다. `DefaultOidcUser`가 핵심 통합을 수행한다.

```java
// DefaultOidcUser.java
public class DefaultOidcUser extends DefaultOAuth2User implements OidcUser {
    public DefaultOidcUser(... OidcIdToken idToken, @Nullable OidcUserInfo userInfo, String nameAttributeKey) {
        super(authorities, OidcUserAuthority.collectClaims(idToken, userInfo), nameAttributeKey);
        ...
    }
    @Override public Map<String, Object> getClaims() { return ...; }  // idToken + userInfo 통합
}
```

두 가지를 보자:
- **클레임 통합**: `OidcUserAuthority.collectClaims(idToken, userInfo)`가 ID 토큰 클레임과 UserInfo 클레임을 하나의 Map으로 합친다. 이게 `DefaultOAuth2User`의 attributes가 되고, `getClaims()`/`getAttributes()`가 통합 뷰를 노출한다.
- **기본 이름 키**: nameAttributeKey 기본값이 `IdTokenClaimNames.SUB`("sub")다 — OIDC는 OAuth2와 달리 표준 식별자(`sub`)가 있으므로 키를 추측할 필요가 없다.

`OidcUserAuthority`는 `OAuth2UserAuthority`를 확장해 `getIdToken()`/`getUserInfo()`를 추가로 들고, `equals`/`hashCode`도 ID 토큰·UserInfo 기준으로 재정의한다.

## 동작 흐름 — OIDC 로그인이 프린시펄을 만들기까지

```
인가 코드 교환 응답 (04장)
   │  access_token + id_token (JWT)
   ▼
OidcIdTokenDecoderFactory  (oauth2-client) — JWT 디코드/서명·nonce·exp 검증 (01·02장 활용)
   │  → OidcIdToken (claims)
   ▼
(scope에 따라) UserInfo 엔드포인트 호출 → OidcUserInfo
   │
   ▼
OidcUserService
   │  collectClaims(idToken, userInfo) → 통합 attributes
   │  권한 = OidcUserAuthority + SCOPE_* (06장)
   ▼
new DefaultOidcUser(authorities, idToken, userInfo, "sub")
   │
   ▼
OAuth2LoginAuthenticationToken.getPrincipal()  →  SecurityContext 저장
```

`oauth2-core`는 이 흐름의 **값 객체와 접근자 계약**(OidcIdToken, OidcUserInfo, OidcUser, *ClaimAccessor)만 정의한다. 실제 디코딩·UserInfo 호출·서비스 조립은 `oauth2-client`가 한다.

## 설계 포인트 / 확장점

- **상속으로 점진적 특화**: `AuthenticatedPrincipal → OAuth2AuthenticatedPrincipal → OAuth2User → OidcUser`. 각 단계가 직전 계약에 의미를 더한다. 리소스 서버(introspection)는 중간 단계(`DefaultOAuth2AuthenticatedPrincipal`)에서 분기.
- **nameAttributeKey로 비표준 흡수**: 표준화 안 된 OAuth2 사용자 이름 문제를, 키를 주입받아 푼다. OIDC는 `sub` 기본값으로 이 부담을 없앤다.
- **클레임 통합 지점의 단일화**: ID 토큰과 UserInfo의 합성을 `collectClaims` 한 곳에 모아, `getClaims()`/`getAttributes()`/`getName()`이 모두 일관된 뷰를 본다.
- **URL equals 함정 회피**: 권한 동등성 비교에서 `URL`을 문자열화 — 미묘하지만 실전에서 중요한 안전장치.

## 정리

프린시펄 계층은 `AuthenticatedPrincipal`에서 출발해 OAuth2(attributes+authorities), 다시 OIDC(ID 토큰+UserInfo 클레임)로 좁혀진다. `DefaultOAuth2User`는 nameAttributeKey로 제각각인 사용자 이름 키 문제를 풀고, `DefaultOidcUser`는 ID 토큰과 UserInfo 클레임을 `collectClaims`로 통합해 단일한 사용자 뷰를 만든다. ID 토큰은 토큰이면서 클레임 접근자라는 이중 정체성을 가지며, 표준 클레임 게터는 모두 02장의 `ClaimAccessor` default 메서드로 공짜로 얻는다. 이 모듈은 계약과 값 객체를, 실제 조립은 `oauth2-client`가 맡는다.
