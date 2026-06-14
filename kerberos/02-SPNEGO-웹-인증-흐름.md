# 02. SPNEGO 웹 인증 흐름

## 무엇을 / 왜

`kerberos-web` 모듈은 브라우저와 서버 사이의 SPNEGO 협상을 서블릿 필터 체인 위에 구현한다. 책임은 두 가지로 나뉜다.

1. **협상 시작** — 인증 안 된 요청에 `401 + WWW-Authenticate: Negotiate` 를 돌려보내 브라우저가 티켓을 보내도록 유도한다. → `SpnegoEntryPoint`
2. **티켓 수신** — 다음 요청의 `Authorization: Negotiate ...` 헤더를 파싱해 `AuthenticationManager` 로 넘긴다. → `SpnegoAuthenticationProcessingFilter`

소스: `kerberos/src/main/java/org/springframework/security/kerberos/web/authentication/` (실제 경로는 `kerberos-web/...`).

## 핵심 타입

```
 ┌───────────────────────────────────────────────────────────┐
 │ SpnegoEntryPoint  (AuthenticationEntryPoint)              │
 │   commence(): 401 + "WWW-Authenticate: Negotiate"        │
 │   (옵션) forwardUrl → 폼 로그인 폴백으로 forward          │
 └───────────────────────────────────────────────────────────┘
 ┌───────────────────────────────────────────────────────────┐
 │ SpnegoAuthenticationProcessingFilter (OncePerRequestFilter)│
 │   Negotiate 헤더 파싱 → KerberosServiceRequestToken      │
 │   → authenticationManager.authenticate()                 │
 │   → SecurityContext 저장                                  │
 └───────────────────────────────────────────────────────────┘
 ┌───────────────────────────────────────────────────────────┐
 │ ResponseHeaderSettingKerberosAuthenticationSuccessHandler │
 │   성공 시 응답 토큰을 WWW-Authenticate 헤더로 회신        │
 │   (상호 인증 mutual auth 용)                              │
 └───────────────────────────────────────────────────────────┘
```

## 동작 흐름 — 전체 협상 왕복

```
[1] 브라우저 ──▶ GET /secure  (티켓 없음)
        │
        │  인가 필터가 인증 없음 감지 → ExceptionTranslationFilter
        │  → EntryPoint.commence()
        ▼
    SpnegoEntryPoint
        response.addHeader("WWW-Authenticate", "Negotiate")
        response.setStatus(401)
        │
[2] 브라우저 ◀── 401 Negotiate
        │  (브라우저가 KDC 에서 서비스 티켓 획득)
        ▼
[3] 브라우저 ──▶ GET /secure
        Authorization: Negotiate YIIF... (base64)
        ▼
    SpnegoAuthenticationProcessingFilter.doFilterInternal()
        ├ 이미 인증됨? → skip (skipIfAlreadyAuthenticated=true)
        ├ 헤더가 "Negotiate "/"Kerberos " 로 시작? (NTLM 은 제외)
        ├ base64 디코드 → byte[] kerberosTicket
        ├ new KerberosServiceRequestToken(ticket)  (미인증)
        ├ authenticationManager.authenticate(token)  ── 03장
        ├ sessionStrategy.onAuthentication(...)
        ├ SecurityContext 생성·저장
        └ successHandler?.onAuthenticationSuccess(...)
        ▼
[4] chain.doFilter() → 보호 자원 도달
```

## 핵심 메서드

### `SpnegoEntryPoint.commence(...)`

핵심은 단 두 줄로, 브라우저에게 "Negotiate 로 인증하라"고 알리는 것이다.

```java
response.addHeader("WWW-Authenticate", "Negotiate");
response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
```

생성자에 `forwardUrl` 을 주면 401 을 내면서 동시에 그 URL(예: 폼 로그인 페이지)로 `RequestDispatcher.forward` 한다. 이는 **SPNEGO 를 지원하지 않는 클라이언트(비도메인 기기)** 를 위한 폼 로그인 폴백을 가능하게 한다. `forwardUrl` 은 상대 경로여야 하고(`UrlUtils.isValidRedirectUrl` 검증), `forwardMethod` 로 forward 시 HTTP 메서드를 바꿀 수도 있다(`HttpServletRequestWrapper` 로 `getMethod()` 오버라이드).

### `SpnegoAuthenticationProcessingFilter.doFilterInternal(...)`

이 필터의 동작에서 주목할 설계 디테일들:

- **이미 인증된 요청은 건너뜀**: `skipIfAlreadyAuthenticated`(기본 true)면 `AnonymousAuthenticationToken` 이 아닌 인증이 이미 있을 때 바로 체인을 통과시킨다. 매 요청마다 티켓 재검증을 피한다.
- **NTLM 배제**: 헤더가 `Negotiate ` 로 시작해도 `Negotiate TlRMTVNTUA`(="NTLMSSP" 의 base64) 로 시작하면 무시한다. 윈도우가 Kerberos 티켓 발급에 실패하면 NTLM 으로 폴백하는데, 이 필터는 NTLM 을 처리하지 못하므로 걸러낸다. `Kerberos ` 접두사도 함께 허용한다.

```java
if (header != null && ((header.startsWith("Negotiate ") && !header.startsWith(NTLMSSP_PREFIX))
        || header.startsWith("Kerberos "))) {
    byte[] base64Token = header.substring(header.indexOf(" ") + 1).getBytes("UTF-8");
    byte[] kerberosTicket = Base64.getDecoder().decode(base64Token);
    KerberosServiceRequestToken authenticationRequest = new KerberosServiceRequestToken(kerberosTicket);
    ...
    authentication = this.authenticationManager.authenticate(authenticationRequest);
}
```

- **실패는 거의 항상 서버 설정 오류**: `AuthenticationException` 을 잡으면 경고 로그를 남기고, `failureHandler` 가 없으면 `500 Internal Server Error` 를 낸다. 클라이언트가 잘못된 티켓을 "재시도"하는 일이 드물기 때문에, SPNEGO 실패는 보통 keytab/서비스 주체 설정이 틀린 서버측 문제로 간주된다.
- **컨텍스트 저장 전략**: 기본 `SecurityContextRepository` 가 `RequestAttributeSecurityContextRepository` 다. 즉 기본은 세션에 저장하지 않고 요청 범위에만 둔다. 세션 저장이 필요하면 교체한다. 인증 성공 후 `stopFilterChainOnSuccessfulAuthentication=true` 면 체인을 멈출 수도 있다.

### `ResponseHeaderSettingKerberosAuthenticationSuccessHandler.onAuthenticationSuccess(...)`

상호 인증(mutual authentication)이 필요할 때 사용하는 성공 핸들러다. 검증 결과로 만들어진 `KerberosServiceRequestToken` 에 **응답 토큰**이 있으면, 그것을 다시 `WWW-Authenticate: Negotiate <token>` 헤더로 브라우저에 돌려보내 "서버도 진짜임"을 증명한다.

```java
KerberosServiceRequestToken auth = (KerberosServiceRequestToken) authentication;
if (auth.hasResponseToken()) {
    response.addHeader(this.headerName, this.headerPrefix + auth.getEncodedResponseToken());
}
```

응답 토큰은 `SunJaasKerberosTicketValidator` 가 `context.acceptSecContext(...)` 로 받은 출력 토큰이며(04장), 표준 1패스 SPNEGO 에서는 비어 있을 수 있다.

## 설계 포인트 / 확장점

- **EntryPoint 와 Filter 의 역할 분리**: 협상 시작(401)과 티켓 처리(헤더 파싱)는 서로 다른 요청 라이프사이클에서 일어나므로 별 컴포넌트로 나뉜다. Spring Security 설정에서는 `exceptionHandling().authenticationEntryPoint(spnegoEntryPoint)` 와 커스텀 필터 등록(BASIC 위치)으로 묶는다.
- **폼 로그인 폴백**: `SpnegoEntryPoint(forwardUrl)` 한 줄로 SSO 와 폼 로그인을 공존시킬 수 있다.
- **세션/디테일 커스터마이즈**: `setSessionAuthenticationStrategy`, `setAuthenticationDetailsSource`, `setSecurityContextRepository` 로 세션 고정 방지·요청 메타데이터·컨텍스트 저장 위치를 조정한다.

## 정리

`kerberos-web` 은 SPNEGO 의 두 박자(401 유도 → 티켓 수신)를 각각 `SpnegoEntryPoint` 와 `SpnegoAuthenticationProcessingFilter` 로 구현한다. 필터는 헤더에서 티켓 바이트를 꺼내 `KerberosServiceRequestToken` 으로 감싼 뒤 `AuthenticationManager` 에 위임할 뿐, 실제 검증은 다음 장의 프로바이더가 맡는다. NTLM 배제·이미 인증된 요청 스킵·폼 폴백 같은 실전 디테일이 이 모듈의 가치다.
