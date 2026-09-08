# 04. API와 Worker를 SQS 없이 `InvocationType: Event` 로 분리

> 메모: Phase 2 회고가 남긴 첫 질문 "Lambda 가 60초 넘게 일해야 하면?" 에 대한 답. 원본이 SQS 없이 Lambda async invoke 만 쓰길래 확인해 보니 Lambda 자체가 큐를 제공

---

## 상황

Day 10 Phase 2 회고에서 답 못 한 질문 여섯 개를 적어 뒀다. 첫 번째가 이거였다.

> **"Lambda 가 60초 넘게 일해야 하면?"**

Day 7 까지 구조는 한 Lambda 가 전부 한다 — HTTP 받고, DDB 읽고, Bedrock 부르고, 응답 흘리고, DDB 쓴다.
Bedrock 이 한 번 응답하는 데 1~5초면 괜찮다. 그런데 Day 13 에 넣을 Agent Loop 는
도구를 여러 번 왕복하므로 **분 단위**로 갈 수 있다. HTTP 요청 하나가 그걸 다 붙들고 있을 수는 없다.

## 결정

Lambda 를 **API 와 Worker 둘로 쪼개고**, 사이를 Lambda 자체의 async invoke 로 잇는다.

```js
await lambda.send(new InvokeCommand({
  FunctionName: process.env.AGENT_WORKER_FUNCTION_NAME,
  InvocationType: InvocationType.Event,        // ← fire-and-forget
  Payload: JSON.stringify({ type: "run_chat", userId, sessionId, message, userSk }),
}));
return c.json({ sessionId, status: "queued", userSk }, 202);
```

```
POST /chat ─▶ API Lambda: 검증 → user 메시지 Put → Worker invoke(Event) → 202 즉시 반환
                                                        │ async
                                                        ▼
                                   Worker Lambda: 히스토리 Query → Bedrock → assistant Put
```

**큐 서비스를 따로 두지 않는다.**

## 왜

- **Lambda 의 async invoke 가 이미 큐다.** `InvocationType: Event` 로 부르면 Lambda 서비스가
  내부 큐에 넣고 즉시 반환한다. 재시도(기본 2회)와 DLQ 연결도 Lambda 쪽 설정으로 된다.
  SQS 를 붙이면 **큐를 하나 더 얹는 것**이지 없던 큐가 생기는 게 아니다.
- **원본이 정확히 이 모양이다.** `packages/backend/scripts/lib/backend-stack.ts` 에
  `Handler` 람다와 `Worker` 람다, 그리고 `workerAlias.grantInvoke(fn)` 한 줄.
  Phase 3 플랜을 짤 때 SQS·EventBridge 를 넣어뒀다가, 원본 실구성을 확인하고 **뺐다.**
- **권한 분리가 구조로 강제된다.** API 는 Bedrock 을 모르고, Worker 는 Lambda invoke 를 못 한다.
  Day 11 에서 API 스택의 `bedrock:*` 를 **명시적으로 제거**했다 (함정 #16 — 이전 day 의 IAM 을
  복붙하면 그대로 남는다).
- **HTTP 시간과 LLM 시간이 분리된다.** 캡스톤 회고에서 이 day 를 "가장 큰 설계 전환점" 중 하나로 꼽았다.
  여기서 "채팅 API" 가 "에이전트 작업 큐" 가 됐다.

## 왜 다른 건 안 썼나

**SQS 를 사이에 두기**

가시성 타임아웃, 배치 처리, FIFO, 정교한 DLQ 정책이 필요하면 SQS 가 맞다.
그런데 이 프로젝트에는 **그중 필요한 게 하나도 없다.** 세션당 요청 하나, 순서 보장은 세션 안에서만
필요하고 그건 DDB SK 가 이미 한다. 리소스와 IAM 을 하나 더 늘리는 비용만 남는다.

원본이 안 썼다는 것도 근거지만, 더 중요한 건 **왜 안 썼는지 설명이 되더라는 것**이다.

**EventBridge**

여러 소비자에게 팬아웃하거나 규칙 기반 라우팅이 필요할 때 쓴다. 소비자가 Worker 하나뿐이다.

**Step Functions**

Agent Loop 를 상태 기계로 표현하는 건 매력적이다. 각 스텝이 상태로 보이고 재시도도 선언적이다.
안 쓴 이유: **원본이 안 쓰고**, 루프 자체가 Worker 안 `for` 문 열 몇 줄이라
Step Functions 의 상태 정의가 코드보다 길어진다. 그리고 Bedrock 왕복마다
상태 전이 비용이 붙는다.

**응답 스트리밍을 유지하고 HTTP 를 길게 잡기**

Day 6~7 이 그 구조였다. Lambda timeout 을 15분까지 늘릴 수는 있다.
하지만 **클라이언트 연결이 끊기면 진행 중인 작업이 통째로 날아간다.** Phase 2 회고의 두 번째 질문이
정확히 이거였다 — "응답이 stream 인데 클라이언트 연결이 끊기면?"
비동기로 돌리면 브라우저를 닫아도 Worker 는 끝까지 돌고 결과는 DDB 에 남는다.

## 결과

- `/chat` 응답이 스트림에서 **202 Accepted** 로 바뀌었다. `invokeMode` 도 `RESPONSE_STREAM` →
  `BUFFERED` 로 되돌렸다. → [결정 02](02-API-Gateway-대신-Function-URL.md) 가 여기서 반쯤 뒤집힌다.
- **결과를 볼 방법이 사라졌다.** Day 11 에서는 `GET /sessions/:id/messages` 를 폴링해서 확인했다.
  이 빈칸이 Day 14~15 의 IoT MQTT 로 이어진다. → [결정 07](07-실시간을-IoT-MQTT-push로.md)
- Day 13 에서 Worker timeout 을 60초 → 300초로 늘렸다. 분리해 놨기 때문에
  **API 는 30초 그대로** 두고 Worker 만 늘릴 수 있었다.

## 남긴 자산

- **`workerAlias.grantInvoke(apiFn)` + env(`AGENT_WORKER_FUNCTION_NAME`) 한 쌍** — 두 Lambda 와이어링 표준
- **discriminated union payload (`{type: "run_chat", ...}`)** — Day 18 에서 `channel: "discord"` 를
  같은 payload 에 얹는 데 그대로 재사용했다
- **"user 먼저 Put → invoke" 순서** — Worker 가 히스토리를 읽을 때 마지막 user 가 이미 있다고 가정해도 안전

## 관련 함정

→ [트러블 03](../troubleshooting/03-분리했는데-API가-Worker를-기다린다.md) (#13 · #15)
