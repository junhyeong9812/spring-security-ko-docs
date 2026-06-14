# acl 모듈 — 도메인 객체 수준 접근 제어 (ACL)

> Spring Security 7.1.1-SNAPSHOT 기준. 소스: `acl/src/main/java/org/springframework/security/acls/`

## 한 줄 정의

`acl` 모듈은 "이 사용자가 **이 특정 도메인 객체 인스턴스**에 대해 읽기/쓰기/삭제 권한을 갖는가?"를 판정하는 **도메인 객체 수준(instance-level) 인가** 서브시스템이다.

## 이 모듈이 푸는 문제

일반적인 Spring Security 인가는 "역할(ROLE_ADMIN)을 가졌는가", "이 URL/메서드에 접근할 수 있는가" 같은 **타입 수준** 결정을 한다. 하지만 현실에서는 더 잘게 쪼갠 결정이 필요하다.

- "사용자 A는 자신이 만든 게시글 #42 는 수정할 수 있지만, 게시글 #43 은 읽기만 가능하다."
- "프로젝트 폴더의 권한이 그 안의 문서들에게 상속된다."
- "관리자 역할이 없어도, 특정 객체에 대한 ADMINISTRATION 권한만 있으면 그 객체의 ACL을 바꿀 수 있다."

이런 결정은 객체 **인스턴스마다 다른 권한 테이블(ACL)** 을 들고 있어야 풀 수 있다. `acl` 모듈은 이 ACL을 모델링하고(누가 = `Sid`, 무엇에 = `ObjectIdentity`, 어떤 권한 = `Permission`, 허용/거부 = `AccessControlEntry`), 이를 관계형 DB에 저장·조회·캐싱하며, 최종적으로 `@PreAuthorize("hasPermission(#post, 'WRITE')")` 같은 SpEL 표현식과 연결한다.

## 핵심 도메인 언어 (4요소)

```
   누가         무엇에            어떤 권한          허용/거부
  ┌─────┐    ┌──────────────┐   ┌──────────┐    ┌──────────┐
  │ Sid │ ×  │ ObjectIdentity│ × │Permission│ →  │ granting │
  └─────┘    └──────────────┘   └──────────┘    └──────────┘
  (보안주체)   (도메인객체 식별)    (비트마스크)      (true/false)
       └──────────────┬───────────────────┘
                      ▼
            AccessControlEntry (ACE) = ACL의 한 줄
                      ▼
                  Acl = 한 ObjectIdentity 에 대한 ACE 목록 + 부모/상속
```

## 의존 / 연관 관계

- **`access` (spring-security-core 내 access 패키지)** — `PermissionEvaluator`, `PermissionCacheOptimizer` 인터페이스를 이 모듈이 구현한다. SpEL `hasPermission()` 의 진입점.
- **`core`** — `Authentication`, `GrantedAuthority`, `SecurityContextHolder`, `RoleHierarchy` 를 사용해 현재 principal 로부터 `Sid` 목록을 만들고 ACL 변경 권한을 검사한다.
- **`spring-jdbc` / `spring-tx`** — 관계형 DB 영속화(`JdbcAclService`)에 `JdbcTemplate` 과 트랜잭션 동기화를 사용.
- **`spring-context` (Cache 추상화)** — `SpringCacheBasedAclCache` 가 `org.springframework.cache.Cache` 에 위임.

사용처: 메서드 보안(`@PreAuthorize`/`@PostFilter`) 또는 표현식 기반 웹 인가에서 `AclPermissionEvaluator` 를 `PermissionEvaluator` 빈으로 등록하면, `hasPermission(...)` 호출이 이 모듈의 ACL 조회·판정으로 흘러간다.

## 하위 챕터 목차

1. [01-핵심-개념과-도메인-모델.md](01-핵심-개념과-도메인-모델.md) — `model` 패키지의 인터페이스들(Acl/Sid/ObjectIdentity/AccessControlEntry/Permission)과 서비스 계약, 그리고 ACL의 유비쿼터스 언어.
2. [02-권한-마스크와-Permission.md](02-권한-마스크와-Permission.md) — 권한을 32비트 정수 마스크로 표현하는 방식, `BasePermission`/`CumulativePermission`, `DefaultPermissionFactory`.
3. [03-인가-결정-흐름.md](03-인가-결정-흐름.md) — `AclImpl.isGranted()` → `DefaultPermissionGrantingStrategy` 의 ACE 스캔/거부/상속 알고리즘, 그리고 ACL 변경 시의 `AclAuthorizationStrategy` 검사.
4. [04-JDBC-영속화와-조회-전략.md](04-JDBC-영속화와-조회-전략.md) — `JdbcAclService`/`JdbcMutableAclService`/`BasicLookupStrategy` 와 5개 테이블 스키마, 배치 조회·부모 해석·캐싱.
5. [05-Spring-Security-통합.md](05-Spring-Security-통합.md) — `AclPermissionEvaluator`/`AclPermissionCacheOptimizer` 로 `hasPermission()` SpEL 과 연결되는 전체 시퀀스.

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.acls
│
├─ AclPermissionEvaluator          ← PermissionEvaluator 구현 (hasPermission 진입점)  [05]
├─ AclPermissionCacheOptimizer     ← 컬렉션 필터링 시 ACL 일괄 선적재               [05]
│
├─ model/   (계약 = 인터페이스·예외)                                              [01]
│   ├─ Acl, MutableAcl, AuditableAcl, OwnershipAcl   ← ACL 본체와 변경/감사/소유 확장
│   ├─ AccessControlEntry, AuditableAccessControlEntry ← ACE(권한 한 줄)
│   ├─ Sid                          ← 보안 주체 추상화 (principal / authority)
│   ├─ ObjectIdentity, ObjectIdentityGenerator       ← 도메인 객체 식별 키
│   ├─ Permission                   ← 32비트 마스크 권한                            [02]
│   ├─ PermissionGrantingStrategy   ← 인가 판정 알고리즘 SPI                        [03]
│   ├─ AclService, MutableAclService ← 조회/생성·수정·삭제 서비스 계약              [04]
│   ├─ AclCache                     ← ACL 캐시 계약
│   ├─ *RetrievalStrategy           ← Sid/ObjectIdentity 추출 전략
│   └─ NotFoundException, UnloadedSidException, AlreadyExistsException ...
│
├─ domain/  (model 인터페이스의 표준 구현)
│   ├─ AclImpl                      ← Acl+MutableAcl+Auditable+Ownership 일체형 구현 [03]
│   ├─ AccessControlEntryImpl       ← 불변 ACE 구현
│   ├─ DefaultPermissionGrantingStrategy ← 핵심 판정 알고리즘                       [03]
│   ├─ AclAuthorizationStrategyImpl ← ACL 변경 권한 검사(소유자/권위/ADMIN)         [03]
│   ├─ AbstractPermission, BasePermission, CumulativePermission ← 권한 마스크       [02]
│   ├─ DefaultPermissionFactory, AclFormattingUtils ← 마스크↔Permission 변환·출력   [02]
│   ├─ PrincipalSid, GrantedAuthoritySid ← Sid 구현                                [01]
│   ├─ ObjectIdentityImpl, ObjectIdentityRetrievalStrategyImpl ← 식별 구현          [01]
│   ├─ SidRetrievalStrategyImpl     ← Authentication → List<Sid>                    [05]
│   ├─ SpringCacheBasedAclCache     ← Cache 추상화 기반 ACL 캐시                    [04]
│   └─ AuditLogger, ConsoleAuditLogger ← 권한 부여/거부 감사 로깅
│
├─ jdbc/    (관계형 DB 영속화)                                                    [04]
│   ├─ JdbcAclService               ← AclService (읽기 전용 조회)
│   ├─ JdbcMutableAclService        ← MutableAclService (생성/수정/삭제)
│   ├─ LookupStrategy, BasicLookupStrategy ← ANSI SQL 배치 조회·부모 해석
│   └─ AclClassIdUtils              ← class_id_type 컬럼으로 식별자 타입 변환(UUID 등)
│
└─ aot/hint/  ← AclRuntimeHints (GraalVM 네이티브 이미지 리플렉션 힌트)
```
