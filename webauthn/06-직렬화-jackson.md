# 06. 직렬화 (jackson) 와 AOT 힌트

> 소스: `webauthn/src/main/java/org/springframework/security/web/webauthn/jackson/`, `.../aot/`

## 무엇을 / 왜

WebAuthn은 브라우저 JS API와 서버가 **정확히 합의된 JSON 형태**로 통신해야 한다. 예컨대 모든 바이너리(challenge, rawId, signature…)는 Base64URL 문자열이어야 하고, enum은 스펙이 정한 소문자 문자열(`"public-key"`, `"platform"` 등)이어야 한다. 이 형태가 한 글자라도 어긋나면 브라우저의 `navigator.credentials.*` 호출이 실패한다.

문제는 `api` 패키지의 값 객체들에 Jackson 애너테이션을 직접 박지 않았다는 점이다(02장). 도메인 모델을 직렬화 기술과 분리하기 위해서다. 그래서 `jackson` 패키지가 **외부에서** 직렬화 규칙을 입힌다. 그 도구가 Jackson의 **Mixin**과 커스텀 `Serializer`/`Deserializer`다.

## 핵심 타입

```
jackson 패키지
 ├ WebauthnJacksonModule    ── Jackson 3 (tools.jackson) 용 모듈, SecurityJacksonModule 상속
 ├ WebauthnJackson2Module   ── Jackson 2 (com.fasterxml) 용 모듈 (호환)
 │
 ├ *Mixin / *Jackson2Mixin            ── 대상 타입에 직렬화 메타데이터를 주입
 ├ *Serializer / *Deserializer        ── 값↔JSON 커스텀 변환 (Jackson 3)
 └ *Jackson2Serializer / *Jackson2Deserializer ── 동일 기능의 Jackson 2 판
```

같은 기능이 두 벌(Jackson 3 / Jackson 2) 존재하는 이유는 Spring Framework가 Jackson 3(`tools.jackson`)로 이행하는 과도기이기 때문이다. 필터들의 기본 컨버터(`JacksonJsonHttpMessageConverter`)는 Jackson 3판 `WebauthnJacksonModule`을 쓴다.

### WebauthnJacksonModule — 규칙을 한 곳에 모음

모듈의 `setupModule`이 "어느 타입에 어느 Mixin을 입힐지"를 일괄 등록한다.

```java
public class WebauthnJacksonModule extends SecurityJacksonModule {
    @Override
    public void setupModule(SetupContext context) {
        context.setMixIn(Bytes.class, BytesMixin.class);
        context.setMixIn(AuthenticatorTransport.class, AuthenticatorTransportMixin.class);
        context.setMixIn(PublicKeyCredential.class, PublicKeyCredentialMixin.class);
        context.setMixIn(PublicKeyCredentialCreationOptions.class, PublicKeyCredentialCreationOptionsMixin.class);
        context.setMixIn(PublicKeyCredentialRequestOptions.class, PublicKeyCredentialRequestOptionsMixin.class);
        context.setMixIn(WebAuthnAuthentication.class, WebAuthnAuthenticationMixin.class);
        // ... 등록/인증에 오가는 거의 모든 api 타입
    }
    @Override
    public void configurePolymorphicTypeValidator(BasicPolymorphicTypeValidator.Builder builder) {
        builder.allowIfSubType(WebAuthnAuthentication.class)
               .allowIfSubType(ImmutablePublicKeyCredentialUserEntity.class);
    }
}
```

두 가지 일을 한다. (1) `setMixIn`으로 각 도메인 타입에 직렬화 규칙(생성자/빌더 바인딩, 무시할 필드 등)을 외부 주입한다. (2) `SecurityJacksonModule`을 상속하므로, `WebAuthnAuthentication` 같은 인증 객체를 세션/Redis에 안전하게 저장하기 위한 **다형성 타입 검증**(허용 서브타입 화이트리스트)을 설정한다 — 역직렬화 가젯 공격을 막는 보안 장치다.

### 커스텀 Serializer — Bytes 예시

가장 단순하고 핵심적인 변환은 `Bytes` ↔ Base64URL이다.

```java
class BytesSerializer extends StdSerializer<Bytes> {
    @Override
    public void serialize(Bytes bytes, JsonGenerator jgen, SerializationContext provider) {
        jgen.writeString(bytes.toBase64UrlString());   // byte[] → "dYF7EGnRFFIXkpXi9XU2wg"
    }
}
```

대응 `BytesDeserializer`는 역으로 Base64URL 문자열을 `Bytes.fromBase64(...)`로 되돌린다. 같은 패턴으로 enum들도 스펙 문자열로 변환된다 — `AuthenticatorTransportSerializer`는 `AuthenticatorTransport.USB`를 `"usb"`로, `COSEAlgorithmIdentifierSerializer`는 알고리즘을 정수 식별자로 직렬화하는 식이다.

### 직렬화가 흐름에 끼어드는 지점

03~04장의 필터들이 이 모듈을 어디서 쓰는지 정리하면 다음과 같다.

```
 [등록]
  CreationOptions 응답 쓰기 ── converter.write(options) ── Mixin/Serializer ─▶ JSON(브라우저)
  /register 본문 읽기       ── converter.read(WebAuthnRegistrationRequest) ◀── JSON(클라이언트)
 [인증]
  RequestOptions 응답 쓰기  ── converter.write(options)  ─▶ JSON
  /login/webauthn 본문 읽기 ── converter.read(PublicKeyCredential<Assertion>) ◀── JSON
 [세션]
  WebAuthnAuthentication / CreationOptions 를 HttpSession에 저장 시에도 동일 모듈 사용 가능
```

필터들은 모두 `JsonMapper.builder().addModule(new WebauthnJacksonModule()).build()`로 컨버터를 초기화하므로, 도메인 객체를 `write`/`read`하기만 하면 올바른 WebAuthn JSON 형태가 자동으로 보장된다.

## AOT 런타임 힌트 (aot 패키지)

GraalVM 네이티브 이미지에서는 리플렉션·리소스 접근이 빌드 타임에 등록돼 있어야 한다. `aot` 패키지가 그 힌트를 제공한다.

```java
class UserCredentialRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, @Nullable ClassLoader classLoader) {
        hints.resources()
            .registerPattern("org/springframework/security/user-credentials-schema.sql")
            .registerPattern("org/springframework/security/user-credentials-schema-postgres.sql");
    }
}
```

`JdbcUserCredentialRepository`/`JdbcPublicKeyCredentialUserEntityRepository`가 클래스패스의 스키마 SQL(`user-credentials-schema.sql`, `user-entities-schema.sql`)에 의존하므로(05장), 네이티브 이미지에서도 그 리소스를 찾을 수 있도록 패턴을 등록한다. `PublicKeyCredentialUserEntityRuntimeHints`도 같은 역할을 한다.

## 설계 포인트

- **도메인-직렬화 분리**: `api` 값 객체는 Jackson을 모른다. Mixin으로 외부에서 규칙을 입혀, 도메인 모델을 순수하게 유지하고 직렬화 전략(Jackson 2/3)을 갈아끼울 수 있게 했다.
- **스펙 정확성의 단일 출처**: 모든 변환 규칙이 `WebauthnJacksonModule`에 모여 있어, WebAuthn JSON 호환성을 한 곳에서 관리한다.
- **보안 역직렬화**: `SecurityJacksonModule` 상속 + 다형성 타입 화이트리스트로, 세션에 저장된 인증 객체를 되살릴 때의 가젯 공격을 차단한다.
- **이중 버전 지원**: Jackson 3(`tools.jackson`) 우선, Jackson 2 호환(`*Jackson2*`)을 병행 제공해 마이그레이션 기간을 흡수한다.
- **네이티브 대비**: JDBC 스키마 리소스를 AOT 힌트로 등록해 네이티브 이미지에서도 동작을 보장한다.

## 정리

`jackson` 패키지는 `api` 도메인 객체를 WebAuthn이 요구하는 정확한 JSON(바이너리=Base64URL, enum=스펙 문자열)으로 변환하는 규칙층이다. 도메인에 애너테이션을 박는 대신 Mixin과 커스텀 Serializer를 외부에서 입히며, 그 규칙을 `WebauthnJacksonModule`에 일괄 등록한다. 상속한 `SecurityJacksonModule`은 인증 객체의 안전한 역직렬화까지 책임진다. `aot` 패키지는 JDBC 스키마 리소스를 네이티브 이미지 힌트로 등록해 마무리한다. 이로써 [02장](02-api-도메인-모델.md)의 도메인 모델이 [03](03-등록-흐름.md)·[04](04-인증-흐름.md)장의 HTTP 경계를 정확한 형태로 오갈 수 있게 된다.
