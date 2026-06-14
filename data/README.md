# data — Spring Data 통합

> Spring Security 7.1.1-SNAPSHOT 해설서 · 모듈 챕터

## 한 줄 정의

`@Query` 같은 Spring Data 쿼리 안에서 `?#{ principal?.id }` 처럼 **현재 로그인한 사용자를 SpEL로 참조**할 수 있게 해 주고, Spring Data 리포지토리의 `@AuthorizeReturnObject` 반환값 프록시를 위한 **AOT(네이티브 이미지) 힌트**를 등록해 주는 얇은 통합 모듈이다.

## 이 모듈이 푸는 문제

데이터 접근 계층은 "현재 사용자"를 알아야 할 때가 많다. 예를 들어 "내게 온 메시지만 조회" 같은 쿼리는 조건절에 현재 principal의 id가 들어가야 한다. 이걸 매번 서비스 코드에서 인자로 넘기는 대신, **쿼리 정의 자체에 현재 사용자를 박아 넣고 싶다**.

Spring Data는 쿼리 SpEL에 임의의 객체를 확장(extension)으로 노출하는 SPI(`EvaluationContextExtension`)를 제공한다. 이 모듈은 그 SPI를 구현해 `SecurityContextHolder`의 `Authentication`을 `security`라는 이름으로 쿼리 SpEL에 꽂아 준다. 결과적으로 개발자는 쿼리 문자열에서 `principal`, `authentication`, `hasRole(...)` 등을 바로 쓸 수 있다.

추가로, GraalVM 네이티브 이미지처럼 런타임 리플렉션/프록시 생성이 제한된 환경에서 `@AuthorizeReturnObject`로 감싸지는 리포지토리 엔티티 타입을 빌드 타임에 미리 발견해 프록시 힌트를 등록하는 책임도 진다.

## 의존 / 연관 관계

```
        ┌────────────────────────────────────────────┐
        │                  data                       │
        │  ┌──────────────────────┐ ┌──────────────┐  │
        │  │ repository.query     │ │ aot.hint     │  │
        │  │ (SpEL 쿼리 확장)      │ │ (AOT 힌트)    │  │
        │  └──────────┬───────────┘ └──────┬───────┘  │
        └─────────────┼────────────────────┼──────────┘
                      │                    │
            ┌─────────▼─────────┐  ┌───────▼──────────────┐
            │ spring-security-  │  │ spring-security-core  │
            │ core (인증/표현식)  │  │ (aot.hint, authz)     │
            └───────────────────┘  └──────────────────────┘
                      │
            ┌─────────▼─────────────────────┐
            │ spring-data-commons            │
            │ (EvaluationContextExtension SPI)│
            └────────────────────────────────┘
```

- **spring-security-core**: `SecurityExpressionRoot`(SpEL 루트 객체), `SecurityContextHolder`/`Authentication`, `AuthorizationManagerFactory`, `PermissionEvaluator`, 그리고 AOT 쪽의 `SecurityHintsRegistrar`·`AuthorizeReturnObjectHintsRegistrar`·`AuthorizationProxyFactory`를 가져다 쓴다. ([../core/README.md](../core/README.md))
- **spring-data-commons**: 쿼리 SpEL 확장 SPI인 `EvaluationContextExtension`과, AOT에서 스캔 대상이 되는 `RepositoryFactoryBeanSupport`를 제공한다.
- **spring-security-config**: 이 모듈의 두 빈(특히 `AuthorizeReturnObjectDataHintsRegistrar`)을 인프라 빈으로 발행해 주는 쪽이다. data 모듈은 설정 자동화를 직접 하지 않고 "구현체"만 제공한다.

## 내부 목차

1. [01-SpEL-쿼리에-현재-사용자-노출.md](01-SpEL-쿼리에-현재-사용자-노출.md) — `SecurityEvaluationContextExtension`이 어떻게 `@Query` SpEL에 `principal`/`authentication`/`hasRole(...)`을 꽂는가
2. [02-AOT-반환객체-프록시-힌트.md](02-AOT-반환객체-프록시-힌트.md) — `AuthorizeReturnObjectDataHintsRegistrar`가 네이티브 이미지를 위해 리포지토리 엔티티 프록시 힌트를 등록하는 과정

## 전체 패키지 구조 ASCII 맵

```
data/src/main/java/org/springframework/security/data/
│
├── repository/query/
│   ├── SecurityEvaluationContextExtension.java   ← @Query SpEL에 "security" 확장 노출
│   └── package-info.java
│
└── aot/hint/
    ├── AuthorizeReturnObjectDataHintsRegistrar.java ← 리포지토리 엔티티 프록시 AOT 힌트
    └── package-info.java
```

> 모듈이 작다(클래스 2개). 하지만 두 클래스 각각이 "Spring Security ↔ Spring Data"라는 서로 다른 두 접점(런타임 SpEL / 빌드타임 AOT)을 담당하므로 챕터를 둘로 나눈다.
