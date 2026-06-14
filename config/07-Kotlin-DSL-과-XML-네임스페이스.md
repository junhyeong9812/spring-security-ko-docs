# 07. Kotlin DSL 과 XML 네임스페이스

## 무엇을 / 왜

같은 보안 정책을 표현하는 표면(surface)은 자바 DSL만이 아니다. `config` 모듈은 **코틀린 DSL**과 **XML 네임스페이스** 두 갈래를 더 제공한다. 둘 다 결국 1~6장에서 본 동일한 빌더/Configurer 또는 그에 상응하는 빈으로 번역된다. 이 장은 이 두 표면이 "내부의 자바 빌더"와 어떻게 연결되는지, 그리고 네이티브 이미지를 위한 AOT 힌트가 무엇인지 본다.

## 핵심 타입

```
   [Kotlin DSL]   config/src/main/kotlin/.../annotation/web/
     operator fun HttpSecurity.invoke { ... }   ←  http { ... } 문법
        └ HttpSecurityDsl  (각 기능마다 XxxDsl, 예: FormLoginDsl)
                 │  XxxDsl → Customizer<XxxConfigurer> 로 변환
                 ▼  내부적으로 자바 HttpSecurity.xxx(customizer) 호출

   [XML 네임스페이스]   config/src/main/java/.../config/
     SecurityNamespaceHandler  ←  <security:http>, <security:authentication-manager> ...
        └ 요소명 → BeanDefinitionParser  사전
                 │  XML → BeanDefinition 들로 번역
                 ▼  (자바 빌더 대신) 직접 빈 정의를 등록

   [AOT]   config/src/main/java/.../config/aot/hint/
     SecurityHintsAware / *RuntimeHints  ←  네이티브 이미지 리플렉션 힌트
```

- 소스: `config/src/main/kotlin/org/springframework/security/config/annotation/web/HttpSecurityDsl.kt`
- 소스: `config/src/main/kotlin/.../web/FormLoginDsl.kt`
- 소스: `config/src/main/java/org/springframework/security/config/SecurityNamespaceHandler.java`
- 소스: `config/src/main/java/org/springframework/security/config/http/HttpSecurityBeanDefinitionParser.java`
- 소스: `config/src/main/java/org/springframework/security/config/aot/`

## Kotlin DSL — invoke 로 자바 빌더를 감싸기

코틀린 DSL의 진입점은 `HttpSecurity`에 대한 확장 `operator fun invoke`다.

```kotlin
operator fun HttpSecurity.invoke(httpConfiguration: HttpSecurityDsl.() -> Unit) =
    HttpSecurityDsl(this, httpConfiguration).build()
```

덕분에 사용자는 `http { ... }` 블록을 쓸 수 있다(`http.invoke { ... }`의 설탕). 블록 안의 각 함수는 코틀린 친화적인 `XxxDsl` 객체를 만든 뒤, 그것을 자바 쪽 `Customizer<XxxConfigurer>`로 변환해 **자바 `HttpSecurity`의 같은 이름 메서드**에 그대로 위임한다.

```kotlin
fun formLogin(formLoginConfiguration: FormLoginDsl.() -> Unit) {
    val loginCustomizer = FormLoginDsl().apply(formLoginConfiguration).get()  // DSL → Customizer
    this.http.formLogin(loginCustomizer)                                       // 자바 빌더로 위임
}
```

`FormLoginDsl.get()`은 코틀린 프로퍼티(`loginPage`, `loginProcessingUrl` 등)를 자바 Configurer 호출로 펼치는 람다를 만든다.

```kotlin
internal fun get(): (FormLoginConfigurer<HttpSecurity>) -> Unit = { login ->
    loginPage?.also { login.loginPage(loginPage) }
    loginProcessingUrl?.also { login.loginProcessingUrl(loginProcessingUrl) }
    ...
    if (disabled) login.disable()
}
```

즉 **코틀린 DSL은 자바 DSL의 얇은 래퍼**다. 새 보안 로직을 더하지 않고, 코틀린 관용구(널 안전 프로퍼티, 트레일링 람다)로 같은 Configurer를 호출할 뿐이다. `build()`는 마지막에 `init()` 블록을 실행하고, 지정됐다면 `authenticationManager`를 자바 빌더에 넘긴다.

```kotlin
internal fun build() {
    init()
    authenticationManager?.also { this.http.authenticationManager(authenticationManager) }
}
```

`HttpSecurityDsl`의 `init` 블록은 `applyFunction1HttpSecurityDslBeans(...)`로 컨테이너의 `HttpSecurityDsl.() -> Unit` 빈들을 자동 적용한다 — 02장의 `Customizer<HttpSecurity>` 빈 자동 적용과 같은 발상의 코틀린판이다. `@SecurityMarker`(코틀린 `@DslMarker`)는 중첩 DSL 블록에서 바깥 수신자가 잘못 노출되는 것을 막는다.

리액티브용은 `config/src/main/kotlin/.../web/server/`의 `ServerHttpSecurityDsl` 등으로 대칭 제공된다.

## XML 네임스페이스 — 파서가 빈을 직접 찍어내기

자바/코틀린 DSL이 빌더를 거치는 반면, XML 설정(`<security:http>` 등)은 **빈 정의(BeanDefinition)를 직접 생성**하는 길이다. 진입점은 `SecurityNamespaceHandler`다.

```java
public final class SecurityNamespaceHandler implements NamespaceHandler {
    private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();

    private void loadParsers() {
        this.parsers.put(Elements.HTTP, new HttpSecurityBeanDefinitionParser());
        this.parsers.put(Elements.AUTHENTICATION_MANAGER, new AuthenticationManagerBeanDefinitionParser());
        this.parsers.put(Elements.USER_SERVICE, new UserServiceBeanDefinitionParser());
        this.parsers.put(Elements.METHOD_SECURITY, new MethodSecurityBeanDefinitionParser());
        this.parsers.put(Elements.FILTER_CHAIN, new FilterChainBeanDefinitionParser());
        this.parsers.put(Elements.LDAP_PROVIDER, new LdapProviderBeanDefinitionParser());
        ...  // oauth2, saml2, websocket, debug, http-firewall 등
    }

    public BeanDefinition parse(Element element, ParserContext pc) {
        BeanDefinitionParser parser = this.parsers.get(...);
        ...
        return parser.parse(element, pc);
    }
}
```

각 XML 요소는 전담 `BeanDefinitionParser`로 매핑된다. `config/src/main/java/org/springframework/security/config/http/`에만 40여 개의 파서가 있다. 예를 들어 `<security:http>` 하나는 `HttpSecurityBeanDefinitionParser`가 처리하는데, 내부적으로 `HttpConfigurationBuilder`와 `AuthenticationConfigBuilder`를 써서 자바 DSL이 Configurer로 하던 일(필터 생성·순서 부여·`FilterChainProxy` 구성)을 **BeanDefinition 수준에서** 재현한다.

```
   <security:http>
      <security:form-login .../>          FormLoginBeanDefinitionParser
      <security:logout .../>              LogoutBeanDefinitionParser
      <security:csrf .../>                CsrfBeanDefinitionParser
   </security:http>
        │  HttpSecurityBeanDefinitionParser
        ▼
   HttpConfigurationBuilder + AuthenticationConfigBuilder
        │  필터 빈 정의들을 순서대로 등록
        ▼
   FilterChainProxy 빈 정의  (자바 DSL의 WebSecurity.performBuild() 와 같은 결과물)
```

흥미로운 점은 **자바 DSL과 XML이 서로 다른 코드 경로**라는 것이다. XML은 `HttpSecurity` 빌더를 쓰지 않고 자체 빌더(`HttpConfigurationBuilder` 등)로 같은 종류의 빈을 만든다. 산출물(`FilterChainProxy`, 필터들, `AuthenticationManager`)은 같지만, 조립 코드는 별도다. 그래서 XML 파서들은 자바 Configurer와 기능 패리티를 맞추기 위해 별도로 유지된다. `loadParsers()`는 `init()`이 아니라 실제 요소를 만나는 시점에도 호출될 수 있게 되어 있어(주석 SEC-1455), 필요할 때 로드된다.

`DefaultFilterChainValidator`는 XML로 구성된 필터 체인의 순서·구성이 올바른지(예: 인증 필터가 인가 필터보다 앞인지) 검증한다.

## AOT / 네이티브 이미지 힌트

`config/src/main/java/org/springframework/security/config/aot/`는 GraalVM 네이티브 이미지를 위한 리플렉션/프록시 힌트를 등록한다. 보안 설정은 리플렉션(`applyTopLevelCustomizers`의 메서드 스캔, 표현식 평가 등)과 CGLIB 프록시(메서드 보안)를 많이 쓰므로, 네이티브에서 이들이 동작하도록 힌트가 필요하다. 01장에서 본 `AutowireBeanFactoryObjectPostProcessor.initializeBeanIfNeeded`가 네이티브에서 CGLIB 프록시를 특별 취급하는 것도 같은 맥락이다(이슈 gh-14825).

## 핵심 메서드

- **`operator fun HttpSecurity.invoke { }`** — 코틀린 `http { }` 진입점. 자바 빌더를 감싼다.
- **`XxxDsl.get()`** — 코틀린 DSL 상태 → 자바 `Customizer<XxxConfigurer>` 변환.
- **`SecurityNamespaceHandler.loadParsers()`** — XML 요소명 → 파서 사전 구축.
- **`HttpSecurityBeanDefinitionParser.parse(...)`** — `<http>`를 필터 체인 빈 정의들로 번역.

## 설계 포인트 / 확장점

- **표면의 다양성, 결과의 단일성**: 자바 DSL·코틀린 DSL·XML 모두 같은 런타임 객체(`FilterChainProxy`, 필터들, `AuthenticationManager`)로 수렴한다.
- **코틀린 DSL = 얇은 위임**: 새 의미 없이 관용구만 제공 → 자바 기능과 자동으로 보조를 맞춘다.
- **XML = 독립 경로**: BeanDefinition을 직접 만들어 빌더 없이 같은 결과를 낸다. 레거시/세밀한 빈 제어용.
- **확장점**: 커스텀 XML 요소는 새 `BeanDefinitionParser`를 `loadParsers()`에 추가하는 식으로(스프링 네임스페이스 메커니즘) 확장 가능. 코틀린은 새 `XxxDsl`을 추가해 `HttpSecurityDsl`에 위임 함수를 더한다.

## 정리

`config` 모듈은 동일한 보안 정책을 세 표면으로 받는다. 코틀린 DSL은 `http { }`의 `invoke` 확장으로 자바 빌더/Configurer를 얇게 감싸고, XML 네임스페이스는 `SecurityNamespaceHandler`가 요소별 `BeanDefinitionParser`로 빈 정의를 직접 찍어낸다. 표면이 무엇이든 최종 산출물은 1~6장에서 본 필터 체인·인증 매니저·메서드 인터셉터로 같다. AOT 힌트는 이 리플렉션/프록시 기반 조립이 네이티브 이미지에서도 동작하도록 보조한다. 이로써 `config` 모듈 전체 — 빌더 엔진에서 시작해 웹/인증/메서드 보안을 거쳐 다양한 설정 표면까지 — 의 한 바퀴가 완성된다.
