# 05. Spring Security 통합 — hasPermission 으로 가는 길

## 무엇을 / 왜

지금까지의 모든 부품(Sid·ObjectIdentity·Permission·Acl·AclService)은 `acl` 모듈 안에 갇혀 있다. 이것을 실제 애플리케이션의 인가 표현식과 연결하는 다리가 이 챕터의 주제다.

```java
@PreAuthorize("hasPermission(#post, 'WRITE')")
public void edit(BlogPost post) { ... }
```

이 SpEL `hasPermission(...)` 호출이 어떻게 ACL 조회·판정으로 흘러가는가? 핵심은 단 두 클래스, `AclPermissionEvaluator`(판정 진입점)와 `AclPermissionCacheOptimizer`(컬렉션 필터링 최적화)다. 둘 다 `acl` 패키지 루트(`org.springframework.security.acls`)에 있고, `access` 패키지의 `PermissionEvaluator`/`PermissionCacheOptimizer` 인터페이스를 구현한다.

## 핵심 타입

```
   SpEL: hasPermission(domainObject, 'WRITE')
        │
        ▼
   PermissionEvaluator            (spring-security-core, access 패키지)
        │ 구현
        ▼
   AclPermissionEvaluator         (acl 모듈)
        ├─ ObjectIdentityRetrievalStrategy  : 도메인객체 → ObjectIdentity
        ├─ ObjectIdentityGenerator          : (id,type) → ObjectIdentity
        ├─ SidRetrievalStrategy             : Authentication → List<Sid>
        ├─ PermissionFactory                : 'WRITE'/2 → Permission
        └─ AclService                       : ObjectIdentity → Acl   (→ 04장)
```

이 평가기는 ACL 모듈의 여러 전략들을 **조립(orchestrate)** 하는 역할이다. 자기 로직은 얇고, 실제 일은 각 전략과 `AclService` 가 한다.

## 동작 흐름 — hasPermission 한 번의 전체 시퀀스

```
 메서드보안 인터셉터 (@PreAuthorize 평가)
   │  hasPermission(authentication, post, "WRITE")
   ▼
 AclPermissionEvaluator.hasPermission(auth, domainObject, permission)
   │
   ├─ domainObject == null ?  → return false                 (널이면 거부)
   │
   ├─ [1] ObjectIdentity 추출
   │     objectIdentityRetrievalStrategy.getObjectIdentity(post)
   │        = new ObjectIdentityImpl(post)  → post.getId() 리플렉션
   │        → ObjectIdentity(type=BlogPost, id=42)
   │
   ├─ [2] checkPermission(auth, oid, "WRITE")
   │     │
   │     ├─ Sid 목록 추출
   │     │    sidRetrievalStrategy.getSids(auth)
   │     │      = [PrincipalSid(alice), GrantedAuthoritySid(ROLE_USER), ...]
   │     │
   │     ├─ 권한 해석
   │     │    resolvePermission("WRITE")
   │     │      = permissionFactory.buildFromName("WRITE") → [BasePermission.WRITE]
   │     │
   │     ├─ ACL 조회 (→ 04장: 캐시→배치→부모해석)
   │     │    aclService.readAclById(oid, sids)  → Acl
   │     │
   │     └─ 판정 (→ 03장: ACE 스캔·거부·상속)
   │          acl.isGranted([WRITE], sids, false)
   │            ├─ true  → "Access is granted" → return true
   │            ├─ false → 권한 부족          → return false
   │            └─ NotFoundException 던져짐    → catch → return false
   ▼
 true / false  → 인터셉터가 메서드 실행 허용/AccessDenied
```

### 진입점 코드 (`AclPermissionEvaluator.java`)

```java
public boolean hasPermission(Authentication authentication, Object domainObject, Object permission) {
    if (domainObject == null) {
        return false;                          // 도메인 객체 없으면 거부
    }
    ObjectIdentity objectIdentity = this.objectIdentityRetrievalStrategy.getObjectIdentity(domainObject);
    return checkPermission(authentication, objectIdentity, permission);
}
```

`PermissionEvaluator` 는 오버로드가 둘이다. 위는 도메인 객체를 직접 받는 경우, 아래는 객체를 로딩하기 전에 `(targetId, targetType)` 만으로 검사하는 경우다(예: `hasPermission(42, 'com.example.BlogPost', 'READ')`).

```java
public boolean hasPermission(Authentication auth, Serializable targetId, String targetType, Object permission) {
    ObjectIdentity oid = this.objectIdentityGenerator.createObjectIdentity(targetId, targetType);
    return checkPermission(auth, oid, permission);
}
```

### checkPermission — 조립의 핵심

```java
private boolean checkPermission(Authentication authentication, ObjectIdentity oid, Object permission) {
    List<Sid> sids = this.sidRetrievalStrategy.getSids(authentication);
    List<Permission> requiredPermission = resolvePermission(permission);
    try {
        Acl acl = this.aclService.readAclById(oid, sids);        // ACL 조회
        if (acl.isGranted(requiredPermission, sids, false)) {     // 판정
            return true;
        }
        return false;                                             // ACL은 있지만 권한 부족
    }
    catch (NotFoundException nfe) {
        return false;                                             // ACL/ACE 정보 자체가 없음
    }
}
```

여기서 [03장](03-인가-결정-흐름.md)의 `NotFoundException` 이 어떻게 소비되는지가 드러난다. ACL 모듈 내부에서는 "정보 없음"을 예외로 신호하지만, **인가 평가기 경계에서 그 예외는 조용히 `false` 로 흡수**된다. 즉 ACL이 없으면 "거부"로 귀결된다.

### resolvePermission — 다양한 입력 형태 흡수

SpEL에서 권한은 정수, `Permission` 객체, 배열, 문자열 등으로 들어올 수 있다. 이를 모두 `List<Permission>` 으로 정규화한다.

```java
List<Permission> resolvePermission(Object permission) {
    if (permission instanceof Integer)       return List.of(permissionFactory.buildFromMask((Integer) permission));
    if (permission instanceof Permission)    return List.of((Permission) permission);
    if (permission instanceof Permission[])  return Arrays.asList((Permission[]) permission);
    if (permission instanceof String s) {
        Permission p = buildPermission(s);   // 이름으로, 실패 시 대문자 재시도
        if (p != null) return List.of(p);
    }
    throw new IllegalArgumentException("Unsupported permission: " + permission);
}
```

문자열은 `'write'` 처럼 소문자로 와도 동작하도록, 먼저 원문으로 `buildFromName` 을 시도하고 실패하면 대문자로 재시도한다. 여러 권한을 배열로 넘기면 그중 **하나라도** 만족하면 통과한다([03장]의 권한 목록 OR 의미).

### SidRetrievalStrategyImpl — Authentication을 Sid로 (`SidRetrievalStrategyImpl.java`)

```java
public List<Sid> getSids(Authentication authentication) {
    Collection<? extends GrantedAuthority> authorities =
        this.roleHierarchy.getReachableGrantedAuthorities(authentication.getAuthorities());
    List<Sid> sids = new ArrayList<>(authorities.size() + 1);
    sids.add(new PrincipalSid(authentication));            // 맨 앞에 개인 Sid
    for (GrantedAuthority authority : authorities) {
        sids.add(new GrantedAuthoritySid(authority));       // 그 뒤로 권위 Sid들
    }
    return sids;
}
```

로그인 사용자 하나가 **여러 Sid** 로 펼쳐진다 — 개인(`PrincipalSid("alice")`) 하나 + 보유 권위 각각(`GrantedAuthoritySid("ROLE_USER")` 등). 판정 시 이 Sid 중 어느 것에 권한 ACE가 걸려 있어도 매칭된다. `RoleHierarchy` 를 주입하면 권위가 계층 확장된다(예: ROLE_ADMIN이 ROLE_USER를 포함). `PrincipalSid` 를 항상 맨 앞에 두는 것은 [03장]에서 Sid 순서가 판정에 영향을 주기 때문이다.

## AclPermissionCacheOptimizer — 컬렉션 필터링 N+1 방지

`@PostFilter("hasPermission(filterObject, 'READ')")` 처럼 **컬렉션의 각 원소마다** `hasPermission` 을 부르면, 원소 수만큼 ACL 조회가 일어나 N+1 문제가 된다. `AclPermissionCacheOptimizer` 는 필터링 직전에 **모든 원소의 ACL을 한 번에 선적재**해 캐시를 데운다.

```java
public void cachePermissionsFor(Authentication authentication, Collection<?> objects) {
    if (objects.isEmpty()) return;
    List<ObjectIdentity> oidsToCache = new ArrayList<>(objects.size());
    for (Object domainObject : objects) {
        if (domainObject != null) {
            oidsToCache.add(this.oidRetrievalStrategy.getObjectIdentity(domainObject));
        }
    }
    List<Sid> sids = this.sidRetrievalStrategy.getSids(authentication);
    this.aclService.readAclsById(oidsToCache, sids);       // 일괄 조회 → BasicLookupStrategy가 캐시 적재
}
```

[04장](04-JDBC-영속화와-조회-전략.md)의 `BasicLookupStrategy.readAclsById` 가 배치 조회 후 결과를 캐시에 넣으므로, 이후 각 원소의 `hasPermission` 은 모두 캐시 히트로 끝난다. 이 옵티마이저를 `PermissionCacheOptimizer` 빈으로 등록하면 표현식 인가 인프라가 필터링 전에 자동 호출한다.

## 핵심 메서드 요약

| 메서드 | 하는 일 |
|--------|---------|
| `AclPermissionEvaluator.hasPermission(auth, obj, perm)` | 도메인 객체 → ObjectIdentity → 판정 |
| `AclPermissionEvaluator.checkPermission` | Sid·Permission 해석 + AclService 조회 + isGranted, NotFoundException→false |
| `resolvePermission` | 정수/이름/객체/배열 입력을 `List<Permission>` 으로 정규화 |
| `SidRetrievalStrategyImpl.getSids` | Authentication → [PrincipalSid, GrantedAuthoritySid...] |
| `AclPermissionCacheOptimizer.cachePermissionsFor` | 컬렉션 ACL 일괄 선적재(N+1 방지) |

## 설계 포인트 / 확장점

- **얇은 오케스트레이터**: `AclPermissionEvaluator` 는 4개 전략 + `AclService` 를 조립할 뿐, 각각을 setter로 교체할 수 있다(`setSidRetrievalStrategy`, `setObjectIdentityRetrievalStrategy`, `setPermissionFactory` 등).
- **예외의 경계 변환**: 모듈 내부의 `NotFoundException`(정보 부재)이 평가기 경계에서 `false`(거부)로 바뀐다 — "ACL 없음 = 접근 거부"라는 보안 기본값.
- **두 가지 진입**: 객체를 이미 들고 있으면 `(auth, obj, perm)`, 객체 로딩 전이면 `(auth, id, type, perm)`. 후자는 불필요한 도메인 객체 로딩을 피한다.
- **커스텀 권한/식별자**: `PermissionFactory` 교체로 애플리케이션 고유 권한 이름을, `ObjectIdentityRetrievalStrategy` 교체로 `getId()` 가 없는 도메인 객체를 지원.

## 정리

`AclPermissionEvaluator` 는 SpEL `hasPermission()` 과 ACL 모듈을 잇는 다리다. 도메인 객체에서 `ObjectIdentity` 를, Authentication에서 `List<Sid>` 를, 표현식 인자에서 `List<Permission>` 을 뽑아내고, `AclService` 로 ACL을 조회한 뒤 `acl.isGranted(...)` 로 최종 판정한다. ACL 정보가 없어 발생하는 `NotFoundException` 은 경계에서 `false` 로 흡수된다. 컬렉션 필터링의 N+1은 `AclPermissionCacheOptimizer` 가 ACL을 일괄 선적재해 해결한다. 이로써 [01]~[04]장의 모든 부품이 한 줄의 `@PreAuthorize` 표현식 아래에서 함께 동작한다.
