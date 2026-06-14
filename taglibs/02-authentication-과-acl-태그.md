# 02 — `<authentication>` 출력 태그와 `<accesscontrollist>` ACL 태그

## 무엇을 / 왜

`<sec:authorize>`가 "노출/숨김"을 결정한다면, 이 챕터의 두 태그는 각각 다른 일을 한다.

- **`<sec:authentication>`** — 현재 로그인한 사용자의 정보(이름, principal의 프로퍼티 등)를 화면에 **출력**하거나 변수에 담는다. JSP에서 `SecurityContextHolder`를 직접 만지면 `null` 체크와 캐스팅이 번거로운데, 이 태그가 그 안전장치를 대신한다.
- **`<sec:accesscontrollist>`** — 객체 단위 보안(domain object security, ACL)을 위한 태그. "이 사용자가 **이 특정 도메인 객체**에 대해 특정 권한을 가졌는가?"를 `PermissionEvaluator`로 검사해 본문 노출을 제어한다. 역할 기반(`hasRole`)으로는 표현할 수 없는 인스턴스 단위 권한을 다룬다.

소스: `taglibs/src/main/java/org/springframework/security/taglibs/authz/AuthenticationTag.java`, `AccessControlListTag.java`

## 핵심 타입

```
  TagSupport (Jakarta JSP)
     ▲                     ▲
     │                     │
  AuthenticationTag     AccessControlListTag
  ─────────────────     ────────────────────
  - property            - domainObject  (검사 대상 객체)
  - var, scope          - hasPermission (콤마 구분 권한 목록)
  - htmlEscape          - var
  - SCHStrategy         - PermissionEvaluator (ApplicationContext에서 1개)
                        - SCHStrategy

  doEndTag():           doStartTag():
    Authentication에서     domainObject 권한 검사 후
    property 추출 → 출력    EVAL_BODY_INCLUDE | SKIP_BODY
```

두 태그 모두 `TagSupport`를 직접 상속하는 독립 JSP 태그다(앞 챕터의 `AbstractAuthorizeTag` 계층과는 별개). 공통적으로 ApplicationContext에 `SecurityContextHolderStrategy` 빈이 정확히 하나 있으면 그것을, 아니면 정적 holder를 쓴다.

## 동작 흐름

### `<sec:authentication>` — 프로퍼티 출력

```jsp
<sec:authentication property="principal.username" />
```

```
JSP 컨테이너 ── doEndTag()
   ▼
AuthenticationTag.doEndTag()
   ├─ property != null ?
   │     ├─ context = SCHStrategy.getContext()
   │     ├─ context == null || authentication == null ? → return EVAL_PAGE (아무것도 안 함)
   │     ├─ auth.getPrincipal() == null ? → return EVAL_PAGE
   │     └─ result = new BeanWrapperImpl(auth).getPropertyValue("principal.username")
   │              (중첩 프로퍼티 지원, BeansException → JspException)
   │
   ├─ var != null ?
   │     ├─ result != null  : pageContext.setAttribute(var, result, scope)
   │     └─ result == null  : removeAttribute(var[, scope])
   │
   └─ var == null (직접 출력)
         ├─ htmlEscape(default true) : TextEscapeUtils.escapeEntities(result) 출력
         └─ else                     : result 그대로 출력
```

설계의 요점은 두 가지다. 첫째, **`null` 안전** — 미인증 상태나 principal이 없으면 조용히 `EVAL_PAGE`만 반환해 빈 출력이 된다(예외/`null` 노출 없음). 둘째, **XSS 방어** — `var` 없이 직접 출력할 때 기본값 `htmlEscape=true`로 `TextEscapeUtils.escapeEntities`를 거친다. 사용자명 같은 값에 들어간 HTML 특수문자를 이스케이프해 화면 출력 시 스크립트 주입을 막는다.

`property`는 `BeanWrapperImpl`로 평가하므로 `name`, `principal.username`, `authorities` 같은 중첩 경로를 그대로 쓸 수 있다.

### `<sec:accesscontrollist>` — 도메인 객체 ACL 검사

```jsp
<sec:accesscontrollist domainObject="${document}" hasPermission="1,2">
    <a href="/edit">수정</a>
</sec:accesscontrollist>
```

```
JSP 컨테이너 ── doStartTag()
   ▼
AccessControlListTag.doStartTag()
   ├─ hasPermission 비어 있음 ? → skipBody()  (검사할 권한 없음)
   ├─ initializeIfRequired()
   │     ├─ applicationContext = SecurityWebApplicationContextUtils.findRequired…
   │     ├─ permissionEvaluator = 컨텍스트(부모 포함)에서 PermissionEvaluator 1개
   │     │      (0개면 null→NPE 메시지, 2개 이상이면 JspException)
   │     └─ SecurityContextHolderStrategy 빈 1개면 채택
   │
   ├─ domainObject == null ? → evalBody()   ("null 객체엔 누구나 접근 가능")
   ├─ authentication == null ? → skipBody() (미인증이면 숨김)
   │
   ├─ requiredPermissions = parseHasPermission("1,2")
   │      └─ 각 토큰을 Integer로 파싱 시도, 실패하면 String 그대로
   │
   └─ for each permission :
         permissionEvaluator.hasPermission(auth, domainObject, permission)
            == false 인 것이 하나라도 있으면 → skipBody()   (AND 조건)
         모두 통과 → evalBody()

   skipBody()/evalBody() : var 지정 시 page scope에 Boolean 저장 후
                           TagLibConfig.evalOrSkip(...) 반환
```

`hasPermission`은 콤마로 구분된 권한 목록이며 **AND** 로 평가된다(전부 가져야 본문 노출). 각 토큰은 우선 `Integer`로 파싱을 시도하고(ACL 권한 마스크는 보통 정수), 실패하면 문자열 권한명으로 둔다. 실제 판단은 전적으로 `PermissionEvaluator`에 위임된다 — 이 모듈은 ACL 저장소나 권한 마스크 의미를 전혀 모른다.

## 핵심 메서드

- **`AuthenticationTag.doEndTag()`** — 인증 객체에서 `property`를 추출해 출력/바인딩하는 전체 로직. `null` 단계마다 `EVAL_PAGE`로 빠져나가 안전하게 동작.
- **`AuthenticationTag.setPageContext(...)`** — page context가 설정될 때 ApplicationContext를 찾아 `SecurityContextHolderStrategy` 빈이 하나면 그것을 채택한다. 즉 전략 선택이 태그 초기화 시점에 이뤄진다.
- **`AccessControlListTag.initializeIfRequired()`** — 첫 호출 때만 ApplicationContext·`PermissionEvaluator`·전략을 해결한다. `getBeanOfType`은 **부모 컨텍스트까지 거슬러 올라가며** 빈을 모으고, 0개면 명시적 NPE 메시지, 2개 이상이면 `JspException("must have only have one")`을 던진다.
- **`AccessControlListTag.parseHasPermission(...)`** — `"1,2"` 같은 문자열을 권한 객체 리스트로 변환. 정수 우선, 실패 시 문자열.
- **`skipBody()` / `evalBody()`** — `var` 바인딩(각각 `FALSE`/`TRUE`)과 `TagLibConfig.evalOrSkip` 반환을 한곳에서 처리하는 헬퍼.

## 설계 포인트 / 확장점

- **`PermissionEvaluator` 교체** — ACL 검사 로직은 전적으로 ApplicationContext에 등록된 `PermissionEvaluator` 빈에 위임된다. `AclPermissionEvaluator`를 쓰든 커스텀 구현을 쓰든 태그 코드 변경이 없다. 단, 컨텍스트 내(부모 포함)에 정확히 하나만 있어야 한다.
- **`SecurityContextHolderStrategy` 빈 인지** — 두 태그 모두 정적 holder 대신 빈으로 등록된 전략을 우선한다. 리액티브/멀티테넌트 등 커스텀 컨텍스트 전략과 호환된다.
- **출력 안전 기본값** — `AuthenticationTag`의 `htmlEscape` 기본값이 `true`라는 점이 보안상 중요하다. 직접 출력 경로에서 항상 이스케이프가 적용된다.
- **`var`/`scope`** — 결과를 변수로 내보내 다른 태그나 EL에서 재사용. `AuthenticationTag`는 scope까지 지정 가능하며, `result`가 `null`이면 변수를 제거한다.

## 정리

`<sec:authentication>`은 현재 `Authentication`에서 프로퍼티를 안전하게(미인증·`null`·XSS 고려) 꺼내 출력하거나 변수에 담는 출력 태그다. `<sec:accesscontrollist>`는 도메인 객체 인스턴스에 대한 권한을 `PermissionEvaluator`에게 AND 조건으로 물어 본문 노출을 제어하는 객체 단위 보안 태그다. 두 태그 모두 인가 로직 자체는 ApplicationContext의 인프라 빈에 위임하고, JSP 표면만 담당한다.
