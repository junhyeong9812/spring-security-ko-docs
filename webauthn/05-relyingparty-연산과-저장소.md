# 05. Relying Party 연산과 저장소 (management)

> 소스: `webauthn/src/main/java/org/springframework/security/web/webauthn/management/`

## 무엇을 / 왜

`management` 패키지는 이 모듈의 **두뇌이자 영속화 계층**이다. 앞 두 챕터의 필터들은 HTTP 껍데기일 뿐, 실제 "WebAuthn 규칙대로 옵션을 만들고 검증하고 저장하는 일"은 모두 여기서 일어난다. 중심에는 4대 연산 인터페이스 `WebAuthnRelyingPartyOperations`가 있고, 그 유일 구현 `Webauthn4JRelyingPartyOperations`가 암호 검증을 외부 라이브러리 webauthn4j에 위임한다. 그리고 두 종류의 저장소(사용자/자격증명)와 소유권 인가 매니저가 함께 산다.

## 핵심 타입

```
management 패키지
 ├ WebAuthnRelyingPartyOperations  ── 4대 연산 인터페이스
 │    └ Webauthn4JRelyingPartyOperations  ── webauthn4j 기반 유일 구현
 │
 ├ 저장소 (영속화)
 │    ├ PublicKeyCredentialUserEntityRepository  사용자(handle)  ─┬ Map* (메모리)
 │    │                                                          └ Jdbc*
 │    └ UserCredentialRepository                  CredentialRecord ─┬ Map*
 │                                                                  └ Jdbc*
 │
 ├ CredentialRecordOwnerAuthorizationManager  ── 자격증명 소유권 인가(삭제 보호)
 │
 └ 요청 DTO
      ├ PublicKeyCredentialCreationOptionsRequest / RequestOptionsRequest  (Authentication 운반)
      ├ RelyingPartyRegistrationRequest  (CreationOptions + RelyingPartyPublicKey)
      ├ RelyingPartyPublicKey            (credential + label)
      └ RelyingPartyAuthenticationRequest (RequestOptions + PublicKeyCredential<Assertion>)
```

### WebAuthnRelyingPartyOperations — 4대 연산

```java
public interface WebAuthnRelyingPartyOperations {
    PublicKeyCredentialCreationOptions createPublicKeyCredentialCreationOptions(PublicKeyCredentialCreationOptionsRequest r);
    CredentialRecord registerCredential(RelyingPartyRegistrationRequest r);
    PublicKeyCredentialRequestOptions createCredentialRequestOptions(PublicKeyCredentialRequestOptionsRequest r);
    PublicKeyCredentialUserEntity authenticate(RelyingPartyAuthenticationRequest r);
}
```

01장에서 본 두 의식의 4개 박스가 그대로 메서드가 됐다. 필터들은 이 인터페이스에만 의존하므로 webauthn4j의 존재를 모른다.

## 동작 흐름

### create 옵션 생성 — createPublicKeyCredentialCreationOptions

등록 옵션을 만드는 메서드다. 인증된 사용자를 확인하고, 그 사용자 엔티티를 찾거나 새로 만들며, 기본 정책을 채운다.

```java
public PublicKeyCredentialCreationOptions createPublicKeyCredentialCreationOptions(...request) {
    Authentication authentication = request.getAuthentication();
    if (!this.trustResolver.isAuthenticated(authentication))
        throw new IllegalArgumentException("Authentication must be authenticated");   // 로그인 필수

    var authenticatorSelection = AuthenticatorSelectionCriteria.builder()
        .userVerification(UserVerificationRequirement.PREFERRED)
        .residentKey(ResidentKeyRequirement.REQUIRED)        // 진짜 패스키(사용자명 없는 로그인) 지향
        .build();

    PublicKeyCredentialUserEntity userEntity = findUserEntityOrCreateAndSave(authentication.getName());
    List<CredentialRecord> credentialRecords = this.userCredentials.findByUserId(userEntity.getId());

    return PublicKeyCredentialCreationOptions.builder()
        .attestation(AttestationConveyancePreference.NONE)
        .pubKeyCredParams(EdDSA, ES256, RS256)               // 허용 서명 알고리즘
        .authenticatorSelection(authenticatorSelection)
        .challenge(Bytes.random())                           // ★ 일회용 챌린지(32바이트 SecureRandom)
        .extensions(clientInputs)                            // credProps
        .timeout(Duration.ofMinutes(5))
        .user(userEntity)
        .rp(this.rp)
        .excludeCredentials(credentialDescriptors(credentialRecords)) // 이미 등록된 건 제외(중복 방지)
        .customize(this.customizeCreationOptions)            // 사용자 커스터마이즈 훅
        .build();
}
```

두 가지가 눈여겨볼 점이다. `findUserEntityOrCreateAndSave`는 username에 해당하는 user handle이 없으면 `Bytes.random()`으로 새 handle을 만들어 저장한다 — 즉 첫 등록 시 WebAuthn용 사용자 식별자가 생성된다. `excludeCredentials`에는 이미 등록된 자격증명을 넣어, 같은 인증기로 중복 등록되는 것을 막는다.

### registerCredential — attestation 검증과 공개키 저장

등록의 핵심. 첫 요청의 옵션(챌린지)과 클라이언트가 보낸 attestation을 받아 webauthn4j로 검증하고, 추출한 공개키를 `CredentialRecord`로 저장한다.

```java
public CredentialRecord registerCredential(RelyingPartyRegistrationRequest rpRegistrationRequest) {
    Bytes credentialId = rpRegistrationRequest.getPublicKey().getCredential().getRawId();
    if (this.userCredentials.findByCredentialId(credentialId) != null)
        throw new IllegalArgumentException("Credential ... already exists");          // 중복 방지

    PublicKeyCredentialCreationOptions creationOptions = rpRegistrationRequest.getCreationOptions();
    ...
    // 서버 기대값 구성: 허용 origin + rpId + 발급했던 challenge
    ServerProperty serverProperty = new ServerProperty(toOrigins(), rpId, new DefaultChallenge(base64Challenge));
    boolean userVerificationRequired = ... == UserVerificationRequirement.REQUIRED;

    RegistrationRequest wa4jReq = new RegistrationRequest(attestationObject, clientDataJSON, transports);
    RegistrationParameters params = new RegistrationParameters(serverProperty, pubKeyCredParams,
            userVerificationRequired, /*userPresenceRequired*/ true);
    RegistrationData data = this.webAuthnManager.verify(wa4jReq, params);   // ★ webauthn4j가 attestation 검증

    // 검증 통과 → 공개키(COSE)와 메타데이터를 CredentialRecord로 저장
    COSEKey coseKey = data.getAttestationObject().getAuthenticatorData().getAttestedCredentialData().getCOSEKey();
    byte[] rawCoseKey = cborConverter.writeValueAsBytes(coseKey);
    ImmutableCredentialRecord userCredential = ImmutableCredentialRecord.builder()
        .userEntityUserId(creationOptions.getUser().getId())
        .credentialId(credential.getRawId())
        .publicKey(new ImmutablePublicKeyCose(rawCoseKey))
        .signatureCount(wa4jAuthData.getSignCount())
        .attestationObject(credential.getResponse().getAttestationObject()) // 인증 때 공개키 재구성에 사용
        .label(publicKey.getLabel())
        ... .build();
    this.userCredentials.save(userCredential);
    return userCredential;
}
```

`ServerProperty`에 **서버가 발급했던 challenge**를 넣는 것이 검증의 핵심이다. webauthn4j는 클라이언트가 보낸 `clientDataJSON` 안의 챌린지가 이것과 일치하는지, origin과 rpId가 맞는지, 서명이 유효한지를 한 번에 검증한다(`webAuthnManager.verify`). 통과하면 attestationObject에서 COSE 공개키를 꺼내 저장한다. 주목할 점은 `attestationObject` 자체도 레코드에 저장하는 것 — 인증 시 이 값으로 공개키를 다시 복원한다.

### authenticate — assertion 서명 검증

인증의 핵심. 저장된 레코드의 공개키로 클라이언트 서명을 검증하고, 서명 카운터를 갱신하며, 사용자 엔티티를 돌려준다.

```java
public PublicKeyCredentialUserEntity authenticate(RelyingPartyAuthenticationRequest request) {
    AuthenticatorAssertionResponse assertionResponse = request.getPublicKey().getResponse();
    Bytes keyId = request.getPublicKey().getRawId();
    CredentialRecord credentialRecord = this.userCredentials.findByCredentialId(keyId);  // 공개키 조회
    if (credentialRecord == null) throw new IllegalArgumentException("Unable to find CredentialRecord ...");

    // 저장해 둔 attestationObject로 공개키(COSE) 복원
    AttestationObject wa4jAttestationObject = cborConverter.readValue(
            credentialRecord.getAttestationObject().getBytes(), AttestationObject.class);

    // 서버 기대값: origins + rpId + 발급했던 challenge
    ServerProperty serverProperty = new ServerProperty(toOrigins(), rpId,
            new DefaultChallenge(requestOptions.getChallenge().getBytes()));

    var authenticationRequest = new com.webauthn4j.data.AuthenticationRequest(
            rawId, assertionResponse.getAuthenticatorData().getBytes(),
            assertionResponse.getClientDataJSON().getBytes(), assertionResponse.getSignature().getBytes());
    var wa4jCredentialRecord = new CredentialRecordImpl(wa4jAttestationObject, null, null, transports);
    var params = new AuthenticationParameters(serverProperty, wa4jCredentialRecord, allowCredentials, userVerificationRequired);

    AuthenticationData data = this.webAuthnManager.verify(authenticationRequest, params); // ★ 서명 검증

    // 서명 카운터 갱신 후 저장 (복제 탐지)
    long updatedSignCount = data.getAuthenticatorData().getSignCount();
    ImmutableCredentialRecord updatedRecord = ImmutableCredentialRecord.fromCredentialRecord(credentialRecord)
        .lastUsed(Instant.now()).signatureCount(updatedSignCount).build();
    this.userCredentials.save(updatedRecord);

    PublicKeyCredentialUserEntity userEntity = this.userEntities.findById(credentialRecord.getUserEntityUserId());
    if (userEntity == null) throw new IllegalArgumentException("Unable to find UserEntity ...");
    return userEntity;   // → Provider가 이 name으로 UserDetailsService 조회
}
```

흐름은 명확하다. (1) `rawId`로 저장된 `CredentialRecord`를 찾아 공개키를 복원, (2) `ServerProperty`에 **발급했던 챌린지**를 실어 webauthn4j로 서명 검증, (3) 검증 통과 시 **서명 카운터를 갱신·저장**(인증기 복제 탐지의 근거), (4) 레코드가 가리키는 user handle로 `PublicKeyCredentialUserEntity`를 조회해 반환. 이 반환값의 `getName()`이 04장 Provider에서 `UserDetailsService.loadUserByUsername`의 인자가 된다.

### createCredentialRequestOptions — 인증 옵션

인증 옵션은 비교적 단순하다. 인증된 사용자가 있으면 그 사용자의 자격증명을 `allowCredentials`로 채우고, 없으면 빈 목록으로 둔다.

```java
public PublicKeyCredentialRequestOptions createCredentialRequestOptions(...request) {
    List<CredentialRecord> credentialRecords = findCredentialRecords(request.getAuthentication());
    return PublicKeyCredentialRequestOptions.builder()
        .allowCredentials(credentialDescriptors(credentialRecords))
        .challenge(Bytes.random())                          // 새 챌린지
        .rpId(this.rp.getId())
        .timeout(Duration.ofMinutes(5))
        .userVerification(UserVerificationRequirement.PREFERRED)
        .customize(this.customizeRequestOptions)
        .build();
}
// findCredentialRecords: 미인증이거나 사용자 없으면 emptyList → 사용자명 없는(discoverable) 로그인 허용
```

비인증 상태에서 `allowCredentials`가 비면, 브라우저는 인증기에 상주한 패스키(resident key) 중에서 사용자가 직접 고르게 한다. 등록 옵션에서 `residentKey(REQUIRED)`로 설정했던 것이 여기서 "사용자명 없는 로그인"으로 결실을 맺는다.

## 저장소 두 종류

```
PublicKeyCredentialUserEntityRepository      UserCredentialRepository
  findById(Bytes) / findByUsername(String)     findByCredentialId(Bytes)
  save(userEntity) / delete(Bytes)             findByUserId(Bytes)
                                               save(record) / delete(Bytes)
  구현: Map* (메모리, 테스트/데모)              구현: Map* / Jdbc*
        Jdbc* (운영, 스키마 SQL 동반)
```

두 저장소는 user handle을 통해 연결된다. `UserCredentialRepository.findByUserId(userEntity.getId())`로 "한 사용자의 여러 패스키"를 조회한다. `Jdbc*` 구현은 각각 `user-entities-schema.sql`, `user-credentials-schema.sql`(클래스패스) 테이블 정의를 전제로 하며, 공개키·attestationObject 같은 바이너리를 LOB로 다룬다.

## 소유권 인가 — CredentialRecordOwnerAuthorizationManager

03장의 자격증명 삭제를 보호하는 `AuthorizationManager<Bytes>`다. 대상은 credential id.

```java
public AuthorizationResult authorize(Supplier<Authentication> authentication, Bytes credentialId) {
    AuthorizationResult decision = this.authenticatedAuthorizationManager.authorize(authentication, credentialId);
    if (!decision.isGranted()) return decision;                    // 1) 로그인 안 했으면 거부
    CredentialRecord credential = this.userCredentials.findByCredentialId(credentialId);
    if (credential == null || credential.getUserEntityUserId() == null) return new AuthorizationDecision(false);
    PublicKeyCredentialUserEntity userEntity = this.userEntities.findByUsername(authentication.get().getName());
    if (userEntity == null) return new AuthorizationDecision(false);
    return new AuthorizationDecision(credential.getUserEntityUserId().equals(userEntity.getId())); // 2) 소유자 일치?
}
```

"로그인 했는가 → 그 자격증명이 현재 사용자 소유인가"를 차례로 확인한다. credential id는 추측 불가능하지만, 노출되더라도 남의 패스키를 삭제하지 못하게 하는 심층 방어다(WebAuthn 스펙 §14.6.3의 credential id 프라이버시 권고를 반영).

## 설계 포인트 / 확장점

- **검증 엔진 위임**: 모든 암호 검증을 `WebAuthnManager.verify(...)`에 맡긴다. 기본은 `createNonStrictWebAuthnManager()`(attestation 신뢰 검증을 느슨하게)이며 `setWebAuthnManager`로 엄격 검증으로 교체 가능.
- **챌린지 = ServerProperty**: 등록·인증 모두 "발급했던 challenge + 허용 origins + rpId"를 `ServerProperty`로 묶어 검증 기준으로 삼는다. 재전송·피싱 방지의 핵심.
- **서명 카운터 갱신**: 인증마다 카운터를 저장해 인증기 복제 탐지의 토대를 남긴다.
- **커스터마이즈 훅**: `setCustomizeCreationOptions` / `setCustomizeRequestOptions`로 기본 정책(알고리즘, UV, residentKey, timeout 등)을 덮어쓸 수 있다.
- **저장소 교체**: 메모리(`Map*`)↔JDBC(`Jdbc*`) 또는 사용자 정의 구현으로 교체 가능. 두 저장소 인터페이스가 명확히 분리돼 있다.

## 정리

`management`는 필터 뒤에서 실제 일을 한다. `WebAuthnRelyingPartyOperations`의 4개 연산이 두 의식의 핵심이며, `Webauthn4JRelyingPartyOperations`는 매번 "발급했던 challenge + origins + rpId"를 `ServerProperty`로 묶어 webauthn4j에 검증을 맡긴다. 등록은 attestation을 검증해 COSE 공개키를 `CredentialRecord`로 저장하고, 인증은 그 공개키로 서명을 검증한 뒤 서명 카운터를 갱신하고 사용자 엔티티를 돌려준다. 데이터는 사용자/자격증명 두 저장소에 user handle로 묶여 영속화되고, 삭제는 소유권 인가로 보호된다. 이 객체들이 네트워크를 오갈 때의 JSON 형태는 [06-직렬화-jackson.md](06-직렬화-jackson.md)에서 본다.
