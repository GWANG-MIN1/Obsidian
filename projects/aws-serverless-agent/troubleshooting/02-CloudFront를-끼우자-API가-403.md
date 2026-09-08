# 02. CloudFront를 끼우자 `/api/*` 가 403 — 정적 파일은 잘 나온다

- **발생**: 2026-06-01 (Day 9) · 저장소 함정 **#11 · #12**
- **증상 한 줄**: `The request could not be satisfied` — 정적 파일은 정상인데 API 만 403

---

## 현상

Day 9 에서 CloudFront 를 세워 정적 파일(S3)과 API(Function URL)를 **한 도메인**으로 묶었다.

- `https://dxxx.cloudfront.net/` → index.html 정상
- `https://dxxx.cloudfront.net/api/chat` → **403**

```
The request could not be satisfied.
```

CloudFront 가 내는 403 이라 백엔드까지 가지도 못한 것이다.

## 환경

- CloudFront 멀티 오리진 — default behavior = S3(OAC), `/api/*` behavior = Function URL
- 백엔드는 Day 7 스택 (Hono, `invokeMode: RESPONSE_STREAM`)
- `cachePolicy: CACHING_DISABLED` (채팅 응답을 캐싱하면 안 되므로)

## 진단

### 1) Function URL 을 직접 치면? → 된다

```
curl https://yyyy.lambda-url.us-east-1.on.aws/chat   → 정상
```

백엔드는 멀쩡하다. CloudFront 를 통과하는 경로에서만 깨진다.

### 2) behavior 설정을 본다 → 맞다

`/api/*` 가 Function URL origin 으로 가게 설정돼 있다. 오리진 자체는 맞게 걸렸다.

### 3) 어떤 헤더가 오리진으로 가는가 → 여기다

CloudFront 는 기본적으로 **viewer 의 Host 헤더를 그대로** 오리진에 전달한다.
즉 `Host: dxxx.cloudfront.net` 이 Function URL 로 간다.

그런데 Function URL 의 TLS 인증서와 SNI 는 자기 도메인
(`*.lambda-url.<region>.on.aws`) 용이다. **모르는 Host 를 받으면 즉시 거절한다.**

브라우저가 아니라 **AWS 인프라 두 개가 서로 안 맞는** 상황이다.

## 원인

**CloudFront 가 viewer 의 Host 헤더를 Function URL 에 그대로 보냈다.**

```
브라우저:  Host: dxxx.cloudfront.net
              ↓ CloudFront 가 그대로 전달
Function URL: "이 Host 는 내 것이 아니다"  → 403
```

## 해결

AWS 가 정확히 이 시나리오를 위해 만들어 둔 managed policy 를 붙인다.

```ts
originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER_EXCEPT_HOST_HEADER,
```

viewer 의 **모든 헤더/쿠키/쿼리를 그대로 전달하되 Host 만 오리진의 것으로 재설정**한다.

직접 정의한다면 이렇게 된다.

```ts
headerBehavior: { behavior: 'allExcept', headers: ['Host'] }
```

이걸 붙이자 403 이 사라졌다.

## 이어서 나온 두 번째 문제 — 404 no route (#12)

403 이 풀리니 이번엔 **백엔드가 404** 를 낸다.

```
브라우저: POST /api/chat
백엔드 라우트: POST /chat        ← /api 라는 라우트가 없다
```

CloudFront 의 `originPath` 는 **PREPEND 만 된다.** `originPath="/v1"` 이면 모든 요청에 `/v1` 을 붙인다.
**STRIP 은 못 한다.**

선택지는 셋이었다.

| | 방법 | 판단 |
|---|---|---|
| 1 | **CloudFront Function** (viewer-request) 로 URI rewrite | 채택 — µs, 월 2M 무료 |
| 2 | Lambda@Edge | 더 강력하지만 콜드스타트 + 비용. Day 16 으로 보류 |
| 3 | Day 7 라우트를 `/api/chat` 으로 변경 | Day 7 스택 침범 + Day 8 과 결합도 깨짐. 안 함 |

CloudFront Function 인라인 4줄로 해결했다.

```js
function handler(event) {
  var req = event.request;
  if (req.uri === '/api' || req.uri === '/api/') req.uri = '/';
  else if (req.uri.startsWith('/api/')) req.uri = req.uri.substring(4);
  return req;
}
```

`substring(4)` 가 `'/api'` 길이 4 만 자르고 뒤의 `/chat` 은 그대로 둔다.

## 이 문제는 Day 16 에서 다시 나온다

Day 16 에서 CloudFront Function 을 Lambda@Edge 로 올리면서 origin 을 런타임에 교체하게 됐는데,
**같은 Host 문제가 다시 터졌다** (함정 #43). 이번엔 managed policy 로 안 되고
엣지 함수가 직접 헤더를 세팅해야 했다.

```js
request.origin = { custom: { domainName: host, port: 443, protocol: "https", ... } };
request.headers.host = [{ key: "Host", value: host }];   // ← 이 줄이 빠지면 403
```

**origin 을 바꾸면 Host 도 같이 바꿔야 한다.** 한 번 배운 걸 다른 계층에서 또 배운 셈이다.

## 재발 방지

- **CloudFront 뒤에 Function URL / ALB / API GW 를 둘 땐 Host 헤더부터 확인한다.**
  `ALL_VIEWER_EXCEPT_HOST_HEADER` 는 거의 항상 정답이다.
- **오리진을 바꾸는 코드를 쓸 때마다 Host 를 같이 바꾼다.** 이 둘은 한 쌍이다.
- 403 이 CloudFront 표현(`The request could not be satisfied`)이면 **오리진까지 안 간 것**이다.
  백엔드 로그를 뒤지기 전에 CloudFront 설정을 본다. 로그가 없는 게 단서다.

## 배운 점

**403 의 발신자를 먼저 구분한다.** CloudFront 의 403, S3 의 403, Function URL 의 403,
애플리케이션의 403 은 전부 다른 문제다. 에러 페이지의 문구가 그걸 알려준다.
`The request could not be satisfied` 는 CloudFront 다.

**같은 도메인으로 합치는 것에는 대가가 있다.** same-origin 을 얻으면 CORS 가 사라지지만
([결정 09](../decisions/09-CF-Function-대신-Lambda-Edge와-SSM.md)),
그 대신 Host 헤더와 URI 재작성이라는 새 층이 생긴다. 공짜로 얻는 건 없다.
