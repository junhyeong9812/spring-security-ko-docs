# 03 — CSRF 토큰 태그

## 무엇을 / 왜

Spring Security의 CSRF 보호가 켜져 있으면, 상태를 바꾸는 요청(POST/PUT/DELETE 등)에는 유효한 CSRF 토큰이 함께 와야 한다. JSP에서 Spring의 `<form:form>` 태그를 쓰면 이 토큰이 hidden 필드로 자동 삽입되지만, **순수 HTML `<form>`을 쓰거나 JavaScript/AJAX로 요청을 보낼 때**는 토큰을 직접 넣어야 한다. 이 챕터의 두 태그가 그 일을 한다.

- **`<sec:csrfInput>`** — HTML `<form>` 안에 `<input type="hidden">`으로 토큰을 삽입한다. (`<form:form>`을 못 쓸 때의 대체재)
- **`<sec:csrfMetaTags>`** — `<head>`에 `<meta>` 3종을 출력해, JavaScript에서 토큰의 파라미터명/헤더명/값을 읽어 AJAX 요청에 실을 수 있게 한다.

소스: `taglibs/src/main/java/org/springframework/security/taglibs/csrf/AbstractCsrfTag.java`, `CsrfInputTag.java`, `CsrfMetaTagsTag.java`

## 핵심 타입

```
  TagSupport (Jakarta JSP)
        ▲
        │
  AbstractCsrfTag                 ── 공통 골격 (package-private)
   doEndTag():
     token = request.getAttribute(CsrfToken.class.getName())
     if token != null:
         out.write( handleToken(token) )    ← 추상 메서드
     return EVAL_PAGE
        ▲                    ▲
        │ extends            │ extends
  CsrfInputTag          CsrfMetaTagsTag
   handleToken():        handleToken():
     <input type=hidden>   <meta _csrf_parameter>
     name=파라미터명        <meta _csrf_header>
     value=토큰값          <meta _csrf>
```

핵심은 **Template Method 패턴**이다. `AbstractCsrfTag.doEndTag()`가 토큰 조회와 `null` 처리라는 공통 흐름을 고정하고, "토큰을 어떤 HTML 문자열로 바꿀지"만 추상 메서드 `handleToken(CsrfToken)`으로 하위 클래스에 위임한다.

## 동작 흐름

```jsp
<%-- <head> 안 --%>
<sec:csrfMetaTags />

<%-- 순수 HTML form 안 --%>
<form method="post" action="/transfer">
    <sec:csrfInput />
    ...
</form>
```

```
JSP 컨테이너 ── doEndTag()
   ▼
AbstractCsrfTag.doEndTag()
   ├─ token = pageContext.getRequest()
   │            .getAttribute("org.springframework.security.web.csrf.CsrfToken")
   │
   ├─ token == null ?  → 아무것도 출력하지 않고 EVAL_PAGE
   │     (CSRF 보호가 꺼져 있으면 이 속성이 없음 → 태그는 조용히 무시됨)
   │
   └─ token != null :
         out.write( handleToken(token) )
                       │
       ┌───────────────┴───────────────┐
       ▼                               ▼
 CsrfInputTag.handleToken         CsrfMetaTagsTag.handleToken
   <input type="hidden"             <meta name="_csrf_parameter"
     name="{parameterName}"           content="{parameterName}" />
     value="{token}" />             <meta name="_csrf_header"
                                      content="{headerName}" />
                                    <meta name="_csrf"
                                      content="{token}" />
```

토큰은 `CsrfToken` 객체로 request 속성에 담겨 있는데, 이것을 거기에 넣어 주는 주체는 이 모듈이 아니라 `web` 모듈의 `CsrfFilter`(또는 토큰 리포지토리/요청 핸들러)다. 즉 태그는 **이미 요청에 실려 온 토큰을 화면 문법으로 렌더링**할 뿐이고, 토큰 생성·검증은 전부 필터 체인의 몫이다. 그래서 CSRF 보호가 비활성화되어 토큰이 없으면 태그는 아무 출력도 하지 않는다.

## 핵심 메서드

- **`AbstractCsrfTag.doEndTag()`** — request 속성 `CsrfToken.class.getName()`에서 토큰을 꺼내고, 있으면 `handleToken` 결과를 출력, 없으면 `EVAL_PAGE`만 반환. `IOException`은 `JspException`으로 감싼다.
- **`CsrfInputTag.handleToken(token)`** — `token.getParameterName()`을 name으로, `token.getToken()`을 value로 한 hidden input 문자열을 만든다. 폼 전송용.
- **`CsrfMetaTagsTag.handleToken(token)`** — `_csrf_parameter`(파라미터명), `_csrf_header`(헤더명), `_csrf`(토큰값) 세 meta 태그를 만든다. JS에서 폼 제출은 파라미터명, AJAX는 헤더명을 사용하라는 의도.

## 설계 포인트 / 확장점

- **Template Method** — 공통 흐름(토큰 조회·`null` 처리)과 가변 부분(렌더링 문자열)을 깔끔히 분리. 새로운 출력 형식이 필요하면 `AbstractCsrfTag`를 상속해 `handleToken`만 구현하면 된다.
- **무설정·무위임 통합** — ApplicationContext 빈 조회조차 하지 않는다. 오직 request 속성 하나(`CsrfToken`)에만 의존하므로 가장 가벼운 태그다. CSRF 인프라와의 결합점은 그 속성 키 하나다.
- **JS 연동 규약** — `csrfMetaTags`가 내보내는 meta 이름(`_csrf_parameter`/`_csrf_header`/`_csrf`)은 클라이언트 스크립트가 `meta[name='_csrf']` 식으로 읽는 사실상의 규약이다. AJAX 요청에는 `_csrf_header` 이름의 헤더로 토큰을 실어 보낸다.

## 정리

CSRF 태그는 요청에 이미 실려 온 `CsrfToken`을 화면 문법으로 렌더링하는 출력 전용 태그다. `AbstractCsrfTag`가 토큰 조회와 `null` 시 무출력이라는 공통 흐름을 고정하고, `CsrfInputTag`는 hidden 폼 필드를, `CsrfMetaTagsTag`는 meta 3종을 만든다. 토큰의 생성·검증은 `web` 모듈의 CSRF 필터가 맡고, 이 태그들은 그 결과를 폼/JS가 사용할 수 있게 출력만 한다.
