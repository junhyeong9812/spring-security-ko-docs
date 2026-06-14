# config 모듈 — 보안 설정 DSL의 중심

> 버전 기준: Spring Security 7.1.1-SNAPSHOT
> 소스 위치: `config/src/main/java`, `config/src/main/kotlin`

## 한 줄 정의

`config` 모듈은 `@EnableWebSecurity`·`HttpSecurity`·`SecurityFilterChain` 빈 같은 **설정용 DSL과 그 DSL을 실제 보안 컴포넌트(필터 체인, 인증 매니저, 메서드 인터셉터)로 조립해 주는 빌더 인프라**를 제공한다. 즉 "내가 원하는 보안 정책"을 자바/코틀린 코드나 XML로 선언하면, 이 모듈이 그것을 `web`·`core` 모듈의 실제 객체 그래프로 번역한다.

## 이 모듈이 푸는 문제

Spring Security의 실제 보안 동작은 `web`(서블릿 필터), `core`(인증/인가 추상화), `oauth2`, `saml2` 등 여러 모듈에 흩어진 수백 개의 클래스로 이뤄진다. 이것들을 사용자가 직접 손으로 연결한다면 다음을 모두 알아야 한다.

- 어떤 필터를 어떤 순서로 둘 것인가 (`SecurityContextHolderFilter` → `CsrfFilter` → `UsernamePasswordAuthenticationFilter` → … → `AuthorizationFilter`)
- 각 필터가 의존하는 `AuthenticationManager`·`SecurityContextRepository`·`RequestCache`를 어떻게 만들고 주입할 것인가
- 그 객체들이 스프링 빈으로서 `@Autowired`·`afterPropertiesSet()`·`Aware` 콜백을 제대로 받게 하려면 어떻게 할 것인가

`config` 모듈은 이 복잡성을 **빌더(Builder) + 설정자(Configurer) + 후처리기(ObjectPostProcessor)** 라는 세 가지 패턴으로 캡슐화한다. 사용자는 `http.formLogin(withDefaults())` 한 줄만 쓰고, 모듈은 그 한 줄을 "필터 생성 → 의존성 주입 → 올바른 위치에 삽입"으로 풀어낸다.

## 의존·연관 관계

```
                         ┌────────────────────────┐
   사용자 설정 코드 ───▶ │      config            │
   (@EnableWebSecurity)  │  (이 모듈: DSL/빌더)    │
                         └───────────┬────────────┘
                                     │ 조립·후처리
            ┌────────────────────────┼─────────────────────────┐
            ▼                        ▼                         ▼
   ┌─────────────────┐    ┌──────────────────┐     ┌──────────────────┐
   │      web         │    │      core         │     │ oauth2 / saml2 / │
   │ FilterChainProxy │    │ AuthenticationMgr │     │ ldap / acl ...    │
   │ 각종 Filter      │    │ AccessDecision    │     │ (선택적 통합)     │
   └─────────────────┘    └──────────────────┘     └──────────────────┘
                                     │
                                     ▼
                    spring-core / spring-context / spring-beans
                    (ApplicationContext, BeanFactory, @Import)
```

- **아래(소비)**: `web`, `core`, `oauth2`, `saml2`, `ldap`, `acl`, `messaging`, `cas` — `config`은 이들의 클래스를 인스턴스화하고 연결만 한다. 실제 보안 로직은 그쪽에 있다.
- **옆(기반)**: Spring Framework의 `@Configuration`/`@Import`/`ImportSelector`/`BeanFactory`. `config`은 사실상 "보안 전용 자바 컨피그 프레임워크"다.
- 관련 모듈 문서: [../web/README.md](../web/README.md), [../core/README.md](../core/README.md)

## 내부 목차

| 장 | 제목 | 다루는 것 |
|----|------|-----------|
| 01 | [빌더와 Configurer 기반구조](01-빌더와-Configurer-기반구조.md) | `SecurityBuilder` / `AbstractConfiguredSecurityBuilder` / `SecurityConfigurer` / `ObjectPostProcessor`. 모든 설정의 공통 엔진과 build 생애주기 |
| 02 | [@EnableWebSecurity 와 빈 구성](02-EnableWebSecurity-와-빈-구성.md) | 애너테이션이 import하는 `@Configuration`들, `springSecurityFilterChain` 빈이 만들어지는 과정 |
| 03 | [HttpSecurity 와 WebSecurity](03-HttpSecurity-와-WebSecurity.md) | 두 핵심 빌더의 역할 분담, 필터 순서 부여, `DefaultSecurityFilterChain`/`FilterChainProxy` 생성 |
| 04 | [HTTP Configurer 카탈로그](04-HTTP-Configurer-카탈로그.md) | `AbstractHttpConfigurer`·`getOrApply`, FormLogin/Csrf/Authorize 등 Configurer가 필터를 끼워 넣는 흐름 |
| 05 | [인증 구성](05-인증-구성.md) | `AuthenticationConfiguration`·`AuthenticationManagerBuilder`·`GlobalAuthenticationConfigurerAdapter`, 전역 `AuthenticationManager` 조립 |
| 06 | [메서드 보안](06-메서드-보안.md) | `@EnableMethodSecurity`·`MethodSecuritySelector`·`PrePostMethodSecurityConfiguration`, Advisor/Pointcut/Interceptor |
| 07 | [Kotlin DSL 과 XML 네임스페이스](07-Kotlin-DSL-과-XML-네임스페이스.md) | 코틀린 DSL의 `invoke` 패턴, `SecurityNamespaceHandler`/XML 파서, AOT 힌트 |

## 전체 패키지 구조 ASCII 맵

```
config/src/main/java/org/springframework/security/config/
│
├── ObjectPostProcessor.java          # 조립된 객체를 빈처럼 초기화하는 핵심 후크
├── Customizer.java                   # DSL 람다 타입 (withDefaults())
├── SecurityNamespaceHandler.java     # XML <security:*> 네임스페이스 진입점
├── Elements.java / BeanIds.java      # XML 요소명/빈 이름 상수
│
├── annotation/                       # ── 자바 애너테이션 설정의 뿌리 ──
│   ├── SecurityBuilder.java               # build() 인터페이스
│   ├── AbstractConfiguredSecurityBuilder  # init→configure→performBuild 엔진
│   ├── SecurityConfigurer(Adapter).java   # 빌더를 부분 설정하는 전략
│   │
│   ├── configuration/                     # ObjectPostProcessor 빈 등록
│   │   └── AutowireBeanFactoryObjectPostProcessor
│   │
│   ├── authentication/                # ── 인증 구성 ──
│   │   ├── builders/AuthenticationManagerBuilder
│   │   ├── configuration/AuthenticationConfiguration
│   │   │   InitializeUserDetailsBeanManagerConfigurer ...
│   │   └── configurers/ (ldap, userdetails, provisioning)
│   │
│   ├── web/                          # ── 웹(서블릿) 보안 구성 ──
│   │   ├── builders/HttpSecurity, WebSecurity, FilterOrderRegistration
│   │   ├── configuration/EnableWebSecurity, WebSecurityConfiguration,
│   │   │                 HttpSecurityConfiguration, *ImportSelector
│   │   ├── configurers/  FormLogin, Csrf, Authorize, Headers, Logout,
│   │   │                 SessionManagement, ExceptionHandling, ... (+ oauth2/ saml2/ ott/)
│   │   └── reactive/, socket/, servlet/
│   │
│   ├── method/configuration/         # ── 메서드 보안 구성 ──
│   │   ├── EnableMethodSecurity, MethodSecuritySelector
│   │   ├── PrePostMethodSecurityConfiguration, Secured..., Jsr250...
│   │   └── EnableReactiveMethodSecurity ...
│   │
│   ├── authorization/, rsocket/, observation/
│
├── http/                             # ── XML <http> 네임스페이스 파서 (40+ 파일) ──
│   ├── HttpSecurityBeanDefinitionParser, HttpConfigurationBuilder
│   ├── AuthenticationConfigBuilder, FormLoginBeanDefinitionParser ...
│
├── authentication/, ldap/, method/, oauth2/, saml2/, websocket/, web/
│   └── 각 영역의 XML BeanDefinitionParser 및 FactoryBean
│
└── aot/                              # 네이티브 이미지용 RuntimeHints

config/src/main/kotlin/org/springframework/security/config/
└── annotation/web/                   # ── Kotlin DSL ──
    ├── HttpSecurityDsl.kt (operator fun HttpSecurity.invoke{...})
    ├── FormLoginDsl, CsrfDsl, AuthorizeHttpRequestsDsl, HeadersDsl ...
    └── web/server/  (리액티브 ServerHttpSecurity DSL)
```

각 장은 위 구조에서 한 덩어리씩을 골라 "왜 존재하고, 요청·기동 시점에 무슨 일이 일어나는가"를 따라간다. 가장 먼저 [01장](01-빌더와-Configurer-기반구조.md)에서 모든 설정의 공통 엔진을 이해하는 것을 권한다.
