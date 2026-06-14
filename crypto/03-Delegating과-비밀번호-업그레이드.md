# 03. DelegatingPasswordEncoder와 비밀번호 업그레이드

## 무엇을 / 왜

10년 전 BCrypt strength 8로 저장한 비밀번호와, 오늘 Argon2로 저장한 비밀번호가 같은 사용자 테이블에 섞여 있다. 게다가 내년에는 더 강한 알고리즘으로 바꾸고 싶다. 이런 현실에서 "인코더 하나를 코드에 박아 두는" 방식은 곧바로 막힌다 — 기존 해시를 검증할 수 없거나, 알고리즘을 바꾸는 순간 전체 사용자가 로그인 불능이 된다.

`DelegatingPasswordEncoder`는 이 문제를 **해시 문자열 앞에 `{id}` 접두사를 붙여** 푼다. 접두사가 "이 해시는 어떤 알고리즘으로 만들어졌는가"를 말해 주므로, 한 애플리케이션이 여러 알고리즘을 동시에 검증하면서도 새 비밀번호는 단 하나의 최신 알고리즘으로 통일할 수 있다. 이것이 Spring Security 5.0 이후 **기본** `PasswordEncoder`다.

## 핵심 타입과 저장 포맷

```
   {id}encodedPassword
   └┬┘ └─────┬───────┘
   prefix    선택된 인코더가 만든 실제 해시
   "{bcrypt}" 등

   예시 (모두 평문 "password"):
   {bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
   {noop}password
   {pbkdf2}5d923b44a6d129f3ddf3e3c8d29412723dcbde72445e8ef6bf3b508fbf17fa4e…
   {scrypt}$e0801$8bWJaSu2IKSn9Z9kM+TPXfOc/9bd…
   {argon2}$argon2id$v=19$m=16384,t=2,p=1$…
```

```
              DelegatingPasswordEncoder
              ┌────────────────────────────────────────┐
   raw ──────▶│ encode: 항상 idForEncode("bcrypt") 사용  │
              │   → "{bcrypt}" + bcryptEncoder.encode()  │
              │                                          │
   raw,enc ──▶│ matches: enc 앞의 {id} 추출              │
              │   → idToPasswordEncoder.get(id) 위임      │
              │                                          │
              │  idToPasswordEncoder (Map<String,Enc>)   │
              │    "bcrypt" → BCryptPasswordEncoder       │
              │    "argon2" → Argon2PasswordEncoder       │
              │    "noop"   → NoOpPasswordEncoder         │
              │    "MD5","SHA-1",... (레거시 검증용)        │
              └────────────────────────────────────────┘
```

소스: `crypto/src/main/java/org/springframework/security/crypto/password/DelegatingPasswordEncoder.java`, `crypto/src/main/java/org/springframework/security/crypto/factory/PasswordEncoderFactories.java`

## 동작 흐름 1 — 인코딩 (항상 최신 알고리즘)

```java
protected String encodeNonNullPassword(String rawPassword) {
    return this.idPrefix + this.idForEncode + this.idSuffix
            + this.passwordEncoderForEncode.encode(rawPassword);
}
```

생성자에 넘긴 `idForEncode`(기본 `"bcrypt"`)에 해당하는 인코더 하나만 인코딩에 쓴다. 결과 앞에 `{bcrypt}`를 붙인다. 즉 **새로 저장되는 모든 비밀번호는 단일 최신 알고리즘으로 통일**된다. 알고리즘을 바꾸고 싶으면 `idForEncode`만 `"argon2"`로 바꾸면 되고, 기존 `{bcrypt}` 해시는 여전히 검증된다.

## 동작 흐름 2 — 검증 (접두사로 라우팅)

```java
protected boolean matchesNonNull(String rawPassword, String prefixEncodedPassword) {
    String id = extractId(prefixEncodedPassword);            // "{bcrypt}…" → "bcrypt"
    PasswordEncoder delegate = this.idToPasswordEncoder.get(id);
    if (delegate == null) {
        return this.defaultPasswordEncoderForMatches.matches(rawPassword, prefixEncodedPassword);
    }
    String encodedPassword = extractEncodedPassword(prefixEncodedPassword);  // 접두사 제거
    return delegate.matches(rawPassword, encodedPassword);
}
```

```
matches("password", "{bcrypt}$2a$10$…")
   │
   ├─ extractId  →  "bcrypt"
   ├─ idToPasswordEncoder.get("bcrypt")  →  BCryptPasswordEncoder
   ├─ extractEncodedPassword  →  "$2a$10$…"   (접두사 떼어냄)
   └─ BCryptPasswordEncoder.matches("password", "$2a$10$…")  →  true
```

`extractId`는 문자열이 정확히 `idPrefix`로 시작하는지(`start != 0`이면 `null`), 그리고 닫는 `idSuffix`가 있는지 확인한 뒤 그 사이를 잘라 낸다. id를 찾지 못하면(또는 매핑에 없으면) `defaultPasswordEncoderForMatches`로 넘어간다.

### id를 못 찾았을 때 — 친절한 예외

기본 `defaultPasswordEncoderForMatches`는 내부 클래스 `UnmappedIdPasswordEncoder`로, 상황별로 다른 `IllegalArgumentException` 메시지를 던진다.

```
{notmapped}foo  → "There is no password encoder mapped for the id 'notmapped'…"
password (접두사 없음) → "…each password must have a password encoding prefix.
                          Please either prefix this password with '{noop}' or set
                          a default password encoder…"
```

이 메시지는 5.0 마이그레이션 때 "왜 갑자기 비밀번호가 안 맞지?"를 겪는 개발자를 위한 안내다. `setDefaultPasswordEncoderForMatches`로 레거시(접두사 없는) 해시를 특정 인코더로 처리하게 바꿀 수도 있다.

## 동작 흐름 3 — 로그인 중 자동 업그레이드

`upgradeEncoding`이야말로 이 클래스의 진짜 가치다.

```java
protected boolean upgradeEncodingNonNull(String prefixEncodedPassword) {
    String id = extractId(prefixEncodedPassword);
    if (!this.idForEncode.equalsIgnoreCase(id)) {
        return true;                                 // ① id 가 최신 알고리즘이 아니면 무조건 업그레이드
    }
    String encodedPassword = extractEncodedPassword(prefixEncodedPassword);
    return this.passwordEncoderForEncode.upgradeEncoding(encodedPassword);  // ② 같은 알고리즘이면 비용 비교
}
```

두 가지 경우에 `true`를 반환한다. ① 저장된 id가 현재 `idForEncode`와 다르면(예: `{MD5}`, `{pbkdf2}`인데 현재는 `bcrypt`) 알고리즘 자체가 구식이므로 업그레이드. ② 같은 알고리즘이면 위임 인코더의 `upgradeEncoding`에 물어 비용 파라미터가 약한지 본다(예: `{bcrypt}` strength 8 < 현재 10).

이 신호는 crypto 모듈 **밖**에서 소비된다. `authentication` 모듈의 `DaoAuthenticationProvider`가 인증 성공 직후 `upgradeEncoding`을 확인하고, `true`면 `UserDetailsPasswordService`를 통해 새 해시를 다시 저장한다.

```
   로그인 요청 ($2a$08$… , MD5 등 옛 해시)
        │
        ▼  DaoAuthenticationProvider (authentication 모듈)
   ① passwordEncoder.matches(raw, stored)   ── false ─▶ 인증 실패
        │ true
   ② passwordEncoder.upgradeEncoding(stored) ── false ─▶ 그대로 통과
        │ true
   ③ newHash = passwordEncoder.encode(raw)   →  "{bcrypt}$2a$10$…"
        │
   ④ userDetailsPasswordService.updatePassword(user, newHash)  (DB 갱신)
        ▼
   사용자는 아무것도 모른 채, 로그인할 때마다 비밀번호가 자동으로 강해진다
```

사용자에게 비밀번호 재설정을 요구하지 않고도 전체 사용자 베이스를 점진적으로 최신 알고리즘으로 이전할 수 있다 — 이것이 `{id}` 접두사 설계가 노리는 궁극 목표다.

## PasswordEncoderFactories — 기본 조립

`DelegatingPasswordEncoder`를 직접 조립할 수도 있지만, 보통은 팩토리가 만들어 준 표준 매핑을 쓴다.

```java
public static PasswordEncoder createDelegatingPasswordEncoder() {
    String encodingId = "bcrypt";
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put(encodingId, new BCryptPasswordEncoder());
    encoders.put("noop",  NoOpPasswordEncoder.getInstance());
    encoders.put("pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_5());
    encoders.put("pbkdf2@SpringSecurity_v5_8", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v4_1());
    encoders.put("scrypt@SpringSecurity_v5_8", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_2());
    encoders.put("argon2@SpringSecurity_v5_8", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("MD5", new MessageDigestPasswordEncoder("MD5"));
    encoders.put("SHA-256", new MessageDigestPasswordEncoder("SHA-256"));
    encoders.put("ldap", ...); encoders.put("MD4", ...); encoders.put("sha256", ...);
    return new DelegatingPasswordEncoder(encodingId, encoders);
}
```

주목할 두 가지. 첫째, **인코딩은 `bcrypt` 하나뿐**이고 나머지는 전부 옛 해시 검증·마이그레이션을 위한 등록이다. 둘째, `argon2`와 `argon2@SpringSecurity_v5_8`처럼 **버전 꼬리표가 붙은 id가 공존**한다. 보안 권장값이 바뀌면 옛 파라미터로 저장된 `{argon2}` 해시는 옛 인코더로 검증하고, 새 비밀번호는 더 강한 설정으로 저장하기 위해서다 — 같은 알고리즘 안에서도 무중단 파라미터 업그레이드가 되도록 한 장치다.

## 설계 포인트 / 확장점

- **자기 기술 포맷(self-describing format)**: `{id}`가 해시 자신에 메타데이터를 담아, 검증 시점에 어떤 인코더가 필요한지 데이터 스스로 말한다. 외부 설정·DB 컬럼이 필요 없다.
- **인코딩 단일 / 검증 다중의 비대칭**: 들어올 때(검증)는 관대하게 여러 알고리즘을 받고, 나갈 때(인코딩)는 엄격하게 하나로 통일한다. 이 비대칭이 점진적 마이그레이션을 가능케 한다.
- **확장**: 자체 매핑을 `new DelegatingPasswordEncoder(idForEncode, customMap)`로 만들거나, `setDefaultPasswordEncoderForMatches`로 접두사 없는 레거시 해시 처리를 커스터마이즈할 수 있다. id prefix/suffix(`{`/`}`)도 생성자로 교체 가능하다.

## 정리

`DelegatingPasswordEncoder`는 해시 앞의 `{id}` 접두사로 "검증은 여러 알고리즘, 인코딩은 하나"라는 비대칭 라우팅을 구현한다. 그 덕에 한 애플리케이션이 BCrypt·Argon2·레거시 MD5 해시를 동시에 검증하면서도, 로그인 성공 시 `upgradeEncoding`이 켜지면(상위 `DaoAuthenticationProvider`가 소비) 사용자 모르게 최신 알고리즘으로 재저장한다. `PasswordEncoderFactories.createDelegatingPasswordEncoder()`가 이 모든 것을 기본 설정으로 조립해, 5.0 이후 Spring Security의 표준 비밀번호 처리 방식이 되었다.
