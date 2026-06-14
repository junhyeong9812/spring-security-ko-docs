# 04. JDBC 영속화와 조회 전략

## 무엇을 / 왜

앞 장들의 `Acl`/`ACE`/`Sid`/`ObjectIdentity` 는 메모리 상의 객체일 뿐이다. 실제로는 **관계형 DB 5개 테이블**에 저장되고, 인가 때마다 조회·재조립돼야 한다. 이 챕터는 `jdbc` 패키지를 다룬다.

- `JdbcAclService` — 읽기 전용 `AclService` 구현.
- `JdbcMutableAclService` — 그 위에 생성/수정/삭제를 더한 `MutableAclService` 구현.
- `BasicLookupStrategy` — 조회의 실제 엔진. ANSI SQL 배치 조회, 부모 ACL 해석, 캐시 연동을 담당.
- `SpringCacheBasedAclCache` — Spring Cache 추상화 기반 ACL 캐시.

ACL 조회는 인가 경로마다 일어나므로 성능이 중요하다. 그래서 배치·캐시·부모 일괄 해석 같은 최적화가 이 패키지에 집중돼 있다.

## DB 스키마 (5개 테이블)

`JdbcMutableAclService` 의 기본 SQL에서 역산한 스키마다(HSQLDB 기준 기본값).

```
 acl_class                    클래스(type) 등록부
   id, class(=ObjectIdentity.type), class_id_type(선택: 식별자 자바타입)
        ▲
        │ object_id_class
 acl_object_identity          도메인 객체 인스턴스 한 건 = 한 ACL
   id, object_id_class→acl_class, object_id_identity(=식별자 문자열),
   parent_object→self(부모 ACL), owner_sid→acl_sid, entries_inheriting
        ▲                                   ▲
        │ acl_object_identity               │ owner_sid
 acl_entry                    ACE 한 줄          │
   id, acl_object_identity→acl_object_identity, │
   ace_order(순서!), sid→acl_sid, mask(권한정수),│
   granting, audit_success, audit_failure       │
                       │ sid                     │
                       ▼                         ▼
 acl_sid                      Sid 등록부
   id, principal(true=PrincipalSid / false=GrantedAuthoritySid), sid(이름)
```

- `acl_class.class_id_type` 컬럼은 식별자가 `Long` 이 아닌 경우(예: `UUID`)를 지원하기 위한 확장이다. `setAclClassIdSupported(true)` 로 켜면 SQL이 이 컬럼을 포함하는 버전으로 바뀌고, `AclClassIdUtils` 가 문자열 식별자를 해당 자바 타입으로 변환한다.
- `ace_order` 가 [03장](03-인가-결정-흐름.md)의 "첫 매칭이 이긴다" 규칙에서 ACE 스캔 순서를 결정한다.

## 동작 흐름 1 — 읽기 (JdbcAclService → BasicLookupStrategy)

`JdbcAclService` 자신은 얇다. 모든 무거운 일을 `LookupStrategy` 에 위임하고, 결과 검증만 한다.

```java
// JdbcAclService.readAclsById
public Map<ObjectIdentity, Acl> readAclsById(List<ObjectIdentity> objects, List<Sid> sids) {
    Map<ObjectIdentity, Acl> result = this.lookupStrategy.readAclsById(objects, sids);
    for (ObjectIdentity oid : objects) {
        if (!result.containsKey(oid)) {
            throw new NotFoundException("Unable to find ACL information for object identity '" + oid + "'");
        }
    }
    return result;
}
```

요청한 ObjectIdentity 중 하나라도 결과에 없으면 `NotFoundException` 을 던진다. 즉 "전부 찾아야 정상"이다. `findChildren` 만 직접 SQL을 쏘아 부모-자식 관계를 조회한다.

### BasicLookupStrategy.readAclsById — 배치 + 캐시 (`BasicLookupStrategy.java`)

```
 for each ObjectIdentity:
   1. 이미 result에 있나?           → skip
   2. aclCache.getFromCache(oid)    → 있으면 result에 추가
   3. 둘 다 아니면 currentBatchToLoad에 모음
   4. 배치가 batchSize(기본50) 차거나 마지막이면
        lookupObjectIdentities(batch) 로 DB 조회
        → result에 합치고, 로딩분을 aclCache.putInCache()
```

캐시 먼저, 없는 것만 모아 배치로 DB 조회, 조회분은 캐시에 적재 — 전형적인 read-through 패턴이다.

> **주의 (소스 명시)**: `BasicLookupStrategy` 는 `sids` 인자를 **완전히 무시**한다. 모든 캐시 항목이 전체 Sid를 담는다고 가정하기 때문이다. Sid별 부분 로딩 최적화가 필요할 만큼 Sid가 많다면 커스텀 `LookupStrategy` 를 만들어야 한다. 그래서 캐시에서 꺼낼 때 `acl.isSidLoaded(sids)` 가 항상 true임을 단언으로 확인한다.

### lookupObjectIdentities — SQL과 부모 해석

조회는 두 단계 SQL을 쓴다. SELECT/ORDER BY는 공유하고 WHERE만 다르다.

- `lookupObjectIdentitiesWhereClause` — `(object_id_identity = ? and class = ?)` 로 ObjectIdentity 기준 조회.
- `lookupPrimaryKeysWhereClause` — `(acl_object_identity.id = ?)` 로 PK(주로 부모) 기준 조회.

SELECT는 `acl_object_identity LEFT JOIN acl_sid(owner) LEFT JOIN acl_class LEFT JOIN acl_entry LEFT JOIN acl_sid(ace)` 형태로, **ACL 한 건과 그 모든 ACE를 한 번에** 끌어온다. `ORDER BY object_id_identity, ace_order` 로 ACE 순서를 보존한다.

부모 처리가 까다롭다. 한 ResultSet 안에서 부모의 전체 정보를 알 수 없으므로, 일단 **StubAclParent(부모 PK만 든 더미)** 로 채워두고, 부모 PK들을 모아 재귀적으로 다시 조회한다.

```
 1차 조회 (ObjectIdentity 기준)
   ResultSet → AclImpl 생성, 부모 있으면 parent = StubAclParent(parentId)
   부모 PK들을 parentsToLookup 집합에 수집
        │
        ▼  (DB 커넥션 반납 후 — SEC-547)
 lookupPrimaryKeys(parentsToLookup)  ← PK 기준으로 부모 ACL 조회
   부모의 부모가 또 있으면 다시 재귀
        │
        ▼
 convert(): StubAclParent → 실제 AclImpl 로 치환 (재귀)
   ACE들의 getAcl() 역참조도 새 AclImpl 로 교체 (SEC-951)
```

`convert` 가 마지막에 `StubAclParent` 를 진짜 `AclImpl` 로 바꾸고, 각 ACE가 가리키는 `acl` 필드도 리플렉션으로 새 인스턴스에 맞춘다. 이 리플렉션 접근을 위해 `AclImpl.aces`, `AccessControlEntryImpl.acl` private 필드에 직접 접근하는 `Field` 들을 들고 있다.

```java
private final Field fieldAces = FieldUtils.getField(AclImpl.class, "aces");
private final Field fieldAcl  = FieldUtils.getField(AccessControlEntryImpl.class, "acl");
```

DB의 `mask` 정수는 `permissionFactory.buildFromMask(mask)` 로 `Permission` 객체로 되살아나고([02장](02-권한-마스크와-Permission.md)), `principal` 플래그에 따라 `PrincipalSid`/`GrantedAuthoritySid` 로 `Sid` 가 복원된다(`createSid`).

## 동작 흐름 2 — 쓰기 (JdbcMutableAclService)

### createAcl — 빈 ACL 생성

```java
public MutableAcl createAcl(ObjectIdentity objectIdentity) {
    if (retrieveObjectIdentityPrimaryKey(objectIdentity) != null)
        throw new AlreadyExistsException(...);              // 중복 방지
    Authentication auth = securityContextHolderStrategy.getContext().getAuthentication();
    PrincipalSid sid = new PrincipalSid(auth);              // 현재 사용자가 소유자
    createObjectIdentity(objectIdentity, sid);             // acl_object_identity 행 삽입
    Acl acl = readAclById(objectIdentity);                  // 다시 읽어 캐시 등록까지
    return (MutableAcl) acl;
}
```

생성 직후 다시 `readAclById` 로 읽어 돌려주는 이유는, 조회 경로를 한 번 태워 캐시 등록과 일관된 객체 그래프를 보장하기 위해서다. 소유자는 **ACL을 만든 현재 principal** 로 자동 설정된다.

### updateAcl — 통째로 지우고 다시 쓰기

```java
public MutableAcl updateAcl(MutableAcl acl) {
    Long oidPk = retrieveObjectIdentityPrimaryKey(acl.getObjectIdentity());
    if (oidPk == null) throw new NotFoundException(...);
    deleteEntries(oidPk);                     // 기존 ACE 전부 삭제
    createEntries(acl);                       // 현재 ACE 전부 재삽입 (배치)
    updateObjectIdentity(acl);                // 부모/소유자/상속 플래그 갱신
    clearCacheIncludingChildren(acl.getObjectIdentity());  // 자식까지 캐시 무효화
    return (MutableAcl) super.readAclById(acl.getObjectIdentity());
}
```

소스 주석이 인정하듯, 이 구현은 dirty-checking 없이 **모든 ACE를 지우고 다시 만든다**. 단순하지만 ACL당 ACE가 많으면 비효율적이다(더 정교한 구현은 ORM 권장). `createEntries` 는 `BatchPreparedStatementSetter` 로 일괄 INSERT한다. 부모가 바뀌면 자식들의 상속 결과도 달라지므로 캐시를 **자식까지 재귀적으로 무효화**한다.

### Sid/Class PK 확보 — createOrRetrieve...

ACE를 저장하려면 `acl_sid`, `acl_class` 의 PK가 필요하다. 없으면 INSERT 후 ID를 회수한다.

```java
protected Long createOrRetrieveSidPrimaryKey(String sidName, boolean sidIsPrincipal, boolean allowCreate) {
    List<Long> sidIds = jdbcOperations.queryForList(selectSidPrimaryKey, Long.class, sidIsPrincipal, sidName);
    if (!sidIds.isEmpty() && sidIds.get(0) != null) return sidIds.get(0);
    if (allowCreate) {
        jdbcOperations.update(insertSid, sidIsPrincipal, sidName);
        Assert.isTrue(TransactionSynchronizationManager.isSynchronizationActive(), "Transaction must be running");
        return jdbcOperations.queryForObject(sidIdentityQuery, Long.class);  // call identity()
    }
    return null;
}
```

새 행을 넣은 뒤 `sidIdentityQuery`(기본 `call identity()`)로 방금 생성된 PK를 가져온다. **이 식별자 회수 쿼리는 DB마다 다르다** — HSQLDB가 기본이고, 다른 DB는 `setSidIdentityQuery`/`setClassIdentityQuery` 로 바꿔야 한다. 또한 `identity()` 호출이 의미를 가지려면 **트랜잭션이 살아 있어야** 하므로 단언으로 확인한다. ACL 쓰기 작업은 트랜잭션 안에서 호출돼야 한다.

### deleteAcl — 자식 캐스케이드

```java
public void deleteAcl(ObjectIdentity oid, boolean deleteChildren) {
    if (deleteChildren) {
        for (ObjectIdentity child : findChildren(oid)) deleteAcl(child, true);  // 재귀
    } else if (!foreignKeysInDatabase) {
        if (findChildren(oid) != null) throw new ChildrenExistException(...);    // FK 수동 검사
    }
    Long oidPk = retrieveObjectIdentityPrimaryKey(oid);
    deleteEntries(oidPk);          // acl_entry 삭제
    deleteObjectIdentity(oidPk);   // acl_object_identity 삭제
    aclCache.evictFromCache(oid);  // 캐시 제거
}
```

`acl_class` 행은 일부러 지우지 않는다(데드락 회피). `foreignKeysInDatabase=true`(기본)이면 자식 존재 검사를 DB의 FK 제약에 맡겨 추가 조회로 인한 데드락을 피한다.

## SpringCacheBasedAclCache — transient 필드 복원

ACL을 캐시에 직렬화/역직렬화하면 문제가 하나 생긴다. `AclImpl` 의 `aclAuthorizationStrategy` 와 `permissionGrantingStrategy` 는 `transient` 라 캐시를 거치며 사라진다. 이 캐시는 꺼낼 때마다 그 두 전략을 **리플렉션으로 다시 주입**한다.

```java
private MutableAcl initializeTransientFields(MutableAcl value) {
    if (value instanceof AclImpl) {
        FieldUtils.setProtectedFieldValue("aclAuthorizationStrategy", value, this.aclAuthorizationStrategy);
        FieldUtils.setProtectedFieldValue("permissionGrantingStrategy", value, this.permissionGrantingStrategy);
    }
    if (value.getParentAcl() != null) initializeTransientFields((MutableAcl) value.getParentAcl());  // 부모까지
    return value;
}
```

`putInCache` 는 ACL을 `ObjectIdentity` 키와 `id`(PK) 키 두 가지로 저장해, 양쪽 조회 경로를 모두 지원한다. 이 캐시는 모든 `AclImpl` 이 같은 전략 인스턴스를 공유한다고 가정한다.

## 설계 포인트 / 확장점

- **조회/영속화 분리**: `JdbcAclService` 는 얇고, 진짜 엔진은 `LookupStrategy` 다. SQL 최적화나 특수 DB 기능이 필요하면 `LookupStrategy` 만 교체한다(BasicLookupStrategy는 subclassing 미지원, 교체 권장).
- **SQL 전면 커스터마이즈**: select/where/order-by/insert/update가 모두 setter로 노출돼 스키마·컬럼명·DB 방언에 맞출 수 있다. 단 서로 일관돼야 한다.
- **식별자 타입 확장**: `setAclClassIdSupported(true)` + `class_id_type` 컬럼으로 `UUID` 등 비-Long PK 지원.
- **캐시 전략**: `SpringCacheBasedAclCache` 로 Caffeine/Redis 등 어떤 Spring Cache 백엔드든 연결. 변경 시 자식까지 무효화하는 책임은 `JdbcMutableAclService` 가 진다.

## 정리

ACL은 `acl_class`/`acl_object_identity`/`acl_entry`/`acl_sid` 4개 테이블(+자기참조 부모)에 저장된다. 읽기는 `JdbcAclService` → `BasicLookupStrategy` 로 흐르며 **캐시 우선·배치 조회·StubAclParent를 통한 부모 재귀 해석**으로 ACL 그래프를 재조립한다. 쓰기는 `JdbcMutableAclService` 가 담당하되 `updateAcl` 은 ACE를 통째로 지우고 다시 쓰며, `identity()` 류 PK 회수 쿼리는 DB별로 교체해야 한다. `SpringCacheBasedAclCache` 는 캐시 왕복에서 사라지는 transient 전략 필드를 리플렉션으로 복원한다.
