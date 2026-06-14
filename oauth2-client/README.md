# oauth2-client — OAuth 2.0 / OpenID Connect 클라이언트

> 한 줄 정의: 우리 애플리케이션을 **OAuth 2.0 클라이언트 / OIDC Relying Party**로 만들어, 외부 인가 서버(Google, GitHub, Keycloak 등)에 사용자를 위임 로그인시키고, 발급받은 액세스 토큰으로 보호된 리소스를 호출하게 해 주는 모듈.

---

## 1. 이 모듈이 푸는 문제

웹 애플리케이션이 직접 비밀번호를 받지 않고, 신뢰할 수 있는 **인가 서버(Authorization Server)** 에 인증을 위임하고 싶을 때가 있다. "Google로 로그인", "GitHub로 로그인" 버튼이 그것이다. 이때 클라이언트(우리 앱) 입장에서 처리해야 할 일은 생각보다 많다.

- 사용자를 인가 서버의 **Authorization Endpoint** 로 리다이렉트시키고(`/oauth2/authorization/{registrationId}`),
- 돌아온 **authorization code** 를 **Token Endpoint** 에서 access token / refresh token / id_token 으로 교환하고,
- (OIDC라면) id_token 서명·클레임을 검증하고 **UserInfo Endpoint** 에서 사용자 속성을 가져오고,
- 발급된 토큰을 **OAuth2AuthorizedClient** 로 저장해 두었다가 만료되면 refresh token으로 갱신하고,
- 그 토큰을 다시 꺼내 **리소스 서버 호출 시 Bearer 헤더로** 실어 보낸다.

이 모듈은 위 흐름 전체를 **서블릿(Web MVC)** 과 **리액티브(WebFlux)** 두 스택 모두에서 표준 컴포넌트로 제공한다. 핵심은 두 갈래다.

1. **OAuth2 Login** — 위임 로그인 자체를 인증 수단으로 사용 (필터 + AuthenticationProvider 체인).
2. **Authorized Client 관리** — 로그인과 무관하게, 토큰을 획득·갱신·보관하고 리소스 호출에 사용하는 인프라(`OAuth2AuthorizedClientManager`).

---

## 2. 의존 모듈 / 연관 관계

```
   ┌─────────────────────────────────────────────────────────────┐
   │ oauth2-client (이 모듈)                                       │
   │   - ClientRegistration / Repository                          │
   │   - OAuth2 Login Filter/Provider                             │
   │   - OAuth2AuthorizedClientManager / Provider                 │
   │   - Token Endpoint Client / UserInfo Service                 │
   └───────┬───────────────────────────┬─────────────────────────┘
           │ 사용                        │ 사용
           ▼                            ▼
   ┌──────────────────┐        ┌──────────────────────────────┐
   │ oauth2-core      │        │ spring-security-web          │
   │  OAuth2AccessToken│        │  AbstractAuthenticationProcess-│
   │  OAuth2Error      │        │  Filter, RedirectStrategy,   │
   │  AuthorizationGrant│       │  SecurityContextRepository   │
   │  OAuth2User/OidcUser│      └──────────────────────────────┘
   └──────────────────┘
           │ (OIDC id_token 디코딩)
           ▼
   ┌──────────────────┐        ┌──────────────────────────────┐
   │ oauth2-jose (JWT)│        │ spring-web (RestClient/WebClient)│
   │  JwtDecoder       │        │  HTTP 통신                    │
   └──────────────────┘        └──────────────────────────────┘
```

- **../oauth2-core/README.md** — `OAuth2AccessToken`, `OAuth2RefreshToken`, `OAuth2User`, `OidcUser`, `AuthorizationGrantType`, `OAuth2Error` 등 도메인 타입을 제공. 이 모듈은 그 위에 "흐름"을 얹는다.
- **../oauth2-jose/README.md** — OIDC `id_token`(JWT) 서명 검증을 위한 `JwtDecoder`.
- **../web/README.md** — 필터 체인, `AbstractAuthenticationProcessingFilter`, `RedirectStrategy`, `SecurityContextRepository` 등 웹 인증 인프라.
- **리소스 서버 측**은 별도 모듈(`oauth2-resource-server`). 이 모듈은 토큰을 **발급/관리/사용하는 클라이언트** 측이다.

---

## 3. 내부 목차 (읽는 순서)

1. [01-클라이언트-등록과-인가된-클라이언트-모델.md](01-클라이언트-등록과-인가된-클라이언트-모델.md) — `ClientRegistration` / `Repository`, `OAuth2AuthorizedClient`, `AuthorizedClientService` / `Repository`. 모든 흐름의 데이터 토대.
2. [02-인가-코드-그랜트-흐름과-필터.md](02-인가-코드-그랜트-흐름과-필터.md) — 사용자를 리다이렉트시키고(`OAuth2AuthorizationRequestRedirectFilter`) 돌아온 코드를 받는(`OAuth2LoginAuthenticationFilter`) 서블릿 필터 두 개와 state/PKCE.
3. [03-인증-프로바이더와-토큰-엔드포인트.md](03-인증-프로바이더와-토큰-엔드포인트.md) — `OAuth2LoginAuthenticationProvider`, code→token 교환을 담당하는 `OAuth2AccessTokenResponseClient`(RestClient/WebClient) 체계.
4. [04-OIDC-로그인과-유저인포.md](04-OIDC-로그인과-유저인포.md) — `OidcAuthorizationCodeAuthenticationProvider`(id_token 검증·nonce), `OidcUserService` / `DefaultOAuth2UserService`, RP-Initiated Logout.
5. [05-인가된-클라이언트-매니저와-프로바이더.md](05-인가된-클라이언트-매니저와-프로바이더.md) — `OAuth2AuthorizedClientManager` / `Provider`, refresh/client_credentials/jwt-bearer/token-exchange 그랜트, 리소스 호출 인터셉터, `@RegisteredOAuth2AuthorizedClient`.
6. [06-리액티브-스택-대응.md](06-리액티브-스택-대응.md) — WebFlux용 `Reactive*` 미러 컴포넌트 개관과 서블릿판과의 대응표.

---

## 4. 전체 패키지 구조 ASCII 맵

```
org.springframework.security.oauth2.client
│
├── (root)                      ── Authorized Client "관리" 계층 (서블릿+리액티브 공용 인터페이스)
│     OAuth2AuthorizedClient            인가된 클라이언트 = 등록 + principal + 토큰
│     OAuth2AuthorizedClientManager     관리 진입점 (authorize)
│     OAuth2AuthorizedClientProvider    그랜트별 인가 전략 (authorization_code/refresh/client_credentials/jwt-bearer/token-exchange)
│     OAuth2AuthorizationContext        Provider에 넘기는 상태 컨텍스트
│     OAuth2AuthorizedClientService     영속화(서비스): InMemory / Jdbc
│     *AuthorizedClientProviderBuilder  Provider 조합 빌더
│     RefreshToken/ClientCredentials/JwtBearer/TokenExchange ...Provider
│     Reactive* (리액티브 미러)
│
├── registration                ── 클라이언트 등록 메타데이터
│     ClientRegistration                clientId/secret, grant, scope, endpoint URI
│     ClientRegistrationRepository      registrationId → ClientRegistration 조회
│     ClientRegistrations               issuer URI로 OIDC discovery 자동 구성
│     InMemory* / Supplier* / Reactive*
│
├── authentication              ── OAuth2 Login 인증 토큰/프로바이더
│     OAuth2LoginAuthenticationFilter용 토큰들 + OAuth2LoginAuthenticationProvider
│     OAuth2AuthenticationToken         최종 Authentication (SecurityContext에 저장)
│
├── endpoint                    ── Token Endpoint 호출 (code/refresh/client_credentials/jwt-bearer/token-exchange)
│     OAuth2AccessTokenResponseClient   추상 인터페이스
│     RestClient*TokenResponseClient    (서블릿) 5종 그랜트
│     WebClientReactive*                (리액티브) 5종 그랜트
│
├── userinfo                    ── UserInfo Endpoint 호출
│     OAuth2UserService / DefaultOAuth2UserService
│
├── oidc                        ── OpenID Connect 특화
│     authentication/  OidcAuthorizationCodeAuthenticationProvider, id_token 디코더/검증
│     userinfo/        OidcUserService, OidcUserRequest
│     session/         OidcSessionRegistry (Back-Channel Logout)
│     web/logout/      OidcClientInitiatedLogoutSuccessHandler (RP-Initiated Logout)
│     authentication/logout/  OidcLogoutToken
│
├── web                         ── 서블릿(Web MVC) 통합
│     OAuth2AuthorizationRequestRedirectFilter   인가 요청 시작(리다이렉트)
│     OAuth2LoginAuthenticationFilter            인가 응답 처리(로그인)
│     OAuth2AuthorizationCodeGrantFilter         로그인 아닌 순수 code grant
│     *AuthorizationRequestResolver/Repository   state/PKCE 보관
│     DefaultOAuth2AuthorizedClientManager       서블릿용 Manager
│     OAuth2AuthorizedClientRepository           요청 컨텍스트 기반 토큰 저장
│     client/   OAuth2ClientHttpRequestInterceptor (RestClient에 토큰 주입)
│     method/   @RegisteredOAuth2AuthorizedClient 인자 리졸버
│     reactive/ WebClient ExchangeFilterFunction
│     server/   WebFlux(WebFilter) 미러
│
├── annotation                  ── @RegisteredOAuth2AuthorizedClient, @ClientRegistrationId
├── event                       ── OAuth2AuthorizedClientRefreshedEvent
├── http                        ── OAuth2ErrorResponseErrorHandler
├── jackson / jackson2          ── 직렬화(JSON) 모듈
└── aot                         ── AOT/Native 힌트
```
