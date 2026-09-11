# 08. JWT 를 HttpOnly 쿠키와 URL 파라미터 양쪽으로

> 메모: 크로스 오리진에서 쿠키가 안 붙는 환경이 있어서 토큰을 URL 에도 실었다.
> 그 대가로 "CSRF 를 끈 근거"가 배포 환경에서 사라졌다. 2026-03-31 / 05-04~06

---

## 상황

프론트(Next.js)와 백엔드(Spring)가 **다른 도메인**에 배포됐다.
Google OAuth2 콜백은 **백엔드**가 받고, 사용자는 **프론트**로 돌아가야 한다.

```
프론트  https://riskdetectorpeuronteuendeu.onrender.com
백엔드  https://riskdetector-backend…onrender.com
콜백    백엔드가 받음  →  프론트로 리다이렉트하면서 토큰을 넘겨야 함
```

토큰을 프론트에 넘기는 방법이 문제였다. 쿠키는 크로스 사이트에서
`SameSite=None; Secure` 가 아니면 안 붙는다. 그리고 일부 환경(Safari 의 추적 방지,
인앱 브라우저)에서는 **그래도 안 붙는다.**

## 결정

**쿠키와 URL 파라미터 양쪽으로 보냈다.**

```java
// OAuth2SuccessHandler
// 크로스 오리진 환경을 위해 SameSite=None; Secure 쿠키 설정
ResponseCookie cookie = ResponseCookie.from("auth_token", token)
        .httpOnly(true)
        .secure(cookieSecure)
        .sameSite(cookieSecure ? "None" : "Lax")   // 배포(HTTPS)에서는 None
        .path("/")
        .maxAge(Duration.ofMillis(jwtUtil.getExpirationMs()))
        .build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());

// 토큰을 URL 파라미터로도 전달 (크로스 오리진 쿠키 미전달 환경 대비)
String redirectUri = frontendRedirectUri + "?token=" + token;
getRedirectStrategy().sendRedirect(request, response, redirectUri);
```

받는 쪽도 두 경로를 다 본다.

```java
// JwtAuthenticationFilter.resolveToken
// 1. Authorization 헤더 확인 (API 클라이언트용)
// 2. HttpOnly 쿠키 확인 (브라우저 클라이언트용)
```

그리고 토큰이 URL 을 지나가는 걸 알고 **`Referrer-Policy: NO_REFERRER`** 를 걸었다.

```java
.referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER))
```

## 왜

- **로그인이 안 되면 시연이 끝난다.** 전시장에서 심사위원이 자기 폰으로 Google 로그인을
  시도할 텐데, 그 브라우저가 서드파티 쿠키를 막고 있으면 쿠키만으로는 실패한다.
  **두 경로를 두면 하나가 막혀도 로그인은 된다.**
- **쿠키는 `HttpOnly` 로 두고 싶었다.** JS 가 못 읽으니 XSS 로 토큰이 새지 않는다.
  그래서 쿠키를 버리고 URL/localStorage 만 쓰는 선택은 안 했다.
- `Referrer-Policy: NO_REFERRER` 로 **외부 링크 클릭 시 토큰이 새는 경로**는 막았다.

## 왜 다른 건 안 썼나

**토큰을 POST 본문으로 프론트에 전달**

리다이렉트는 GET 이라 본문을 실을 수 없다. 자동 제출 폼 페이지를 백엔드가 렌더링하면 가능한데,
**백엔드에 뷰를 하나 추가**해야 하고 그 페이지 자체가 공격면이 된다.

**일회용 코드(authorization code)를 URL 로 주고 프론트가 교환**

정석이다. URL 에 노출되는 게 **단명 코드**라 유출돼도 대가가 작다.
안 쓴 이유는 교환 엔드포인트 + 코드 저장소(TTL) + 재사용 방지를 새로 만들어야 했고,
`/api/**` 가 전부 `permitAll` 인 상태([결정 06](06-인가를-서비스-계층에서.md))에서
교환 엔드포인트만 제대로 보호할 자신이 없었다.

**프론트와 백엔드를 같은 도메인에 두기 (프록시)**

프론트는 `next.config` 의 rewrites 로 `/api/*` 호출을 Spring 백엔드에 프록시하고 있었다.
일반 API 호출은 그래서 동일 출처처럼 동작했다.
**그런데 OAuth 콜백은 프록시로 해결되지 않는다** —
Google 이 등록된 redirect URI 로 직접 보내고, 그 호스트는 백엔드여야 했다.
(같은 도메인으로 완전히 합쳤다면 이 결정 자체가 필요 없었다. 그게 정답이었을 수 있다.)

## 결과 — 근거가 무너진 자리

`SecurityConfig` 에 이렇게 적혀 있다.

```java
// JWT + Stateless 환경 → CSRF 비활성화 (SameSite=Lax 쿠키로 대체 보호)
.csrf(AbstractHttpConfigurer::disable)
```

그런데 `OAuth2SuccessHandler` 는 **운영에서 `SameSite=None`** 으로 내린다
(`cookieSecure=true` → `"None"`). 즉

- 주석: *CSRF 를 끈 대신 `SameSite=Lax` 가 막아 준다*
- 실제: **운영에서는 `SameSite=None`** — 크로스 사이트 요청에 쿠키가 붙는다

**대체 보호라고 적은 것이 운영에서 존재하지 않는다.** 두 파일을 따로 고치다가 생긴 어긋남이고,
`/api/**` 가 `permitAll` 이라 상태 변경 엔드포인트(`POST /api/ocr/upload`, `POST /api/analysis`)가
CSRF 로 호출될 수 있다. 실제 피해는 "남의 계정으로 계약서를 올려 주는" 정도라 낮지만,
**논리가 깨진 걸 발견한 게 이 노트를 쓰면서다.**

토큰이 URL 에 남는 문제도 완전히 막힌 게 아니다.
`Referrer-Policy` 는 나가는 요청만 막고, **브라우저 히스토리와 프론트 서버 액세스 로그**에는 남는다.

## 배운 점

**타협을 적을 때는 그 타협이 성립하는 조건도 같이 적어야 한다.**
*"CSRF 를 끈다 (SameSite 가 막아 준다)"* 는 문장은 `SameSite` 값이 바뀌는 순간 거짓이 된다.
주석은 **작성 시점의 사실**을 적은 것이고, 그 사실이 다른 파일에서 바뀌는 걸 막지 못한다.

[serverless-uptime-monitor 결정 09](../../serverless-uptime-monitor/decisions/09-tfsec을-soft-fail로.md)
에서 나는 *"게이트가 있다"와 "게이트가 막는다"* 의 차이를 적었다.
여기는 한 칸 더 나쁘다 — **막는다고 적어 뒀는데 안 막는다.**
