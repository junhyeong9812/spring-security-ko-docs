# 01. 왜 AspectJ 위빙인가 — 프록시 모드와의 차이

## 무엇을 / 왜

이 챕터는 코드를 보기 전에 **이 모듈이 왜 따로 존재하는지**를 잡는다. 핵심 질문은 하나다. 메서드 보안은 이미 `@EnableMethodSecurity` 만 붙이면 동작하는데, 왜 `aspects` 라는 별도 모듈과 `.aj` 파일이 필요한가?

답은 **위빙(weaving) 메커니즘의 선택지**에 있다. `@EnableMethodSecurity` 에는 `mode` 속성이 있고, 기본값은 `AdviceMode.PROXY` 다. 이 모듈은 `mode = AdviceMode.ASPECTJ` 일 때만 쓰인다.

```java
// config: EnableMethodSecurity.java
AdviceMode mode() default AdviceMode.PROXY;
```

## 두 가지 위빙 모델

메서드에 인가 검사를 "끼워 넣는" 방법은 두 가지다.

### (1) PROXY 모드 — Spring AOP 프록시

스프링이 빈을 감싸는 **프록시 객체**를 만든다. 호출자는 실제 객체가 아니라 프록시를 들고 있고, 프록시의 메서드를 호출하면 그 안에서 인가 인터셉터가 먼저 돌고 나서 진짜 메서드로 위임한다.

```
호출자 ──▶ [프록시 객체] ──▶ 인가 인터셉터 ──▶ 실제 빈.method()
                  │
                  └─ 프록시를 "거쳐야만" 검사가 일어난다
```

### (2) ASPECTJ 모드 — 바이트코드 위빙

프록시를 만들지 않는다. 대신 **클래스의 바이트코드 자체**에 인가 호출을 직접 짜 넣는다. 컴파일 타임 위빙(CTW, ajc 컴파일러)이나 로드 타임 위빙(LTW, java agent)이 `.aj` 애스펙트를 실제 클래스에 섞는다.

```
호출자 ──▶ 실제 빈.method()
              │ (메서드 본문 진입 지점에 인가 코드가 이미 박혀 있음)
              └─ around advice ──▶ 인가 인터셉터 ──▶ 원래 본문
```

## PROXY 모드의 한계 (= 이 모듈의 존재 이유)

`PreAuthorizeAspect.aj` 의 클래스 주석은 둘의 차이를 직접 언급한다("This will vary from Spring AOP"). 프록시 모델은 다음 상황에서 인가를 **놓친다**.

| 상황 | PROXY 모드 | ASPECTJ 모드 |
|---|---|---|
| 외부 호출자 → 빈의 보안 메서드 | 검사됨 | 검사됨 |
| **같은 클래스 내부 self-invocation** (`this.m()`) | **검사 안 됨** (프록시 우회) | **검사됨** |
| **스프링 빈이 아닌 객체** (`new Foo().m()`) | **불가** (프록시 없음) | **검사됨** |
| final 클래스/메서드 | 제약 있음 | 가능 |

ASPECTJ 모드는 join point(메서드가 실제로 실행되는 그 지점)에 코드를 박기 때문에, 누가 어떻게 호출했는지와 무관하게 메서드 본문이 도는 순간 항상 인가가 적용된다. 이것이 self-invocation 과 비-빈 객체까지 보호할 수 있는 이유다.

## 같은 인가 엔진, 다른 진입점

여기서 자주 오해하는 부분을 짚어 둔다. ASPECTJ 모드가 **별도의 인가 로직**을 갖는 것은 아니다. 인가 판단(SpEL 평가, `AuthorizationManager` 호출, 필터링)은 두 모드 모두 `spring-security-core` 의 동일한 `MethodInterceptor` 들이 수행한다.

```
            ┌─ PROXY 모드  ──┐
메서드 호출 ─┤                ├──▶ AuthorizationManagerBeforeMethodInterceptor
            └─ ASPECTJ 모드 ─┘     PreFilterAuthorizationMethodInterceptor ...
              (aspects 모듈)        (= core, 두 모드가 공유)
```

`aspects` 모듈이 제공하는 것은 **"AspectJ join point 를 그 인터셉터가 이해하는 `MethodInvocation` 으로 바꿔 전달하는 어댑터 + pointcut 정의"** 뿐이다. 그래서 이 모듈은 작고, 인가 규칙이 바뀌어도 손댈 일이 거의 없다.

## 트레이드오프

ASPECTJ 모드는 강력하지만 공짜가 아니다.

- **빌드/런타임 구성이 필요**하다. CTW 는 AspectJ 컴파일러를, LTW 는 `-javaagent` 또는 `spring-instrument` 를 요구한다.
- 위빙 대상이 넓어 **성능·디버깅 비용**이 올라갈 수 있다.
- 그래서 대부분의 애플리케이션은 PROXY 모드로 충분하고, AspectJ 모드는 self-invocation/도메인 객체 보안 같은 **명확한 요구가 있을 때** 선택한다.

## 정리

`aspects` 모듈은 메서드 보안의 "두 번째 위빙 백엔드"다. PROXY 모드가 프록시 객체에 의존해 self-invocation 과 비-빈 객체를 놓치는 반면, AspectJ 위빙은 메서드 실행 지점 자체에 인가를 박아 그 사각지대를 없앤다. 단, 인가 판단은 core 의 인터셉터에 그대로 위임하므로 이 모듈의 책임은 **위빙 진입점과 어댑팅**으로 한정된다. 다음 챕터에서 그 애스펙트들의 실제 구조를 본다.
