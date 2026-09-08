# Lambda async invoke로 시간 분리하기

> 메모: HTTP 응답 시간과 실제 작업 시간을 떼어 놓는 패턴. Lambda 자체가 큐라 SQS를 안 끼워도 된다

---

## 개념

HTTP 요청 하나가 오래 걸리는 작업을 끝까지 붙들고 있으면 두 가지가 깨진다.

- **클라이언트가 끊기면 작업이 통째로 날아간다** — 브라우저를 닫으면 진행 중이던 게 사라진다
- **외부 플랫폼의 응답 제한을 넘는다** — Discord·Slack Interactions는 3초, API Gateway는 29초

그래서 **받는 쪽과 일하는 쪽을 나눈다.**

```
POST /work ──▶ API Lambda
                1) 검증
                2) 입력을 먼저 저장
                3) Worker 를 async 로 호출
                4) 202 즉시 반환
                      │ (fire-and-forget)
                      ▼
                 Worker Lambda ── 오래 걸리는 일 ──▶ 결과 저장
```

### Lambda 자체가 큐다

`InvocationType: Event`로 부르면 Lambda 서비스가 **내부 큐에 넣고 즉시 반환**한다.
재시도(기본 2회)와 DLQ 연결도 Lambda 쪽 설정으로 된다.

**SQS를 붙이는 건 큐를 하나 더 얹는 것**이지 없던 큐가 생기는 게 아니다.

| SQS가 필요한 경우 | 이 프로젝트 |
|---|---|
| 가시성 타임아웃 조절 | 필요 없음 |
| 배치 처리 | 요청당 하나 |
| FIFO 순서 보장 | 세션 안 순서는 정렬키가 이미 보장 |
| 정교한 DLQ·재처리 정책 | 학습 범위 밖 |

소비자가 하나고 위 항목이 다 아니라면 **리소스와 IAM만 하나 늘어난다.**

### 기본값이 동기다

```js
await lambda.send(new InvokeCommand({
  FunctionName: WORKER,
  Payload: JSON.stringify(payload),
  // InvocationType 생략 → 기본값 RequestResponse (동기!)
}));
```

**에러가 안 난다.** 정상 동작이고 의도만 다르다. 응답이 느린 것 외에 증상이 없다.
쪼갰는데 동기로 붙어 있는 상태가 된다.

```js
InvocationType: InvocationType.Event,     // 항상 명시한다
```

### 권한은 qualifier 단위다

alias로 호출할 거면 **alias에 grant해야 한다.**

```ts
workerFn.grantInvoke(apiFn);      // ✗ $LATEST 만 호출 가능
workerAlias.grantInvoke(apiFn);   // ✅ alias ARN 으로 호출할 때
```

env로 넘기는 값도 같은 qualifier여야 한다.

```ts
environment: { WORKER_ARN: workerAlias.functionArn }
```

grant 대상과 호출 대상이 어긋나면 `AccessDenied`다.

### 순서가 계약이 된다

Worker가 입력을 읽는다면, **API가 먼저 저장하고 그다음 invoke**해야 한다.

```
① 입력 저장  →  ② Worker invoke
```

이 순서를 뒤집으면 Worker가 아직 없는 데이터를 읽는다.
"항상 저장 → invoke"를 코드 규칙으로 고정하고, Worker에도 방어 가드를 둔다.

## 실행

```ts
// CDK
const workerAlias = new lambda.Alias(this, 'WorkerLive', {
  aliasName: 'live', version: workerFn.currentVersion,
});
workerAlias.grantInvoke(apiFn);
apiFn.addEnvironment('WORKER_ARN', workerAlias.functionArn);

// 책임 분리 — API 는 모델/외부 서비스 권한을 갖지 않는다
workerFn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['bedrock:InvokeModel'], resources: ['*'],
}));
```

```js
// API
import { LambdaClient, InvokeCommand, InvocationType } from "@aws-sdk/client-lambda";

await putItem(userInput);                        // ① 먼저 저장
await lambda.send(new InvokeCommand({
  FunctionName: process.env.WORKER_ARN,
  InvocationType: InvocationType.Event,          // ② 비동기 호출
  Payload: JSON.stringify({ type: "run_job", id, ...}),
}));
return json({ id, status: "queued" }, 202);      // ③ 즉시 반환
```

**payload는 discriminated union으로** 둔다. 나중에 `type`을 추가하기 쉽고,
채널·모드 같은 칸을 한 개씩 늘려도 기존 경로가 안 깨진다.

```js
{ type: "run_job", ... }
{ type: "run_job", channel: "discord", interactionToken, ... }   // 나중에 한 칸 추가
```

### 검증은 시간으로 한다

```bash
curl -i -X POST "$URL/work" -d '{...}' -w "\n총 %{time_total}s\n"
```

**"202가 왔다"만 보면 통과한다.** 0.2초인지 30초인지를 봐야
동기로 붙어 있는 상태가 잡힌다.

### 결과를 어떻게 돌려주나

즉시 반환하면 결과를 볼 경로가 사라진다. 셋 중 하나를 고른다.

| 방법 | 특징 |
|---|---|
| 폴링 (`GET /jobs/:id`) | 가장 단순. 진행 과정은 안 보인다 |
| **push** (MQTT·WebSocket) | 진행 단계가 실시간으로 흐른다 → [SigV4 presigned WSS](SigV4-presigned-WebSocket-URL.md) |
| 콜백 (webhook PATCH) | 외부 플랫폼이 그 방식을 정해 준 경우 (Discord followup 등) |

### 추적이 끊긴다

`InvocationType: Event`는 컨텍스트를 전파하지 않는다.
X-Ray를 켜도 **API와 Worker가 다른 trace로 따로 기록된다.**
호출하는 쪽 Lambda 클라이언트를 래핑해야 이어진다.
→ [X-Ray로 Lambda 계측하기](X-Ray-Lambda-계측.md)

## 배운 점

**"분리했다"는 구조 얘기지 동작 얘기가 아니다.** Lambda를 둘로 나눈 것과
둘이 비동기로 도는 것은 별개다. 구조를 바꿨으면 **동작이 바뀌었는지 따로 재야 한다.**

**기본값이 조용히 틀린 값인 경우가 가장 찾기 어렵다.** 에러도 로그도 없이 의도만 다르다.
비동기 호출은 `InvocationType`을 항상 명시해서 **코드에서 읽히게** 둔다.

**같은 패턴이 계층을 바꿔 다시 나온다.** Discord의 3초 제한과 `type 5` deferred 응답은
이 패턴을 프로토콜로 강제한 것이다. 플랫폼이 짧은 응답 제한을 두면
대개 "즉시 응답 + 나중에 채우기" 경로도 같이 정의해 뒀다.

**큐를 넣기 전에 큐가 이미 있는지 본다.** Lambda async invoke, SNS, EventBridge는
각자 큐 성격을 갖는다. 관리형 큐를 하나 더 얹기 전에 **무엇이 부족한지**를 먼저 적어 본다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 04](../../projects/aws-serverless-agent/decisions/04-API와-Worker를-async-invoke로-분리.md)
- 동기로 붙어 있던 진단: [트러블 03](../../projects/aws-serverless-agent/troubleshooting/03-분리했는데-API가-Worker를-기다린다.md)
- 3초 제한이 같은 패턴을 강제한 사례: [트러블 10](../../projects/aws-serverless-agent/troubleshooting/10-봇이-응답-실패만-띄운다.md)
