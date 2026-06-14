# 02. api 도메인 모델

> 소스: `webauthn/src/main/java/org/springframework/security/web/webauthn/api/`

## 무엇을 / 왜

`api` 패키지는 이 모듈 전체가 주고받는 **데이터 모델**이다. 거의 모든 타입이 W3C WebAuthn Level 3 스펙의 딕셔너리/인터페이스와 1:1로 대응한다. 동작 로직은 거의 없고, 빌더로 만들어 불변(immutable)으로 들고 다니는 값 객체들이다. 이 챕터를 먼저 잡아두면 등록/인증 흐름에서 어떤 데이터가 오가는지 바로 읽힌다.

크게 네 묶음으로 나뉜다. (1) 클라이언트가 보내는 **자격증명 봉투**, (2) 서버가 발급하는 **옵션**, (3) 서버가 저장하는 **레코드**, (4) 공통 토대인 `Bytes`와 enum들.

## 핵심 타입

```
                       AuthenticatorResponse (마커)
                         ▲                 ▲
   AuthenticatorAttestationResponse   AuthenticatorAssertionResponse
     (등록: attestationObject,           (인증: authenticatorData,
      clientDataJSON, transports)         clientDataJSON, signature, userHandle)
                         ▲                 ▲
                         └── PublicKeyCredential<R> ──┘
                              (id, rawId, type, response:R,
                               authenticatorAttachment, clientExtensionResults)

  서버가 발급하는 옵션                       서버가 저장하는 레코드
   PublicKeyCredentialCreationOptions        CredentialRecord (interface)
   PublicKeyCredentialRequestOptions          └ ImmutableCredentialRecord

  사용자 식별                                 토대
   PublicKeyCredentialUserEntity (interface)   Bytes  (byte[] 래퍼)
    └ ImmutablePublicKeyCredentialUserEntity   enum 다수
```

### PublicKeyCredential&lt;R&gt; — 클라이언트가 보내는 봉투

브라우저 JS가 `create()`/`get()` 결과로 받아 서버로 POST하는 최상위 객체다. 제네릭 `R`이 응답 종류를 가른다.

- `PublicKeyCredential<AuthenticatorAttestationResponse>` → 등록 요청 본문
- `PublicKeyCredential<AuthenticatorAssertionResponse>` → 인증 요청 본문

```java
public final class PublicKeyCredential<R extends AuthenticatorResponse> implements Serializable {
    private final String id;        // rawId의 Base64URL 문자열
    private final Bytes rawId;      // 자격증명 id (credential id)의 원본 바이트
    private final R response;       // attestation 또는 assertion
    private final @Nullable AuthenticatorAttachment authenticatorAttachment; // platform/cross-platform
    private final @Nullable AuthenticationExtensionsClientOutputs clientExtensionResults;
}
```

`rawId`가 곧 등록된 자격증명을 식별하는 키다. 인증 시 서버는 이 `rawId`로 저장소에서 `CredentialRecord`를 찾아 공개키를 꺼낸다.

### 두 종류의 AuthenticatorResponse

`AuthenticatorResponse`는 마커 인터페이스이고, 의식에 따라 두 구현이 갈린다.

- **AuthenticatorAttestationResponse(등록)**: `attestationObject`(인증기가 만든 새 공개키 + 메타데이터를 담은 CBOR), `clientDataJSON`(타입·챌린지·origin), `transports`(usb/internal/hybrid…). 서버는 이를 검증해 공개키를 추출한다.
- **AuthenticatorAssertionResponse(인증)**: `authenticatorData`, `clientDataJSON`, `signature`(챌린지에 대한 개인키 서명), `userHandle`(사용자 식별자). 서버는 저장된 공개키로 `signature`를 검증한다.

### 두 종류의 Options — 서버가 발급

`PublicKeyCredentialCreationOptions`(등록)와 `PublicKeyCredentialRequestOptions`(인증)는 서버가 만들어 브라우저로 내려보내는 파라미터다. 둘 다 빌더 + `customize(Consumer)` 패턴을 쓴다(아래 `CreationOptions` 발췌).

```java
public final class PublicKeyCredentialCreationOptions implements Serializable {
    private final PublicKeyCredentialRpEntity rp;          // 우리 서비스(도메인) 정보
    private final PublicKeyCredentialUserEntity user;       // 등록 대상 사용자
    private final Bytes challenge;                          // 일회용 난수 (핵심!)
    private final List<PublicKeyCredentialParameters> pubKeyCredParams; // 허용 알고리즘 (EdDSA/ES256/RS256)
    private final List<PublicKeyCredentialDescriptor> excludeCredentials; // 중복 등록 방지
    private final AuthenticatorSelectionCriteria authenticatorSelection;  // residentKey, userVerification
    private final @Nullable AttestationConveyancePreference attestation;
    private final @Nullable AuthenticationExtensionsClientInputs extensions;
}
```

여기서 **challenge가 안전성의 핵심**이다. 서버가 매 요청 새 난수를 발급하고 세션에 보관했다가, 둘째 요청에서 인증기가 서명/반영한 챌린지가 그와 일치하는지 확인한다. `RequestOptions`는 비슷하되 `allowCredentials`(이 사용자가 쓸 수 있는 자격증명 목록), `rpId`, `userVerification` 중심이다.

### CredentialRecord — 서버가 저장하는 것

등록이 성공하면 서버는 이후 인증에 필요한 모든 것을 `CredentialRecord`로 영속화한다. 이 인터페이스가 곧 "패스키 1개"의 서버 측 표현이다.

```java
public interface CredentialRecord {
    PublicKeyCredentialType getCredentialType();
    Bytes getCredentialId();          // = PublicKeyCredential.rawId, 조회 키
    PublicKeyCose getPublicKey();      // 검증에 쓸 공개키(COSE)
    long getSignatureCount();          // 복제 탐지용 서명 카운터
    boolean isUvInitialized();
    Set<AuthenticatorTransport> getTransports();
    boolean isBackupEligible(); boolean isBackupState();
    Bytes getUserEntityUserId();       // 소유자(PublicKeyCredentialUserEntity.id) 참조
    @Nullable Bytes getAttestationObject();  // 인증 시 공개키 재구성에 사용
    String getLabel();                 // 사용자가 붙인 별칭 (예: "1password")
    Instant getLastUsed(); Instant getCreated();
}
```

`signatureCount`는 보안상 중요한 필드다. 인증기는 서명할 때마다 카운터를 올리고, 서버는 인증 성공 시마다 이 값을 갱신·저장한다(05장 `authenticate()` 참고). 카운터가 거꾸로 가면 복제된 인증기를 의심할 단서가 된다. 불변 구현은 `ImmutableCredentialRecord`이며 `fromCredentialRecord(...)`로 일부 필드만 바꿔 다시 빌드할 수 있다.

### PublicKeyCredentialUserEntity — WebAuthn의 사용자

WebAuthn은 username 문자열이 아니라 **user handle**(불투명 바이트, 최대 64바이트)로 사용자를 식별한다.

```java
public interface PublicKeyCredentialUserEntity extends Serializable {
    String getName();          // 사람이 읽는 식별자(보통 username)
    Bytes getId();             // user handle (불투명 바이트)
    @Nullable String getDisplayName();
}
```

`getId()`가 `CredentialRecord.getUserEntityUserId()`와 연결된다. 즉 "사용자 ↔ 그 사용자의 여러 패스키"는 이 handle을 통해 묶인다.

### Bytes — 토대가 되는 값 객체

WebAuthn JSON은 모든 바이너리를 Base64URL 문자열로 주고받는다. `Bytes`는 `byte[]`를 감싸 그 변환과 안전한 동치 비교를 책임진다.

```java
public final class Bytes implements Serializable {
    private final byte[] bytes;
    public String toBase64UrlString() { ... }            // 직렬화용
    public static Bytes fromBase64(String s) { ... }      // 역직렬화용
    public static Bytes random() {                        // 챌린지/user handle 생성
        byte[] bytes = new byte[32];
        RANDOM.nextBytes(bytes);                          // SecureRandom, 32바이트=256비트
        return new Bytes(bytes);
    }
    // equals/hashCode는 Base64URL 문자열 기준으로 비교
}
```

`Bytes.random()`이 챌린지와 user handle을 만든다(32바이트 SecureRandom). `equals`를 Base64URL 문자열로 비교하므로 저장소에서 자격증명 id를 키로 찾을 때 일관되게 동작한다.

### enum과 보조 타입

스펙의 열거값들이 그대로 enum으로 들어온다. 각각 WebAuthn 정책의 손잡이다.

- `COSEAlgorithmIdentifier`: 서명 알고리즘(EdDSA, ES256, RS256 …).
- `UserVerificationRequirement`: `REQUIRED`/`PREFERRED`/`DISCOURAGED` — 생체/PIN 등 사용자 검증 강제 수준.
- `ResidentKeyRequirement`: 인증기에 자격증명을 상주시킬지(=진짜 "패스키", 사용자명 없는 로그인 가능 여부).
- `AttestationConveyancePreference`: attestation 정보를 얼마나 받을지(`NONE`/`INDIRECT`/`DIRECT`).
- `AuthenticatorTransport`: `USB`/`NFC`/`BLE`/`INTERNAL`/`HYBRID`.
- `AuthenticatorAttachment`: `PLATFORM`(기기 내장)/`CROSS_PLATFORM`(외장 보안키).
- 확장: `AuthenticationExtensionsClientInputs/Outputs`, `CredProtectAuthenticationExtensionsClientInput`, `CredentialPropertiesOutput`(credProps) 등.

## 설계 포인트

- **스펙 1:1 + 불변 + 빌더**: 타입 이름과 필드가 WebAuthn 스펙 용어를 그대로 따르므로, 스펙 문서를 옆에 두고 읽으면 그대로 매핑된다. 모든 옵션 객체는 빌더로만 생성되고 `customize(Consumer)` 훅으로 기본값을 덮어쓴다.
- **인터페이스 + Immutable 구현 쌍**: `CredentialRecord`/`ImmutableCredentialRecord`, `PublicKeyCredentialUserEntity`/`ImmutablePublicKeyCredentialUserEntity`처럼 인터페이스와 불변 구현을 분리해, 저장소가 자신만의 구현을 제공할 여지를 둔다.
- **제네릭으로 의식 구분**: `PublicKeyCredential<R>`의 타입 파라미터 하나로 등록/인증 봉투를 같은 클래스로 표현하면서도 컴파일 타임에 구분한다.
- **직렬화는 위임**: 이 값 객체들에는 Jackson 애너테이션이 거의 없다. WebAuthn JSON 매핑은 `jackson` 패키지의 Mixin이 외부에서 입힌다(06장). 도메인 모델을 직렬화 기술과 분리하기 위함이다.

## 정리

`api` 패키지는 동작이 아니라 데이터다. 클라이언트가 보내는 `PublicKeyCredential<R>` 봉투, 서버가 발급하는 두 종류의 Options, 서버가 저장하는 `CredentialRecord`, 사용자를 가리키는 `PublicKeyCredentialUserEntity`, 그리고 이 모두의 바이너리를 다루는 `Bytes`가 중심축이다. 챌린지(`Bytes.random()`)와 자격증명 id(`rawId`/`credentialId`), 서명 카운터가 보안의 핵심 데이터이며, 이어지는 등록/인증 흐름은 이 객체들을 만들고 검증하고 저장하는 과정이다.
