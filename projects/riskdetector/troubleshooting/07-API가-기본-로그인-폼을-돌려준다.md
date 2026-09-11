# 07. API 를 부르면 Spring 기본 로그인 폼이 나온다

- **발생**: 2026-04-10 · 커밋 `5604c30`
- **증상 한 줄**: JSON 을 기대한 요청에 Spring Security 의 기본 로그인 **HTML 페이지**가 돌아왔다

---

## 현상

프론트가 API 를 호출하면 JSON 파싱이 깨졌다. 응답 본문이 이렇게 시작했다.

```html
<!DOCTYPE html><html lang="en"><head><title>Please sign in</title>
```

브라우저에서 직접 열면 **Spring 이 자동 생성한 로그인 폼**(Username / Password 입력창)이 떴다.
우리는 그런 폼을 만들지 않았다.

`fetch` 쪽에서는 이렇게 보인다.

```
SyntaxError: Unexpected token '<', "<!DOCTYPE "... is not valid JSON
```

## 환경

- Spring Boot 3.5 + Spring Security + OAuth2 Client
- 인증은 **Google OAuth2 + JWT** 만 쓸 계획 (자체 아이디/비밀번호 로그인 없음)
- `SecurityConfig` 에 `httpBasic(disable)` 은 이미 걸려 있었다

## 진단

### 1) 그 폼은 우리가 만든 게 아니다

`formLogin` 은 **Spring Security 의 기본값이다.** 명시적으로 끄지 않으면
`spring-boot-starter-security` 가 붙는 순간

- `/login` 에 로그인 폼 페이지를 자동 등록하고
- 인증이 필요한 요청을 **그 페이지로 302 리다이렉트**한다

`httpBasic` 은 껐지만 `formLogin` 은 안 껐다. 둘은 별개 설정이다.

### 2) 왜 JSON API 가 리다이렉트를 받나

인증이 안 된 요청이 `authenticated()` 경로에 닿으면 Spring 은
**`AuthenticationEntryPoint`** 를 호출한다. `formLogin` 이 켜져 있으면 그 entry point 가
`LoginUrlAuthenticationEntryPoint` 이고, 하는 일은 **로그인 페이지로 보내기**다.

```
GET /api/auth/me   (JWT 없음)
  → 인증 실패
    → LoginUrlAuthenticationEntryPoint
      → 302 Location: /login
        → 200 text/html (Please sign in)
```

`fetch` 는 리다이렉트를 자동으로 따라가므로, 프론트에는 **HTML 200** 이 도착한다.
**401 이 아니라 200 이 오는 게 이 증상의 핵심**이다 — 에러 처리 분기에도 안 걸린다.

## 원인

**`formLogin` 을 끄지 않았다.** Spring Security 의 기본 동작이라
아무 코드도 쓰지 않았는데 존재한다.

## 해결

한 줄.

```diff
 http
         .httpBasic(AbstractHttpConfigurer::disable)
+        .formLogin(AbstractHttpConfigurer::disable)
         // JWT + Stateless 환경 → CSRF 비활성화 (SameSite=Lax 쿠키로 대체 보호)
         .csrf(AbstractHttpConfigurer::disable)
```

그리고 인증 실패를 **401 로 명시**했다 (같은 설정 파일에 있다).

```java
.exceptionHandling(exception -> exception
        .authenticationEntryPoint((request, response, authException) -> {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
        })
)
```

## 재발 방지

- **REST API 전용 Spring 앱은 기본값 3종을 같이 끈다.** `httpBasic` · `formLogin` ·
  `logout`(기본 로그아웃 페이지). 하나만 끄면 나머지가 남는다.
- **`AuthenticationEntryPoint` 를 항상 명시한다.** 이걸 지정해 두면
  다른 기본값이 켜져 있어도 응답이 HTML 로 새지 않는다. **끄는 것보다 이게 더 근본적이다.**
- **HTML 응답을 프론트에서 감지한다.** `Content-Type` 이 `application/json` 이 아니면
  파싱 전에 에러로 처리하는 래퍼가 있으면, *"JSON 파싱 실패"* 대신
  *"서버가 HTML 을 돌려줬다"* 는 진단이 바로 나온다.

## 배운 점

**프레임워크의 기본값은 "안 쓰면 없는 것"이 아니다.**
로그인 폼을 만든 적이 없는데 로그인 폼이 응답했다. `spring-boot-starter-security` 를
의존성에 넣은 것만으로 **켜지는 기능이 여러 개**고, 그중 어떤 게 켜졌는지는
설정 파일을 봐서는 알 수 없다 — **안 적힌 것이 기본값이기 때문**이다.

**그리고 인증 실패가 200 으로 오는 건 조용한 실패의 한 종류다.**
401 이면 프론트가 로그인 유도를 했을 것이다. 200 + HTML 은
*"성공했는데 이상한 데이터가 왔다"* 로 보여서 **엉뚱한 곳(프론트 파싱)을 먼저 의심하게 만든다.**
이 프로젝트에서 반복된 형태다 → [트러블 04](04-OCR-결과에-1페이지만-남았다.md) ·
[트러블 02](02-환각을-막으려고-정상-질문까지-막았다.md)
