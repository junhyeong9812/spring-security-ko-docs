# saml2 — SAML 2.0 서비스 제공자(Service Provider)

> 버전: Spring Security 7.1.1-SNAPSHOT
> 소스: `saml2/saml2-service-provider/src/main/java` (+ `src/opensaml5Main/java`)

## 한 줄 정의

이 모듈은 우리 애플리케이션을 **SAML 2.0 서비스 제공자(SP, Relying Party)** 로 만들어, 외부 **신원 제공자(IdP, Asserting Party)** 가 발급한 SAML 응답(Response/Assertion)으로 사용자를 로그인시키고 다시 로그아웃시키는 일을 담당한다.

## 이 모듈이 푸는 문제

기업 환경에서는 사용자 비밀번호를 우리 앱이 직접 다루지 않고, 사내 IdP(ADFS, Okta, Keycloak, SimpleSAMLphp 등)에 위임하는 경우가 많다. SAML 2.0은 그 위임을 위한 XML 기반 표준 프로토콜이다. SP가 직접 구현하려면 다음을 모두 처리해야 한다.

- IdP/SP 쌍의 설정(엔드포인트, 인증서, 바인딩)을 어떻게 보관할 것인가 → **RelyingPartyRegistration**
- 로그인을 시작할 때 서명된 `<AuthnRequest>` XML을 만들고 Redirect/POST 바인딩으로 IdP에 보내기
- IdP가 돌려보낸 `<Response>` 를 디코딩·서명검증·복호화·assertion 검증하고 `Authentication` 으로 바꾸기
- 우리 SP의 메타데이터 XML을 IdP에게 노출하기
- 단일 로그아웃(SLO): 우리가 시작하거나 IdP가 시작하는 `<LogoutRequest>`/`<LogoutResponse>` 교환

이 모든 XML 처리는 **OpenSAML 5** 라이브러리에 위임하되, Spring Security는 그것을 필터 체인·`AuthenticationProvider`·빌더 API로 감싸 친숙한 모양으로 제공한다.

## 의존 / 연관 관계

- **OpenSAML 5** (`org.opensaml:*`): XML 직렬화/서명/암호화의 실제 엔진. `src/main/java` 의 `Base*` 클래스는 버전 중립이고, `src/opensaml5Main/java` 의 `OpenSaml5*` 클래스가 OpenSAML 5 구현을 채운다.
- `spring-security-web`: `AbstractAuthenticationProcessingFilter`, `OncePerRequestFilter`, `RedirectStrategy`, `LogoutHandler` 등 필터/로그아웃 인프라.
- `spring-security-core`: `AuthenticationProvider`, `AuthenticationManager`, `Authentication`.
- `config` 모듈의 `Saml2LoginConfigurer` 가 이 모듈의 필터들을 조립해 `http.saml2Login()` / `http.saml2Logout()` DSL로 노출한다. (이 모듈 자체에는 DSL이 없다.)

## 하위 챕터

1. [01-핵심-개념과-OpenSAML-통합.md](01-핵심-개념과-OpenSAML-통합.md) — SP/IdP 용어, 패키지 지도, `Base*` ↔ `OpenSaml5*` 소스셋 분리, `Saml2X509Credential`·`OpenSamlOperations`·초기화
2. [02-RelyingPartyRegistration과-레지스트리.md](02-RelyingPartyRegistration과-레지스트리.md) — RP/AP 설정 모델, 빌더, 레지스트리(InMemory/Caching/Iterable), IdP 메타데이터 로딩
3. [03-AuthnRequest-생성으로-로그인-시작.md](03-AuthnRequest-생성으로-로그인-시작.md) — `Saml2WebSsoAuthenticationRequestFilter` + `AuthenticationRequestResolver`, Redirect/POST 바인딩, 요청 서명
4. [04-응답검증과-인증프로바이더.md](04-응답검증과-인증프로바이더.md) — `Saml2AuthenticationTokenConverter` → `Saml2WebSsoAuthenticationFilter` → `OpenSaml5AuthenticationProvider` 검증 파이프라인
5. [05-메타데이터와-싱글로그아웃.md](05-메타데이터와-싱글로그아웃.md) — `Saml2MetadataFilter`, RP/AP 주도 로그아웃 필터·리졸버·밸리데이터

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.saml2
│
├── Saml2Exception                         최상위 언체크 예외
│
├── core/                                  버전 중립 값 타입 + OpenSAML 부트스트랩
│   ├── Saml2X509Credential                인증서+키 + 용도(SIGNING/VERIFICATION/ENCRYPTION/DECRYPTION)
│   ├── Saml2Error / Saml2ErrorCodes       검증 실패 코드
│   ├── Saml2ResponseValidatorResult       에러 누적 결과(모나드 비슷)
│   ├── Saml2ParameterNames                SAMLRequest/SAMLResponse/RelayState/SigAlg/Signature
│   └── OpenSamlInitializationService      OpenSAML 1회 초기화(멱등)
│
├── provider.service/                      SP의 핵심
│   ├── registration/                      "누구와 SAML을 하는가" 설정
│   │   ├── RelyingPartyRegistration       RP(우리)+AP(IdP) 한 쌍, registrationId로 식별
│   │   ├── AssertingPartyMetadata         IdP 메타데이터 인터페이스
│   │   ├── RelyingPartyRegistrationRepository   findByRegistrationId / findUniqueByAssertingPartyEntityId
│   │   ├── InMemory / Caching / Iterable...Repository
│   │   ├── AssertingPartyMetadataRepository + Jdbc / BaseOpenSaml...
│   │   └── RelyingPartyRegistrations      IdP 메타데이터 XML → 빌더
│   │
│   ├── authentication/                    Response/Assertion → Authentication
│   │   ├── Saml2AuthenticationToken       검증 전 입력 토큰(SAMLResponse 보유)
│   │   ├── BaseOpenSamlAuthenticationProvider   검증 파이프라인(패키지 전용)
│   │   ├── Saml2Authentication / Saml2AssertionAuthentication   검증 후 결과
│   │   ├── Saml2Redirect/PostAuthenticationRequest   보낸 AuthnRequest 보관본
│   │   └── logout/                        LogoutRequest/Response 밸리데이터 + 결과
│   │
│   ├── metadata/                          SP 메타데이터 XML 생성
│   │   ├── Saml2MetadataResolver          RelyingPartyRegistration → XML
│   │   └── Saml2MetadataResponseResolver  HttpServletRequest → 메타데이터 응답
│   │
│   └── web/                               서블릿 필터 계층
│       ├── Saml2WebSsoAuthenticationRequestFilter   /saml2/authenticate → IdP로 redirect/post
│       ├── Saml2MetadataFilter                       /saml2/service-provider-metadata/{id}
│       ├── Saml2AuthenticationTokenConverter         /login/saml2/sso → Saml2AuthenticationToken
│       ├── *RelyingPartyRegistrationResolver         요청 → RelyingPartyRegistration
│       ├── *Saml2AuthenticationRequestRepository     보낸 AuthnRequest 저장(세션/캐시)
│       ├── authentication/
│       │   ├── Saml2WebSsoAuthenticationFilter        SSO 진입 필터
│       │   ├── Saml2AuthenticationRequestResolver     AuthnRequest 생성 전략
│       │   └── logout/                                 로그아웃 필터·리졸버·성공 핸들러
│       └── metadata/
│
├── jackson / jackson2/                    토큰/예외 JSON 직렬화 믹스인(세션 저장용)
└── aot.hint/                              GraalVM 네이티브 힌트

src/opensaml5Main/java   (별도 소스셋, OpenSAML 5 구체 구현)
   ├── OpenSaml5AuthenticationProvider          ← BaseOpenSamlAuthenticationProvider 위임
   ├── OpenSaml5AuthenticationRequestResolver   ← BaseOpenSamlAuthenticationRequestResolver
   ├── OpenSaml5AuthenticationTokenConverter
   ├── OpenSaml5MetadataResolver
   ├── OpenSaml5AssertingPartyMetadataRepository
   ├── OpenSaml5LogoutRequest/ResponseValidator, ...Resolver
   └── OpenSaml5Template                        OpenSamlOperations 구현(직렬화/서명/검증/복호화)
```
