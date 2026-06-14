# rsocket 모듈 — RSocket 보안

## 한 줄 정의

`rsocket` 모듈은 **RSocket 프로토콜의 모든 메시지 교환(payload)을 가로채 인증·인가를 적용하는 보안 계층**이다. HTTP의 서블릿 필터 체인에 해당하는 역할을, RSocket의 `SocketAcceptorInterceptor`와 `PayloadInterceptor` 체인으로 리액티브하게 구현한다.

## 이 모듈이 푸는 문제

RSocket은 HTTP처럼 단일 요청-응답만 있는 프로토콜이 아니다. 연결을 여는 SETUP 프레임 한 번과, 그 위에서 흐르는 여러 종류의 상호작용(fire-and-forget, request-response, request-stream, request-channel, metadata-push)이 한 연결 안에 공존한다. 그래서 "요청 한 번에 필터 한 번"이라는 서블릿식 모델이 그대로 맞지 않는다.

이 모듈은 두 가지 시점을 분리해서 보안을 건다.

- **연결 수립 시점(SETUP)**: 클라이언트가 처음 붙을 때 자격증명(자격증명 메타데이터)을 검사해 인증을 확정하고, 그 인증을 연결 전체에 흐르는 Reactor `Context`에 심는다.
- **개별 메시지 시점(REQUEST_*)**: 연결 위에서 오가는 각 요청마다 라우트(목적지)를 보고 인가를 판단한다.

핵심 설계는 **`PayloadInterceptor`라는 단일 추상화**다. 인증 인터셉터든 인가 인터셉터든 익명 인터셉터든 모두 같은 인터페이스를 구현하고, 순서를 가진 체인으로 묶여 `Mono<Void>`를 반환하는 리액티브 파이프라인을 형성한다.

## 의존·연관 모듈

```
   rsocket
     │
     ├─▶ core (Authentication, GrantedAuthority, ReactiveSecurityContextHolder)
     ├─▶ core (ReactiveAuthenticationManager / ReactiveAuthorizationManager 계약)
     ├─▶ oauth2-resource-server (BearerTokenAuthenticationToken)
     │
     ├─▶ io.rsocket  (RSocket, SocketAcceptor, SocketAcceptorInterceptor, RSocketProxy)
     ├─▶ spring-messaging (MetadataExtractor, RouteMatcher — 라우트/메타데이터 추출)
     └─▶ reactor-core (Mono, Flux, Context)
```

- 실제 보안 설정(어떤 인터셉터를 어떤 순서로 꽂을지)은 `config` 모듈의 `RSocketSecurity` DSL이 담당하고, 이 모듈은 그 DSL이 조립해 쓰는 **부품(인터셉터, 매처, 인코더)**을 제공한다.
- 인증 토큰 타입과 인증/인가 매니저 계약은 모두 Spring Security 코어의 리액티브 계약을 그대로 재사용한다. 이 모듈은 RSocket 프로토콜과 그 계약 사이를 잇는 **어댑터**다.

## 하위 챕터 목차

1. [01-개요와-인터셉터-체인.md](./01-개요와-인터셉터-체인.md) — RSocket에 보안이 끼어드는 진입점(`SocketAcceptorInterceptor`)과 `PayloadInterceptor` 체인이 어떻게 SETUP과 개별 요청을 감싸는지.
2. [02-인증.md](./02-인증.md) — `AuthenticationPayloadInterceptor`가 payload 메타데이터에서 자격증명을 꺼내 인증하고 `ReactiveSecurityContextHolder`에 심는 흐름, 익명 처리.
3. [03-인가와-페이로드-매칭.md](./03-인가와-페이로드-매칭.md) — `AuthorizationPayloadInterceptor` + `PayloadExchangeMatcher` 기반 라우트 매칭과 `ReactiveAuthorizationManager` 결정 흐름.
4. [04-메타데이터-인코딩.md](./04-메타데이터-인코딩.md) — 클라이언트 측에서 자격증명을 payload 메타데이터로 싣는 인코더/디코더(Simple, Bearer, 레거시 Basic).

## 전체 패키지 구조 ASCII 맵

```
rsocket/src/main/java/org/springframework/security/rsocket/
│
├── api/                         보안 인터셉터의 핵심 계약 (프로토콜 비의존)
│   ├── PayloadExchange          한 번의 payload 상호작용 추상화 (type/payload/mimeType)
│   ├── PayloadExchangeType      SETUP / FIRE_AND_FORGET / REQUEST_* / PAYLOAD / METADATA_PUSH
│   ├── PayloadInterceptor       intercept(exchange, chain) — 보안 단위
│   └── PayloadInterceptorChain  next(exchange) — 다음 인터셉터로 위임
│
├── core/                        RSocket 진입점 어댑터 + 체인 실행 엔진
│   ├── SecuritySocketAcceptorInterceptor   보안을 지연 적용하는 래퍼
│   ├── PayloadSocketAcceptorInterceptor    SocketAcceptor에 인터셉터들을 꽂음
│   ├── PayloadSocketAcceptor               SETUP을 가로채고 RSocket을 프록시로 감쌈
│   ├── PayloadInterceptorRSocket           각 요청 메서드마다 체인 실행 (RSocketProxy)
│   ├── ContextPayloadInterceptorChain      체인 노드 + Reactor Context 캡처
│   └── DefaultPayloadExchange              PayloadExchange 기본 구현
│
├── authentication/              인증 인터셉터 + payload→Authentication 변환
│   ├── AuthenticationPayloadInterceptor             자격증명 변환→인증→Context 저장
│   ├── AnonymousPayloadInterceptor                  인증 없으면 익명 토큰 주입
│   ├── PayloadExchangeAuthenticationConverter       변환 계약
│   ├── AuthenticationPayloadExchangeConverter       표준 Simple/Bearer 인증 추출
│   ├── BasicAuthenticationPayloadExchangeConverter  (deprecated) 레거시 Basic 추출
│   └── BearerPayloadExchangeConverter               (deprecated) 레거시 Bearer 추출
│
├── authorization/               인가 인터셉터 + 인가 매니저
│   ├── AuthorizationPayloadInterceptor                   Context의 인증을 꺼내 verify
│   └── PayloadExchangeMatcherReactiveAuthorizationManager  매처→매니저 매핑 테이블
│
├── util/matcher/                payload 매칭 (HTTP의 RequestMatcher 대응)
│   ├── PayloadExchangeMatcher        matches(exchange) → MatchResult
│   ├── RoutePayloadExchangeMatcher   라우트 패턴 매칭 + 경로 변수 추출
│   ├── PayloadExchangeMatchers       setup() / anyRequest() / anyExchange()
│   ├── PayloadExchangeMatcherEntry   (matcher, entry) 쌍
│   └── PayloadExchangeAuthorizationContext  exchange + 경로 변수
│
└── metadata/                    클라이언트 측 자격증명 인코딩
    ├── SimpleAuthenticationEncoder       표준 Simple 인증 인코더
    ├── BearerTokenAuthenticationEncoder  표준 Bearer 인코더
    ├── BasicAuthenticationEncoder/Decoder (deprecated) 레거시 Basic
    ├── UsernamePasswordMetadata          username/password 표현
    └── BearerTokenMetadata               bearer token 표현
```
