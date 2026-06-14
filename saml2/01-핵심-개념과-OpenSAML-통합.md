# 01. 핵심 개념과 OpenSAML 통합

## 무엇을 / 왜

SAML 2.0을 코드로 읽기 전에 두 가지를 먼저 정리해야 한다. 첫째는 **용어**(누가 누구인가), 둘째는 이 모듈의 가장 독특한 설계인 **OpenSAML 의존성을 격리하는 소스셋 분리**다. 이 둘을 알아야 나머지 챕터의 클래스 이름이 자연스럽게 읽힌다.

### 등장인물

```
   ┌─────────────────────┐   1. AuthnRequest    ┌─────────────────────┐
   │  Relying Party (RP) │ ───────────────────▶ │ Asserting Party(AP) │
   │  = Service Provider │                      │ = Identity Provider │
   │  = 우리 애플리케이션  │ ◀─────────────────── │  = IdP (Okta 등)     │
   └─────────────────────┘   2. Response         └─────────────────────┘
                                (+ Assertion)
```

- **Relying Party(RP) / Service Provider(SP)**: 우리 앱. IdP의 주장(assertion)에 "기대어(rely)" 사용자를 신뢰한다.
- **Asserting Party(AP) / Identity Provider(IdP)**: 사용자를 실제로 인증하고 "이 사람은 누구다"라고 서명된 **Assertion** 을 발급하는 쪽.
- **AuthnRequest**: RP가 "이 사용자를 인증해 달라"고 IdP에 보내는 요청 XML.
- **Response / Assertion**: IdP가 돌려주는 응답. `<Response>` 안에 한 개 이상의 `<Assertion>` 이 들어 있고, 그 안의 `<Subject><NameID>` 가 사용자 식별자다.
- **바인딩(Binding)**: XML을 HTTP로 실어 나르는 방식. `HTTP-Redirect`(쿼리 파라미터, deflate+base64) 와 `HTTP-POST`(폼 자동 제출, base64) 두 가지를 지원한다 (`Saml2MessageBinding`).

## 핵심 타입

### OpenSAML 격리: `Base*` ↔ `OpenSaml5*`

XML 직렬화·서명·암호화의 실제 작업은 전부 **OpenSAML 5** 라이브러리가 한다. Spring Security는 OpenSAML API에 직접 묶이지 않도록, 모듈을 두 소스셋으로 쪼갰다.

```
src/main/java                         src/opensaml5Main/java
─────────────────────────             ──────────────────────────────
OpenSamlOperations (interface)  ◀──── OpenSaml5Template (구현)
BaseOpenSamlAuthenticationProvider ◀─ OpenSaml5AuthenticationProvider (위임 래퍼, public)
BaseOpenSamlAuthenticationRequestResolver ◀─ OpenSaml5AuthenticationRequestResolver
BaseOpenSamlMetadataResolver        ◀─ OpenSaml5MetadataResolver
...                                   ...
```

- `src/main/java` 에는 **버전 중립 로직** 과 공개 값 타입(`RelyingPartyRegistration`, `Saml2X509Credential`, 필터 등)만 둔다. 여기서 OpenSAML XML 작업이 필요하면 패키지 전용 인터페이스 `OpenSamlOperations` 를 통해서만 호출한다.
- `src/opensaml5Main/java` 에는 `OpenSamlOperations` 를 구현한 `OpenSaml5Template` 과, `Base*` 를 감싸 공개 API로 만드는 `OpenSaml5*` 구체 클래스가 산다.

즉 애플리케이션이 빈으로 등록하는 것은 항상 `OpenSaml5AuthenticationProvider` 같은 `OpenSaml5*` 클래스이고, 실제 검증 알고리즘은 `src/main/java` 의 `Base*` 가 갖는다. 과거 OpenSAML 4 소스셋이 함께 존재했으나 7.x에는 `opensaml5Main` 만 남아 있다.

### `OpenSamlOperations` — XML 작업의 단일 창구

`saml2/saml2-service-provider/src/main/java/org/springframework/security/saml2/provider/service/authentication/OpenSamlOperations.java`

```java
interface OpenSamlOperations {
    <T extends XMLObject> T deserialize(String serialized);   // XML → 객체
    SerializationConfigurer<?> serialize(XMLObject object);   // 객체 → XML
    SignatureConfigurer<?>   withSigningKeys(Collection<Saml2X509Credential> creds);   // 서명
    VerificationConfigurer   withVerificationKeys(Collection<Saml2X509Credential> creds); // 서명검증
    DecryptionConfigurer     withDecryptionKeys(Collection<Saml2X509Credential> creds);   // 복호화
}
```

`withSigningKeys(...).algorithms(...).sign(authnRequest)` 처럼 fluent 하게 쓰며, 각 SAML 패키지(`authentication`, `web`, `metadata`, `logout`)가 자기 패키지 안에 같은 이름의 `OpenSamlOperations` 를 패키지 전용으로 두고 있다. 이 인터페이스 덕분에 `Base*` 로직은 OpenSAML 타입을 최소한으로만 만진다.

### `Saml2X509Credential` — 키에 "용도"를 못 박는다

`saml2/.../core/Saml2X509Credential.java`

SAML은 한 인증서를 서명용/검증용/암호화용/복호화용으로 구분해 쓴다. 이 클래스는 인증서(+선택적 개인키)에 **용도 enum** 을 붙여, 잘못된 자리에 잘못된 키가 들어가는 것을 생성 시점에 막는다.

```java
public enum Saml2X509CredentialType { VERIFICATION, ENCRYPTION, SIGNING, DECRYPTION }

// 팩토리: 용도가 이름에 드러난다
Saml2X509Credential.signing(privateKey, cert);       // 우리 AuthnRequest 서명
Saml2X509Credential.verification(cert);              // IdP 응답 서명 검증
Saml2X509Credential.decryption(privateKey, cert);    // 암호화된 assertion 복호화
Saml2X509Credential.encryption(cert);                // IdP에게 보낼 것 암호화
```

`RelyingPartyRegistration` 의 생성자는 `signingX509Credentials` 컬렉션의 모든 원소에 대해 `c.isSigningCredential()` 을 `Assert` 로 검사한다(§02 참고). 개인키 없는 인증서로 서명용을 만들면 생성자에서 바로 실패한다.

### `OpenSamlInitializationService` — 1회성 부트스트랩

`saml2/.../core/OpenSamlInitializationService.java`

OpenSAML은 사용 전에 `InitializationService.initialize()` 로 XML 빌더/마샬러 레지스트리를 채워야 한다. 이 서비스는 그것을 **멱등하게** 감싼다.

```java
private static final AtomicBoolean initialized = new AtomicBoolean(false);

public static boolean initialize() {                 // 여러 번 불러도 안전
    if (initialized.compareAndSet(false, true)) { ... return true; }
    return false;                                    // 이미 됐으면 조용히 false
}
```

OpenSAML을 쓰는 모든 컴포넌트(`BaseOpenSamlAuthenticationProvider`, `BaseOpenSamlAuthenticationRequestResolver`, `BaseOpenSamlAuthenticationTokenConverter` 등)는 `static { OpenSamlInitializationService.initialize(); }` 블록을 갖는다. 그래서 어느 컴포넌트가 먼저 로드되든 OpenSAML은 정확히 한 번 초기화된다. 레지스트리 기본값을 바꾸고 싶으면 `requireInitialize(Consumer)` 를 애플리케이션에서 가장 먼저 호출해야 한다(두 번째 호출은 예외).

### 에러 모델: `Saml2Error` / `Saml2ResponseValidatorResult`

SAML 검증은 "여러 군데가 동시에 틀릴 수 있다". 그래서 단일 예외 대신 에러를 **누적**한다.

```java
Saml2ResponseValidatorResult result = Saml2ResponseValidatorResult.success();
result = result.concat(new Saml2Error(Saml2ErrorCodes.INVALID_ISSUER, "..."));
result = result.concat(validateInResponseTo(...));   // 계속 이어붙임
// 마지막에 result.getErrors() 로 한꺼번에 보고
```

`Saml2ErrorCodes` 에는 `INVALID_SIGNATURE`, `INVALID_ISSUER`, `INVALID_DESTINATION`, `INVALID_IN_RESPONSE_TO`, `SUBJECT_NOT_FOUND`, `MALFORMED_RESPONSE_DATA` 등 검증 단계마다의 코드가 정의돼 있다. 이 누적 결과 패턴은 §04의 인증 파이프라인 전체를 관통한다.

## 동작 흐름 — 큰 그림에서의 두 갈래

이 모듈이 하는 일은 결국 두 개의 HTTP 왕복으로 요약된다.

```
[로그인 시작]                                  [로그인 완료]
브라우저 ── GET /saml2/authenticate/{id} ──▶ SP
   SP가 <AuthnRequest> 생성·서명·인코딩
브라우저 ◀── 302 redirect 또는 POST 폼 ─────  SP        (§03)
브라우저 ── AuthnRequest ───────────────────▶ IdP
   ...IdP에서 사용자 인증...
브라우저 ◀── <Response>(+Assertion) ────────  IdP
브라우저 ── POST /login/saml2/sso/{id} ─────▶ SP
   SP가 Response 디코딩·검증 → Authentication  (§04)
브라우저 ◀── 인증 성공, 보호 자원으로 ─────────  SP
```

여기에 부가적으로 SP 메타데이터 노출과 단일 로그아웃이 붙는다(§05). 이 챕터의 개념 — RP/AP, 바인딩, 용도별 키, OpenSAML 격리 — 을 들고 §02부터 따라가면 된다.

## 설계 포인트

- **포트/어댑터(헥사고날) 변형**: `OpenSamlOperations` 가 포트, `OpenSaml5Template` 이 어댑터. OpenSAML major 버전이 바뀌어도 `src/main/java` 로직은 손대지 않고 새 소스셋만 추가하면 된다.
- **값 타입 불변성**: `Saml2X509Credential`, `RelyingPartyRegistration` 등은 모두 불변 + 빌더. 세션에 저장돼 직렬화되므로 `Serializable` 이다.
- **방어적 생성자**: 키 용도, 필수 필드를 생성 시점에 `Assert` 로 검증해 잘못된 설정을 부팅 단계에서 잡는다.

## 정리

이 모듈은 SAML 2.0의 XML 노동을 OpenSAML 5에 위임하되, `OpenSamlOperations` 포트와 `Base*`/`OpenSaml5*` 소스셋 분리로 그 의존성을 격리한다. 핵심 값 타입은 용도가 박힌 `Saml2X509Credential`, 멱등 부트스트랩 `OpenSamlInitializationService`, 그리고 에러를 누적하는 `Saml2ResponseValidatorResult` 다. 다음 장에서는 "누구와 SAML을 하는가"를 표현하는 `RelyingPartyRegistration` 으로 들어간다.
