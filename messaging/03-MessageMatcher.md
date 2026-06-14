# 03. MessageMatcher — 메시지를 무엇으로 식별하는가

## 무엇을 / 왜

인가든 CSRF든, 보안 규칙은 항상 "**어떤 메시지에** 이 규칙을 적용할 것인가"라는 선택에서 시작한다. HTTP에서 `RequestMatcher`가 그 역할을 하듯, 메시징에서는 `MessageMatcher`가 메시지를 식별한다.

메시지의 식별 기준은 크게 두 축이다.

1. **목적지(destination)** — `/topic/admin`, `/app/chat/{roomId}` 같은 STOMP 목적지 경로.
2. **메시지 타입(`SimpMessageType`)** — `CONNECT`, `SUBSCRIBE`, `MESSAGE`, `DISCONNECT` 등 STOMP 프레임의 종류.

이 장은 이 두 축을 표현하는 매처 타입들과, 이들이 어떻게 조합되는지를 본다.

## 핵심 타입

```
MessageMatcher<T> (interface)
│   boolean matches(Message<? extends T> message)
│   default MatchResult matcher(Message) ─ match 여부 + 추출 변수(Map)
│   상수 ANY_MESSAGE ─ 항상 true
│
├── SimpMessageTypeMatcher        타입 한 종류와 일치하는지 (CONNECT 등)
│
├── PathPatternMessageMatcher     destination 패턴 매칭 + 경로 변수 추출 (6.5+)
│       NULL_DESTINATION_MATCHER  destination이 null인 메시지(CONNECT 등) 매칭
│
└── AbstractMessageMatcherComposite<T>   여러 매처를 묶는 베이스
        ├── AndMessageMatcher    전부 일치해야 true
        └── OrMessageMatcher     하나라도 일치하면 true
```

### MatchResult — 단순 boolean을 넘어서

`MessageMatcher`의 핵심은 단순 `matches()`가 아니라 `matcher()`가 돌려주는 `MatchResult`다. 매칭 성공 여부와 함께 **경로에서 추출한 변수 맵**을 담는다.

```java
// MessageMatcher.matcher 기본 구현
default MatchResult matcher(Message<? extends T> message) {
    boolean match = matches(message);
    return new MatchResult(match, Collections.emptyMap());
}
```

기본 구현은 변수 없는 단순 매칭이지만, `PathPatternMessageMatcher`는 이걸 오버라이드해서 `/app/chat/{roomId}`의 `roomId` 같은 변수를 채워 넣는다. 이 변수가 04장에서 인가 표현식(`#roomId`)으로 흘러간다.

## 동작 흐름 — PathPatternMessageMatcher의 매칭

`PathPatternMessageMatcher`는 6.5부터 도입된 표준 목적지 매처다. Spring Web의 `PathPattern` 엔진을 그대로 빌려 쓴다.

```
matcher(message)
   │
   ├─ 1. (타입 지정 시) messageTypeMatcher.matches(message)?
   │        타입 불일치 → MatchResult.notMatch()
   │
   ├─ 2. SimpMessageHeaderAccessor.getDestination(headers) 로 destination 추출
   │        destination == null → MatchResult.notMatch()
   │        (즉, CONNECT처럼 목적지 없는 프레임은 이 매처로 안 잡힌다)
   │
   ├─ 3. PathContainer.parsePath(destination, options)
   │
   └─ 4. pattern.matchAndExtract(pathContainer)
            매칭 O → MatchResult.match(uriVariables)  ← 경로 변수 포함
            매칭 X → MatchResult.notMatch()
```

빌더 사용법:

```java
// 타입 무시, 목적지 패턴만
PathPatternMessageMatcher.withDefaults()
    .matcher("/topic/admin/**");

// SimpMessageType.SUBSCRIBE 이면서 목적지가 /topic/admin/{id} 인 메시지
PathPatternMessageMatcher.withDefaults()
    .matcher(SimpMessageType.SUBSCRIBE, "/topic/admin/{id}");
```

`withDefaults()`는 `PathPatternParser.defaultInstance`(슬래시 `/` 구분자, HTTP 경로 옵션)를 쓴다. STOMP에서 흔한 **점(.) 기반 목적지**(`topic.admin.1`)를 쓰려면 `withPathPatternParser(...)`로 점 구분 파서를 넣으면 된다 — 패턴 매칭 자체는 동일한 `PathPattern` 엔진이 처리한다.

### NULL_DESTINATION_MATCHER

CONNECT, DISCONNECT, HEARTBEAT, UNSUBSCRIBE 등은 목적지가 없다. 이런 프레임을 한 번에 잡기 위한 상수가 `PathPatternMessageMatcher.NULL_DESTINATION_MATCHER`다. 단순히 `getDestination(message) == null`을 검사하는 람다 매처다. 04장 빌더의 `nullDestMatcher()`가 이걸 사용한다.

## 동작 흐름 — SimpMessageTypeMatcher

`SimpMessageTypeMatcher`는 더 단순하다. 메시지 헤더에서 `SimpMessageType`을 읽어 생성자에 받은 타입과 `==` 비교한다.

```java
SimpMessageType messageType = SimpMessageHeaderAccessor.getMessageType(headers);
return this.typeToMatch == messageType;
```

CSRF 인터셉터(05장)가 "CONNECT 프레임만 검증"하기 위해 `new SimpMessageTypeMatcher(SimpMessageType.CONNECT)`를 그대로 쓴다. `equals`/`hashCode`가 타입 기준으로 구현돼 있어 매처 자체를 컬렉션 키로 비교·중복 제거할 수 있다.

## 조합 매처 — And / Or

`AndMessageMatcher`와 `OrMessageMatcher`는 `AbstractMessageMatcherComposite`를 상속해, 여러 매처를 불리언 조합한다. 둘 다 단락 평가(short-circuit)를 한다.

```
AndMessageMatcher : 하나라도 false면 즉시 false, 전부 true여야 true
OrMessageMatcher  : 하나라도 true면 즉시 true
```

예를 들어 "SUBSCRIBE 타입이면서 목적지가 `/topic/admin`인 메시지"는 `SimpMessageTypeMatcher`와 목적지 매처를 `AndMessageMatcher`로 묶어 표현할 수 있다(또는 `PathPatternMessageMatcher`에 타입을 함께 지정해 한 번에).

## 설계 포인트 / 확장점

- **`RequestMatcher`와의 의도적 평행 구조** — API 모양(`matches`, `MatchResult`, And/Or 조합, ANY 상수)을 HTTP 쪽과 맞춰, web 모듈을 아는 사람이 그대로 옮겨 올 수 있게 했다.
- **변수 추출의 표준화** — `MatchResult.getVariables()`를 통해 경로 변수가 인가 컨텍스트로 자연스럽게 전달된다. 매처가 단순 필터를 넘어 "데이터 추출기" 역할까지 한다.
- **파서 교체 훅** — 슬래시/점 구분 등 목적지 컨벤션 차이를 `PathPatternParser` 교체로 흡수한다. config에서 `PathPatternMessageMatcher.Builder` 빈을 등록하면 04장 빌더가 그것을 자동으로 집어 쓴다.
- **커스텀 매처** — `MessageMatcher<T>`는 함수형 인터페이스처럼 `matches` 하나만 구현하면 되므로, 람다로 임의 조건 매처를 즉석에서 만들 수 있다(`NULL_DESTINATION_MATCHER`가 그 예).

## 정리

`MessageMatcher`는 보안 규칙의 "대상 선택" 추상화다. 목적지는 `PathPatternMessageMatcher`(+경로 변수 추출), 프레임 종류는 `SimpMessageTypeMatcher`, 둘의 조합은 And/Or 매처가 맡는다. 매칭 결과인 `MatchResult`는 단순 참/거짓을 넘어 추출 변수까지 실어, 다음 장의 인가 표현식으로 흘러간다.
