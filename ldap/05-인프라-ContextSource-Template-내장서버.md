# 05. 인프라 — ContextSource, Template, 내장 서버

## 무엇을 / 왜

앞 챕터들이 "인증/권한 전략"이었다면, 이 챕터는 그 전략들이 딛고 서는 **하부 인프라**다. LDAP 서버와의 연결(접속 풀링 포함), 안전한 검색을 위한 인코딩, 권한 검색용 템플릿, 비밀번호 정책 컨트롤, 그리고 테스트·개발용 내장 서버를 다룬다. 보안 관점에서 특히 중요한 건 **풀링과 인증의 충돌**, **LDAP 인젝션 방지**다.

## 핵심 타입

```
ContextSource (Spring LDAP)
   └ LdapContextSource
        └ DefaultSpringSecurityContextSource   URL 파싱 + 풀링 제어
             └ PasswordPolicyAwareContextSource  ppolicy 컨트롤 수신

LdapTemplate (Spring LDAP)
   └ SpringSecurityLdapTemplate               compare / 단일엔트리 / 다중속성 검색

LdapEncoder (root)                            DN·필터 RFC 인코딩 (인젝션 방지)
LdapUtils  (root)                             URL→rootDn 파싱 등

EmbeddedLdapServerContainer (interface)       내장 서버 수명주기
   └ UnboundIdContainer                       UnboundID In-Memory 서버
```

## 동작 흐름 ① — ContextSource와 풀링의 함정

`DefaultSpringSecurityContextSource`는 LDAP URL을 받아 `LdapContextSource`를 설정한다 (`.../ldap/DefaultSpringSecurityContextSource.java`). 생성자에서 URL(`ldap://host:389/dc=example,dc=com`)을 파싱해 서버 URL과 base DN(`rootDn`)을 분리하고, 여러 URL은 base DN이 같은지 검증한다(페일오버).

핵심은 **풀링 제어**다:

```
setPooled(true)   ← 연결 풀링 켬 (성능)
   │
   └ 그러나 인증은 "사용자 비번으로 접속"이라 풀링하면 위험
     (다른 사용자 연결이 재사용될 수 있음)
   ▼
SimpleDirContextAuthenticationStrategy.setupEnvironment 오버라이드:
   접속 DN != userDn(관리자) 이면 → 풀링 플래그(SUN_LDAP_POOLING_FLAG) 제거
```

즉 관리자 컨텍스트(검색용)는 풀링하되, `getContext(userDn, password)`로 특정 사용자로 bind하는 인증 접속은 풀링에서 빼서 연결이 섞이지 않게 한다. `BindAuthenticator`가 사용자로 접속할 때 이 보호가 작동한다.

### PasswordPolicyAwareContextSource

`ppolicy` 패키지의 이 서브클래스는 사용자로 bind할 때 먼저 관리자로 접속한 뒤 `reconnect`로 사용자 자격증명과 함께 `PasswordPolicyControl`을 실어 보낸다 (`.../ppolicy/PasswordPolicyAwareContextSource.java`). 그래야 bind 실패 시 서버가 보낸 정책 컨트롤(계정 잠김, 비번 만료 임박 등)을 추출할 수 있다. 잠김이면 `PasswordPolicyException`을 던지고, `LdapAuthenticationProvider`가 이를 `LockedException`으로 번역한다(02챕터).

## 동작 흐름 ② — SpringSecurityLdapTemplate

Spring LDAP의 `LdapTemplate`을 확장해 보안 모듈이 필요로 하는 연산을 더했다 (`.../ldap/SpringSecurityLdapTemplate.java`). 핵심 3개:

```
compare(dn, attr, value)
  └ OBJECT_SCOPE 검색으로 "(attr={value})" 매칭 여부 → boolean
    (PasswordComparisonAuthenticator 의 비번 비교에 사용)

searchForSingleEntry(base, filter, params)
  └ 정확히 1건을 요구. 0건/2건+ 면 IncorrectResultSizeDataAccessException.
    PartialResultException(AD) 은 무시. (FilterBasedLdapUserSearch 가 사용)

searchForMultipleAttributeValues(base, filter, params, attrNames)
  └ 매칭 엔트리들의 지정 속성값을 Map<String,List<String>> 집합으로.
    각 record 에 DN_KEY 로 전체 DN 도 포함.
    (DefaultLdapAuthoritiesPopulator 의 그룹 검색에 사용)
```

여기서 보안상 결정적인 부분은 `searchForMultipleAttributeValues`가 필터 파라미터를 치환하기 전에 **모두 인코딩**한다는 점이다:

```java
encodedParams[i] = LdapEncoder.filterEncode(params[i].toString());
String formattedFilter = MessageFormat.format(filter, encodedParams);
```

## 동작 흐름 ③ — LDAP 인젝션 방지 (LdapEncoder)

사용자 입력(username 등)이 그대로 DN이나 검색 필터에 들어가면 LDAP 인젝션이 가능하다. 이 모듈은 두 지점에서 인코딩한다:

```
[DN 조립]   AbstractLdapAuthenticator.getUserDns():
              args = LdapEncoder.nameEncode(username)
              → "uid={0},ou=people" 의 {0} 에 안전하게 치환

[필터 검색] SpringSecurityLdapTemplate.searchForMultipleAttributeValues / 단일검색:
              LdapEncoder.filterEncode(param)  (RFC 2254: * ( ) \ NUL 이스케이프)
```

`nameEncode`는 DN 특수문자(`, + " \ < > ;` 등)를, `filterEncode`는 필터 메타문자를 이스케이프한다. 덕분에 `*)(uid=*` 같은 입력이 필터 구조를 깨뜨리지 못한다.

## 동작 흐름 ④ — 내장 LDAP 서버 (UnboundIdContainer)

실 디렉터리 없이 테스트/개발하려면 in-memory 서버가 필요하다. `UnboundIdContainer`는 UnboundID In-Memory Directory Server를 Spring 빈 수명주기에 얹는다 (`.../server/UnboundIdContainer.java`):

```
afterPropertiesSet() → start():
  InMemoryDirectoryServerConfig(defaultPartitionSuffix="dc=example,dc=com")
    + addAdditionalBindCredentials("uid=admin,ou=system","secret")
    + listener(port; 0 이면 랜덤 포트)
  루트 엔트리(dc=example...) 추가
  importLdif(ldif 리소스) → 초기 데이터 적재
  startListening()
  port = 실제 리슨 포트  ← 0(에페메랄) 이었으면 여기서 확정

destroy()/stop() → shutDown   (에페메랄 + 컨텍스트 살아있으면 보존)
```

`EmbeddedLdapServerContainer` 인터페이스는 `getPort/setPort`만 노출한다. 포트 0을 주면 랜덤 가용 포트를 잡고(`isEphemeral`), 시작 후 실제 포트를 다시 읽어올 수 있어 테스트 병렬 실행에 유리하다. LDIF 파일로 사용자·그룹을 미리 심어 두고, 앞 챕터의 provider들이 이 서버를 향해 bind/검색하도록 구성한다.

## 설계 포인트 / 확장점

- **풀링 vs 인증의 분리** — 검색용 관리자 연결만 풀링하고 사용자 bind는 제외하는 전략으로 성능과 안전을 동시에 잡는다.
- **인코딩을 인프라 층에 못박음** — DN 조립과 필터 검색의 단일 진입점에서 인코딩해, 상위 전략 코드가 인젝션을 신경 쓰지 않아도 되게 한다.
- **템플릿 메서드 재사용** — `SpringSecurityLdapTemplate`은 Spring LDAP의 콜백/예외 변환을 그대로 활용하면서 보안 특화 연산만 얹는다.
- **수명주기 통합** — 내장 서버가 `InitializingBean`/`DisposableBean`/`Lifecycle`을 구현해 컨테이너가 자동으로 띄우고 내린다.
- **ppolicy 확장점** — 정책 컨트롤이 필요한 디렉터리에서는 ContextSource만 `PasswordPolicyAwareContextSource`로 바꿔 끼우면 잠김/만료가 인증 예외로 흘러든다.

## 정리

전략들이 동작하려면 그 아래에 연결·검색·인코딩 인프라가 있어야 한다. `DefaultSpringSecurityContextSource`는 URL을 파싱하고 인증 접속을 풀링에서 제외해 안전하게 연결하며, `SpringSecurityLdapTemplate`은 compare·단일/다중 검색을 제공하되 파라미터를 `LdapEncoder`로 이스케이프해 LDAP 인젝션을 막는다. `PasswordPolicyAwareContextSource`는 정책 컨트롤을, `UnboundIdContainer`는 테스트용 내장 서버를 책임진다.
