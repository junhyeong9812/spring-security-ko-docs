# taglibs — JSP 보안 태그 라이브러리

> Spring Security 7.1.1-SNAPSHOT 기준

## 한 줄 정의

JSP(또는 Facelets) 화면 안에서 **현재 인증/인가 상태에 따라 화면 일부를 보이거나 숨기고**, 인증 정보를 출력하며, CSRF 토큰을 폼/메타 태그로 삽입할 수 있게 해 주는 JSP 태그 라이브러리 모듈이다.

## 이 모듈이 푸는 문제

서버에서 인가(authorization)를 거는 것만으로는 화면이 깔끔하지 않다. 권한 없는 사용자에게도 "관리자 메뉴" 링크나 버튼이 그대로 보이면 혼란스럽고, 클릭하면 403만 뜬다. 또 JSP에서 `SecurityContextHolder`를 직접 뒤지면 `null` 체크와 캐스팅이 지저분해진다. 이 모듈은 이런 **뷰 레벨의 보안 표현(UI security)** 을 선언적인 커스텀 태그로 해결한다.

- `<sec:authorize>` — Spring EL 표현식 또는 URL 접근 가능 여부로 태그 본문 노출을 제어한다.
- `<sec:authentication>` — 현재 `Authentication` 객체의 프로퍼티를 안전하게 꺼내 출력하거나 변수에 담는다.
- `<sec:accesscontrollist>` — 특정 도메인 객체에 대한 ACL 권한(`PermissionEvaluator`)을 검사해 본문 노출을 제어한다.
- `<sec:csrfInput>` / `<sec:csrfMetaTags>` — CSRF 토큰을 hidden 필드 또는 meta 태그로 출력한다.

> 주의: 뷰 레벨 노출 제어는 **편의/UX 기능**이지 진짜 인가가 아니다. 실제 접근 통제는 `web` 모듈의 `AuthorizationFilter`/메서드 보안이 담당하며, 태그는 그 결정을 화면에 "투영"할 뿐이다.

## 의존 / 연관 모듈

- **`web`** — `FilterInvocation`, `WebInvocationPrivilegeEvaluator`, `WebAttributes`, `SecurityWebApplicationContextUtils`, `CsrfToken`을 가져다 쓴다. 인가 판단의 실제 두뇌는 web 모듈에 있다. ([../web/README.md](../web/README.md))
- **`core`** — `Authentication`, `SecurityContext`, `SecurityContextHolder(Strategy)`. ([../core/README.md](../core/README.md))
- **`access` (core 내 expression 패키지)** — `SecurityExpressionHandler`, `ExpressionUtils`, `PermissionEvaluator`. EL 평가와 ACL 권한 검사의 기반.
- **Jakarta Servlet/JSP API** — `Tag`, `TagSupport`, `PageContext`. 태그 생명주기를 제공하는 컨테이너 계약.
- **Spring EL(`spring-expression`)** — `Expression`, `EvaluationContext`. `access` 속성을 파싱·평가.

이 모듈에는 인가 알고리즘 자체가 없다. **이미 만들어진 인가/인증 인프라를 JSP에서 호출하는 얇은 어댑터 계층**이다.

## 하위 챕터 목차

1. [01-authorize-인가-태그.md](01-authorize-인가-태그.md) — `<sec:authorize>`의 두뇌인 `AbstractAuthorizeTag`와 JSP 어댑터 `JspAuthorizeTag`, 그리고 노출/숨김을 결정하는 `TagLibConfig`.
2. [02-authentication-과-acl-태그.md](02-authentication-과-acl-태그.md) — 인증 정보를 출력하는 `AuthenticationTag`와 도메인 객체 ACL을 검사하는 `AccessControlListTag`.
3. [03-csrf-태그.md](03-csrf-태그.md) — CSRF 토큰을 폼/메타로 내보내는 `CsrfInputTag` · `CsrfMetaTagsTag`와 공통 골격 `AbstractCsrfTag`.

## 전체 패키지 구조 ASCII 맵

```
taglibs/src/main/java/org/springframework/security/taglibs
│
├── TagLibConfig.java          ── 시스템 프로퍼티 기반 UI 보안 on/off,
│                                 EVAL_BODY_INCLUDE / SKIP_BODY 결정 (내부용)
│
├── authz/                     ── 인가·인증 태그
│   ├── AbstractAuthorizeTag   ── <authorize> 핵심 두뇌 (렌더링 기술 독립)
│   │                             access(EL) / url+method 두 방식의 authorize()
│   ├── JspAuthorizeTag        ── AbstractAuthorizeTag의 JSP Tag 어댑터
│   │                             (+ PageContext 변수 조회 EvaluationContext)
│   ├── AuthenticationTag      ── 현재 Authentication 프로퍼티 출력/변수 바인딩
│   └── AccessControlListTag   ── 도메인 객체 + hasPermission ACL 검사
│
└── csrf/                      ── CSRF 토큰 출력 태그
    ├── AbstractCsrfTag        ── request에서 CsrfToken 추출 후 handleToken() 위임
    ├── CsrfInputTag           ── <input type="hidden"> 형태로 토큰 출력
    └── CsrfMetaTagsTag        ── <meta> 3종(parameter/header/token)으로 출력

taglibs/src/main/resources/META-INF/security.tld
    └── 태그 ↔ 클래스 매핑 (uri: http://www.springframework.org/security/tags)
        authorize / authentication / accesscontrollist / csrfInput / csrfMetaTags
```

각 태그가 JSP에서 어떤 이름으로 노출되는지는 `security.tld`가 정의한다. 예를 들어 `<sec:authorize>`는 `JspAuthorizeTag`로, `<sec:csrfInput>`은 `CsrfInputTag`로 연결된다. (`uri`: `http://www.springframework.org/security/tags`)
