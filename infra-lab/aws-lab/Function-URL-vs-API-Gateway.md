# Lambda Function URL vs API Gateway

> 메모: Lambda를 HTTP로 노출하는 세 방법. 응답 스트리밍이 필요하면 선택지가 하나로 줄어든다

---

## 개념

Lambda를 HTTP로 부르는 방법은 셋이다. 대부분의 비교표는 비용과 인증 기능을 나열하는데,
**실제로 선택을 강제하는 칸은 응답 스트리밍 한 줄**인 경우가 많다.

|  | Function URL | API GW v2 (HTTP) | API GW v1 (REST) |
|---|---|---|---|
| 비용 | Lambda 호출비만 | $1.00 / 1M req | $3.50 / 1M req |
| **응답 스트리밍** | **✅** | **❌** | **❌** |
| CORS | 옵션 한 덩어리 | 내장 | 직접 |
| 인증 | NONE / AWS_IAM | + JWT | + Cognito / Lambda Authorizer |
| API key · usage plan | ❌ | ❌ | ✅ |
| request validator | ❌ | ❌ | ✅ |
| WAF | CloudFront 경유 | 도메인 직접 | 직접 |

LLM 응답처럼 **첫 바이트 도착 시각이 중요한 워크로드**는 API Gateway를 끼우는 순간
Lambda가 끝날 때까지 버퍼링된다. 스트리밍이 목적이면 그 앞에 버퍼를 놓는 셈이다.

### 스트리밍은 두 군데가 맞아야 동작한다

인프라와 핸들러 코드가 **둘 다** 스트리밍 모드여야 한다. 하나만 맞으면 조용히 버퍼링된다.

```ts
invokeMode: lambda.InvokeMode.RESPONSE_STREAM   // ① 인프라
```

```js
export const handler = awslambda.streamifyResponse(   // ② 핸들러
  async (event, responseStream, context) => { ... }
);
```

`awslambda`는 import가 아니라 **Node.js Lambda 런타임이 전역으로 주입하는 객체**다.
로컬에는 없으므로 ESLint가 `no-undef`로 화내고, 로컬 단위 테스트도 그대로는 안 된다.

`RESPONSE_STREAM` 모드인데 핸들러가 `responseStream`을 안 쓰고 `return`하면,
그 값이 마지막에 한 번에 흐른다. **에러 없이 스트리밍만 사라진다.**

### IAM action이 갈린다

| | 커맨드 | IAM action |
|---|---|---|
| 버퍼드 | `ConverseCommand` | `bedrock:InvokeModel` |
| 스트리밍 | `ConverseStreamCommand` | `bedrock:InvokeModelWithResponseStream` |

앞의 것만 허용하면 스트림 호출이 `AccessDeniedException`으로 거부된다. 별개 action이다.

### CORS — `OPTIONS`를 넣으면 배포가 깨진다

```ts
cors: {
  allowedOrigins: ['*'],
  allowedMethods: [lambda.HttpMethod.POST, lambda.HttpMethod.GET],
  allowedHeaders: ['content-type'],
  maxAge: cdk.Duration.hours(1),
}
```

`allowedMethods` enum에 **`OPTIONS`가 없다.** Function URL이 preflight를 자동 처리하기 때문이다.
명시하면 `OPTIONS is not a valid enum value`로 deploy가 실패한다.
(지원값: GET/PUT/HEAD/POST/PATCH/DELETE/`*`)

### event 포맷은 API GW v2와 호환된다

```js
{ version: "2.0", requestContext: { http: { method, path } }, body, headers, isBase64Encoded }
```

`body`는 JSON 문자열이라 `JSON.parse(event.body)`가 필요하다.
이 호환성 덕에 나중에 API Gateway로 갈아끼워도 핸들러는 거의 그대로다.

> `isBase64Encoded`를 흘려보내면 안 되는 경우가 있다. 웹훅 서명 검증처럼 **원본 바이트**가
> 필요한 곳에서는 디코딩한 raw 문자열로 검증해야 한다.

## 실행

```ts
const alias = new lambda.Alias(this, 'Live', { aliasName: 'live', version: fn.currentVersion });

const url = alias.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.NONE,
  invokeMode: lambda.InvokeMode.RESPONSE_STREAM,
  cors: { allowedOrigins: ['*'], allowedMethods: [lambda.HttpMethod.POST] },
});
```

**Alias 위에 붙이는 이유** — 함수 자체에 붙이면 URL이 항상 `$LATEST`를 가리킨다.
alias에 붙이면 배포 중에도 URL이 안정적이고, canary(`additionalVersions`)를 나중에 끼울 수 있다.
안 끼우다가 나중에 끼우면 **URL이 바뀐다.** 처음부터 두는 쪽이 안전하다.

스트리밍 검증 — 버퍼링이면 두 값이 거의 같아진다.

```bash
curl --no-buffer -N -X POST "$URL" \
  -H "content-type: application/json" \
  -d '{"message":"긴 문장으로 자기소개 해줘"}' \
  -w "\n--- first-byte: %{time_starttransfer}s | total: %{time_total}s ---\n"
```

```
--- first-byte: 1.995s | total: 4.443s ---
```

첫 토큰 2.0초, 전체 4.4초 — 그 사이 2.4초 동안 청크가 흘렀다는 뜻이다.

> `-N` / `--no-buffer`를 빠뜨리면 서버가 청크로 보내도 curl이 모아서 한 번에 출력한다.
> **클라이언트 버퍼링과 서버 버퍼링을 구분해야 한다.**

CloudFront를 앞에 세워도 스트리밍은 유지된다 — 2023년부터 origin의 chunked transfer-encoding을
viewer까지 passthrough한다. 단 `CACHING_DISABLED`와 함께 써야 한다.
브라우저 DevTools의 Response Headers에 `transfer-encoding: chunked`가 보이면 통과다.

## 배운 점

**비교표에서 결정을 내리는 칸은 대개 하나다.** 비용·인증·WAF를 다 늘어놓아도
"스트리밍이 되는가"가 워크로드 요구사항이면 나머지는 읽을 필요가 없다.
**먼저 탈락 조건을 찾고 그다음에 비교한다.**

**Function URL을 고르면 API key·usage plan·request validator를 잃는다.**
CloudFront를 앞에 세우면 HTTPS·도메인·캐싱·WAF는 되찾을 수 있지만 나머지는 안 돌아온다.
없는 것은 없다고 적어 두는 편이 낫다.

**모드와 코드가 둘 다 맞아야 하는 설정은 한쪽만 틀리면 조용히 실패한다.**
에러가 아니라 "그냥 안 되는" 상태가 되므로, 검증은 동작 여부가 아니라 **시간으로** 재야 한다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 02](../../projects/aws-serverless-agent/decisions/02-API-Gateway-대신-Function-URL.md)
- 이 결정은 나중에 반쯤 뒤집혔다 (async 분리로 202 반환) →
  [결정 04](../../projects/aws-serverless-agent/decisions/04-API와-Worker를-async-invoke로-분리.md)
