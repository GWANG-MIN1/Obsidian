# 02. API Gateway 대신 Lambda Function URL

> 메모: 로드맵엔 "API Gateway 연동"이라 적어놨는데, API GW v1/v2 둘 다 응답 스트리밍을 지원 안 해서 노선 변경 (Day 6)

---

## 상황

Day 6 계획은 "API Gateway 로 HTTP 노출"이었다. `aws lambda invoke` 대신 `curl` 로 부를 수 있게 만드는 단계.
HTTP API(v2) 와 REST API(v1) 를 비교하는 것까지 해 뒀다.

그런데 같은 day 에 하려던 일이 하나 더 있었다 — **Bedrock 응답을 토큰 단위로 흘리기.**
챗봇에서 첫 토큰이 도착하는 시점은 UX 의 핵심이다. Day 5 는 Bedrock 이 다 끝난 뒤 한 번에 응답했다.

## 결정

**Lambda Function URL** 을 쓴다. `invokeMode: RESPONSE_STREAM` + `awslambda.streamifyResponse`.

```ts
const url = alias.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.NONE,
  invokeMode: lambda.InvokeMode.RESPONSE_STREAM,   // ← 이것 때문
  cors: { allowedOrigins: ['*'], allowedMethods: [POST, GET], ... },
});
```

## 왜

- **API Gateway 는 v1·v2 모두 응답 스트리밍을 지원하지 않는다.** API GW 를 끼우면 Lambda 가 다 끝날
  때까지 버퍼링된다. 토큰 스트리밍이 목적인데 그 앞에 버퍼를 놓는 셈이다.
- **실측으로 확인했다.** Function URL + `ConverseStreamCommand` 로:

  ```
  --- first-byte: 1.995s | total: 4.443s ---
  ```

  첫 토큰까지 2초, 전체 4.4초. 그 사이 2.4초 동안 청크가 계속 흘렀다.
  Buffered 였다면 두 값이 거의 같았을 것이다.
- **원본도 같은 이유로 Function URL 을 택했다.** 원본 확인이 결정의 계기였지만,
  근거는 "원본이 그래서"가 아니라 **스트리밍 지원 여부**다.
- **비용도 0 추가.** Function URL 은 Lambda 호출비에 포함된다.
  API GW HTTP API 는 $1/M req, REST 는 $3.50/M req.
- **event 포맷이 API GW v2 호환**이라 나중에 갈아끼워도 핸들러가 거의 그대로다.
  `{ version: "2.0", requestContext, body, headers }` 모양이 같다.

## 왜 다른 건 안 썼나

**API Gateway HTTP API (v2)**

$1/M 로 싸고 JWT 인증이 내장이다. 하지만 **스트리밍이 안 된다.** 이 프로젝트에서는 그게 전부다.

**API Gateway REST API (v1)**

API key + usage plan, request validator, WAF 직접 연결, 정교한 throttling — 이 프로젝트에서
포기한 것들이 전부 여기 있다. $3.50/M 이고 역시 **스트리밍이 안 된다.**

## 무엇을 잃었나 (명시적으로)

Function URL 로 가면서 실제로 없어진 것들:

| 잃은 것 | 대체 경로 |
|---|---|
| API key 발급 / usage plan | 없음 — 이 프로젝트엔 인증 자체가 없다 |
| request validator | Hono 핸들러 안에서 직접 검증 |
| WAF 직접 연결 | CloudFront 뒤에 두면 가능 (Day 9/16) |
| 정교한 throttling | Lambda 동시성 제한으로만 |
| 커스텀 도메인 | CloudFront 경유 (Day 16) |

Day 9 에서 CloudFront 를 앞에 세우면서 HTTPS·도메인·캐싱은 되찾았다.
남은 건 **API key 와 usage plan** 이고, 이건 인증을 안 넣기로 한 것과 같은 줄에 있다.

## 결과

- Day 6~7 에서 스트리밍 동작을 실측으로 확인 (first-byte ~2s / total ~4.4s).
- Day 9 에서 CloudFront 를 끼웠는데도 chunked 가 유지됐다 — CloudFront 는 2023년부터
  origin 의 chunked transfer-encoding 을 viewer 까지 passthrough 한다.
- **Day 11 에서 이 결정이 반쯤 뒤집혔다.** API ↔ Worker 를 분리하면서
  `/chat` 이 202 를 즉시 돌려주는 구조가 되어 `invokeMode` 를 `BUFFERED` 로 되돌렸다.
  스트리밍의 자리를 IoT MQTT push 가 대신 가져갔다.
  → [결정 04](04-API와-Worker를-async-invoke로-분리.md) · [결정 07](07-실시간을-IoT-MQTT-push로.md)

**결정이 5일 만에 뒤집혔지만 낭비는 아니었다.** Day 6 에서 스트리밍을 직접 만들어 봤기 때문에,
Day 11 에서 "HTTP 응답 시간과 LLM 실행 시간을 분리한다"는 말이 무슨 뜻인지 알 수 있었다.

## 관련 함정

- **#5** 응답이 토막이 아니라 한 번에 옴 — `invokeMode` 와 curl `-N` 이 **둘 다** 필요하다.
- **#6** `AccessDeniedException: bedrock:InvokeModelWithResponseStream` —
  `InvokeModel` 만 허용하면 스트림 호출은 거부된다. action 이 별개다.
- **#7** `responseStream.end()` 누락 → Lambda 가 timeout 까지 매달린다. try/finally 로 보장.
- **#8** `cors.allowedMethods` 에 `OPTIONS` 를 넣으면 deploy 가 깨진다.
  Function URL 이 preflight 를 자동 처리하므로 명시하면 안 된다.
