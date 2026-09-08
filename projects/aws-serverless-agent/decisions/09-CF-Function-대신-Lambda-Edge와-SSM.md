# 09. CloudFront Function 대신 Lambda@Edge + SSM Parameter Store

> 메모: Day 9 는 backend host 를 distribution 에 구웠음 → 백엔드 URL 바뀌면 CloudFront 재배포. SSM 에 두고 엣지가 런타임에 읽으면 디커플링

---

## 상황

Day 9 에서 CloudFront 를 세우면서 `/api/*` 를 Function URL 로 보냈다.
`/api` 접두어를 떼는 일은 **CloudFront Function** (viewer-request) 4줄로 처리했다.

```js
function handler(event) {
  var req = event.request;
  if (req.uri === '/api' || req.uri === '/api/') req.uri = '/';
  else if (req.uri.startsWith('/api/')) req.uri = req.uri.substring(4);
  return req;
}
```

문제는 **backend host 가 distribution 에 구워져 있다**는 것이다.
origin 을 CDK deploy 시점의 Function URL 로 박아 뒀으니,
백엔드를 다시 배포해서 URL 이 바뀌면 **CloudFront 를 다시 배포해야 한다.**
CloudFront 재배포는 전파에 5~15분이 걸린다.

## 결정

`/api` 처리를 **origin-request Lambda@Edge** 로 올리고,
backend URL 은 **SSM Parameter Store** 에 둔다. 엣지가 런타임에 읽는다.

```js
const PROJECT = "serverless-agent";      // 엣지는 env 불가 → 소스 상수
const SSM_REGION = "us-east-1";

let cached = null;                        // 엣지 인스턴스가 사는 동안 60초 재사용
async function getBackendUrl() {
  if (cached && cached.expires > Date.now()) return cached.value;
  const out = await ssm.send(new GetParameterCommand({ Name: `/${PROJECT}/backend/url` }));
  cached = { value: out.Parameter?.Value ?? null, expires: Date.now() + 60_000 };
  return cached.value;
}

export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  if (request.uri === "/api" || request.uri.startsWith("/api/")) {
    const host = new URL(await getBackendUrl()).hostname;
    request.origin = { custom: { domainName: host, port: 443, protocol: "https", ... } };
    request.headers.host = [{ key: "Host", value: host }];
    request.uri = request.uri === "/api" || request.uri === "/api/" ? "/" : request.uri.slice(4);
  } else if (!(request.uri.split("/").pop() || "").includes(".")) {
    request.uri = "/index.html";          // SPA fallback
  }
  return request;
};
```

## 왜

- **CloudFront Function 은 origin 을 못 바꾼다.** 이게 결정적이다.
  CF Function 은 **요청/응답 객체의 문자열만** 만질 수 있다 — URI, 헤더, 쿠키, 쿼리.
  `request.origin` 을 교체하려면 Lambda@Edge 의 origin-request 이벤트가 필요하다.
- **AWS 호출을 할 수 있다.** CF Function 은 순수 JS 실행 환경이라 네트워크가 없다.
  Lambda@Edge 는 Node 런타임이라 SSM·DDB 를 부를 수 있다.
- **백엔드와 CDN 이 분리된다.** 백엔드를 재배포하고 SSM 파라미터만 갱신하면,
  **CloudFront 는 손도 안 대고** 다음 요청부터(최대 60초 뒤) 새 origin 으로 간다.
  Blue-green 이나 백엔드 교체가 CDN 전파 시간에 묶이지 않는다.
- **원본이 이 패턴이다.** 원본은 `packages/edge` 를 **별도 스택**으로 두고,
  SSM 이 backend 와의 **유일한 연결고리**다. 우리는 한 스택이지만 같은 SSM 패턴을 써서
  디커플링을 그대로 보여준다.

## 왜 다른 건 안 썼나

**CloudFront Function 유지 (Day 9 구조)**

µs 단위로 빠르고 월 2M 호출까지 무료다. `/api` strip 만 하면 그만이라면 이게 맞다.
안 쓴 이유는 하나 — **origin 을 못 바꾼다.** 이 day 의 목적이 정확히 그거였다.

**backend URL 을 엣지 함수에 하드코딩**

Lambda@Edge 는 환경변수를 못 쓰니까 어차피 상수를 박아야 한다.
그럼 URL 도 상수로 박으면 되지 않나? — 안 된다.
**백엔드 URL 이 바뀔 때마다 엣지 함수를 재배포해야 하고**, 엣지 함수 재배포는
CloudFront 재배포보다 더 느리다(복제본이 전 리전에 퍼진다). 문제가 더 나빠진다.

**Origin Group / 여러 origin 을 미리 등록해 두고 behavior 로 분기**

origin 이 유한하고 미리 안다면 가능하다. 그런데 CDK 로 매번 새로 배포하면
Function URL 은 **매번 새 도메인**이다. 미리 등록할 수가 없다.

**DDB 에 backend URL 두기**

가능하다. SSM 을 고른 이유는 **원본이 SSM 을 쓰고**, 설정값 저장은 SSM Parameter Store 의
용도 그 자체이며, 무료 티어(Standard 파라미터)라서다. DDB 는 테이블을 하나 더 만들어야 한다.

## 잃은 것

| | CloudFront Function | Lambda@Edge |
|---|---|---|
| 지연 | µs | ms (콜드스타트 있음) |
| 비용 | 2M/월 무료 | 요청 + 실행시간 과금 |
| 리전 | 제약 없음 | **us-east-1 필수** |
| 환경변수 | — | **불가** |
| 삭제 | 즉시 | 복제본 정리까지 지연 |

정적 파일 요청은 여전히 엣지 함수를 탄다(SPA fallback 때문에).
`/api` 만 태우고 싶으면 behavior 를 더 쪼개야 하는데, 학습 범위에서 안 했다.

## 결과

- 브라우저가 `/api/...` 만 치게 되면서 **CORS 가 사라졌다.** same-origin 이라 preflight 도 안 나간다.
- Day 16 이후 Day 17~21 이 전부 이 호스팅 위에 얹혔다. 엣지 함수는 그대로 재사용됐다
  (Day 18 커밋에 `chore(day-18): edge-origin-request.mjs 동봉 (Day 16 그대로, 스택이 참조)` 가 있다).
- **실시간 WSS 는 여전히 CDN 을 우회한다.** presigned URL 에 IoT host 가 박혀 있어서
  브라우저 → IoT Core 직결이다. → [결정 08](08-브라우저-권한을-세션정책으로.md)

## 원본과 다르게 간 것

원본은 `/api` 를 **떼지 않는다.** 백엔드 라우트가 `/api/...` 로 시작하기 때문이다.
우리 Hono 는 `/chat`, `/health` 라서 **엣지에서 strip 한다** — Day 9 의 CF Function 이 하던 일을
그대로 이관했다.

돌아보면 백엔드 라우트를 `/api/chat` 으로 바꾸는 게 더 깔끔했을 수 있다.
그러면 엣지가 origin 만 바꾸고 URI 는 안 건드린다. 안 한 이유는 Day 7 스택을 침범하기 싫어서였는데,
Day 16 시점에는 이미 Day 7 과 코드가 갈라져 있었으니 유효한 이유가 아니었다.

## 관련 함정

→ [트러블 07](../troubleshooting/07-Lambda-Edge는-환경변수를-못-쓴다.md) (#39 · #40 · #41)

그 외:
- **#42** Lambda@Edge 는 us-east-1 필수 — `bin` 에서 리전 고정
- **#43** origin 교체 후 백엔드가 403/SNI 에러 — 엣지가 `request.headers.host` 도 같이 세팅해야 한다
  ([트러블 02](../troubleshooting/02-CloudFront를-끼우자-API가-403.md) 와 같은 원인이 엣지에서 재현된 것)
- **#44** 엣지에서 SSM 403 / 빈 값 — EdgeRole 에 `ssm:GetParameter` + `SSM_REGION` 일치
- **#45** `/api/chat` 이 백엔드에서 404 — strip 누락
- **#46** `destroy` 가 엣지 함수 삭제 실패 — 복제본이 남아서. **정상**이고 시간 두고 재시도
