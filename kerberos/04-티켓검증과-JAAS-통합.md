# 04. 티켓 검증과 JAAS 통합

## 무엇을 / 왜

03장의 프로바이더는 검증을 `KerberosTicketValidator`/`KerberosClient` 인터페이스에 위임했다. 이 장은 그 인터페이스의 SUN JRE 구현이 **실제로 JAAS/GSS-API 를 어떻게 호출**하는지를 본다. 여기가 모듈에서 가장 저수준이고, "keytab 으로 로그인해서 티켓을 검증한다"는 추상 문장이 코드로 구현되는 지점이다.

소스: `kerberos/src/main/java/org/springframework/security/kerberos/authentication/sun/`.

## 핵심 타입

```
 SunJaasKerberosTicketValidator   (KerberosTicketValidator, InitializingBean)
   afterPropertiesSet(): keytab 으로 LoginContext 로그인 → serviceSubject 보관
   validateTicket(token): Subject.doAs(serviceSubject, GSS accept)

 SunJaasKerberosClient            (KerberosClient)
   login(user, pw): Krb5LoginModule 로 KDC 로그인 → JaasSubjectHolder

 GlobalSunJaasKerberosConfig      (BeanPostProcessor, InitializingBean)
   System.setProperty 로 krb5.conf 위치/디버그 전역 설정

 KerberosMultiTier                위임(delegation) — 사용자 자격으로 다른 서비스 티켓 발급
 JaasUtil                         Subject 복제
 KerberosRuntimeHints (aot.hint)  GraalVM 네이티브 리플렉션 힌트
```

## 동작 흐름 — SunJaasKerberosTicketValidator

### 시작 시점: keytab 으로 서비스 신원 로그인

`afterPropertiesSet()` 가 애플리케이션 기동 때 한 번 실행되어, keytab 에 든 서비스 비밀키로 **JAAS 로그인**해 `serviceSubject`(우리 웹앱의 Kerberos 신원)를 메모리에 올린다.

```java
LoginConfig loginConfig = new LoginConfig(keyTabLocationAsString, servicePrincipal, realmName,
        multiTier, debug, refreshKrb5Config);
Subject sub = new Subject(false, princ /* KerberosPrincipal(servicePrincipal) */, ..., ...);
LoginContext lc = new LoginContext("", sub, null, loginConfig);
lc.login();
this.serviceSubject = lc.getSubject();
```

여기서 `LoginConfig` 는 JAAS `Configuration` 의 인라인 구현으로, **별도 jaas.conf 파일 없이** 코드로 `Krb5LoginModule` 옵션을 채운다. 핵심 옵션(`getAppConfigurationEntry`):

```
useKeyTab=true, keyTab=<경로>, principal=<servicePrincipal>,
storeKey=true, doNotPrompt=true, [isInitiator=false (멀티티어 아닐 때)]
```

`isInitiator=false` 는 "우리는 티켓을 *받기만* 하는 서비스(acceptor)지 개시자가 아니다"라는 뜻이라, 단방향 SPNEGO 검증에 맞다.

### 요청 시점: 티켓 검증 (GSS accept)

```
validateTicket(byte[] token)
  └ Subject.doAs(serviceSubject, KerberosValidateAction(token))
         │  (서비스 신원 권한으로 실행)
         ▼
      GSSContext context = GSSManager.createContext((GSSCredential) null)
      while (!context.isEstablished())
          responseToken = context.acceptSecContext(token, ...)   ★티켓 수락·검증
      gssName = context.getSrcName()        ── 검증된 클라이언트 username
      if (context.getCredDelegState())
          delegationCredential = context.getDelegCred()   ── 위임 자격(있으면)
      if (!holdOnToGSSContext) context.dispose()
      return new KerberosTicketValidation(gssName, servicePrincipal, responseToken, context, delegationCredential)
```

- `acceptSecContext(...)` 가 keytab 으로 티켓을 복호화·검증한다. 성공하면 `getSrcName()` 이 클라이언트의 주체 이름(=username)을 준다. 이 username 이 03장 프로바이더로 흘러가 `UserDetailsService` 의 입력이 된다.
- **`holdOnToGSSContext`**: 기본은 검증 후 `context.dispose()` 로 즉시 폐기한다. true 로 두면 `GSSContext` 를 보존해, 이후 `KerberosServiceRequestToken.encrypt/decrypt`(03장)로 메시지 암복호화를 할 수 있다.
- **위임 자격(`delegationCredential`)**: 클라이언트가 자격 위임을 허용했다면(`getCredDelegState()`) 사용자를 대신해 백엔드 서비스를 호출할 자격을 얻는다.

### 멀티티어 분기

`multiTier=true` 면 `validateTicket` 이 다른 경로를 탄다. `serviceSubject` 를 `JaasUtil.copySubject` 로 복제한 뒤 `KerberosMultitierValidateAction` 을 실행하고, `GSSUtil.createSubject(srcName, delegCred)` 로 **사용자의 위임 자격이 든 Subject** 를 만든다. 이 Subject 가 03장의 `JaasSubjectHolder` 에 실려, 백엔드 서비스에 대한 새 티켓 발급(아래 멀티티어)에 쓰인다.

## 동작 흐름 — SunJaasKerberosClient ([B] 경로)

ID/PW 를 받아 **KDC 에 직접 로그인**한다. 키는 keytab 이 아니라 사용자 비밀번호다.

```java
LoginContext loginContext = new LoginContext("", null,
        new KerberosClientCallbackHandler(username, password), new LoginConfig(debug));
loginContext.login();   // KDC 에 username/password 로 인증
Subject jaasSubject = loginContext.getSubject();
String validatedUsername = jaasSubject.getPrincipals().iterator().next().toString();
Subject subjectCopy = JaasUtil.copySubject(jaasSubject);
result = new JaasSubjectHolder(subjectCopy, validatedUsername);
if (!this.multiTier) { loginContext.logout(); }   // 멀티티어 아니면 즉시 로그아웃
```

- `KerberosClientCallbackHandler` 가 JAAS 의 `NameCallback`/`PasswordCallback` 에 ID/PW 를 주입한다. 그 외 콜백은 `UnsupportedCallbackException` 으로 거부한다.
- `login()` 성공 = KDC 가 그 비밀번호를 인정했다는 뜻이므로, 그 자체로 인증이 성립한다.
- `multiTier` 가 아니면 바로 `logout()` 한다. 멀티티어면 Subject 의 티켓을 보존해 위임에 쓴다.

## 멀티티어 / 위임 — KerberosMultiTier

`KerberosMultiTier.authenticateService(authentication, username, lifetime, targetService)` 는 인증된 사용자의 `JaasSubjectHolder` 를 꺼내, 그 Subject 권한으로 **다른 서비스(targetService)에 대한 새 GSS 토큰**을 만들어 홀더에 저장한다.

```java
Subject.doAs(subject, () -> {
    GSSCredential clientCredential = manager.createCredential(clientName, lifetime, KERBEROS_OID, INITIATE_ONLY);
    GSSContext securityContext = manager.createContext(serverName, KERBEROS_OID, clientCredential, DEFAULT_LIFETIME);
    securityContext.requestCredDeleg(true);
    byte[] outToken = securityContext.initSecContext(new byte[0], 0, 0);
    jaasContext.addToken(targetService, outToken);   // 발급 토큰 캐싱
});
```

이는 "사용자 → 우리 서비스 → 백엔드 서비스" 로 신원을 이어가는 **constrained delegation** 시나리오를 가능하게 한다. 발급된 토큰은 `getTokenForService(authentication, principalName)` 로 꺼낸다.

## 전역 설정 — GlobalSunJaasKerberosConfig

`BeanPostProcessor` 이자 `InitializingBean` 으로, **다른 빈보다 먼저** 생성되어 JVM 시스템 프로퍼티를 세팅한다. JAAS/Krb5 는 이 프로퍼티를 기동 초기에 읽기 때문에 타이밍이 중요하다.

```java
if (debug) System.setProperty("sun.security.krb5.debug", "true");
if (krbConfLocation != null) System.setProperty("java.security.krb5.conf", krbConfLocation);
```

`krb5.conf`(KDC·realm 매핑) 위치와 디버그 출력을 코드로 지정한다.

## AOT / 네이티브 이미지 — KerberosRuntimeHints

`Krb5LoginModule` 은 `LoginConfig` 안에서 **문자열 클래스명**(`"com.sun.security.auth.module.Krb5LoginModule"`)으로 참조되어 JAAS 가 런타임에 `Class.forName` 으로 로드한다. GraalVM 네이티브 이미지는 이런 리플렉션 로딩을 자동 추적하지 못하므로, `KerberosRuntimeHints` 가 명시적으로 리플렉션 힌트를 등록한다.

```java
hints.reflection().registerType(TypeReference.of("com.sun.security.auth.module.Krb5LoginModule"),
        builder -> builder.withMembers(INVOKE_DECLARED_CONSTRUCTORS, INVOKE_DECLARED_METHODS, ACCESS_DECLARED_FIELDS));
```

## 설계 포인트 / 확장점

- **별도 JAAS 설정 파일 불필요**: `Configuration` 을 인라인 구현(`LoginConfig`)으로 두어, 한 JVM 안에서 서비스별로 서로 다른 keytab/옵션을 코드로 구성할 수 있다. 클래스 주석도 "no additional JAAS configuration is needed" 라고 명시한다.
- **JRE 격리**: `com.sun.security.*`/`GSSUtil` 의존을 이 패키지에만 가둔다. 다른 JRE 는 `KerberosTicketValidator`/`KerberosClient` 만 새로 구현하면 된다.
- **keytab 보안 경고**: keytab 이 `ClassPathResource` 면 경고 로그를 남긴다 — keytab 은 비밀이므로 `file:/etc/http.keytab` 처럼 보호된 경로를 쓰라는 의도다.
- **GSSContext 수명 정책**: `holdOnToGSSContext` 로 "검증 후 즉시 폐기 vs 보존(암복호화 가능)" 를 선택한다.

## 정리

`sun` 패키지는 모듈의 엔진룸이다. `SunJaasKerberosTicketValidator` 는 기동 시 keytab 으로 서비스 신원에 로그인해 두고, 요청마다 `Subject.doAs` + `GSSContext.acceptSecContext` 로 티켓을 검증해 username·위임 자격·응답 토큰을 뽑아낸다. `SunJaasKerberosClient` 는 반대로 ID/PW 로 KDC 에 로그인해 검증한다. 멀티티어·전역 krb5 설정·AOT 힌트까지, "JAAS/GSS 의 거친 부분"을 모두 이 패키지가 흡수해 위층(프로바이더·필터)을 깔끔하게 유지한다.
