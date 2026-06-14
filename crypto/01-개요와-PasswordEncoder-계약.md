# 01. 개요와 PasswordEncoder 계약

## 무엇을 / 왜

비밀번호 인증의 핵심은 단 하나의 질문이다. "사용자가 방금 입력한 평문이, 저장된 해시와 같은 비밀번호에서 나온 것인가?" 이 질문에 답하려면 평문을 저장해선 안 되고, 복호화도 불가능해야 하며, 같은 입력이라도 매번 다른 해시가 나와야(솔트) 안전하다. crypto 모듈은 이 모든 요구를 `PasswordEncoder`라는 **단 세 메서드짜리 인터페이스** 뒤에 숨긴다.

이 챕터는 그 계약(`PasswordEncoder`)과, 모든 구현이 공유하는 골격(`AbstractValidatingPasswordEncoder`)을 본다. 구체 알고리즘은 다음 챕터에서 다룬다.

## 핵심 타입

```
            ┌─────────────────────────────────────────────┐
            │           «interface» PasswordEncoder        │
            │  String  encode(CharSequence raw)            │
            │  boolean matches(CharSequence raw, String enc)│
            │  boolean upgradeEncoding(String enc)  =false  │
            └───────────────────────┬─────────────────────┘
                                    │ implements
            ┌───────────────────────▼─────────────────────┐
            │     AbstractValidatingPasswordEncoder (7.0)  │
            │  · encode/matches/upgradeEncoding 을 final 로 │
            │    구현하고 null·빈 문자열을 먼저 거른 뒤        │
            │  · ...NonNull(...) 추상 메서드로 위임          │
            └───────────────────────┬─────────────────────┘
                ┌───────────────────┼───────────────────────┐
                ▼                   ▼                         ▼
        BCryptPasswordEncoder  Argon2/SCrypt/Pbkdf2   DelegatingPasswordEncoder
                                                      AbstractPasswordEncoder
                                                      (솔트+digest 공통 골격)
```

소스 경로:
- `crypto/src/main/java/org/springframework/security/crypto/password/PasswordEncoder.java`
- `crypto/src/main/java/org/springframework/security/crypto/password/AbstractValidatingPasswordEncoder.java`
- `crypto/src/main/java/org/springframework/security/crypto/password/AbstractPasswordEncoder.java`

## PasswordEncoder의 세 메서드

### encode — 평문을 해시로

```java
@Contract("!null -> !null; null -> null")
@Nullable String encode(@Nullable CharSequence rawPassword);
```

평문을 받아 저장 가능한 해시 문자열을 만든다. JavaDoc이 못 박는 두 가지 계약이 중요하다. 첫째, "좋은 인코딩은 **적응형 단방향 함수**(adaptive one-way function)를 쓴다" — 즉 비용을 키울 수 있는 BCrypt/Argon2 류여야 한다. 둘째, **평문이 `null`이면 결과도 반드시 `null`** 이다. 비밀번호가 없는 사용자를 표현하기 위해서다.

### matches — 평문과 저장 해시 비교

```java
boolean matches(@Nullable CharSequence rawPassword, @Nullable String encodedPassword);
```

저장된 해시를 **복호화하지 않는다.** 대신 입력 평문을 (저장 해시에 들어 있는 솔트로) 다시 해싱해 두 해시를 비교한다. JavaDoc에 "둘 중 하나라도 `null`이거나 빈 문자열이면 절대 `true`가 아니다"라고 명시돼 있고, 이 가드는 아래 템플릿이 책임진다.

### upgradeEncoding — 더 강하게 다시 해싱할까?

```java
default boolean upgradeEncoding(@Nullable String encodedPassword) {
    return false;
}
```

기본값은 `false`. 이 메서드가 `true`를 돌려주면 "이 해시는 지금 기준으로 너무 약하니, 다음 로그인 때 더 강하게 다시 저장하라"는 신호다. BCrypt라면 저장된 strength가 현재 설정보다 낮을 때 `true`가 된다. 이 한 메서드가 [03장](03-Delegating과-비밀번호-업그레이드.md)의 자동 업그레이드 전체를 떠받친다.

## AbstractValidatingPasswordEncoder — null 가드 템플릿 (7.0)

7.0에서 추가된 이 추상 클래스는 **모든 구현이 반복하던 null·빈 문자열 검사를 한 곳으로 모은** 템플릿 메서드 패턴이다. 공개 메서드를 `final`로 막고, 실제 알고리즘은 `...NonNull` 메서드로 내려보낸다.

```java
public final boolean matches(@Nullable CharSequence rawPassword, @Nullable String encodedPassword) {
    if (!StringUtils.hasLength(rawPassword) || !StringUtils.hasLength(encodedPassword)) {
        return false;                       // ← 빈 입력은 무조건 불일치
    }
    return matchesNonNull(rawPassword.toString(), encodedPassword);
}

protected abstract boolean matchesNonNull(String rawPassword, String encodedPassword);
```

`encode`도 같은 방식으로 `rawPassword == null`이면 `null`을 반환하고, 그 외에는 `encodeNonNullPassword(...)`로 위임한다. `upgradeEncoding`은 빈 문자열이면 `false`, 아니면 `upgradeEncodingNonNull(...)`(기본 `false`)로 위임한다. 덕분에 BCrypt나 Argon2 구현은 "값이 반드시 있는" 행복한 경로만 작성하면 된다.

```
matches(raw, enc) 호출
   │
   ├─ raw 또는 enc 가 비었나? ── 예 ──▶ return false
   │                              (타이밍 차이 없이 즉시)
   └─ 아니오
        └──▶ matchesNonNull(raw, enc)   ← 구현체의 실제 알고리즘
```

## AbstractPasswordEncoder — 솔트+다이제스트 골격

`AbstractValidatingPasswordEncoder`를 한 단계 더 특화한 클래스로, "랜덤 솔트를 만들고 → 솔트와 해시를 이어 붙이고 → Hex로 인코딩"하는 패턴을 공통화한다. `encodeNonNullPassword`는 다음을 한다.

```java
byte[] salt = this.saltGenerator.generateKey();      // KeyGenerators.secureRandom()
byte[] encoded = encodeAndConcatenate(rawPassword, salt);  // salt || digest(raw, salt)
return String.valueOf(Hex.encode(encoded));
```

검증 시에는 저장된 Hex를 디코딩해 앞쪽 `keyLength` 바이트를 솔트로 떼어 내(`EncodingUtils.subArray`) 같은 절차로 다시 해싱한 뒤, **상수 시간 비교**로 맞춰 본다.

```java
protected static boolean matchesNonNull(byte[] expected, byte[] actual) {
    return MessageDigest.isEqual(expected, actual);   // 타이밍 공격 방어
}
```

`MessageDigest.isEqual`은 길이가 같으면 모든 바이트를 끝까지 비교해 "몇 번째 바이트에서 틀렸는지"가 응답 시간으로 새어 나가지 않게 한다. 솔트 생성에 `keygen` 패키지를, Hex 변환에 `codec` 패키지를, 바이트 이어붙이기에 `util.EncodingUtils`를 쓰는 — 이 모듈의 패키지들이 어떻게 맞물리는지 보여 주는 대표 예다.

## 설계 포인트

- **인터페이스 최소화**: 세 메서드뿐이라 누구나 구현할 수 있고, 상위 모듈은 구현을 몰라도 된다. 알고리즘 교체가 빈(bean) 하나 바꾸기로 끝나는 이유다.
- **템플릿 메서드 패턴**: `AbstractValidatingPasswordEncoder`가 null/빈 값 같은 "절대 빠지면 안 되는 안전 검사"를 `final`로 강제한다. 구현자가 실수로 빈 비밀번호를 통과시키는 사고를 구조적으로 막는다.
- **상수 시간 비교**: 모든 비교는 `MessageDigest.isEqual` 또는 직접 작성한 XOR 누적 루프(Argon2)로 처리한다. 일반 `equals`/`String.equals`는 타이밍 공격에 노출되므로 쓰지 않는다.

## 정리

`PasswordEncoder`는 `encode`/`matches`/`upgradeEncoding` 세 메서드로 "비밀번호를 안전하게 저장·검증·갱신"하는 전부를 표현하는 최소 계약이다. 7.0의 `AbstractValidatingPasswordEncoder`는 null·빈 문자열 가드를 템플릿으로 끌어올려 모든 구현이 알고리즘 본질에만 집중하게 해 주고, `AbstractPasswordEncoder`는 솔트+상수시간비교 골격을 더한다. 다음 장에서 이 골격 위에 올라가는 실제 해시 알고리즘들을 본다.
