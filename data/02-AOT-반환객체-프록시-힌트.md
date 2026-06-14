# 02. AOT 반환객체 프록시 힌트 — AuthorizeReturnObjectDataHintsRegistrar

## 무엇을 / 왜

Spring Security에는 `@AuthorizeReturnObject`라는 기능이 있다. 메서드가 돌려주는 도메인 객체를 **보안 프록시로 감싸**, 그 객체의 getter 등을 다시 인가 규칙(`@PreAuthorize` 등)으로 보호하는 기능이다. 이 프록시는 런타임에 CGLIB 등으로 동적으로 생성된다.

문제는 GraalVM **네이티브 이미지**다. 네이티브 이미지는 빌드 타임에 닫힌 세계(closed world)를 가정하므로, 런타임에 어떤 클래스의 프록시가 필요한지 **미리 알려 줘야**(RuntimeHints 등록) 한다. 일반적인 메서드라면 코어의 AOT 처리기가 `@AuthorizeReturnObject`가 붙은 타입을 찾아낸다. 그러나 **Spring Data 리포지토리**는 사정이 다르다. 리포지토리는 인터페이스이고 그 구현체는 `RepositoryFactoryBeanSupport`를 통해 동적으로 만들어지므로, 일반 빈 스캔으로는 "이 리포지토리가 어떤 엔티티를 반환하는지"가 잘 드러나지 않는다.

`AuthorizeReturnObjectDataHintsRegistrar`는 바로 이 틈을 메운다. 모든 빈을 훑어 **Spring Data 리포지토리 팩토리 빈**을 찾아내고, 해당 리포지토리(또는 그 메서드)에 `@AuthorizeReturnObject`가 있으면 **그 리포지토리가 다루는 엔티티 타입**을 프록시 대상으로 등록한다.

## 핵심 타입

```
  ┌─────────────────────────────────────────────────────────┐
  │ SecurityHintsRegistrar  (spring-security-core)           │
  │   void registerHints(RuntimeHints, BeanFactory)          │
  └──────────────────────▲──────────────────────────────────┘
                         │ implements
  ┌──────────────────────┴──────────────────────────────────┐
  │ AuthorizeReturnObjectDataHintsRegistrar                  │
  │   - AuthorizationProxyFactory proxyFactory               │
  │   - SecurityAnnotationScanner<AuthorizeReturnObject>     │
  │   registerHints(...) → 엔티티 수집 후 위임                  │
  └──────────────────────┬──────────────────────────────────┘
                         │ 수집한 엔티티 목록을 위임
  ┌──────────────────────▼──────────────────────────────────┐
  │ AuthorizeReturnObjectHintsRegistrar (spring-security-core)│
  │   실제 프록시 RuntimeHints 등록 + 반환 그래프 순회          │
  └──────────────────────────────────────────────────────────┘
```

- **`SecurityHintsRegistrar`**: 코어가 정의한, AOT 처리 단계에서 호출되는 보안 힌트 등록 인터페이스. `RuntimeHints`와 `ConfigurableListableBeanFactory`를 받는다.
- **`AuthorizeReturnObjectDataHintsRegistrar`**: 이 챕터의 주인공. Spring Data 리포지토리에 특화된 "엔티티 발견기". 발견만 하고, 실제 힌트 등록은 코어의 `AuthorizeReturnObjectHintsRegistrar`에 위임한다.
- **`AuthorizationProxyFactory`**: 보안 프록시를 만드는 코어 팩토리. 어떤 타입을 프록시로 감쌀지 결정·생성하는 주체다.
- **`SecurityAnnotationScanner`**: 메서드/클래스에서 `@AuthorizeReturnObject`를 (메타 애너테이션 포함) 견고하게 찾아 주는 스캐너.

## 동작 흐름

### 빌드 타임 AOT 처리

```
 AOT 처리 단계 (빌드 타임)
   registerHints(RuntimeHints hints, BeanFactory beanFactory)
        │
        ▼
 for 모든 BeanDefinitionName:
        │
        ├─ type = beanDef.getResolvableType()
        │   RepositoryFactoryBeanSupport 가 아니면 ─────────► skip (continue)
        │
        │  (리포지토리 팩토리 빈이면)
        ├─ generics = type.resolveGenerics()
        │     generics[0] = repository 인터페이스
        │     generics[1] = entity 타입
        │
        ├─ [A] 빈에 직접 @AuthorizeReturnObject 가 있나?
        │        findAnnotationOnBean(name, AuthorizeReturnObject.class) != null
        │        → toProxy.add(entity); continue
        │
        └─ [B] 아니면 repository 의 선언 메서드들을 스캔:
                 for method in repository.getDeclaredMethods():
                     scanner.scan(method, repository) != null ?
                         → toProxy.add(entity); break   // 하나라도 있으면 엔티티 추가
        │
        ▼
 new AuthorizeReturnObjectHintsRegistrar(proxyFactory, toProxy)
       .registerHints(hints, beanFactory)
        │
        ▼
 코어가 toProxy 의 각 엔티티에 대해 프록시 RuntimeHints 등록
 + 반환 타입 그래프를 추가로 순회(traverse)
```

흐름의 요점은 세 가지다. (1) **대상 좁히기** — `RepositoryFactoryBeanSupport`가 아닌 빈은 즉시 건너뛴다. (2) **엔티티 추출** — 리포지토리 팩토리 빈의 제네릭 시그니처에서 `[리포지토리, 엔티티]`를 뽑는다. (3) **낙관적 등록** — 리포지토리의 메서드 중 **하나라도** `@AuthorizeReturnObject`를 쓰면, 그 엔티티 전체를 프록시 대상으로 올린다(메서드별로 정밀히 구분하지 않는다).

## 핵심 메서드

### `registerHints(RuntimeHints, ConfigurableListableBeanFactory)`

```java
for (String name : beanFactory.getBeanDefinitionNames()) {
    ResolvableType type = beanFactory.getBeanDefinition(name).getResolvableType();
    if (!RepositoryFactoryBeanSupport.class.isAssignableFrom(type.toClass())) {
        continue;
    }
    @Nullable Class<?>[] generics = type.resolveGenerics();
    @Nullable Class<?> entity = generics[1];
    AuthorizeReturnObject authorize = beanFactory.findAnnotationOnBean(name, AuthorizeReturnObject.class);
    if (authorize != null) {
        toProxy.add(entity);
        continue;
    }
    ...
}
```

- `getResolvableType()`으로 빈 정의의 제네릭 타입 정보를 얻는다. 리포지토리 팩토리 빈은 보통 `RepositoryFactoryBeanSupport<T extends Repository<S, ID>, S, ID>` 형태라 `resolveGenerics()`의 인덱스 0이 리포지토리 인터페이스, 인덱스 1이 엔티티(S)에 해당한다.
- **경로 [A]**: 빈 레벨(즉 리포지토리 전체)에 `@AuthorizeReturnObject`가 붙어 있으면 모든 반환을 감싸겠다는 의도이므로, 엔티티를 바로 등록하고 다음 빈으로 넘어간다.

```java
    @Nullable Class<?> repository = generics[0];
    Assert.state(repository != null, "Repository resolved from " + type + " cannot be null");
    for (Method method : repository.getDeclaredMethods()) {
        AuthorizeReturnObject returnObject = this.scanner.scan(method, repository);
        if (returnObject == null) {
            continue;
        }
        // optimistically assume that the entity needs wrapping if any of the
        // repository methods use @AuthorizeReturnObject
        toProxy.add(entity);
        break;
    }
```

- **경로 [B]**: 리포지토리의 선언 메서드를 하나씩 스캐너로 검사한다. `@AuthorizeReturnObject`가 붙은 메서드를 **처음 발견하는 즉시** 엔티티를 등록하고 `break`로 멈춘다. 주석대로 "메서드 하나라도 쓰면 엔티티 래핑이 필요하다고 낙관적으로 가정"하는 전략이다.

### 마지막 위임

```java
new AuthorizeReturnObjectHintsRegistrar(this.proxyFactory, toProxy)
    .registerHints(hints, beanFactory);
```

이 클래스는 "어떤 엔티티가 프록시 대상인가"만 결정한다. 실제 `RuntimeHints` 등록과, 그 엔티티가 반환하는 다른 타입들까지 따라가며 추가 힌트를 등록하는 **그래프 순회**는 코어의 `AuthorizeReturnObjectHintsRegistrar`가 담당한다. 책임을 깔끔히 분리한 구조다.

## 설계 포인트 / 확장점

- **인프라 빈으로 발행**: 보통 `spring-security-config` 모듈이 이 클래스를 인프라 빈(`@Role(BeanDefinition.ROLE_INFRASTRUCTURE)`)으로 자동 등록한다. 직접 등록해야 한다면 Javadoc 예시처럼 정적 `@Bean` + `@Role(ROLE_INFRASTRUCTURE)`로 발행해, AOT 처리 시점에 `BeanFactory`를 깨우지 않도록 한다.

  ```java
  @Bean
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  static SecurityHintsRegistrar proxyThese(AuthorizationProxyFactory proxyFactory) {
      return new AuthorizeReturnObjectDataHintsRegistrar(proxyFactory);
  }
  ```

- **불변/스레드 무관 설계**: `final` 필드만 갖고, 스캐너는 `SecurityAnnotationScanners.requireUnique(...)`로 한 번 구성한다. `visitedClasses` 같은 보조 상태가 있지만 AOT는 빌드 타임 1회 실행이라 동시성 문제는 없다.

- **낙관적 정밀도 트레이드오프**: 메서드 단위로 "어느 반환만 프록시 대상"인지 가리지 않고, 하나라도 걸리면 엔티티 전체를 등록한다. AOT 힌트는 "혹시 모를 것을 빠뜨리지 않는 것"이 더 중요하기 때문에, 정밀도보다 누락 방지(보수성)를 택한 합리적 선택이다.

## 정리

`AuthorizeReturnObjectDataHintsRegistrar`는 네이티브 이미지 빌드 시, Spring Data 리포지토리에서 `@AuthorizeReturnObject`로 감싸질 엔티티 타입을 찾아 코어의 힌트 등록기에 넘겨 주는 AOT 어댑터다. 모든 빈 중 `RepositoryFactoryBeanSupport`만 골라 제네릭에서 엔티티를 뽑고, 빈 레벨 또는 메서드 레벨 애너테이션이 있으면 엔티티를 프록시 대상에 올린다. 발견은 이 클래스가, 실제 `RuntimeHints` 등록과 그래프 순회는 `AuthorizeReturnObjectHintsRegistrar`가 맡는 책임 분리 구조다.
