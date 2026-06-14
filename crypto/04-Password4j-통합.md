# 04. Password4j 통합 (7.0 신규)

## 무엇을 / 왜

7.0에서 `password4j` 패키지가 새로 추가됐다. [Password4j](https://github.com/Password4j/password4j)는 비밀번호 해싱에 특화된 외부 라이브러리로, BCrypt·Argon2·SCrypt·PBKDF2뿐 아니라 **Balloon Hashing** 같은 최신 알고리즘과 더 풍부한 튜닝 API를 제공한다. 이 패키지는 Spring Security의 `PasswordEncoder` 계약과 Password4j 사이의 **얇은 어댑터 계층**이다.

기존 인코더(BouncyCastle 기반 Argon2/SCrypt 등)가 있는데도 이 계층을 둔 이유는 선택지를 넓히기 위해서다. Password4j를 이미 쓰는 팀, 또는 Balloon Hashing처럼 기존 모듈에 없는 알고리즘이 필요한 팀이 같은 `PasswordEncoder` 인터페이스로 갈아끼울 수 있게 한다. 클래스패스에 `com.password4j` 의존이 있을 때만 동작하는 선택적 기능이다.

## 핵심 타입

```
   AbstractValidatingPasswordEncoder
        │
        └── Password4jPasswordEncoder   (package-private 베이스, HashingFunction 위임)
              │
              ├── BcryptPassword4jPasswordEncoder
              ├── Argon2Password4jPasswordEncoder
              ├── ScryptPassword4jPasswordEncoder
              ├── Pbkdf2Password4jPasswordEncoder
              └── BalloonHashingPassword4jPasswordEncoder   (★ 베이스 미사용, 솔트 직접 관리)
```

소스: `crypto/src/main/java/org/springframework/security/crypto/password4j/`

핵심은 패키지 전체에 `@NullMarked`(jSpecify)가 걸려 있고, 베이스 클래스 `Password4jPasswordEncoder`가 **package-private**이라는 점이다. 외부에서 직접 상속할 수 없고, 검증된 알고리즘별 공개 서브클래스만 쓰도록 강제한다.

## 동작 흐름 — 베이스 클래스의 위임

`Password4jPasswordEncoder`는 Password4j의 `HashingFunction` 하나를 받아 세 메서드를 그대로 위임한다.

```java
abstract class Password4jPasswordEncoder extends AbstractValidatingPasswordEncoder {
    private final HashingFunction hashingFunction;

    protected String encodeNonNullPassword(String rawPassword) {
        Hash hash = Password.hash(rawPassword).with(this.hashingFunction);
        return hash.getResult();                       // 솔트 포함 해시 문자열
    }

    protected boolean matchesNonNull(String rawPassword, String encodedPassword) {
        return Password.check(rawPassword, encodedPassword).with(this.hashingFunction);
    }

    protected boolean upgradeEncodingNonNull(String encodedPassword) {
        return false;                                  // 현재는 항상 false
    }
}
```

`Password.hash(...).with(fn)`이 솔트 생성·해싱·직렬화를 한 번에 처리하고, `Password.check(...)`가 검증(내부적으로 상수시간 비교)을 담당한다. 즉 Spring 쪽 어댑터는 **null/빈 문자열 가드만 책임지고**(상위 `AbstractValidatingPasswordEncoder`), 알고리즘 디테일은 전부 라이브러리에 맡긴다. `upgradeEncoding`은 주석대로 현재 항상 `false` — Password4j가 알고리즘별로 내부 처리하므로 보수적으로 비활성화해 두었다.

서브클래스는 그저 적절한 `HashingFunction`을 골라 넘길 뿐이다.

```java
public class BcryptPassword4jPasswordEncoder extends Password4jPasswordEncoder {
    public BcryptPassword4jPasswordEncoder() {
        super(AlgorithmFinder.getBcryptInstance());    // 라이브러리 기본 설정
    }
    public BcryptPassword4jPasswordEncoder(BcryptFunction bcryptFunction) {
        super(bcryptFunction);                          // 커스텀 라운드 등
    }
}
```

`AlgorithmFinder.getXxxInstance()`가 Password4j의 권장 기본값을 제공하고, 명시적 `Function`을 넘기면 라운드·메모리 같은 파라미터를 직접 조절할 수 있다.

## 예외 — BalloonHashing은 솔트를 직접 관리한다

`BalloonHashingPassword4jPasswordEncoder`만 베이스를 쓰지 않고 `AbstractValidatingPasswordEncoder`를 직접 상속한다. 이유는 JavaDoc에 명시돼 있다 — **Password4j의 Balloon 구현은 출력 해시에 솔트를 포함하지 않기** 때문이다. 그래서 이 인코더는 솔트를 직접 만들어 별도 형식으로 합친다.

```
저장 포맷:  {salt}:{hash}   (둘 다 Base64, DELIMITER = ":")
            DEFAULT_SALT_LENGTH = 32 bytes, SecureRandom 으로 생성
```

`encode`는 `SecureRandom`으로 32바이트 솔트를 만들고 `Base64(salt) + ":" + Base64(hash)`를 반환하며, `matches`는 `:`로 분리해 앞쪽 솔트로 재해싱한 뒤 비교한다. 다른 인코더(BCrypt/Argon2/SCrypt)는 알고리즘 표준 포맷에 솔트가 들어 있어 이런 수작업이 필요 없다 — 솔트가 해시에 포함되는지 여부가 어댑터 설계를 가르는 분기점임을 보여 준다.

## 설계 포인트 / 확장점

- **얇은 어댑터(Adapter) 패턴**: Spring의 `PasswordEncoder`와 Password4j의 `HashingFunction` 사이를 잇기만 한다. 해싱 로직을 다시 구현하지 않고 신뢰받는 라이브러리에 위임한다.
- **package-private 베이스 + 공개 서브클래스**: 검증된 알고리즘만 노출해 사용자가 위험한 함수를 끼워 넣지 못하게 막는다.
- **`DelegatingPasswordEncoder`와 호환**: 이들도 `PasswordEncoder`이므로 `idToPasswordEncoder` 맵에 임의 id로 등록해 [03장](03-Delegating과-비밀번호-업그레이드.md)의 라우팅·업그레이드 체계에 그대로 합류시킬 수 있다.
- **선택적 의존**: `com.password4j`가 클래스패스에 없으면 이 패키지는 로드되지 않는다. 기본 비밀번호 처리에는 필요 없는 부가 기능이다.

## 정리

`password4j` 패키지는 7.0에서 추가된, Spring `PasswordEncoder`와 Password4j 라이브러리 사이의 어댑터 계층이다. 베이스 `Password4jPasswordEncoder`가 `encode`/`matches`를 `Password.hash/check`에 위임하고, BCrypt·Argon2·SCrypt·PBKDF2·Balloon용 공개 서브클래스가 알맞은 `HashingFunction`만 골라 넘긴다. 솔트를 자체 포맷에 담지 않는 Balloon만 솔트를 직접 관리하는 예외다. 모두 `PasswordEncoder`이므로 `DelegatingPasswordEncoder`에 등록해 기존 마이그레이션 체계에 자연스럽게 끼울 수 있다.
