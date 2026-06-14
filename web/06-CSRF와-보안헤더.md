# 06 · CSRF 보호와 보안 응답 헤더 — 브라우저를 거드는 방어

## 무엇을 / 왜

인증·인가가 "서버 쪽 결정"이라면, 이 장의 두 주제는 **브라우저의 동작을 이용·통제해 공격을 막는** 방어다.

- **CSRF(Cross-Site Request Forgery)** — 로그인된 피해자의 브라우저가 공격자 사이트의 유도로 의도치 않은 상태 변경 요청을 보내는 공격. 브라우저는 쿠키를 자동 첨부하므로 인증만으로는 막을 수 없다. 해법은 **공격자가 알 수 없는 토큰**을 상태 변경 요청에 요구하는 것(synchronizer token pattern).
- **보안 헤더** — `X-Frame-Options`(클릭재킹), `Strict-Transport-Security`(HTTPS 강제), `Content-Security-Policy`(XSS 완화) 등 브라우저에 방어를 지시하는 응답 헤더들.

## 핵심 타입

```
CsrfFilter (OncePerRequestFilter)
   ├── CsrfTokenRepository      토큰 로드/저장
   │     ├── HttpSessionCsrfTokenRepository  (세션, 기본 성향)
   │     └── CookieCsrfTokenRepository        (쿠키, SPA 친화)
   ├── CsrfTokenRequestHandler  토큰을 요청 속성으로 노출 + 요청에서 토큰 값 추출
   │     ├── XorCsrfTokenRequestAttributeHandler (기본, BREACH 완화 마스킹)
   │     └── CsrfTokenRequestAttributeHandler
   └── requireCsrfProtectionMatcher  보호 대상 판단 (기본: GET/HEAD/TRACE/OPTIONS 제외)

HeaderWriterFilter (OncePerRequestFilter)
   └── List<HeaderWriter>
         ├── XFrameOptionsHeaderWriter, HstsHeaderWriter
         ├── ContentSecurityPolicyHeaderWriter, ReferrerPolicyHeaderWriter
         ├── XContentTypeOptionsHeaderWriter, CacheControlHeadersWriter
         └── CrossOrigin{Opener,Embedder,Resource}PolicyHeaderWriter ...
```

## 동작 흐름 1 — CSRF 검증

`CsrfFilter.doFilterInternal()`(`csrf/CsrfFilter.java:107`)이 매 요청에서 두 가지를 한다: 토큰을 **제공**하고(읽기 요청), 토큰을 **검증**한다(상태 변경 요청).

```
요청 ──▶ CsrfFilter
   │ deferredToken = tokenRepository.loadDeferredToken(req, res)
   │ request.setAttribute(DeferredCsrfToken, deferredToken)  ← 뷰가 토큰 렌더링하도록
   │ requestHandler.handle(req, res, deferredToken)           ← _csrf 속성 노출
   │
   │ requireCsrfProtectionMatcher.matches(req)?
   │   아니오(GET/HEAD/TRACE/OPTIONS) → chain.doFilter (검증 생략, 토큰만 제공)
   ▼ 예 (POST/PUT/DELETE 등)
  csrfToken = deferredToken.get()                  ← 저장된(또는 새로 생성한) 기대 토큰
  actualToken = requestHandler.resolveCsrfTokenValue(req, csrfToken)
                                                    ← 파라미터 "_csrf" 또는 헤더 "X-CSRF-TOKEN"
   │
   ├─ equalsConstantTime(expected, actual) 실패
   │     missing = deferredToken.isGenerated()       ← 기대 토큰이 막 생성됐다면 = 클라가 안 보냄
   │     throw missing ? MissingCsrfTokenException : InvalidCsrfTokenException
   │     → accessDeniedHandler.handle (403)
   │
   └─ 일치 → chain.doFilter (통과)
```

핵심 포인트:

- **읽기 요청은 토큰을 제공만, 쓰기 요청은 검증**. `DefaultRequiresCsrfMatcher`가 GET·HEAD·TRACE·OPTIONS를 "안전한 메서드"로 보고 검증에서 제외한다(REST 의미론 준수가 전제).
- **상수 시간 비교**(`equalsConstantTime` → `MessageDigest.isEqual`)로 타이밍 공격을 막는다.
- **지연 토큰(`DeferredCsrfToken`)** — 2장의 지연 컨텍스트와 같은 발상. 토큰이 실제 필요해질 때만 저장소/세션을 건드린다. `isGenerated()`로 "기존 토큰이 없어 방금 만들었는지"를 구분해, 토큰 누락(클라이언트가 안 보냄)과 토큰 불일치(잘못 보냄)를 다른 예외로 처리한다.

### XOR 마스킹 — BREACH 완화

기본 핸들러 `XorCsrfTokenRequestAttributeHandler`는 응답에 노출하는 토큰을 매번 랜덤 값으로 XOR 마스킹한다. 같은 토큰이라도 요청마다 다른 문자열로 보이게 해, 압축+추측 기반 사이드채널 공격(BREACH)으로 토큰을 알아내기 어렵게 만든다. 검증 시에는 마스크를 풀어 원래 토큰과 비교한다.

### 저장소 선택

- **`HttpSessionCsrfTokenRepository`** — 토큰을 세션에 둔다. 서버가 신뢰하는 저장소.
- **`CookieCsrfTokenRepository`** — 토큰을 쿠키(`XSRF-TOKEN`)에 둔다. JS가 쿠키를 읽어 헤더로 되돌려보내는 SPA 패턴에 맞다. `withHttpOnlyFalse()`로 JS 접근을 허용한다.

## 동작 흐름 2 — 보안 헤더 작성

`HeaderWriterFilter.doFilterInternal()`(`header/HeaderWriterFilter.java:69`)은 기본적으로 **응답이 커밋되기 직전**에 헤더를 쓴다.

```
요청 ──▶ HeaderWriterFilter
   │ shouldWriteHeadersEagerly?
   │   true  → 즉시 writeHeaders 후 체인 진행
   │   false(기본) → 응답을 HeaderWriterResponse 로 래핑해 체인 진행
   │                  └─ 응답 커밋 시점에 writeHeaders 호출
   ▼ writeHeaders:
   for (writer : headerWriters) writer.writeHeaders(req, res)
        XFrameOptions → "X-Frame-Options: DENY"
        Hsts         → "Strict-Transport-Security: max-age=...; includeSubDomains"
        ContentSecurityPolicy → "Content-Security-Policy: ..."
        XContentTypeOptions   → "X-Content-Type-Options: nosniff"
        ...
```

지연 작성(`doHeadersAfter`)을 기본으로 하는 이유는, 체인 하류(예: 인증 실패로 다른 응답을 쓰는 경우)에서도 최종 응답에 헤더가 확실히 붙도록 하기 위해서다. `HeaderWriterResponse`가 커밋 직전 콜백으로 헤더를 보장한다.

각 `HeaderWriter`는 한 종류의 헤더만 책임진다(SRP). `DelegatingRequestMatcherHeaderWriter`로 특정 경로에만 헤더를 적용하거나, `StaticHeadersWriter`로 임의 정적 헤더를 추가할 수 있다.

## 핵심 메서드

- **`CsrfFilter.doFilterInternal` (:107)** — 위 흐름 전체. "제공 + (조건부)검증"을 한 자리에서 처리한다.
- **`CsrfTokenRequestHandler.resolveCsrfTokenValue`** — 요청에서 실제 토큰을 꺼내는 방식(파라미터명 `_csrf`, 헤더명 `X-CSRF-TOKEN`)을 캡슐화. XOR 핸들러는 여기서 마스크를 해제한다.
- **`CsrfToken` / `DeferredCsrfToken.get()` / `isGenerated()`** — 토큰 본문과 "방금 생성 여부". 누락 vs 불일치를 가르는 신호.
- **`HeaderWriterFilter.writeHeaders` (:97)** — 등록된 모든 `HeaderWriter`를 순회 호출. 헤더 정책 추가 = 라이터 추가.

## 설계 포인트 / 확장점

- **CSRF와 인증의 협력** — `CsrfAuthenticationStrategy`(`SessionAuthenticationStrategy` 구현)는 로그인 시 CSRF 토큰을 새로 발급해 세션 고정과 유사한 토큰 고정을 막는다. `CsrfLogoutHandler`는 로그아웃 시 토큰을 제거한다. 즉 CSRF는 3·5장의 인증/세션 생애주기와 맞물려 동작한다.
- **`requireCsrfProtectionMatcher` 교체** — 특정 엔드포인트(웹훅 수신 등)를 CSRF 검증에서 제외하거나, `CsrfFilter.skipRequest(request)`로 런타임에 건너뛴다.
- **헤더 라이터 조합** — 필요한 `HeaderWriter`만 골라 리스트로 주입한다. config의 `headers {}` DSL이 이 리스트를 구성한다.
- **CORS** — 모듈 힌트에 언급된 CORS는 본 모듈이 아니라 Spring Web의 `CorsFilter`/`CorsConfigurationSource`로 처리되며, config가 보안 체인 적절한 위치(인증 앞)에 끼운다. `web` 모듈은 그 필터가 놓일 자리를 제공할 뿐 CORS 로직 자체를 소유하지 않는다.

## 정리

`CsrfFilter`는 안전한 메서드에는 토큰을 제공만 하고, 상태 변경 요청에는 저장된 기대 토큰과 요청 토큰을 상수 시간으로 비교해 CSRF를 막는다. 토큰은 지연 로딩되고 XOR로 마스킹되어 사이드채널을 줄이며, 누락과 불일치를 구분해 다른 예외로 처리한다. `HeaderWriterFilter`는 한 헤더당 하나씩의 `HeaderWriter`들을 응답 커밋 직전에 실행해 브라우저 방어 헤더를 일관되게 붙인다. 둘 다 "브라우저의 동작을 전제로 한 방어"라는 점에서, 서버 측 인증·인가를 보완하는 마지막 겹의 보호막이다.
