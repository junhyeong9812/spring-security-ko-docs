# 02. 권한 마스크와 Permission

## 무엇을 / 왜

ACL의 한 줄(ACE)이 표현하는 "어떤 권한"은 `Permission` 으로 모델링된다. Spring Security ACL은 권한을 **32비트 정수 비트마스크**로 표현한다. 왜 enum이 아니라 비트마스크일까? 두 가지 이유가 있다.

1. **DB 친화적**: ACE 한 줄의 권한을 `acl_entry.mask` 라는 `INTEGER` 컬럼 하나에 저장할 수 있다.
2. **조합 가능**: READ(1) | WRITE(2) | DELETE(8) = 11 처럼 여러 권한을 한 정수로 합칠 수 있다(`CumulativePermission`).

이 챕터는 `Permission` 인터페이스와 그 구현 계층(`AbstractPermission` → `BasePermission`/`CumulativePermission`), 그리고 정수↔Permission 변환을 담당하는 `DefaultPermissionFactory` 를 다룬다.

## 핵심 타입

```
        Permission  (interface)              model 패키지
        │  int    getMask()      ← 비트마스크 (판정에 쓰임)
        │  String getPattern()   ← 32자 비트 패턴 문자열 (UI/로깅 전용)
        ▼
        AbstractPermission  (abstract)        domain 패키지
        │  final equals/hashCode = 마스크 비교
        │  mask, code(활성 비트 출력 문자)
        ├──────────────┐
        ▼              ▼
   BasePermission   CumulativePermission
   (정적 단일 권한)   (런타임 조합 권한)
```

### Permission 인터페이스의 두 메서드

`getMask()` 만이 인가 계산에 쓰인다. `getPattern()` 은 사람이 보기 위한 32자 문자열로, 켜진 비트는 코드 문자(예: `R`,`W`)로, 꺼진 비트는 `.` 로 표시한다. Javadoc은 패턴 문자열을 **권한 계산에 절대 쓰지 말라**고 명시한다 — 순수히 UI·로그용이다.

`Permission` 에 정의된 상수들:

```java
char RESERVED_ON  = '~';   // 내부 계산용으로만 허용, 출력엔 금지
char RESERVED_OFF = '.';   // 꺼진 비트 표시
String THIRTY_TWO_RESERVED_OFF = "................................"; // 비어있는 32비트
```

### AbstractPermission — equals는 마스크가 전부 (`AbstractPermission.java`)

모든 표준 권한의 공통 부모다. 핵심은 동등성 정의다.

```java
@Override
public final boolean equals(Object obj) {
    if (!(obj instanceof Permission other)) return false;
    return (this.mask == other.getMask());   // 마스크가 같으면 같은 권한
}
@Override public final int hashCode() { return this.mask; }
```

`equals`/`hashCode`/`getMask` 가 모두 `final` 이다. 즉 어떤 `Permission` 구현이든 **마스크 정수가 신원(identity)** 이다. 이것이 [03장](03-인가-결정-흐름.md)에서 ACE 매칭이 `ace.getPermission().getMask() == p.getMask()` 한 줄로 끝나는 근거다.

`getPattern()` 은 `AclFormattingUtils.printBinary(mask, code)` 에 위임해 32자 문자열을 만든다.

### BasePermission — 다섯 개의 표준 권한 (`BasePermission.java`)

```java
public static final Permission READ           = new BasePermission(1 << 0, 'R'); //  1
public static final Permission WRITE          = new BasePermission(1 << 1, 'W'); //  2
public static final Permission CREATE         = new BasePermission(1 << 2, 'C'); //  4
public static final Permission DELETE         = new BasePermission(1 << 3, 'D'); //  8
public static final Permission ADMINISTRATION = new BasePermission(1 << 4, 'A'); // 16
```

각 권한은 **서로 다른 비트 하나**를 차지한다(`1 << n`). 생성자가 `protected` 이므로, 이 클래스를 상속해 애플리케이션 고유 권한(예: `APPROVE = 1 << 5`)을 추가하는 것이 권장 확장 방식이다. `ADMINISTRATION` 권한은 특별하다 — 그 객체의 ACL 자체를 변경할 수 있는 권한으로, `AclAuthorizationStrategyImpl` 이 별도로 검사한다([03장](03-인가-결정-흐름.md)).

### CumulativePermission — 런타임 조합 (`CumulativePermission.java`)

여러 권한을 비트 OR로 합치거나 빼서 하나의 `Permission` 으로 만든다. 메서드가 `this` 를 반환해 체이닝된다.

```java
Permission p = new CumulativePermission()
        .set(BasePermission.READ)    // mask |= 1
        .set(BasePermission.WRITE);  // mask |= 2   → mask = 3
// p.clear(BasePermission.WRITE)     // mask &= ~2  → mask = 1
```

```java
public CumulativePermission set(Permission permission) {
    this.mask |= permission.getMask();
    this.pattern = AclFormattingUtils.mergePatterns(this.pattern, permission.getPattern());
    return this;
}
```

`set` 은 마스크를 OR로 켜고, 패턴 문자열도 함께 병합해 `RWC...` 같은 가독 문자열을 유지한다. `clear` 는 그 반대(`&= ~mask`).

> 주의: `CumulativePermission(mask=3)` 은 "READ **와** WRITE를 동시에 표현하는 단일 ACE"다. 이것을 인가 판정에 쓰면 의미가 달라진다 — `DefaultPermissionGrantingStrategy` 의 기본 매칭은 **마스크 정확히 일치**라서, mask=3 ACE는 READ(1)만 요구한 질의에는 매칭되지 않는다. 이 함정은 03장에서 다시 짚는다.

## 동작 흐름 — 정수 ↔ Permission 변환 (`DefaultPermissionFactory.java`)

DB에는 권한이 정수(`mask`)로 저장돼 있고, SpEL 표현식에서는 `'WRITE'` 같은 이름이나 `2` 같은 정수로 들어온다. 이 변환을 `PermissionFactory` 가 담당한다.

```
 생성 시점: registerPublicPermissions(BasePermission.class)
   리플렉션으로 public static Permission 필드를 모두 스캔
        ▼
   registeredPermissionsByInteger : {1→READ, 2→WRITE, 4→CREATE, 8→DELETE, 16→ADMIN}
   registeredPermissionsByName    : {"READ"→READ, "WRITE"→WRITE, ...}
```

### buildFromMask — 정수를 Permission으로

```java
public Permission buildFromMask(int mask) {
    if (registeredPermissionsByInteger.containsKey(mask)) {
        return registeredPermissionsByInteger.get(mask);   // 단일 등록 권한이면 그대로
    }
    // 등록된 단일 권한이 아니면 → 비트별로 분해해 CumulativePermission 조립
    CumulativePermission permission = new CumulativePermission();
    for (int i = 0; i < 32; i++) {
        int bit = 1 << i;
        if ((mask & bit) == bit) {
            Permission p = registeredPermissionsByInteger.get(bit);
            Assert.state(p != null, "Mask '" + bit + "' has no corresponding static Permission");
            permission.set(p);
        }
    }
    return permission;
}
```

핵심 로직: 마스크가 **정확히 등록된 단일 권한**이면 그 싱글턴을 반환하고, 그렇지 않으면(예: mask=3) 32개 비트를 하나씩 검사해 각 켜진 비트에 해당하는 등록 권한을 `CumulativePermission` 으로 합쳐 돌려준다. 등록되지 않은 비트가 켜져 있으면 예외를 던진다. 이것이 `BasicLookupStrategy` 가 DB의 `mask` 컬럼을 읽어 `Permission` 객체로 되살리는 경로다.

### buildFromName — 이름을 Permission으로

```java
public Permission buildFromName(String name) {
    Permission p = registeredPermissionsByName.get(name);
    Assert.notNull(p, "Unknown permission '" + name + "'");
    return p;
}
```

`@PreAuthorize("hasPermission(#post, 'WRITE')")` 의 `'WRITE'` 가 여기서 `BasePermission.WRITE` 로 변환된다. `AclPermissionEvaluator` 는 대소문자 차이를 흡수하려고, 먼저 원문으로 시도하고 실패하면 대문자로 재시도한다([05장](05-Spring-Security-통합.md)).

커스텀 권한을 쓰려면 `new DefaultPermissionFactory(MyPermission.class)` 또는 이름→권한 맵 생성자를 사용해 팩토리를 교체하면 된다.

## 설계 포인트 / 확장점

- **비트마스크 = 신원**: `equals` 가 마스크 비교라서, 권한의 동일성·저장·매칭이 모두 정수 하나로 환원된다. 단순하지만 강력하다.
- **확장은 상속으로**: `BasePermission` 을 상속해 새 비트를 추가하고, `DefaultPermissionFactory(MyPermission.class)` 로 등록하면 끝.
- **CumulativePermission의 함정**: 조합 권한을 ACE에 저장하면 기본 판정(정확 일치)과 어긋날 수 있다. "READ 또는 WRITE 중 하나면 허용"을 원하면 ACE를 두 줄로 나누거나(권장), `DefaultPermissionGrantingStrategy.isGranted(ace, p)` 를 비트 AND 방식으로 오버라이드해야 한다.

## 정리

`Permission` 은 32비트 정수 마스크다. `BasePermission` 이 READ/WRITE/CREATE/DELETE/ADMINISTRATION 다섯 비트를 정의하고, `CumulativePermission` 으로 런타임 조합한다. `equals` 가 마스크 비교이므로 권한의 신원은 정수 하나로 귀결되고, 이 단순함이 판정 알고리즘과 DB 저장을 모두 떠받친다. `DefaultPermissionFactory` 는 리플렉션 레지스트리로 정수·이름과 `Permission` 객체를 양방향 변환한다.
