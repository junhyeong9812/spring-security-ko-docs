# webauthn 모듈 — WebAuthn / 패스키(Passkey)

> Spring Security 7.1.1-SNAPSHOT 기준 해설. 소스 위치: `webauthn/src/main/java/org/springframework/security/web/webauthn/`

## 한 줄 정의

비밀번호 대신 인증기(스마트폰 지문, 윈도우 Hello, 보안키 등)가 만든 공개키 자격증명(패스키)으로 사용자를 등록하고 로그인시키는, W3C WebAuthn Level 3 Relying Party(검증 서버) 구현 모듈이다.

## 이 모듈이 푸는 문제

비밀번호는 피싱·재사용·유출에 취약하다. WebAuthn은 인증기 안에 갇힌 개인키로 서버가 보낸 챌린지(challenge)에 서명하게 하여, 서버는 미리 등록해 둔 공개키로 그 서명만 검증한다. 공유 비밀이 네트워크를 오가지 않으므로 피싱·재사용이 원천 차단된다.

이 모듈은 그 "Relying Party" 측을 담당한다. 구체적으로 다음 4가지 서버 책임을 HTTP 엔드포인트와 필터로 엮어 제공한다.

- **등록 옵션 발급**: 새 패스키를 만들 때 브라우저 `navigator.credentials.create()`에 넘길 `PublicKeyCredentialCreationOptions`(rp 정보, 챌린지, 허용 알고리즘 등)를 생성.
- **등록 검증·저장**: 인증기가 만든 attestation 응답을 검증하고 `CredentialRecord`(공개키 + 메타데이터)로 저장.
- **인증 옵션 발급**: 로그인 시 `navigator.credentials.get()`에 넘길 `PublicKeyCredentialRequestOptions`(챌린지, 허용 자격증명)를 생성.
- **인증(assertion) 검증**: 인증기가 챌린지에 서명한 응답을 저장된 공개키로 검증하고 Spring Security `Authentication`을 발급.

암호학적 검증(서명/attestation 파싱)은 직접 구현하지 않고 외부 라이브러리 **webauthn4j**에 위임한다. 이 모듈은 그 위에 Spring Security의 도메인 모델·필터·저장소 추상화·직렬화를 입힌다.

## 의존 / 연관 관계

- **webauthn4j** (`com.webauthn4j:*`): attestation/assertion의 실제 암호 검증 엔진. `Webauthn4JRelyingPartyOperations`만 이 라이브러리를 직접 참조한다.
- **spring-security-web**: `AbstractAuthenticationProcessingFilter`, `OncePerRequestFilter`, `RequestMatcher`, `SecurityContextRepository` 등 웹 인증 기반 위에 올라탄다. 패키지가 `org.springframework.security.web.webauthn`인 이유다. → [../web/README.md](../web/README.md)
- **spring-security-core**: `AuthenticationProvider`, `AuthorizationManager`, `UserDetailsService`, `Authentication`. → [../core/README.md](../core/README.md)
- **spring-web / Jackson(`tools.jackson`)**: 요청·응답 본문을 WebAuthn JSON 포맷으로 직렬화/역직렬화.
- **DSL/설정**: 실제 필터 조립(`WebAuthnConfigurer`)은 이 모듈이 아니라 `config` 모듈에 있다. 이 모듈은 "부품"만 제공한다. → [../config/README.md](../config/README.md)

## 하위 챕터

1. [01-개념과-전체-구조.md](01-개념과-전체-구조.md) — WebAuthn/패스키 개념, Relying Party 4대 연산, 두 개의 의식(ceremony), 모듈 전체 조감도.
2. [02-api-도메인-모델.md](02-api-도메인-모델.md) — `api` 패키지. `PublicKeyCredential`, 두 종류의 Options, `CredentialRecord`, `Bytes`와 enum들.
3. [03-등록-흐름.md](03-등록-흐름.md) — `registration` 패키지. 옵션 발급 필터 → 브라우저 → 등록 검증 필터 → 저장.
4. [04-인증-흐름.md](04-인증-흐름.md) — `authentication` 패키지. 옵션 발급 → assertion 제출 → `WebAuthnAuthenticationProvider` → `Authentication`.
5. [05-relyingparty-연산과-저장소.md](05-relyingparty-연산과-저장소.md) — `management` 패키지. `WebAuthnRelyingPartyOperations`와 webauthn4j 위임, 두 저장소, 소유권 인가.
6. [06-직렬화-jackson.md](06-직렬화-jackson.md) — `jackson` 패키지. WebAuthn JSON 포맷과 Mixin 직렬화 전략, `aot` 힌트.

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.web.webauthn
│
├── api ──────────────  WebAuthn 데이터 모델 (스펙 1:1 매핑, 순수 값 객체)
│     ├── PublicKeyCredential<R>          클라이언트가 보낸 자격증명 봉투
│     ├── PublicKeyCredentialCreationOptions   등록 옵션 (create)
│     ├── PublicKeyCredentialRequestOptions    인증 옵션 (get)
│     ├── AuthenticatorAttestationResponse     등록 응답(attestation)
│     ├── AuthenticatorAssertionResponse       인증 응답(서명)
│     ├── CredentialRecord / ImmutableCredentialRecord  저장되는 공개키 레코드
│     ├── PublicKeyCredentialUserEntity        WebAuthn 사용자 식별자
│     ├── Bytes                            byte[] 래퍼 + Base64URL + 랜덤
│     └── enums: COSEAlgorithmIdentifier, UserVerificationRequirement,
│                ResidentKeyRequirement, AttestationConveyancePreference ...
│
├── management ───────  Relying Party "두뇌" + 영속화
│     ├── WebAuthnRelyingPartyOperations        4대 연산 인터페이스
│     ├── Webauthn4JRelyingPartyOperations      └ webauthn4j 기반 구현
│     ├── PublicKeyCredentialUserEntityRepository  사용자 저장소 (Map/Jdbc)
│     ├── UserCredentialRepository                 자격증명 저장소 (Map/Jdbc)
│     ├── CredentialRecordOwnerAuthorizationManager  자격증명 소유권 인가
│     └── *Request DTO (RelyingParty*Request, *OptionsRequest)
│
├── registration ─────  등록(create) HTTP 진입점
│     ├── PublicKeyCredentialCreationOptionsFilter   POST /webauthn/register/options
│     ├── WebAuthnRegistrationFilter                 POST /webauthn/register, DELETE .../{id}
│     ├── PublicKeyCredentialCreationOptionsRepository (+HttpSession 구현)
│     └── DefaultWebAuthnRegistrationPageGeneratingFilter  기본 등록 UI
│
├── authentication ───  인증(get) HTTP 진입점
│     ├── PublicKeyCredentialRequestOptionsFilter    POST /webauthn/authenticate/options
│     ├── WebAuthnAuthenticationFilter               POST /login/webauthn
│     ├── WebAuthnAuthenticationProvider             서명검증 → UserDetails → Authentication
│     ├── WebAuthnAuthenticationRequestToken / WebAuthnAuthentication
│     └── PublicKeyCredentialRequestOptionsRepository (+HttpSession 구현)
│
├── jackson ──────────  WebAuthn JSON 직렬화 (Mixin + 커스텀 Serializer)
│     └── WebauthnJacksonModule / WebauthnJackson2Module
│
└── aot ──────────────  GraalVM 네이티브 이미지용 리플렉션 힌트
```
