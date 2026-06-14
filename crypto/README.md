# crypto 모듈 — 암호화 유틸리티

> Spring Security 7.1.1-SNAPSHOT 기준. 소스 위치: `crypto/src/main/java/org/springframework/security/crypto`

## 한 줄 정의

비밀번호 해싱(`PasswordEncoder`), 대칭/비대칭 데이터 암호화(`Encryptors`), 키 생성(`KeyGenerators`), 인코딩(Hex/Base64) 같은 **순수 암호화 도구**를 모아 둔, Spring Security의 가장 밑바닥 모듈이다.

## 이 모듈이 푸는 문제

웹 인증의 출발점은 "사용자가 보낸 평문 비밀번호가 저장된 값과 같은가?"라는 질문이다. 이 질문을 안전하게 답하려면 비밀번호를 평문으로 저장하지 않고, 복호화가 불가능한 단방향 함수로 해싱해 두어야 한다. crypto 모듈은 이 단방향 함수(BCrypt, Argon2, SCrypt, PBKDF2)를 `PasswordEncoder`라는 단일 계약 뒤에 숨기고, 시간이 지나 알고리즘이 약해졌을 때 **로그인하는 그 순간에 더 강한 해시로 자동 업그레이드**하는 메커니즘(`DelegatingPasswordEncoder`)까지 제공한다.

그 외에도 토큰·쿠키·설정값처럼 **복호화가 필요한** 데이터를 위한 대칭/비대칭 암호화(`BytesEncryptor`/`TextEncryptor`), 솔트·IV·세션 키 같은 무작위 바이트를 만드는 키 생성기(`KeyGenerators`), 바이트 배열을 문자열로 안전하게 옮기는 인코더(`Hex`, `Base64`)가 함께 들어 있다.

이 모듈은 **인증/인가 흐름 자체를 모른다.** 필터도, `Authentication`도, `UserDetails`도 여기엔 없다. 오직 "바이트를 안전하게 다루는 법"만 안다. 그래서 의존성이 거의 없고, 상위 모듈(`authentication`, `web`, `oauth2`, `ldap`)이 공통으로 끌어다 쓰는 토대가 된다.

## 의존 / 연관 관계

```
        ┌──────────────────────────────────────────────┐
        │  상위 모듈 (이 모듈을 소비)                     │
        │  authentication · web · config · oauth2 · ldap │
        └───────────────────────┬──────────────────────┘
                                 │ uses
                                 ▼
        ┌──────────────────────────────────────────────┐
        │                crypto 모듈                     │
        │  PasswordEncoder · Encryptors · KeyGenerators  │
        └───────────────────────┬──────────────────────┘
                                 │ depends on
                                 ▼
   ┌──────────────┬─────────────────┬─────────────────────┐
   │ JCA (javax.  │ BouncyCastle    │ Password4j          │
   │ crypto, JDK) │ (Argon2/SCrypt/ │ (선택, 7.0+ 신규)    │
   │ 필수          │ AES-GCM) 선택    │ 선택                 │
   └──────────────┴─────────────────┴─────────────────────┘
```

- **JDK 표준 JCA**(`javax.crypto`, `java.security`): BCrypt(자체 구현), PBKDF2, AES-CBC/GCM, `SecureRandom`은 표준 라이브러리만으로 동작한다.
- **BouncyCastle**(선택): Argon2, SCrypt, AES-GCM의 BouncyCastle 변형은 `org.bouncycastle` 의존이 클래스패스에 있어야 한다.
- **Password4j**(선택, 7.0 신규): `password4j` 패키지의 인코더들은 `com.password4j` 라이브러리에 위임한다.
- 상위 모듈들은 이 모듈의 `PasswordEncoder`/`Encryptors`만 알고 구현 알고리즘은 모른다 — 그래서 알고리즘 교체가 설정 한 줄로 끝난다.

## 하위 챕터 목차

1. [01-개요와-PasswordEncoder-계약.md](01-개요와-PasswordEncoder-계약.md) — 모듈 전체 지도, `PasswordEncoder` 인터페이스의 세 메서드, `AbstractValidatingPasswordEncoder` 템플릿
2. [02-단방향-해시-인코더.md](02-단방향-해시-인코더.md) — BCrypt·Argon2·SCrypt·PBKDF2의 적응형 해싱, 솔트·반복비용, 상수시간 비교, 레거시(MessageDigest/NoOp)
3. [03-Delegating과-비밀번호-업그레이드.md](03-Delegating과-비밀번호-업그레이드.md) — `{id}` 접두사 포맷, `DelegatingPasswordEncoder`, `PasswordEncoderFactories`, 로그인 중 자동 업그레이드
4. [04-Password4j-통합.md](04-Password4j-통합.md) — 7.0에서 추가된 외부 라이브러리 위임 인코더 계층
5. [05-데이터암호화-키생성-인코딩.md](05-데이터암호화-키생성-인코딩.md) — `Encryptors`/`BytesEncryptor`/`TextEncryptor`(AES, RSA), `KeyGenerators`, Hex/Base64/Utf8

## 전체 패키지 구조 ASCII 맵

```
org.springframework.security.crypto
│
├── password/                  ← PasswordEncoder 계약 + 템플릿 + 레거시 구현
│   ├── PasswordEncoder              (핵심 인터페이스: encode/matches/upgradeEncoding)
│   ├── AbstractValidatingPasswordEncoder  (null/빈 문자열 처리 템플릿, 7.0)
│   ├── AbstractPasswordEncoder            (솔트+다이제스트 공통 골격)
│   ├── DelegatingPasswordEncoder          (★ {id} 라우팅 + 업그레이드)
│   ├── Pbkdf2PasswordEncoder              (PBKDF2)
│   ├── MessageDigestPasswordEncoder       (@Deprecated, MD5/SHA)
│   ├── StandardPasswordEncoder/Md4/Ldap…  (@Deprecated 레거시)
│   └── NoOpPasswordEncoder                (@Deprecated, 평문)
│
├── bcrypt/                    ← BCrypt 자체 구현
│   ├── BCrypt                       (순수 알고리즘, hashpw/checkpw/gensalt)
│   └── BCryptPasswordEncoder        (PasswordEncoder 어댑터, strength/version)
│
├── argon2/   Argon2PasswordEncoder + Argon2EncodingUtils  (BouncyCastle)
├── scrypt/   SCryptPasswordEncoder                        (BouncyCastle)
│
├── factory/                   ← 기본 조립 지점
│   └── PasswordEncoderFactories     (createDelegatingPasswordEncoder())
│
├── password4j/                ← 7.0 신규, com.password4j 위임
│   ├── Password4jPasswordEncoder        (package-private 베이스)
│   ├── BcryptPassword4jPasswordEncoder
│   ├── Argon2Password4jPasswordEncoder
│   ├── ScryptPassword4jPasswordEncoder
│   ├── Pbkdf2Password4jPasswordEncoder
│   └── BalloonHashingPassword4jPasswordEncoder
│
├── encrypt/                   ← 복호화 가능한 데이터 암호화
│   ├── BytesEncryptor / TextEncryptor   (인터페이스)
│   ├── Encryptors                       (팩토리: standard/stronger/delux/text)
│   ├── AesBytesEncryptor                (AES-CBC/GCM, PBKDF2 키 유도)
│   ├── BouncyCastleAes(Cbc|Gcm)BytesEncryptor
│   ├── Rsa(Raw|Secret)Encryptor         (비대칭)
│   └── KeyStoreKeyFactory / CipherUtils
│
├── keygen/                    ← 무작위 키/솔트/IV 생성
│   ├── BytesKeyGenerator / StringKeyGenerator  (인터페이스)
│   ├── KeyGenerators                    (팩토리: secureRandom/shared/string)
│   ├── SecureRandomBytesKeyGenerator
│   └── Base64StringKeyGenerator / HexEncodingStringKeyGenerator
│
├── codec/                     ← 바이트 ↔ 문자열 인코딩 (내부용)
│   ├── Hex                          (16진 인코딩/디코딩)
│   └── Utf8                         (UTF-8 인코딩/디코딩)
│
└── util/
    └── EncodingUtils                (concatenate / subArray 바이트 유틸)
```

각 챕터는 위 패키지를 책임 단위로 묶어 "왜 존재하고 호출되면 무슨 일이 벌어지는가"를 설명한다.
