# 05. WebSocket CSRF — 핸드셰이크 토큰 적재와 CONNECT 검증

## 무엇을 / 왜

WebSocket에는 고유한 CSRF 위험이 있다. WebSocket 핸드셰이크는 일반 HTTP 요청이지만, 브라우저의 **same-origin 정책이 WebSocket에는 적용되지 않는다**. 즉 악성 사이트가 사용자의 쿠키(세션)를 실어 다른 출처의 WebSocket 엔드포인트로 연결을 시도할 수 있다("Cross-Site WebSocket Hijacking"). 이를 막으려면 연결 수립 시점에 CSRF 토큰을 검증해, **같은 출처에서 온 연결만** 받아들여야 한다.

문제는 검증 시점이 둘로 나뉜다는 것이다.

1. **핸드셰이크(HTTP)** — 이때는 아직 STOMP가 시작되기 전이라 토큰 검증을 할 수 없지만, 서버 세션에 저장된 **기대 토큰**을 확보할 수 있다.
2. **CONNECT(STOMP 프레임)** — 실제 STOMP 세션이 열리는 첫 프레임. 클라이언트가 보낸 토큰을 1에서 저장해 둔 기대 토큰과 비교한다.

`web/socket/server`는 1을, `web/csrf`는 2를 담당한다.

## 핵심 타입

```
web/socket/server/
└── CsrfTokenHandshakeInterceptor      (HandshakeInterceptor)
        핸드셰이크 시 HTTP 요청의 CsrfToken을 WebSocket 세션 속성으로 복사

web/csrf/
├── CsrfChannelInterceptor             (ChannelInterceptor)
│       CONNECT 프레임의 토큰을 기대 토큰과 평문 비교
├── XorCsrfChannelInterceptor          (ChannelInterceptor)
│       XOR 마스킹된 토큰을 디코딩 후 상수시간 비교 (BREACH 방어)
└── XorCsrfTokenUtils                  XOR 토큰 디코딩 유틸
```

`CsrfChannelInterceptor`와 `XorCsrfChannelInterceptor`는 **둘 중 하나만** 쓴다. web 모듈에서 CSRF 토큰을 어떻게 발급했는지(평문 vs `XorCsrfTokenRequestAttributeHandler`로 마스킹)에 맞춰 짝을 맞춘다.

## 동작 흐름 — 토큰의 두 단계 여정

```
[1단계: 핸드셰이크 - HTTP]
브라우저 ──GET /ws (Upgrade)──▶ 서버
                                  │
   CsrfTokenHandshakeInterceptor.beforeHandshake
        │  HttpServletRequest 속성에서 DeferredCsrfToken 조회
        │    (web 모듈의 CsrfFilter가 미리 심어 둠)
        │  deferred == null 이면 그냥 통과(토큰 미사용 설정)
        │  csrfToken = deferred.get()
        │  ★ 값만 복사한 새 DefaultCsrfToken 생성
        │     (원본은 handshake의 request/response를 참조 → GC 위해 끊음)
        │  attributes.put(CsrfToken.class.getName(), resolvedToken)
        ▼
   WebSocket 세션 속성에 "기대 토큰" 저장됨
                                  │
[2단계: CONNECT - STOMP 프레임]
브라우저 ──STOMP CONNECT (X-CSRF-TOKEN 헤더)──▶ clientInboundChannel
                                  │
   CsrfChannelInterceptor.preSend
        │  SimpMessageType이 CONNECT가 아니면 → return message (통과)
        │  sessionAttributes에서 기대 토큰 조회
        │     없으면 → MissingCsrfTokenException
        │  STOMP 네이티브 헤더에서 실제 토큰 값 추출
        │  expected.getToken().equals(actual)?
        │     불일치 → InvalidCsrfTokenException
        ▼
   검증 통과 → CONNECT 진행, 이후 프레임은 CSRF 재검증 없음
```

핵심 설계는 **"한 번만 검증한다"**는 것이다. CONNECT 프레임에서만 CSRF를 본다. 일단 STOMP 세션이 수립되면 그 연결은 신뢰된 출처에서 온 것이므로, 이후 SEND/SUBSCRIBE마다 토큰을 요구하지 않는다(애초에 STOMP 클라이언트가 매 프레임에 CSRF 헤더를 붙이지도 않는다).

`CsrfTokenHandshakeInterceptor`가 토큰 값을 **새 객체로 복사**하는 디테일도 중요하다. 원본 `CsrfToken`(특히 지연 토큰)은 핸드셰이크의 `HttpServletRequest`/`Response`를 참조할 수 있는데, 이걸 WebSocket 세션(수명이 긴) 속성에 그대로 넣으면 요청/응답 객체가 연결 내내 GC되지 못한다. 값만 뽑아 `DefaultCsrfToken`으로 새로 담아 이 참조를 끊는다.

## XOR 변형 — XorCsrfChannelInterceptor

`XorCsrfChannelInterceptor`는 흐름이 동일하되 토큰 비교 방식이 다르다. web 모듈이 BREACH 공격 방어를 위해 매 응답마다 토큰을 **랜덤 값으로 XOR 마스킹**해 내보내는 경우, 클라이언트가 보내는 토큰도 마스킹돼 있다. 따라서:

```
XorCsrfChannelInterceptor.preSend (CONNECT만)
   │  actualToken = STOMP 헤더의 마스킹된 토큰 문자열
   │  actualTokenValue = XorCsrfTokenUtils.getTokenValue(actualToken, expected.getToken())
   │       └ XOR 디코딩으로 원래 토큰 복원
   │  equalsConstantTime(expected.getToken(), actualTokenValue)?
   │       └ MessageDigest.isEqual 로 상수시간 비교 (타이밍 공격 방어)
   │  불일치 → InvalidCsrfTokenException
```

평문판이 `String.equals`로 비교하는 것과 달리, XOR판은 **상수시간 비교**(`MessageDigest.isEqual`)를 쓴다. 토큰 비교에 걸리는 시간으로 토큰을 추측하는 타이밍 공격을 막기 위해서다.

## 핵심 메서드 정리

- `CsrfTokenHandshakeInterceptor.beforeHandshake` — HTTP 핸드셰이크 요청에서 `DeferredCsrfToken`을 꺼내 실제 토큰을 해석하고, 값만 복사한 토큰을 WebSocket 세션 속성에 적재한다. 토큰이 없으면 막지 않고 통과시킨다(설정상 CSRF 미사용).
- `CsrfChannelInterceptor.preSend` — `SimpMessageTypeMatcher(CONNECT)`로 CONNECT만 골라, 세션의 기대 토큰과 프레임의 토큰을 비교한다. 기대 토큰 부재는 `MissingCsrfTokenException`, 불일치는 `InvalidCsrfTokenException`.
- `XorCsrfTokenUtils.getTokenValue` — XOR 마스킹된 토큰 문자열을 기대 토큰 길이에 맞춰 디코딩해 원래 토큰을 복원한다.

## 설계 포인트 / 확장점

- **시점 분리 + 채널 분리** — 토큰 *적재*는 핸드셰이크(`HandshakeInterceptor`)에서, 토큰 *검증*은 인바운드 채널(`ChannelInterceptor`)에서. 서로 다른 Spring 확장 지점을 한 보안 목표로 엮었다.
- **web 모듈 토큰 재사용** — `CsrfToken`, `DeferredCsrfToken`, `MissingCsrfTokenException`, `InvalidCsrfTokenException`을 그대로 가져와, HTTP CSRF와 동일한 토큰 발급 인프라를 공유한다. WebSocket이라고 새 토큰 체계를 만들지 않는다.
- **평문/XOR 페어링** — 토큰 발급 핸들러(평문 `CsrfTokenRequestAttributeHandler` vs `XorCsrfTokenRequestAttributeHandler`)에 맞춰 인터셉터를 골라 쓴다. 잘못 짝지으면 항상 검증 실패한다.
- **CONNECT 한정 검증** — 검증 비용과 클라이언트 호환성을 위해 CONNECT 프레임에만 적용. 다른 프레임 타입에 CSRF를 요구하려면 매처를 바꾼 커스텀 인터셉터가 필요하다.

## 정리

WebSocket CSRF는 same-origin 정책 밖의 연결 탈취를 막기 위해, 핸드셰이크 시점에 세션으로 옮겨 둔 기대 토큰과 CONNECT 프레임의 토큰을 한 번 대조하는 방식이다. `CsrfTokenHandshakeInterceptor`가 토큰을 적재(참조 끊은 복사본으로)하고, `CsrfChannelInterceptor`(평문) 또는 `XorCsrfChannelInterceptor`(마스킹+상수시간 비교)가 CONNECT에서 검증한다. 토큰 인프라 자체는 web 모듈 것을 그대로 빌려 쓴다.
