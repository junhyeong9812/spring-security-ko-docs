# 01. 빌더와 Configurer 기반구조

## 무엇을 / 왜

`config` 모듈의 거의 모든 설정 클래스(`HttpSecurity`, `WebSecurity`, `AuthenticationManagerBuilder`)는 똑같은 골격 위에 서 있다. 그 골격이 바로 **`SecurityBuilder` + `SecurityConfigurer` + `ObjectPostProcessor`** 세 가지다. 이 장을 이해하면 나머지 장은 "이 골격에 무엇을 끼웠는가"의 변형으로 읽힌다.

핵심 아이디어는 이렇다. 하나의 복잡한 객체(예: `FilterChainProxy`)를 만드는 일을, **(a) 결과물을 책임지는 빌더 하나**와 **(b) 그 빌더의 일부분만 설정하는 작은 전략 객체 여러 개(Configurer)** 로 쪼갠다. 그리고 빌더가 조립한 객체가 스프링 빈처럼 동작하도록 **후처리기(ObjectPostProcessor)** 가 `@Autowired`·`afterPropertiesSet()`·`Aware` 콜백을 대신 걸어 준다.

## 핵심 타입

```
        SecurityBuilder<O>            "O 를 build() 해서 돌려준다"
              ▲
              │ extends
   AbstractSecurityBuilder<O>         build() 1회 보장 (AlreadyBuiltException)
              ▲
              │ extends
   AbstractConfiguredSecurityBuilder<O,B>   ◀── 진짜 엔진. Configurer 들을 모아
              ▲                               init→configure→performBuild 로 실행
      ┌───────┼─────────────────┐
      │       │                 │
 HttpSecurity WebSecurity  AuthenticationManagerBuilder
 (→DefaultSecurityFilterChain) (→Filter/FilterChainProxy) (→ProviderManager)


   SecurityConfigurer<O,B>           init(B) / configure(B) 두 콜백
              ▲
   SecurityConfigurerAdapter<O,B>    빈 구현 + ObjectPostProcessor 보유
              ▲
   AbstractHttpConfigurer<T,B>       HttpSecurity 전용 베이스 (04장)


   ObjectPostProcessor<T>            postProcess(obj) → 빈 초기화/Aware/autowire
              ▲
   AutowireBeanFactoryObjectPostProcessor  실제 BeanFactory 위임 구현
```

- 소스: `config/src/main/java/org/springframework/security/config/annotation/AbstractConfiguredSecurityBuilder.java`
- 소스: `.../annotation/SecurityConfigurer.java`, `.../SecurityConfigurerAdapter.java`
- 소스: `config/src/main/java/org/springframework/security/config/ObjectPostProcessor.java`

### SecurityBuilder / SecurityConfigurer

`SecurityBuilder<O>`는 메서드가 `O build()` 단 하나다. `SecurityConfigurer<O, B>`는 두 콜백을 가진다.

```java
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {
    void init(B builder);       // 공유 상태만 만든다 (다른 Configurer가 참조할 것)
    void configure(B builder);  // 실제 속성/필터를 빌더에 세팅한다
}
```

주석이 강조하는 규칙이 중요하다. `init()`에서는 **공유 객체(shared object)만** 만들고, 빌더에 최종 속성을 박는 일은 `configure()`에서 한다. 이 2단계 분리 덕분에 모든 Configurer의 `init()`이 먼저 끝난 뒤에 `configure()`가 돌아, 서로의 공유 상태를 안전하게 참조할 수 있다.

### ObjectPostProcessor — 빈이 아닌 객체를 빈처럼

`config`이 만드는 필터·매니저들은 `new`로 생성된, 스프링이 모르는 객체다. 그대로 두면 `@Autowired`도 `afterPropertiesSet()`도 호출되지 않는다. 그래서 모든 생성 지점은 `postProcess()`를 거친다.

```java
final class AutowireBeanFactoryObjectPostProcessor implements ObjectPostProcessor<Object>, ... {
    public <T> T postProcess(T object) {
        T result = initializeBeanIfNeeded(object);   // initializeBean(): Aware + afterPropertiesSet
        this.autowireBeanFactory.autowireBean(object); // @Autowired 필드 주입
        if (result instanceof DisposableBean) this.disposableBeans.add(...);   // destroy() 추적
        if (result instanceof SmartInitializingSingleton) this.smartSingletons.add(...);
        return result;
    }
}
```

즉 `postProcess(new CsrfFilter(...))` 한 번이면 그 필터는 의존성 주입을 받고, 컨테이너 종료 시 `destroy()`까지 호출된다. 이것이 "DSL로 만든 객체가 스프링 빈처럼 동작하는" 비밀이다. 이 빈은 [02장](02-EnableWebSecurity-와-빈-구성.md)의 `ObjectPostProcessorConfiguration`이 등록한다.

## 동작 흐름: doBuild 생애주기

`AbstractConfiguredSecurityBuilder.doBuild()`가 전체 조립을 지휘한다. 상태 머신(UNBUILT → INITIALIZING → CONFIGURING → BUILDING → BUILT)을 거친다.

```
   build() (한 번만, AbstractSecurityBuilder가 보장)
     │
     ▼
   doBuild()  ──────────────────────────────────────────────
     │  state = INITIALIZING
     ├─▶ beforeInit()                       (서브클래스 훅)
     ├─▶ init()  ── 모든 Configurer.init(this) 호출
     │        └─ init 중 새로 추가된 Configurer 도 빠짐없이 init  (while 루프)
     │  state = CONFIGURING
     ├─▶ beforeConfigure()                  (서브클래스 훅; HttpSecurity는 여기서
     │                                       AuthenticationManager 를 build 해 공유객체로)
     ├─▶ configure() ── 모든 Configurer.configure(this) 호출
     │  state = BUILDING
     ├─▶ performBuild()  ◀── 추상 메서드. 진짜 결과 객체 생성
     │        HttpSecurity → DefaultSecurityFilterChain(필터 정렬)
     │        WebSecurity  → FilterChainProxy
     │        AuthManagerBuilder → ProviderManager
     │  state = BUILT
     ▼
   결과 객체 반환
```

코드로 보면 핵심은 이 짧은 블록이다.

```java
protected final O doBuild() {
    synchronized (this.configurers) {
        this.buildState = BuildState.INITIALIZING;
        beforeInit();  init();
        this.buildState = BuildState.CONFIGURING;
        beforeConfigure();  configure();
        this.buildState = BuildState.BUILDING;
        O result = performBuild();
        this.buildState = BuildState.BUILT;
        return result;
    }
}
```

### Configurer를 어떻게 모으는가 — apply / with / getOrApply

빌더는 Configurer들을 `LinkedHashMap<Class, List<Configurer>>`에 보관한다(삽입 순서 유지). 추가 경로는 세 가지다.

- **`apply(configurer)`** — 같은 클래스의 Configurer를 덮어쓴다(기본). `WebSecurity`에 `WebSecurityConfigurer`를 등록할 때 쓰인다.
- **`with(configurer, customizer)`** (since 6.2/7.0) — `setBuilder(this)`로 빌더 역참조를 걸고 `ObjectPostProcessor`를 넘겨준 뒤 추가한다. 외부에서 만든 커스텀 Configurer를 끼울 때의 공개 API.
- **`getOrApply(...)`** — `HttpSecurity` 내부에서 `http.formLogin(...)` 같은 호출이 쓰는 비공개 헬퍼. 이미 같은 Configurer가 있으면 그걸 반환, 없으면 새로 `apply`한다. 그래서 `formLogin()`을 두 번 호출해도 하나의 `FormLoginConfigurer`가 누적 설정된다([04장](04-HTTP-Configurer-카탈로그.md)).

`with`의 본문이 이 패턴을 잘 보여 준다.

```java
public <C extends SecurityConfigurerAdapter<O, B>> B with(C configurer, Customizer<C> customizer) {
    configurer.addObjectPostProcessor(this.objectPostProcessor);
    configurer.setBuilder((B) this);
    add(configurer);
    customizer.customize(configurer);   // 사용자의 람다 { ... } 실행
    return (B) this;
}
```

## 핵심 메서드

- **`setSharedObject(Class, obj)` / `getSharedObject(Class)`** — Configurer들이 서로 데이터를 주고받는 칠판. 예: `HttpSecurity`는 `AuthenticationManager`, `ApplicationContext`, `PathPatternRequestMatcher.Builder`를 공유 객체로 둔다. 한 Configurer가 `init`에서 채워 두면 다른 Configurer가 `configure`에서 읽는다.
- **`postProcess(obj)`** — 빌더가 가진 `ObjectPostProcessor`로 위임. Configurer가 필터를 만들 때마다 호출해 빈 초기화를 보장한다.
- **`getConfigurer(Class)` / `removeConfigurer(Class)`** — 특정 Configurer를 조회/제거. `AbstractHttpConfigurer.disable()`이 `removeConfigurer(getClass())`로 자기 자신을 빼는 식으로 `csrf(c -> c.disable())`을 구현한다.

### Customizer — DSL 람다의 타입

```java
public interface Customizer<T> {
    void customize(T t);
    static <T> Customizer<T> withDefaults() { return (t) -> {}; }
}
```

`http.csrf(withDefaults())`의 `withDefaults()`는 "아무것도 더 손대지 않고 기본값을 쓴다"는 의미의 빈 람다다. 이 한 타입이 자바 DSL 전체의 일관된 모양(`xxx(Customizer<XxxConfigurer> c)`)을 만든다.

## 설계 포인트 / 확장점

- **Builder + Strategy(Configurer)**: 거대한 `if/else` 설정 메서드 대신, 관심사별 Configurer로 쪼개 조합·교체가 가능하다. `formLogin`을 끄려면 그 Configurer만 제거하면 된다.
- **2-phase(init/configure)**: Configurer 간 의존 순서 문제를 "전부 init 먼저, 그다음 전부 configure"로 해소. 공유 객체 패턴과 짝을 이룬다.
- **후처리기 추상화**: 테스트에서는 `ObjectPostProcessor.identity()`(아무 것도 안 하는 구현)를 끼워 BeanFactory 없이도 빌더를 돌릴 수 있다.
- **확장점**: 사용자는 `AbstractHttpConfigurer`를 상속해 자기 필터를 추가하는 Configurer를 만들고 `http.with(myConfigurer, ...)`로 끼울 수 있다. `ObjectPostProcessor`를 직접 구현해 모든 생성 객체에 후크를 걸 수도 있다.

## 정리

`config` 모듈의 모든 설정은 결국 `AbstractConfiguredSecurityBuilder`라는 한 엔진이 `init → configure → performBuild`로 Configurer들을 실행하고, 그 산출물을 `ObjectPostProcessor`가 빈처럼 초기화하는 과정이다. 빌더는 "무엇을 만들지", Configurer는 "어느 부분을 어떻게", 후처리기는 "스프링과 어떻게 통합되는지"를 각각 책임진다. 다음 장부터는 이 엔진이 웹 보안(`HttpSecurity`/`WebSecurity`)과 인증·메서드 보안에서 구체적으로 어떻게 쓰이는지를 본다.
