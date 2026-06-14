# 02. SecurityContext와 컨텍스트 전파

## 무엇을 / 왜

인증이 끝나 `Authentication`을 손에 쥐었다고 끝이 아니다. 그 다음 코드(컨트롤러, 서비스, 메서드 보안 인터셉터)가 "지금 누가 요청 중인가"를 알아야 한다. 매 메서드마다 `Authentication`을 인자로 넘기는 건 비현실적이다. 그래서 Spring Security는 **현재 실행 흐름(보통 스레드)에 인증 정보를 묶어두고, 어디서든 정적으로 꺼내 쓰는** 방식을 택한다.

이 장은 그 저장소인 `SecurityContextHolder`와, 저장 전략(`SecurityContextHolderStrategy`), 그리고 스레드 경계를 넘는 전파(리액티브·비동기) 문제를 다룬다.

소스: `core/src/main/java/org/springframework/security/core/context/`

## 핵심 타입

```
  애플리케이션 코드
       │ SecurityContextHolder.getContext().getAuthentication()
       ▼
 ┌─────────────────────────────┐
 │ SecurityContextHolder        │  (정적 파사드, JVM 전역)
 │   - static strategy 필드      │
 └──────────────┬──────────────┘
                │ 위임 (delegate)
                ▼
 ┌─────────────────────────────────────────────┐
 │ SecurityContextHolderStrategy (전략 인터페이스)  │
 │   getContext / setContext / clearContext      │
 │   getDeferredContext / createEmptyContext     │
 └───┬──────────────┬───────────────┬───────────┘
     │              │               │
     ▼              ▼               ▼
ThreadLocal   InheritableThread   Global
Strategy      LocalStrategy       Strategy   (+ Listening/Observation 데코레이터)
     │
     ▼ 담는 것
 ┌─────────────────────────────┐
 │ SecurityContext              │  getAuthentication() / setAuthentication()
 │   └ SecurityContextImpl      │
 └─────────────────────────────┘
```

### SecurityContext — 인증을 감싸는 얇은 봉투

`SecurityContext`는 `getAuthentication()` / `setAuthentication()` 두 메서드뿐인 인터페이스다. "왜 `Authentication`을 바로 들고 다니지 않고 한 겹 더 감싸는가?"라는 의문이 들 수 있다. 이유는 **교체 가능성**이다. 컨텍스트라는 봉투를 두면, 인증 객체를 바꾸거나(세션 고정 보호 시 새 컨텍스트 생성) 향후 부가 정보를 더하기 쉬워진다. 기본 구현 `SecurityContextImpl`은 사실상 `Authentication` 하나를 담는 직렬화 가능 컨테이너다.

### SecurityContextHolder — JVM 전역 정적 파사드

`SecurityContextHolder.java`는 전부 `static` 메서드로 이루어진 파사드다. 실제 일은 하지 않고, 내부 `static` 필드인 `strategy`에 위임한다.

```java
public static SecurityContext getContext() { return strategy.getContext(); }
public static void setContext(SecurityContext context) { strategy.setContext(context); }
public static void clearContext() { strategy.clearContext(); }
```

이 정적 파사드 + 위임 전략 구조 덕분에, 호출 코드는 `SecurityContextHolder.getContext()`만 알면 되고, **저장 메커니즘(스레드로컬/전역/커스텀)은 전략 교체로 바꿀 수 있다**.

### 세 가지 저장 전략(MODE)

`SecurityContextHolder`는 시작 시 `initializeStrategy()`로 전략을 고른다. 선택 기준은 시스템 프로퍼티 `spring.security.strategy` 또는 `setStrategyName(...)` 호출이다.

```java
MODE_THREADLOCAL            → ThreadLocalSecurityContextHolderStrategy        (기본값)
MODE_INHERITABLETHREADLOCAL → InheritableThreadLocalSecurityContextHolderStrategy
MODE_GLOBAL                 → GlobalSecurityContextHolderStrategy
(그 외 문자열)               → 해당 FQCN 클래스를 리플렉션으로 로드
```

- **MODE_THREADLOCAL (기본):** 컨텍스트를 `ThreadLocal`에 저장. 한 요청 = 한 스레드인 서블릿 환경에 자연스럽다. 요청 간 격리가 되며 서버에 안전하다.
- **MODE_INHERITABLETHREADLOCAL:** `InheritableThreadLocal`을 써서, 부모 스레드가 만든 자식 스레드도 같은 컨텍스트를 본다. 직접 스레드를 만드는 경우 유용하지만, 스레드 풀에선 오염 위험이 있다.
- **MODE_GLOBAL:** 모든 스레드가 **하나의** 컨텍스트를 공유. 주석이 명시하듯 서버에는 부적절하고, 독립 실행형 클라이언트 앱(예: 데스크톱)을 위한 모드다.

> 주의: `setStrategyName()`은 호출 시 전략을 **재초기화**하므로, 인플라이트 요청이 있을 때 바꾸면 레이스가 생긴다. JVM당 한 번만 설정하라고 주석이 경고한다. 빈으로 직접 전략 인스턴스를 주입하려면 `setContextHolderStrategy(strategy)`를 쓰며, 이 경우 내부적으로 `MODE_PRE_INITIALIZED`로 표시된다.

### ThreadLocalSecurityContextHolderStrategy — 기본 전략의 실제

가장 많이 쓰이는 기본 전략의 내부를 보면, 저장 단위가 `SecurityContext`가 아니라 `Supplier<SecurityContext>`(지연 공급자)다.

```java
private static final ThreadLocal<Supplier<SecurityContext>> contextHolder = new ThreadLocal<>();

@Override public SecurityContext getContext() { return getDeferredContext().get(); }

@Override public Supplier<SecurityContext> getDeferredContext() {
    Supplier<SecurityContext> result = contextHolder.get();
    if (result == null) {                          // 아직 없으면
        SecurityContext context = createEmptyContext();
        result = () -> context;                    // 빈 컨텍스트를 지연 보관
        contextHolder.set(result);
    }
    return result;
}
```

**왜 `Supplier`인가(5.8+):** 매 요청마다 세션에서 컨텍스트를 역직렬화하는 비용을 아끼기 위해서다. "실제로 누군가 `getAuthentication()`을 호출할 때"까지 컨텍스트 로딩을 미룰 수 있다. `getContext()`는 절대 `null`을 반환하지 않는다 — 없으면 빈 컨텍스트를 만들어 채운다.

> **메모리 누수 주의:** `ThreadLocal`은 요청이 끝나면 반드시 `clearContext()`로 비워야 한다. 서블릿 환경에서는 `web` 모듈의 `SecurityContextHolderFilter`/`SecurityContextPersistenceFilter`가 `finally` 블록에서 이를 책임진다.

### 변경 감지 데코레이터

`ListeningSecurityContextHolderStrategy`는 다른 전략을 감싸, 컨텍스트가 바뀔 때 `SecurityContextChangedEvent`를 `SecurityContextChangedListener`에 발행한다. `ObservationSecurityContextChangedListener`는 이를 Micrometer 관측(observability)과 연결한다. 즉 "컨텍스트가 언제 누구로 바뀌었는가"를 추적하는 훅이다.

## 동작 흐름 — 한 HTTP 요청에서의 컨텍스트 생애

```
요청 시작
   │  (web: SecurityContextHolderFilter)
   ▼
[1] 세션/리포지토리에서 SecurityContext 로딩 → SecurityContextHolder.setDeferredContext(supplier)
   │      (Supplier로 지연 — 실제 인증 필요할 때만 역직렬화)
   ▼
[2] 인증 필터가 로그인 처리 → 새 Authentication 생성
   │      context.setAuthentication(authResult); SecurityContextHolder.setContext(context)
   ▼
[3] 컨트롤러/서비스 실행
   │      SecurityContextHolder.getContext().getAuthentication()  ← 어디서든 동일 스레드면 접근 가능
   │      메서드 보안 인터셉터도 같은 경로로 현재 인증을 읽음 (06장)
   ▼
[4] 응답 후 finally
          SecurityContextHolder.clearContext()   ← ThreadLocal 정리 (누수 방지)
```

`core`는 이 흐름에서 **[1]~[4]의 저장·조회 메커니즘**만 제공한다. 실제로 언제 로딩하고 언제 비울지(필터 배치)는 `web` 모듈의 책임이다.

## 리액티브·비동기 전파

`ThreadLocal`은 스레드를 넘는 순간 무용지물이다. WebFlux는 한 요청이 여러 스레드를 오가므로, 별도 전파 메커니즘이 필요하다.

### ReactiveSecurityContextHolder — Reactor Context 기반

리액티브 세계에서는 `ThreadLocal` 대신 Reactor의 불변 `Context`에 컨텍스트를 싣는다.

```java
public static Mono<SecurityContext> getContext() {
    return Mono.deferContextual(Mono::just).cast(Context.class)
        .filter(ReactiveSecurityContextHolder::hasSecurityContext)
        .flatMap(ReactiveSecurityContextHolder::getSecurityContext);
}

public static Context withAuthentication(Authentication authentication) {
    return withSecurityContext(Mono.just(new SecurityContextImpl(authentication)));
}
```

핵심은 `SecurityContext.class`를 **키로** 하여 Reactor `Context`에 `Mono<SecurityContext>`를 넣고 빼는 것이다. Reactor `Context`는 구독 체인을 따라 스레드 경계와 무관하게 전파되므로, 리액티브 흐름 어디서든 `ReactiveSecurityContextHolder.getContext()`로 현재 인증을 얻을 수 있다.

### ThreadLocalAccessor — 두 세계의 다리

`SecurityContextHolderThreadLocalAccessor`와 `ReactiveSecurityContextHolderThreadLocalAccessor`는 Micrometer Context Propagation의 `ThreadLocalAccessor`를 구현해, **ThreadLocal 기반 컨텍스트와 Reactor Context 기반 컨텍스트를 자동으로 상호 변환**한다. 명령형 코드와 리액티브 코드가 섞일 때 인증이 끊기지 않도록 잇는 역할이다.

### 비동기 실행 전파

`concurrent`/`scheduling`/`task` 패키지의 래퍼들(예: `DelegatingSecurityContextRunnable`, `DelegatingSecurityContextExecutor`)은 작업을 다른 스레드에서 실행하기 전에 **현재 `SecurityContext`를 캡처**해 대상 스레드에 설정하고, 끝나면 복원한다. `@Async`나 스레드 풀에 작업을 던질 때도 인증이 따라가게 한다.

## 설계 포인트 / 확장점

- **정적 파사드 + 전략 패턴:** 호출부는 단순(정적 메서드), 메커니즘은 교체 가능. 테스트에서는 전략을 바꿔 가짜 인증을 주입한다.
- **Supplier 지연 로딩(5.8+):** 인증이 실제로 필요할 때까지 컨텍스트 역직렬화를 미뤄 성능 최적화.
- **전략 인스턴스 직접 주입(`setContextHolderStrategy`, 5.6+):** 빈으로 전략을 관리해 GC·테스트 격리를 제어. 인터셉터·필터들은 `SecurityContextHolder::getContextHolderStrategy`를 `Supplier`로 들고 다녀, 전역 정적 상태에 직접 묶이지 않도록 설계되어 있다(06장의 인터셉터 코드 참고).
- **변경 리스너:** `SecurityContextChangedListener`로 컨텍스트 변경을 감사·관측에 연결.

## 정리

`SecurityContextHolder`는 "현재 인증"을 실행 흐름에 묶어두는 JVM 전역 정적 파사드이며, 실제 저장은 교체 가능한 `SecurityContextHolderStrategy`에 위임한다. 서블릿 환경의 기본은 `ThreadLocal`(+지연 `Supplier`)이고, 전역·상속형 모드와 커스텀 전략도 고를 수 있다. 스레드 경계를 넘는 리액티브·비동기 상황에서는 `ReactiveSecurityContextHolder`(Reactor Context)와 ThreadLocalAccessor·Delegating 래퍼가 인증을 끊김 없이 전파한다. 이렇게 저장된 `Authentication`이 다음 장의 인증 처리(03·04장)와 인가 결정(05·06장)의 입력이 된다.
