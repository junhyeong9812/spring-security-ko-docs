# 07 · RequestMatcher와 요청 매칭 — "이 요청이 대상인가?"를 답하는 공통 어휘

## 무엇을 / 왜

앞선 모든 장에서 같은 질문이 반복됐다. "어떤 `SecurityFilterChain`을 적용할까?"(1장), "이 요청이 로그인 처리 URL인가?"(3장), "이 URL에 어떤 인가 규칙을 적용할까?"(4장), "이 요청이 로그아웃인가?"(5장), "CSRF를 검증할 메서드인가?"(6장). 이 모든 "이 요청이 대상인가?"를 하나의 추상화로 통일한 것이 **`RequestMatcher`** 다.

`RequestMatcher`는 Spring Security 웹 보안 전반을 관통하는 **공통 어휘**다. 한 군데서 동작을 이해하면 다른 모든 곳에서 같은 방식으로 재사용된다.

## 핵심 타입

```
RequestMatcher  (@FunctionalInterface)
   boolean matches(HttpServletRequest)
   MatchResult matcher(HttpServletRequest)   ← 매칭 + 추출 변수

구현 (util/matcher):
   AnyRequestMatcher          항상 true (anyRequest)
   PathPatternRequestMatcher  PathPattern 기반 (servlet/util/matcher) ← 권장 기본
   RegexRequestMatcher        정규식
   RequestHeaderRequestMatcher / ParameterRequestMatcher / MediaTypeRequestMatcher
   IpAddressMatcher           IP/CIDR
   DispatcherTypeRequestMatcher  REQUEST/ERROR/ASYNC 등

결합자:
   AndRequestMatcher / OrRequestMatcher / NegatedRequestMatcher
   RequestMatchers (팩토리 유틸)
```

## 동작 흐름 — 매칭과 변수 추출

`RequestMatcher`(`util/matcher/RequestMatcher.java`)는 본질적으로 메서드 하나짜리 함수형 인터페이스다.

```java
@FunctionalInterface
public interface RequestMatcher {
    boolean matches(HttpServletRequest request);

    default MatchResult matcher(HttpServletRequest request) {
        boolean match = matches(request);
        return new MatchResult(match, Collections.emptyMap());
    }
}
```

`matches`는 단순 불리언이지만, `matcher`는 **매칭 여부 + 추출된 경로 변수**를 함께 돌려준다. 이 변수 추출이 인가에서 결정적이다.

```
"/users/{id}/posts" 패턴, 요청 "/users/42/posts"
   PathPatternRequestMatcher.matcher(req)
      → MatchResult(match=true, variables={"id":"42"})
            │
            ▼ (4장) RequestMatcherDelegatingAuthorizationManager
      new RequestAuthorizationContext(request, {"id":"42"})
            │
            ▼ 인가식에서 #id 로 사용 가능
      "authentication.name == @repo.findOwner(#id)"
```

즉 매처는 "대상인가?"뿐 아니라 "대상이라면 URL의 어느 조각이 무엇인가?"까지 알려준다.

## PathPatternRequestMatcher — 권장 기본

`PathPatternRequestMatcher`(`servlet/util/matcher/PathPatternRequestMatcher.java`)는 Spring Web의 `PathPattern` 엔진을 사용한다. 3·5장에서 본 `pathPattern("/login")`, `pathPattern("/logout")`이 모두 이것이다.

```java
public static PathPatternRequestMatcher pathPattern(String pattern) { ... }
public static PathPatternRequestMatcher pathPattern(@Nullable HttpMethod method, String pattern) { ... }
```

- 경로 패턴(`/admin/**`, `/users/{id}`)과 **선택적 HTTP 메서드**를 함께 매칭한다. 메서드가 `null`이면 메서드 무관.
- 경로는 **컨텍스트 패스를 제외한** 상대 경로로 해석된다. `basePath`로 서블릿 경로 접두사를 한 번 정해 여러 매처에 재사용할 수 있다(`Builder`).
- 내부적으로 `PathPattern`은 1장의 `StrictHttpFirewall`이 정규화한 일관된 경로 위에서 동작한다. 그래서 "매처는 통과시키고 디스패처는 다르게 해석"하는 류의 우회가 차단된다.

`PathPattern` 엔진은 과거의 `AntPathRequestMatcher`(`mvcMatchers`)보다 빠르고 매칭 규칙이 명확해, 7.x에서 서블릿 경로 매칭의 기본으로 자리잡았다.

## 결합자 — 작은 매처를 조립하다

복잡한 조건은 작은 매처를 논리 연산으로 합쳐 만든다.

```
AndRequestMatcher(a, b)   →  a.matches && b.matches
OrRequestMatcher(a, b)    →  a.matches || b.matches
NegatedRequestMatcher(a)  →  !a.matches
```

예: "POST이면서 `/api/**`이고 JSON인 요청"은 `PathPatternRequestMatcher(POST, "/api/**")`와 `MediaTypeRequestMatcher(APPLICATION_JSON)`를 `AndRequestMatcher`로 묶어 만든다. 6장의 CSRF 보호 대상 판단(`DefaultRequiresCsrfMatcher`)도 결국 이 `RequestMatcher` 추상화의 한 구현일 뿐이다.

## 핵심 메서드

- **`matches(request)`** — 가장 기본. 1장 `DefaultSecurityFilterChain.matches`, 3장 `requiresAuthentication`, 5장 `requiresLogout`, 6장 CSRF 매처가 모두 이 한 메서드를 호출한다.
- **`matcher(request)` → `MatchResult`** — 매칭 + 변수. 4장 `RequestMatcherDelegatingAuthorizationManager`가 이걸 써서 경로 변수를 인가 컨텍스트로 넘긴다. `MatchResult.match()/notMatch()/match(variables)` 팩토리로 결과를 만든다.
- **`PathPatternRequestMatcher.pathPattern(...)`** — 가장 많이 쓰는 진입점. 보안 설정 곳곳에서 URL/메서드 매처를 만든다.

## 설계 포인트 / 확장점

- **단일 추상화의 재사용** — 체인 선택·필터 적용 여부·인가 매핑·CSRF/로그아웃 대상 판단이 전부 같은 인터페이스를 공유한다. 새 매칭 기준(예: 커스텀 헤더, 테넌트 식별)이 필요하면 `RequestMatcher` 하나를 구현해 어디에나 끼울 수 있다.
- **함수형 인터페이스** — 람다로 즉석 매처를 만들 수 있다(`request -> request.getServerName().endsWith(".internal")`).
- **`RequestMatchers` / `RequestMatcherEditor`** — 매처 생성 유틸과, 문자열 설정을 매처로 변환하는 `PropertyEditor`. XML/프로퍼티 기반 설정에서 쓰인다.
- **first-match 책임은 호출자에게** — `RequestMatcher` 자체는 우선순위를 모른다. 1장 `FilterChainProxy.getFilters`와 4장 `RequestMatcherDelegatingAuthorizationManager.authorize`처럼 **리스트를 위에서부터 순회하며 첫 매칭을 채택**하는 책임은 매처를 담는 쪽에 있다. 그래서 "구체적 규칙을 위에" 두라는 원칙이 반복된다.

## 정리

`RequestMatcher`는 "이 HTTP 요청이 대상인가?"라는 보편 질문을 메서드 하나로 추상화한 공통 어휘다. 체인 선택, 인증/로그아웃 트리거, URL 인가 매핑, CSRF 대상 판정이 모두 이 인터페이스 위에 서 있고, `matcher()`는 매칭과 동시에 경로 변수를 추출해 인가식까지 데이터를 흘려보낸다. `PathPatternRequestMatcher`가 권장 기본 구현이고, And/Or/Negated 결합자로 복합 조건을 조립한다. 매처를 담는 리스트를 위에서부터 first-match로 평가하는 규칙이 보안 설정 전반의 "순서가 중요하다"는 직관의 근거다.
