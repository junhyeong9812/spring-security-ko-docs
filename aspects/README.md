# aspects — AspectJ 기반 메서드 보안

> **한 줄 정의**: `@PreAuthorize`/`@PostAuthorize`/`@Secured`/`@PreFilter`/`@PostFilter` 가 붙은 메서드를 **Spring AOP 프록시가 아니라 AspectJ 애스펙트(.aj)로 위빙**해서 인가 로직을 메서드 호출에 끼워 넣는 모듈.

---

## 1. 이 모듈이 푸는 문제

Spring Security 의 메서드 보안(method security)은 기본적으로 **Spring AOP 프록시** 위에서 동작한다. 즉 `@PreAuthorize` 가 붙은 빈은 컨테이너가 프록시 객체로 감싸고, 그 프록시를 거쳐 호출될 때만 인가 검사가 일어난다. 이 방식에는 구조적 한계가 있다.

- **프록시를 거치지 않는 호출은 검사되지 않는다.** 같은 클래스 안에서 `this.someSecuredMethod()` 처럼 자기 자신을 직접 호출하면(self-invocation) 프록시를 우회하므로 인가가 적용되지 않는다.
- **스프링 빈이 아닌 객체는 보호할 수 없다.** `new` 로 직접 생성한 도메인 객체의 메서드에는 프록시가 없다.
- **인터페이스 기반 JDK 프록시의 제약**, final 클래스/메서드 보호 불가 등 프록시 모델 고유의 제약이 따라온다.

`aspects` 모듈은 이 문제를 **AspectJ 위빙**으로 해결한다. 컴파일 타임(CTW) 또는 로드 타임(LTW)에 바이트코드 자체에 인가 호출을 직접 짜 넣기 때문에, self-invocation 이든 `new` 로 만든 객체든 **메서드 본문이 실행되는 그 지점(join point)** 에서 항상 인가가 적용된다.

중요한 설계 결정 하나: 이 모듈은 **인가 판단 로직을 직접 구현하지 않는다.** 실제 인가는 `spring-security-core` 의 `AuthorizationManagerBeforeMethodInterceptor`, `PreFilterAuthorizationMethodInterceptor` 같은 `MethodInterceptor` 들이 담당한다. 애스펙트는 단지 AspectJ join point 를 AOP Alliance `MethodInvocation` 으로 변환해 그 인터셉터에 넘겨주는 **얇은 어댑터**일 뿐이다. 그래서 프록시 모드와 AspectJ 모드는 **같은 인가 엔진을 공유**하고, 위빙 메커니즘만 다르다.

---

## 2. 의존 / 연관 관계

```
        ┌──────────────────────────────────────────────┐
        │                  aspects                       │
        │  PreAuthorizeAspect / PostAuthorizeAspect /    │
        │  PreFilterAspect / PostFilterAspect /          │
        │  SecuredAspect  (모두 .aj 애스펙트)             │
        └───────────────┬──────────────────────────────┘
                        │ securityInterceptor 로 위임
                        ▼
        ┌──────────────────────────────────────────────┐
        │  spring-security-core                          │
        │  authorization.method.*MethodInterceptor       │
        │  (실제 인가 판단 엔진 — 프록시 모드와 공유)      │
        └──────────────────────────────────────────────┘
                        ▲
                        │ 애스펙트 빈 등록 + 인터셉터 주입
        ┌──────────────────────────────────────────────┐
        │  spring-security-config                         │
        │  @EnableMethodSecurity(mode = ASPECTJ)         │
        │  MethodSecurityAspectJAutoProxyRegistrar       │
        └──────────────────────────────────────────────┘
```

- **core (`org.springframework.security.authorization.method`)**: 실제 인가/필터링을 수행하는 `MethodInterceptor` 구현들. 애스펙트가 이쪽으로 위임한다. (참고: [../core/README.md](../core/README.md))
- **config (`...config.annotation.method.configuration`)**: `@EnableMethodSecurity(mode = AdviceMode.ASPECTJ)` 가 선택되면 `MethodSecurityAspectJAutoProxyRegistrar` 가 각 애스펙트 빈을 `aspectOf()` 로 등록하고 core 의 인터셉터를 주입한다. (참고: [../config/README.md](../config/README.md))
- **AspectJ 런타임/위버**: `org.aspectj` (aspectjrt, aspectjweaver). `.aj` 파일은 AspectJ 컴파일러(ajc) 또는 로드타임 위버가 처리한다.
- **AOP Alliance (`org.aopalliance.intercept`)**: `MethodInterceptor`, `MethodInvocation` 추상화. 애스펙트와 인터셉터를 잇는 공통 계약.

---

## 3. 내부 목차

1. [01-왜-AspectJ-위빙인가.md](01-왜-AspectJ-위빙인가.md) — 프록시 모드 vs AspectJ 모드, 이 모듈이 존재하는 이유와 차이.
2. [02-애스펙트-구조와-위빙-흐름.md](02-애스펙트-구조와-위빙-흐름.md) — `AbstractMethodInterceptorAspect`, 5개 구체 애스펙트, pointcut/around advice, `JoinPointMethodInvocation` 어댑터의 동작.
3. [03-설정-연결과-레거시-애스펙트.md](03-설정-연결과-레거시-애스펙트.md) — config 의 위빙 활성화 흐름, `aspectOf` 빈 등록, deprecated `AnnotationSecurityAspect`.

---

## 4. 전체 패키지 구조 ASCII 맵

```
aspects/src/main/java/org/springframework/security/
│
├── authorization/method/aspectj/        ← 현행(7.x) 애스펙트
│   ├── AbstractMethodInterceptorAspect.aj   추상 애스펙트: around advice 공통 골격
│   ├── JoinPointMethodInvocation.aj         AspectJ JoinPoint → MethodInvocation 어댑터
│   ├── PreAuthorizeAspect.aj                 @PreAuthorize 메서드 매칭
│   ├── PostAuthorizeAspect.aj                @PostAuthorize 메서드 매칭
│   ├── PreFilterAspect.aj                    @PreFilter 메서드 매칭
│   ├── PostFilterAspect.aj                   @PostFilter 메서드 매칭
│   └── SecuredAspect.aj                      @Secured 메서드/타입 매칭
│
└── access/intercept/aspectj/aspect/      ← 레거시(@Deprecated)
    └── AnnotationSecurityAspect.aj           구버전 단일 애스펙트
                                              (AspectJMethodSecurityInterceptor 사용)
```

> 디렉토리는 단 두 개, 파일은 8개뿐인 **작은 모듈**이다. 핵심 가치는 코드량이 아니라 "프록시 없이도 메서드 보안이 적용되게 하는 위빙 진입점"이라는 역할에 있다.
