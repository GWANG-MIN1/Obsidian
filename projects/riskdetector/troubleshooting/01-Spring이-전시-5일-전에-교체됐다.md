# 01. 두 달 만든 Spring 백엔드가 전시 5일 전에 교체됐다

- **발생**: 2026-06-02 · 커밋 `2863732` (팀원 작업, 내 커밋 아님)
- **증상 한 줄**: `render.yaml` 에서 `riskdetector-backend` 블록이 사라지고, 내 엔드포인트 10개가 977줄짜리 Supabase Edge Function 으로 다시 구현됐다

---

## 현상

2026-06-02, 커밋 하나가 이런 stat 을 남겼다.

```
 render.yaml                                        |    42 +-
 supabase/functions/rd-api/index.ts                 |   977 +++++++
 supabase/migrations/20260602000000_init_riskdetector.sql |  126 +
 supabase/seed.sql                                  |  2923 ++++++++++++++++
 frontend/riskdetector-web/src/api/auth.ts          |    94 +-
 frontend/riskdetector-web/src/lib/supabase.ts      |    33 +
 ...
 28 files changed, 4483 insertions(+), 146 deletions(-)
```

`render.yaml` 의 변화가 결정적이다.

```diff
 services:
-  - type: web
-    name: riskdetector-backend
-    runtime: docker
-    dockerfilePath: ./backend_core/Dockerfile
-    ...   (JWT_SECRET, GOOGLE_CLIENT_ID, AWS_LAMBDA_OCR_FUNCTION … 환경변수 16개)
   - type: web
     name: riskdetector-frontend
```

**Spring Boot 서비스가 배포 대상에서 빠졌다.** 코드는 저장소에 남았지만 아무도 안 띄운다.
프론트의 API 기본값도 바뀌었다.

```ts
const API_BASE = process.env.NEXT_PUBLIC_API_BASE_URL || 'http://localhost:54321/functions/v1/rd-api';
```

내 마지막 백엔드 커밋은 **6월 1일**(`f36f340`)이었다. 하루 뒤였다.

## 환경

- 전시 발표까지 약 5일
- 그때까지 구성: Render(프론트 + Spring + PostgreSQL) + AWS(Lambda 3개 · Bedrock · S3)
- 내가 만든 것: Spring Boot 59파일 2,839줄 — 컨트롤러 7 · 서비스 4 · 엔티티 7 · 시큐리티 4
- 교체 후: Render(프론트만) + Supabase(Auth · Postgres · Edge Function) + AWS(그대로)

## 진단

### 1) 무엇이 대체됐나 — 전부는 아니다

```
대체됨   인증 (Spring OAuth2 + 자체 JWT)       → Supabase Auth
대체됨   DB 접근 (JPA · 엔티티 7개)             → Edge Function 안의 Supabase 클라이언트
대체됨   엔드포인트 10개                        → rd-api/index.ts 의 라우트 분기
대체됨   PostgreSQL (Render)                   → Supabase Postgres (데이터 이관 없음)
살아남음  Lambda 3개 (OCR · 분석 · 적재)         → Edge Function 이 같은 함수를 호출
살아남음  챗봇 라우트 (Next.js route.ts)         → 프론트에 있었으므로 무관
살아남음  analysis_result_loader                → DB 접속 정보만 Supabase 로 교체
```

**설계는 거의 그대로 옮겨졌다.** Edge Function 을 열어 보면 내 구조가 그대로 있다.

```ts
if (req.method === 'GET'  && path === '/auth/me')          return handleAuthMe(scope);
if (req.method === 'POST' && path === '/ocr/upload')       return handleOcrUpload(req, db, scope);
if (req.method === 'POST' && path === '/analysis')         return handleAnalysisStart(req, db, scope);
if (req.method === 'POST' && path === '/chatbot/retrieve') return handleChatbotRetrieve(req);
```

게스트 모델도 그대로다 — `X-Guest-Id` 헤더, `guest_session_id` 컬럼, 소유권 분기.
→ [결정 07](../decisions/07-게스트를-X-Guest-Id로.md)

```ts
const guestSessionId = req.headers.get('X-Guest-Id')?.trim() || null;
...
if (hasText(contract.guest_session_id)) {
  if (scope.guestSessionId === contract.guest_session_id) return contract;
```

그리고 이전 문서의 확인 절차 마지막 항목이 내가 만든 인가 규칙이다.

> 6. 다른 guestId 또는 다른 Google 계정으로 같은 계약 조회가 404가 되는지 확인합니다.

**404 로 돌려주는 것**까지 옮겨졌다 → [결정 06](../decisions/06-인가를-서비스-계층에서.md)

### 2) 왜 교체했나 — 비용

**Render 운영 비용 때문이다.** 학생 팀 프로젝트에 붙은 상시 비용이라,
전시가 끝난 뒤에도 서비스를 살려 두려면 줄여야 했다.

Render 구성은 **웹 서비스 2개 + PostgreSQL** 이었다.

| Render 자원 | 성격 |
|---|---|
| `riskdetector-backend` (Spring Boot, Docker) | 상시 인스턴스 — **잠들면 콜드스타트 30초** |
| `riskdetector-frontend` (Next.js, Docker) | 상시 인스턴스 |
| PostgreSQL | 관리형 DB |

Supabase 로 옮기면 이 셋 중 **둘이 사라진다.** Postgres·Auth·Edge Function 이
한 프로젝트 안에 무료 티어로 들어오고, Render 에는 프론트 하나만 남는다.
`render.yaml` 이 49줄에서 13줄로 줄어든 게 그 결과다.

그리고 비용이 이유였기 때문에 **데이터를 안 옮겼다.**

> 기존 Render PostgreSQL 데이터는 복구하지 않고 새 Supabase DB에서 시작합니다.

기존 Render DB 를 계속 띄워 두고 덤프를 옮기는 대신 **새 DB 에서 시작**하는 쪽이
그 시점에 더 싸고 빨랐다. 시연 데이터는 다시 만들면 되는 것이었다.

### 3) 곁따라온 이득 — 콜드스타트

비용이 목적이었고 콜드스타트 개선은 부수 효과다. 그런데 그게 작지 않았다.

- Spring 인스턴스가 잠들면 **첫 요청이 30초**였다. 발표 자료 체크리스트에도
  *"Render 백엔드 / 프론트 모두 깨어 있는지 (cold start 30초 정도 걸림 — 미리 한번 깨워두기)"* 가
  적혀 있다 — **시연 리스크로 인식돼 있었다.**
- 챗봇에 콜드스타트 대응 커밋이 3개 붙어 있다 (`Prewarm Ardi chatbot cold starts`,
  `Avoid Ardi cold start refusals`, `Prefer OpenAI after chatbot warmup`).
  내 KB 검색 경로가 백엔드를 지나가는데, 그 백엔드가 자고 있으면
  4초 타임아웃을 넘긴다 → [트러블 02](02-환각을-막으려고-정상-질문까지-막았다.md)
- Edge Function 은 콜드스타트가 훨씬 짧다. JVM 부팅이 없다.

즉 **비용을 줄이는 변경이 시연 안정성도 같이 올렸다.** 교체를 안 할 이유가 얇았다.

## 원인

**교체의 이유는 비용이다.** 내가 코드로 고칠 수 있는 종류가 아니었다 —
Spring Boot 를 Render 에 띄우는 한 인스턴스 비용은 그대로고, JVM 기동 시간도 그대로다.

그래서 내가 돌아볼 부분은 "왜 교체됐나"가 아니라 **"교체가 이렇게 쉬웠던 이유"** 다.

**내가 만든 것은 "Spring 구현"이었고, 팀에 필요한 것은 "동작하는 API"였다.**
설계(엔드포인트 · 게스트 모델 · 404 인가 · Lambda 연동)는 살아남았고
구현체만 갈렸다. 즉 **내 기여의 어느 부분이 자산이고 어느 부분이 껍데기였는지가
교체로 드러났다.**

그리고 왜 교체가 쉬웠는지도 분명하다.

| | 상태 | 결과 |
|---|---|---|
| 자동 테스트 | **1개** (`contextLoads`) | 교체 후 동등성을 검증할 기준이 없다 |
| API 스펙 문서 | 없음 (OpenAPI·Swagger 미도입) | 977줄을 **코드 읽고** 재구현해야 했다 |
| 통합 테스트 | 없음 | 회귀를 잡을 그물이 없다 |

**테스트가 있었다면 교체가 어려웠을 거라는 뜻이 아니다.**
반대다 — 테스트가 있었다면 **교체가 안전했을** 것이고, 교체본이 내 스펙을 지키는지
기계가 말해 줬을 것이다. 그 대신 사람이 절차 문서 6단계를 손으로 확인했다.

## 해결

없다. 되돌리지 않았다. 최종 배포는 Supabase 구성으로 갔고, 전시에서 **장려상**을 받았다.

내가 그 뒤에 한 일은 6월 7일의 커밋 3개 — **GitHub Pages 소개 페이지와 README** 다.
코드가 아니라 프로젝트를 설명하는 작업이었다.

## 재발 방지

- **경계를 코드가 아니라 계약으로 만든다.** 엔드포인트 스펙(OpenAPI)이 있었다면
  교체본이 그 스펙을 지키는지 자동으로 확인할 수 있었다. 구현은 갈릴 수 있지만
  **계약은 남는다** — 실제로 이 프로젝트에서 남은 것도 계약(엔드포인트·게스트 모델)이었다.
- **테스트는 내 코드를 보호하는 게 아니라 내 설계를 문서화한다.** 이 프로젝트에서
  테스트를 안 쓴 이유를 "팀이 안 쓰니까"로 넘겼는데, 그 결과 **내 설계 의도가 코드에만 있었다.**
  혼자 만든 [gym-management-db](../../gym-management-db/README.md) 에는 테스트 56개가 있다.
  협업할 때 더 필요한 걸 협업할 때 안 썼다.
- **운영 비용을 스택 선택의 조건으로 처음부터 센다.** 이건 코드 품질 문제가 아니라
  **선택 시점의 문제**다. "Java 경험자가 있으니 Spring"으로 정할 때
  *상시 인스턴스 + 관리형 DB 를 몇 달 유지할 수 있는가* 는 고려하지 않았다.
  전시가 끝나도 살려 둘 서비스라면 그 질문이 처음에 나와야 했다.
- **내 계층의 비용을 내가 안다.** 백엔드 담당인데 Render 요금이 어디서 나오는지,
  내 서비스 하나를 빼면 얼마가 줄어드는지 **계산해 본 적이 없다.**
  같은 누락을 솔로 프로젝트에도 적어 뒀다 —
  [serverless-uptime-monitor](../../serverless-uptime-monitor/README.md) 는
  *"비용을 집계하지 않았다. 프리티어 범위라고 판단했을 뿐"* 이다.
- **배포 파이프라인을 남에게 맡기지 않는다.** `render.yaml` 을 나는 거의 안 봤다.
  내 서비스가 배포 설정에서 빠지는 것을 **커밋 로그로 알았다.**
  그 파일에 내 서비스의 환경변수 16개가 들어 있었는데도 그렇다.

## 배운 점

**구현은 교체되고 설계는 남는다.** 977줄의 Edge Function 은 내 Spring 코드를 한 줄도
쓰지 않았는데, **엔드포인트 이름 · 게스트 식별 방식 · 404 인가 · Lambda 호출 순서는 그대로다.**
두 달의 결과물 중 남은 건 파일이 아니라 판단이었다.

**그리고 그 판단이 남았다는 걸 증명해 준 것도 교체였다.** 교체본이 내 구조를 따라간 것은
그 구조가 이 문제에 맞았다는 증거다. 백엔드 담당으로서 가장 뼈아픈 커밋이
동시에 가장 확실한 검증이었다.

**마지막으로, 코드를 잘 쓰는 것으로 방어되지 않는 선택이 있다.**
교체 사유는 비용이었고, 내 Spring 코드가 더 깔끔했어도 결과는 같았을 것이다.
기술 선택은 *"이걸로 만들 수 있나"* 만이 아니라 *"이걸 유지할 수 있나"* 로도 갈린다 —
후자를 팀에서 아무도 계산하지 않았고, 백엔드 담당인 내가 할 자리였다.

**남은 숙제는 태도 쪽이다.** 이유를 안 물었고, 배포 설정을 안 봤고, 스펙을 안 남겼다.
세 개 다 **내가 할 수 있었던 것**이다.
