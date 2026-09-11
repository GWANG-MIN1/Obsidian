# 08. Google 로그인이 배포 환경에서만 깨졌다

- **발생**: 2026-05-04 · 커밋 `ca9df24` `db04a06` (같은 날 `abb6895` 팀원 작업과 함께)
- **증상 한 줄**: 로컬에서 되는 Google 로그인이 Render 에서 `redirect_uri_mismatch` 로 막혔다

---

## 현상

Google 로그인 버튼을 누르면 Google 동의 화면까지는 가는데, 거기서 에러가 떴다.

```
400: redirect_uri_mismatch
요청의 리디렉션 URI가 승인된 URI와 일치하지 않습니다.
```

**로컬(`http://localhost:8080`)에서는 정상**이었다.

## 환경

- Spring Boot + `spring-boot-starter-oauth2-client`, Google provider
- 배포: Render (프론트·백엔드가 각각 다른 `*.onrender.com` 도메인)
- Google Cloud Console 의 승인된 리디렉션 URI 는 등록해 둔 상태

## 진단

### 1) Google 콘솔 등록값을 의심했다 (원인이 아니었다)

가장 흔한 원인이라 먼저 봤다. 배포 URL 은 등록돼 있었다.
그런데 **Spring 이 보낸 `redirect_uri` 가 등록값과 달랐다.**

### 2) Spring 이 `redirect_uri` 를 스스로 만든다

`spring-security-oauth2-client` 는 `redirect-uri` 를 명시하지 않으면
기본 템플릿으로 **런타임에 조립**한다.

```
{baseUrl}/login/oauth2/code/{registrationId}
```

여기서 `{baseUrl}` 은 **들어온 HTTP 요청에서 유추**한다 — 스킴 · 호스트 · 포트 · 컨텍스트 경로.

```
로컬    요청이 http://localhost:8080 으로 옴
        → redirect_uri = http://localhost:8080/login/oauth2/code/google   ✅ 등록값과 일치

Render  요청이 리버스 프록시를 지나서 옴
        → 스킴/호스트가 프록시 내부 값으로 보일 수 있다
        → redirect_uri = http://<내부호스트>/login/oauth2/code/google      ❌ 불일치
```

**프록시 뒤에서는 "요청에서 유추한 baseUrl"이 외부에서 본 주소와 다르다.**
`X-Forwarded-Proto` / `X-Forwarded-Host` 를 Spring 이 신뢰하도록 설정하지 않으면
`https` 를 `http` 로 보거나 호스트를 내부 이름으로 본다.

### 3) 같은 날 배포 경로 문제도 겹쳐 있었다

`db04a06 fix: Render Docker build path` 가 같은 날이다. Render 의 Docker 빌드 컨텍스트가
저장소 루트가 아니라 `backend_core` 여야 했다. 그리고 팀원이 같은 날
`abb6895 fix: OAuth 리다이렉트 주소 및 CORS 실서버(Render) 주소로 영구 변경` 을 올렸다.

**셋이 같은 증상("배포에서 로그인이 안 된다")으로 뭉쳐 있어서 하나씩 갈라내야 했다.**

## 원인

**`redirect-uri` 를 명시하지 않아서 Spring 이 요청 기준으로 조립했고,
프록시 뒤에서 그 값이 외부 주소와 달랐다.**

## 해결

**환경변수로 못 박았다.** 유추를 아예 안 하게 만드는 방식이다.

```diff
 spring:
   security:
     oauth2:
       client:
         registration:
           google:
             client-id: ${GOOGLE_CLIENT_ID}
             client-secret: ${GOOGLE_CLIENT_SECRET}
+            redirect-uri: ${GOOGLE_OAUTH_REDIRECT_URI}
             scope:
               - email
               - profile
```

`render.yaml` 에 `GOOGLE_OAUTH_REDIRECT_URI` 와 `FRONTEND_OAUTH_REDIRECT_URI` 를
`sync: false`(대시보드에서 입력)로 뒀다. 이틀 뒤 Dockerfile `ARG` 에도
프로덕션 URL 기본값을 넣었다 (`2fd26c1`) — **빌드 시점에 값이 안 들어와도
프로덕션 주소가 기본이 되게** 한 것이다.

## 재발 방지

- **OAuth 의 `redirect_uri` 는 절대 유추하게 두지 않는다.** 환경마다 명시한다.
  이 값은 **Google 콘솔 등록값과 문자 단위로 같아야** 하므로, 유추가 개입할 여지를 없애는 게 맞다.
- **프록시 뒤 배포에서는 `server.forward-headers-strategy=framework` 를 먼저 확인한다.**
  이걸 켜면 Spring 이 `X-Forwarded-*` 를 신뢰해서 `baseUrl` 유추가 정확해진다.
  그러면 `redirect-uri` 를 안 적어도 됐을 것이다. **지금 저장소에는 이 설정이 없다** —
  명시로 우회했을 뿐 근본은 남아 있다. 다른 곳에서 `baseUrl` 유추를 쓰면 또 틀린다.
- **"로컬에서는 된다"를 진단 정보로 쓴다.** 로컬과 배포의 유일한 차이가 **프록시**라면
  의심 목록의 맨 위가 *"요청 헤더로 주소를 만드는 코드"* 여야 한다.
- **한 증상에 여러 원인이 겹쳐 있으면 하나씩 배포해서 갈라낸다.**
  이날 세 커밋이 같이 들어가서, **어느 것이 실제로 고쳤는지 지금도 확실하지 않다.**

## 배운 점

**"설정하지 않음"은 "기본값 사용"이고, 기본값이 환경을 읽는 경우가 있다.**
`redirect-uri` 를 안 적은 것은 선택을 안 한 게 아니라
*"요청에서 유추하겠다"* 를 선택한 것이었다. 그리고 그 유추는 **로컬에서만 맞다.**

같은 형태를 [트러블 07](07-API가-기본-로그인-폼을-돌려준다.md) 에서도 겪었다 —
`formLogin` 을 안 끈 것이 곧 "켠 것"이었다. **Spring 에서 안 적은 것이 가장 위험하다.**

그리고 이 프로젝트 전체를 보면 **배포 환경에서만 깨지는 문제**가 반복된다 —
OAuth 주소, FAB stacking context([트러블 06](06-배포하면-챗봇-버튼이-사라진다.md)),
Render 런타임 포트(PR #124~#129), 콜드스타트.
**로컬에서만 검증하는 습관의 대가**고, 그게 나중에 Supabase 이전의 배경이 됐을 수 있다
([트러블 01](01-Spring이-전시-5일-전에-교체됐다.md) — 확인은 못 했다).
