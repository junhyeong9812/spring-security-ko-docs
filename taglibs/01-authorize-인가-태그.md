# 01 — `<authorize>` 인가 태그

## 무엇을 / 왜

`<sec:authorize>`는 이 모듈에서 가장 핵심적인 태그다. **"이 사용자가 권한이 있을 때만 이 화면 조각을 보여라"** 를 선언적으로 표현한다. 관리자에게만 보일 메뉴, 글쓴이에게만 보일 수정 버튼처럼 화면 일부를 권한에 따라 노출/숨김 처리한다.

이 태그의 설계 핵심은 **렌더링 기술과 인가 로직을 분리**한 것이다. 실제 인가 판단(EL 평가, URL 접근 검사)은 JSP를 전혀 모르는 `AbstractAuthorizeTag`가 담당하고, JSP의 `Tag` 생명주기(`doStartTag`/`doEndTag`)에 끼우는 일은 얇은 어댑터 `JspAuthorizeTag`가 맡는다. 덕분에 같은 두뇌를 Facelets 등 다른 뷰 기술에서도 재사용할 수 있다.

소스: `taglibs/src/main/java/org/springframework/security/taglibs/authz/AbstractAuthorizeTag.java`, `JspAuthorizeTag.java`, `taglibs/src/main/java/org/springframework/security/taglibs/TagLibConfig.java`

## 핵심 타입

```
                 ┌──────────────────────────────────────────┐
                 │  AbstractAuthorizeTag (렌더링 기술 독립)   │
                 │                                            │
   속성:         │  - access  (Spring EL 표현식)              │
   access/url/   │  - url, method (URL 접근 검사)             │
   method        │                                            │
                 │  + authorize() : boolean                   │
                 │      ├─ authorizeUsingAccessExpression()   │──▶ SecurityExpressionHandler
                 │      └─ authorizeUsingUrlCheck()           │──▶ WebInvocationPrivilegeEvaluator
                 │                                            │
                 │  abstract getRequest()/getResponse()/      │
                 │           getServletContext()              │
                 └────────────────────▲─────────────────────┘
                                       │ extends + implements Tag
                 ┌─────────────────────┴─────────────────────┐
                 │  JspAuthorizeTag (JSP 어댑터)              │
                 │  - pageContext, var                        │
                 │  + doStartTag() ─▶ super.authorize()       │
                 │  + doEndTag()                              │
                 │  getRequest() = pageContext.getRequest()…  │
                 │  inner PageContextVariableLookup…Context   │
                 └────────────────────────────────────────────┘

   TagLibConfig (전역 정책)
     evalOrSkip(authorized) ─▶ EVAL_BODY_INCLUDE | SKIP_BODY
     isUiSecurityDisabled() ─▶ -Dspring.security.disableUISecurity=true 면 항상 노출
```

- **`AbstractAuthorizeTag`** — 인가 결정의 두뇌. `access`(EL)와 `url`+`method`(URL 접근) 두 가지 판단 방식을 가진다. 둘은 상호 배타적이며 `access` 우선.
- **`JspAuthorizeTag`** — `AbstractAuthorizeTag`를 상속하고 JSP `Tag`를 구현하는 어댑터. `PageContext`에서 request/response/servletContext를 꺼내 부모에 공급한다.
- **`TagLibConfig`** — 결과 boolean을 JSP의 `EVAL_BODY_INCLUDE`/`SKIP_BODY` 반환값으로 번역하고, 전역 UI 보안 스위치를 관리하는 내부용 유틸.

## 동작 흐름

### 시나리오 A: `access` EL 표현식 인가

```jsp
<sec:authorize access="hasRole('ADMIN')">
    <a href="/admin">관리자</a>
</sec:authorize>
```

```
JSP 컨테이너
   │ doStartTag()
   ▼
JspAuthorizeTag.doStartTag()
   │  super.authorize()
   ▼
AbstractAuthorizeTag.authorize()
   │  access 속성에 텍스트 있음 → authorizeUsingAccessExpression()
   ▼
authorizeUsingAccessExpression()
   ├─ getContext().getAuthentication() == null ?  → return false (미인증이면 숨김)
   ├─ getExpressionHandler()  ── ApplicationContext에서
   │      SecurityExpressionHandler<FilterInvocation> 빈 탐색
   ├─ handler.getExpressionParser().parseExpression("hasRole('ADMIN')")
   ├─ createExpressionEvaluationContext(handler)
   │      └─ FilterInvocation(request,response,deny-chain) 으로
   │         handler.createEvaluationContext(authentication, fi)
   │         (JspAuthorizeTag는 PageContext 변수까지 조회하도록 래핑)
   └─ ExpressionUtils.evaluateAsBoolean(expr, ctx)  → true / false
                       │
                       ▼  authorized
JspAuthorizeTag
   ├─ var 지정 시 pageContext에 boolean 저장 (재사용용)
   └─ return TagLibConfig.evalOrSkip(authorized)
            → true  : EVAL_BODY_INCLUDE  (본문 출력)
            → false : SKIP_BODY          (본문 생략)
```

핵심은 `access` 표현식을 평가하는 컨텍스트가 **웹 인가와 동일한 `SecurityExpressionHandler<FilterInvocation>`** 라는 점이다. 즉 필터 체인에서 `hasRole(...)`을 쓸 때와 같은 표현식 루트(`WebSecurityExpressionRoot`)가 적용되어, JSP와 서버 인가가 같은 의미를 갖는다.

### 시나리오 B: `url` 접근 검사 인가

```jsp
<sec:authorize url="/admin" method="POST">…</sec:authorize>
```

```
authorize()
   │  access 없음, url 있음 → authorizeUsingUrlCheck()
   ▼
authorizeUsingUrlCheck()
   ├─ contextPath = request.getContextPath()
   ├─ currentUser = getContext().getAuthentication()
   └─ getPrivilegeEvaluator()
          .isAllowed(contextPath, "/admin", "POST", currentUser)  → boolean
```

`url` 방식은 "이 사용자가 실제로 이 URL에 접근 가능한가?"를 `WebInvocationPrivilegeEvaluator`에게 직접 묻는다. 이 평가기는 필터 체인이 실제 요청에 적용하는 인가 규칙을 그대로 시뮬레이션하므로, **링크의 노출 여부와 실제 접근 권한이 자동으로 일치**한다. 평가기는 먼저 request 속성(`WebAttributes.WEB_INVOCATION_PRIVILEGE_EVALUATOR_ATTRIBUTE`)에서 찾고, 없으면 ApplicationContext에서 단 하나의 빈을 가져온다.

## 핵심 메서드

- **`AbstractAuthorizeTag.authorize()`** — 디스패처. `access`가 있으면 EL 방식, `url`이 있으면 URL 방식, 둘 다 없으면 `false`(숨김). 우선순위와 상호 배타성을 여기서 결정한다.
- **`authorizeUsingAccessExpression()`** — 미인증(`Authentication == null`)이면 즉시 `false`. 이후 `SecurityExpressionHandler`로 EL을 파싱·평가. `ParseException`은 `IOException`으로 감싼다.
- **`getExpressionHandler()`** — ApplicationContext의 모든 `SecurityExpressionHandler` 중 제네릭 타입 인자가 `FilterInvocation`인 것을 골라낸다(즉 웹용). 없으면 명시적 `IOException`으로 "JSP authorize 태그를 쓰려면 WebSecurityExpressionHandler가 필요하다"고 안내한다.
- **`getContext()`** — `SecurityContext`를 얻는다. ApplicationContext에 `SecurityContextHolderStrategy` 빈이 정확히 하나 있으면 그 전략을, 아니면 정적 `SecurityContextHolder.getContext()`를 사용한다. (커스텀 전략을 빈으로 등록한 앱과 호환)
- **`JspAuthorizeTag.doStartTag()`** — `super.authorize()` 결과를 보관(`doEndTag`에서 재사용)하고, `var` 지정 시 page scope에 저장, `TagLibConfig.evalOrSkip(...)`로 본문 노출 여부를 반환.
- **`JspAuthorizeTag.createExpressionEvaluationContext(...)`** — 부모가 만든 컨텍스트를 `PageContextVariableLookupEvaluationContext`로 감싼다. EL 안에서 변수를 찾을 때 먼저 위임 컨텍스트를 보고, 없으면 `pageContext.findAttribute(name)`까지 뒤져 **JSP 페이지 변수를 표현식에서 참조**할 수 있게 한다.

## 설계 포인트 / 확장점

- **Template Method + Adapter** — `AbstractAuthorizeTag`는 인가 알고리즘(불변)을, 추상 메서드 `getRequest/getResponse/getServletContext`로 환경 접근(가변)을 분리한다. JSP용 구현이 `JspAuthorizeTag`다. 다른 뷰 기술은 이 추상 클래스를 상속해 재사용한다.
- **빈 탐색 기반 통합** — 핸들러/평가기/전략을 코드에 박지 않고 ApplicationContext에서 동적으로 찾는다. 따라서 `HttpSecurity` 설정으로 등록된 인가 인프라를 태그가 자동으로 따라간다.
- **`var` 속성** — 인가 결과를 변수에 담아 한 페이지에서 같은 조건을 여러 번 쓸 때 재평가를 피한다.
- **전역 UI 보안 스위치** — `TagLibConfig`는 시스템 프로퍼티 `spring.security.disableUISecurity=true`면 `evalOrSkip`이 항상 `EVAL_BODY_INCLUDE`를 반환한다. 이때 숨겨졌어야 할 본문은 `securedUIPrefix`/`Suffix`(기본 `<span class="securityHiddenUI">…</span>`)로 감싸 출력된다. 디자이너가 "권한 없는 사용자에게 보일 화면"을 한눈에 점검하기 위한 개발용 토글이다(운영에서 켜면 안 됨).

## 정리

`<sec:authorize>`는 `access`(EL) 또는 `url`(접근 검사) 속성으로 화면 조각의 노출을 결정한다. 판단 두뇌인 `AbstractAuthorizeTag`는 뷰 기술과 무관하게 `SecurityExpressionHandler`·`WebInvocationPrivilegeEvaluator`를 ApplicationContext에서 찾아 웹 인가와 동일한 기준으로 평가하고, `JspAuthorizeTag`는 그 결과를 JSP `EVAL_BODY_INCLUDE`/`SKIP_BODY`로 번역한다. 전역 정책과 번역은 `TagLibConfig`가 담당한다.
