# Bedrock Converse 도구 호출 루프 (Agent Loop)

> 메모: LLM이 도구를 쓴다는 게 실제로 무슨 흐름인가. toolUse/toolResult 짝과 교대 규칙이 안 맞으면 ValidationException

---

## 개념

"에이전트"의 최소 형태는 **모델 호출을 루프로 감싼 것**이다.

```
for step in 0..MAX_STEPS:
    res = Converse(messages, toolConfig)
    messages.push(assistant 턴 원본)

    toolUse 가 없나? ──▶ break (최종 텍스트)
        │ 있다
        ▼
    도구 실행 → 결과
    messages.push({ role: "user", content: [{ toolResult }] })
    ↑ 루프 top 으로
```

모델이 `stopReason: "tool_use"`와 함께 `toolUse` 블록을 내면,
우리가 실행하고 `toolResult`를 되먹인 뒤 다시 부른다. **이 왕복이 곧 agent loop다.**

### 규칙 1 — `toolUseId`로 짝이 맞아야 한다

```
assistant 턴:  [ { toolUse:    { toolUseId: "abc123", name: "...", input: {...} } } ]
                                            ↓ 같은 id
다음 user 턴:  [ { toolResult: { toolUseId: "abc123", content: [...] } } ]
```

**assistant 턴을 재구성하지 않는다.** 모델이 준 `res.output.message`를 **통째로** push하고,
같은 id로 `toolResult`를 만들어 **바로 다음 user 메시지**에 넣는다.
필요한 것만 골라 담으면 id나 순서가 어긋난다.

### 규칙 2 — user/assistant가 교대해야 한다

다음 턴에서 저장소의 이력을 복원할 때, 도구 행을 건너뛰고 텍스트만 쓰면 이렇게 된다.

```
user      "이번 달 비용?"
assistant "계산해 볼게"        ← 도구 호출 직전 텍스트
assistant "$3.12 입니다"       ← 도구 결과 이후 텍스트   ← 연속!
```

Converse가 거부한다. **같은 role이 연속이면 한 메시지로 병합**한다(text 블록 여러 개).
이력을 가공하는 함수 안에서 병합을 보장해야 한다 — 가공할 때마다 깨질 수 있다.

### 규칙 3 — 모델이 보는 것을 명시적으로 정한다

도구가 샌드박스 코드 실행이라면, **무엇이 모델에게 돌아가는지**가 계약이다.

```js
function read(value) { reads.push(JSON.parse(JSON.stringify(value))); }
// console.log 는 버려진다. read() 한 것만 모델에게 간다.
```

이 규칙은 **코드를 봐야만 알 수 있는 사실**이고 모델은 코드를 안 본다.
시스템 프롬프트와 도구 `description` **양쪽에** 적는다.

> 프롬프트에 안 적은 건 문서화 누락이 아니라 **버그**다.

### 도구를 몇 개로 둘 것인가

| | 기능별 도구 여러 개 | 단일 `executeCode` |
|---|---|---|
| 입력 스키마 = 계약 | ✅ 안전·예측 가능 | 코드라 자유롭다 |
| **조합** | ❌ 여러 번 왕복 + 중간값을 컨텍스트에 | ✅ 한 번에 끝 |
| 토큰 비용 | 도구 수만큼 매 호출 실림 | 하나 |
| 확장 | 도구 추가 | **샌드박스에 함수 주입** |

"비용을 조회해서 서비스별로 정렬하고 상위 3개"는 도구 3개면 왕복 3번에
중간 결과를 모델이 들고 있어야 한다. 코드 실행 하나면 세 줄이다.

**확장은 도구가 아니라 샌드박스 주입으로 연다.**

```js
const sandbox = { read, awsCost, calendar, JSON, Date, Math, /* 안전 글로벌 */ };
//                      ↑ 이 함수 하나만 노출. fetch·require 는 안 준다
```

네트워크를 "연" 게 아니라 **그 호출 하나를 연 것**이다.
그리고 호출 기록(`skillCalls`)은 저장·UI 전용으로 두고 **모델에는 안 보낸다** — 토큰 낭비와 혼선을 막는다.

### 루프를 어떻게 멈추나

| 컷 | 무엇을 막나 |
|---|---|
| `MAX_TURN_STEPS` | 모델이 도구만 계속 부르는 것 |
| 샌드박스 타임아웃 | **비동기** 코드가 오래 도는 것 |
| 함수 타임아웃 | 그 위의 모든 것 |

> **`node:vm`은 진짜 격리가 아니다.** `while(true){}` 같은 동기 무한루프는
> `Promise.race` 타임아웃으로도 못 막는다 — 이벤트 루프 자체가 멈춘다.
> vm이 막는 건 **글로벌 접근**이지 실행이 아니다. 진짜 격리는 `worker_threads`나 isolate가 필요하다.

## 실행

```js
const TOOL = {
  toolSpec: {
    name: "executeCode",
    description: "Execute JavaScript in a sandbox. Call read(value) to surface a value back.",
    inputSchema: { json: {
      type: "object",
      properties: { description: { type: "string" }, code: { type: "string" } },
      required: ["code"],
    } },
  },
};

let messages = rowsToConverseMessages(await loadHistory(sessionId));   // 같은 role 병합 포함

for (let step = 0; step < MAX_TURN_STEPS; step++) {
  const res = await bedrock.send(new ConverseCommand({
    modelId: MODEL_ID,
    messages,
    toolConfig: { tools: [TOOL] },
    system: [{ text: SYSTEM_PROMPT }],
  }));

  const msg = res.output.message;
  messages.push(msg);                                  // ★ 원본 그대로
  await persistBlocks(msg);                            // kind: text | tool_call

  const toolUse = msg.content.find((b) => b.toolUse)?.toolUse;
  if (!toolUse) break;                                 // 최종 텍스트

  const { reads, error, skillCalls } = await runSandbox(toolUse.input.code);
  await persistToolResult({ toolUseId: toolUse.toolUseId, reads, error, skillCalls });

  messages.push({                                      // ★ 같은 id 로 짝
    role: "user",
    content: [{ toolResult: {
      toolUseId: toolUse.toolUseId,
      content: [{ json: { reads, error } }],           // skillCalls 는 안 보낸다
    } }],
  });
}
```

### 저장 표현과 전송 표현을 분리한다

| | 형태 | 용도 |
|---|---|---|
| in-memory `messages` | toolUse/toolResult 짝 그대로 | Converse에 보낼 것 |
| 저장 행 | `kind`로 단계를 풀어 적음 | 화면·조회·이벤트 |

하나로 겸하려다 둘 다 어긋난다. 저장 행은 실시간 push 페이로드와 같은 모양으로 두면
**조회 렌더와 실시간 렌더가 코드를 공유**한다.

## 배운 점

**에러 메시지가 원인을 안 가리키는 종류가 있다.** `ValidationException`만 보고
`toolUseId` 짝을 의심하기는 어렵다. 이럴 땐 에러를 읽는 대신
**프로토콜 제약 조건을 하나씩 대조**하는 게 빠르다.

**LLM과의 인터페이스도 계약이다.** 함수 시그니처처럼 "무엇이 보이고 무엇이 안 보이는지"를
명시해야 한다. 모델은 구현을 못 본다.

**도구를 늘리는 대신 도구 안을 늘린다.** 단일 도구 + 주입은
도구 스펙(=매 호출 토큰)을 고정한 채 능력만 확장한다.
skill을 하나 더한 날, 도구 정의·루프·스키마·이벤트가 전부 그대로였다.

**막을 수 있는 것과 없는 것을 구분해 적어 둔다.** 스텝 컷과 타임아웃으로 막히는 건 일부다.
`node:vm`의 한계를 결정 안에 남겨 두지 않으면 다음에 그대로 가져다 쓴다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 06](../../projects/aws-serverless-agent/decisions/06-도구를-executeCode-하나로.md) ·
  [결정 10](../../projects/aws-serverless-agent/decisions/10-skill을-샌드박스에-주입.md)
- 형식 오류 진단: [트러블 04](../../projects/aws-serverless-agent/troubleshooting/04-toolResult를-넣으면-ValidationException.md)
- 루프를 오래 돌리려면 실행을 분리해야 한다 → [Lambda async invoke로 시간 분리](Lambda-async-invoke로-시간-분리.md)
