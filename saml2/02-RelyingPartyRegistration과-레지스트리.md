# 02. RelyingPartyRegistration과 레지스트리

## 무엇을 / 왜

SAML SP가 동작하려면 "우리(RP)는 누구이고, 상대 IdP(AP)는 누구이며, 어떤 엔드포인트·인증서·바인딩을 쓰는가"를 모두 알아야 한다. 이 설정 한 덩어리가 **`RelyingPartyRegistration`** 이고, 여러 개를 모아 요청마다 골라 쓰는 것이 **`RelyingPartyRegistrationRepository`** 다. 이 장의 타입들은 §03(AuthnRequest)·§04(응답검증)·§05(메타데이터/로그아웃) 모든 흐름의 입력이 된다.

## 핵심 타입

```
RelyingPartyRegistration  (RP + AP 한 쌍, registrationId로 식별)
│
├── RP(우리) 측 정보
│   ├── registrationId                  URL 경로에 쓰이는 고유 키
│   ├── entityId                        기본 "{baseUrl}/saml2/service-provider-metadata/{registrationId}"
│   ├── assertionConsumerServiceLocation  기본 "{baseUrl}/login/saml2/sso/{registrationId}" (응답 수신점)
│   ├── assertionConsumerServiceBinding   기본 POST
│   ├── signingX509Credentials          AuthnRequest/LogoutRequest 서명용
│   ├── decryptionX509Credentials       암호화된 assertion 복호화용
│   ├── authnRequestsSigned             RP가 요청에 항상 서명할지
│   ├── nameIdFormat / singleLogoutService...
│
└── assertingPartyMetadata : AssertingPartyMetadata   (AP/IdP 측 정보)
    ├── entityId
    ├── singleSignOnServiceLocation / Binding   IdP의 로그인 수신 엔드포인트
    ├── verificationX509Credentials             IdP 서명 검증용
    ├── encryptionX509Credentials               IdP에게 보낼 것 암호화용
    ├── wantAuthnRequestsSigned (기본 true)     IdP가 서명을 원하는지
    ├── signingAlgorithms (기본 rsa-sha256)
    └── singleLogoutService... (SLO 엔드포인트)
```

`saml2/.../registration/RelyingPartyRegistration.java` 와 `AssertingPartyMetadata.java`.

### `RelyingPartyRegistration` — 불변 설정 + 빌더

생성자는 설정을 그대로 받지 않고 **검증** 한다. 예를 들어 서명 키 컬렉션은 모든 원소가 서명 용도인지 확인한다.

```java
for (Saml2X509Credential c : signingX509Credentials) {
    Assert.isTrue(c.isSigningCredential(),
        "All signingX509Credentials must have a usage of SIGNING set");
}
// SLO 위치를 줬으면 바인딩도 있어야 한다
Assert.isTrue(singleLogoutServiceLocation == null || !isEmpty(singleLogoutServiceBindings), ...);
```

엔드포인트 문자열에는 `{baseUrl}`, `{registrationId}`, `{baseScheme}`, `{baseHost}`, `{basePort}` 같은 **플레이스홀더** 가 허용된다. 실제 HTTP 요청이 오기 전까지는 호스트/포트를 모르므로, 이 치환은 요청 시점에 `RelyingPartyRegistrationPlaceholderResolvers` 가 수행한다(§03·§04에서 `uriResolver.resolve(...)` 로 등장).

### 빌더의 두 진입점

```java
// (1) 처음부터 손으로
RelyingPartyRegistration.withRegistrationId("simplesamlphp")
    .entityId("{baseUrl}/saml2/service-provider-metadata/{registrationId}")
    .signingX509Credentials(c -> c.add(signingCred))
    .assertingPartyMetadata(ap -> ap
        .entityId("https://idp.example/metadata")
        .singleSignOnServiceLocation("https://idp.example/SSO")
        .verifyingX509Credentials(c -> c.add(verifyCred)))   // 인터페이스에선 verificationX509Credentials
    .build();

// (2) 이미 가진 AP 메타데이터로 시작 (registrationId = AP entityId)
RelyingPartyRegistration.withAssertingPartyMetadata(metadata).build();
```

`mutate()` 는 현재 설정을 복사한 새 빌더를 돌려준다. §04의 `Saml2AuthenticationTokenConverter` 와 §05의 로그아웃 리졸버가 이걸로 플레이스홀더가 치환된 새 등록본을 만든다.

### `AssertingPartyMetadata` — IdP 부분을 인터페이스로 추상화 (6.4+)

원래 AP 정보는 구체 클래스 `RelyingPartyRegistration.AssertingPartyDetails` 였는데, 6.4부터 `AssertingPartyMetadata` 인터페이스로 추상화됐다. `AssertingPartyDetails` 는 그 인터페이스를 구현하며, 덕분에 IdP 메타데이터를 DB·캐시 등 다른 저장소가 제공하는 구현으로 바꿀 수 있게 됐다. 빌더의 `wantAuthnRequestsSigned` 기본값은 `true`, `singleSignOnServiceBinding` 기본값은 `REDIRECT` 다.

## 레지스트리 — 요청마다 등록을 고른다

`saml2/.../registration/RelyingPartyRegistrationRepository.java`

```java
public interface RelyingPartyRegistrationRepository {
    @Nullable RelyingPartyRegistration findByRegistrationId(String registrationId);

    // SAML Response의 Issuer(=AP entityId)만으로 찾아야 할 때 (IdP-initiated 등)
    default @Nullable RelyingPartyRegistration findUniqueByAssertingPartyEntityId(String entityId) {
        return findByRegistrationId(entityId);
    }
}
```

두 번째 메서드는 §04에서 URL에 `registrationId` 가 없을 때(IdP가 먼저 보낸 응답) 쓰인다. 구현체:

```
RelyingPartyRegistrationRepository
├── InMemoryRelyingPartyRegistrationRepository   가장 흔함. 부팅 시 등록들을 Map에 보관
│      └── (IterableRelyingPartyRegistrationRepository 구현 — 순회로 entityId 매칭)
└── CachingRelyingPartyRegistrationRepository     원본 조회를 Cache로 감쌈 (원격 메타데이터 등)
```

`InMemoryRelyingPartyRegistrationRepository` 는 `IterableRelyingPartyRegistrationRepository` 도 구현하므로, `findUniqueByAssertingPartyEntityId` 가 전체를 돌며 AP entityId가 유일하게 매칭되는 등록을 찾을 수 있다.

## IdP 메타데이터로부터 설정 만들기

IdP의 엔드포인트·인증서를 손으로 적는 대신, IdP가 게시한 메타데이터 XML(`<EntityDescriptor>`/`<IDPSSODescriptor>`)을 파싱해 빌더를 채울 수 있다.

`saml2/.../registration/RelyingPartyRegistrations.java`

```java
RelyingPartyRegistration reg = RelyingPartyRegistrations
    .fromMetadataLocation("https://idp.example/metadata")  // classpath:/file:/https: 모두 가능
    .registrationId("my-id")
    .build();
```

내부적으로는 OpenSAML로 `<EntityDescriptor>` 들을 읽고, `IDPSSODescriptor` 가 있는 것만 골라 AP 메타데이터로 변환한다.

```java
for (EntityDescriptor descriptor : OpenSamlMetadataUtils.descriptors(source)) {
    if (descriptor.getIDPSSODescriptor(SAMLConstants.SAML20P_NS) != null) {
        OpenSamlAssertingPartyDetails ap =
            OpenSamlAssertingPartyDetails.withEntityDescriptor(descriptor).build();
        builders.add(RelyingPartyRegistration.withAssertingPartyMetadata(ap));
    }
}
```

주의: IdP 메타데이터에는 **AP 정보만** 들어 있으므로, RP 쪽 서명 키 등은 빌더에서 따로 채워야 한다. 메타데이터를 더 동적으로(엔티티ID로 지연 조회) 다루려면 `AssertingPartyMetadataRepository`(6.4+, `JdbcAssertingPartyMetadataRepository` / `BaseOpenSamlAssertingPartyMetadataRepository`)를 쓴다.

## 동작 흐름 — 부팅과 요청 시점의 역할 분담

```
[부팅 시]
 application.yml / @Bean
   └─ RelyingPartyRegistration(들) 빌드  ──▶ InMemory...Repository 에 적재

[매 요청 시]   (§03·§04·§05 공통 전처리)
 HttpServletRequest
   └─ RelyingPartyRegistrationResolver.resolve(request, registrationId)
        └─ repository.findByRegistrationId(...)        설정 조회
        └─ uriResolver.resolve(entityId/acsLocation)   플레이스홀더 → 실제 URL
        └─ registration.mutate()...build()             요청별 확정 등록본
```

즉 레지스트리는 "정적 설정 보관소", 리졸버는 "요청 컨텍스트(스킴/호스트/포트)를 입혀 확정본을 만드는 어댑터"다. 토큰/필터는 항상 **플레이스홀더가 치환된** 등록본을 받는다(`Saml2AuthenticationToken` JavaDoc도 이 점을 명시).

## 설계 포인트 / 확장점

- **빌더 + 불변 + `mutate()`**: 설정을 안전하게 복제·부분수정. 요청별 URL 치환을 부작용 없이 처리.
- **AP를 인터페이스로 분리**(`AssertingPartyMetadata`): IdP 메타데이터 출처를 DB/원격으로 교체 가능.
- **레지스트리 교체**: `RelyingPartyRegistrationRepository` 만 구현하면 멀티테넌트·DB 기반 동적 등록도 가능. `Caching...` 으로 원격 조회를 감쌀 수 있다.
- **메타데이터 우선 설정**: 손 설정 대신 `RelyingPartyRegistrations.fromMetadataLocation` 로 휴먼에러를 줄인다.

## 정리

`RelyingPartyRegistration` 은 RP/AP 한 쌍의 모든 SAML 설정을 담은 불변 값 타입이고, 키 용도와 필수 필드를 생성 시점에 검증한다. `RelyingPartyRegistrationRepository` 는 이를 `registrationId` 또는 AP entityId로 조회하며, 요청 시점에는 리졸버가 플레이스홀더를 실제 URL로 치환한 확정본을 만들어 준다. 이 확정본을 들고 다음 장에서 실제 로그인을 시작한다.
