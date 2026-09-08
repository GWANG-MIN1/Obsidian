# 04. `toolResult` 를 넣으면 Converse 가 ValidationException

- **발생**: 2026-06-04 (Day 13) · 저장소 함정 **#21 · #22 · #23**
- **증상 한 줄**: 도구 실행 결과를 되먹이는 순간 Bedrock 이 요청을 거부한다

---

## 현상

Day 13 에서 Agent Loop 를 만들었다. 모델이 `executeCode` 를 부르면 샌드박스에서 실행하고
결과를 `toolResult` 로 되먹인 뒤 Converse 를 다시 부른다.

그 "다시 부르는" 지점에서 `ValidationException` 이 난다.

에러 메시지가 **원인을 안 가리킨다.** 요청 형식이 틀렸다는 것만 알려주지,
어느 블록의 무엇이 문제인지는 말하지 않는다.

## 환경

- AWS Bedrock `ConverseCommand`, Claude Haiku 4.5
- `toolConfig` 에 `executeCode` 도구 1개 ([결정 06](../decisions/06-도구를-executeCode-하나로.md))
- Worker Lambda 안 in-memory `messages` 배열 + `MessagesTable` 저장

## 진단

### 1) toolUse 와 toolResult 의 짝을 확인한다 → 여기다

Bedrock Converse 의 규칙은 이렇다.

```
assistant 턴:  [ { toolUse: { toolUseId: "abc123", name: "executeCode", input: {...} } } ]
                                            ↓ 같은 id 여야 한다
다음 user 턴:  [ { toolResult: { toolUseId: "abc123", content: [...] } } ]
```

**assistant 의 `toolUse` 블록과 바로 다음 user 메시지의 `toolResult` 블록은
`toolUseId` 로 짝이 맞아야 한다.**

처음에는 assistant 턴에서 필요한 것만 골라 담았다.
그러다 `toolUseId` 가 어긋나거나 짝이 빠지면 거부된다.

### 2) 다음 턴에서 히스토리를 복원하니 또 깨진다

첫 문제를 고쳤더니 **다음 요청**에서 다른 에러가 났다.

DDB 에서 히스토리를 읽어 Converse 메시지로 바꿀 때,
`tool_call`/`tool_result` 행은 건너뛰고 최종 텍스트만 쓴다.
그런데 그러면 이렇게 된다.

```
user      "이번 달 비용?"
assistant "계산해 볼게"        ← tool_call 직전의 텍스트
assistant "$3.12 입니다"       ← tool_result 다음의 텍스트
```

**assistant 가 연달아 두 번** 나온다. Converse 는 user/assistant 교대를 요구한다.

### 3) 모델이 결과를 못 본다

형식 문제를 다 고쳤는데, 모델이 계산 결과를 언급하지 않고 엉뚱한 답을 한다.

샌드박스는 **`read()` 한 값만** 돌려준다. `console.log` 는 버려진다.
모델이 `console.log(result)` 를 쓰면 결과가 아무 데도 안 간다.

## 원인

세 겹이었다.

**#21 — `toolUseId` 짝이 깨졌다.** assistant 턴을 재구성하면서 id 가 어긋났다.

**#22 — 히스토리 복원이 교대 규칙을 위반했다.** tool 행을 건너뛰니 같은 role 이 연속됐다.

**#23 — 모델이 `read()` 를 안 썼다.** 프롬프트에 그 규칙이 없었다.

앞의 둘은 **형식**, 마지막은 **계약**이다. 셋 다 `ValidationException` 이나
"답이 이상하다"로만 드러나서 구분이 안 됐다.

## 해결

### #21 — assistant 턴을 통째로 push 한다

골라 담지 않는다. Bedrock 이 준 `res.output.message` 를 **그대로** in-memory `messages` 에 넣고,
같은 `toolUseId` 로 `toolResult` 를 만들어 **바로 다음 user 메시지**에 넣는다.

```js
messages.push(res.output.message);                       // assistant 턴 원본 그대로
// ...샌드박스 실행...
messages.push({ role: "user", content: [{ toolResult: { toolUseId, content: [...] } }] });
```

**in-memory 는 짝을 그대로, DDB 는 단계별로 풀어서** — 두 표현을 분리했다.
화면에 보여줄 형태와 모델에게 줄 형태가 다르다.

### #22 — 같은 role 이 연속이면 병합한다

```js
// rowsToConverseMessages: 같은 role 이 연속되면 한 메시지로 합친다 (text 블록 여러 개)
[
  { role: "assistant", content: [{ text: "계산해 볼게" }, { text: "$3.12 입니다" }] }
]
```

### #23 — `read()` 를 프롬프트와 도구 설명 양쪽에 명시한다

```
결과는 반드시 read() 로 표시해라. console.log 는 보이지 않는다.
```

시스템 프롬프트에 한 번, 도구 `description` 에 한 번. **두 군데 다 적었다.**

## 재발 방지

- **모델이 준 것은 그대로 돌려준다.** assistant 턴을 재구성하지 않는다.
  필요한 것만 골라 담으면 id 나 순서가 어긋난다.
- **저장 표현과 전송 표현을 분리한다.** DDB 는 `kind` 로 단계를 풀어 적고,
  Converse 에 보낼 때 다시 접는다. 하나로 겸하려다 둘 다 어긋났다.
- **샌드박스의 출력 규칙은 프롬프트에 계약으로 박는다.** `read()` 만 보인다는 건
  코드를 봐야만 알 수 있는 사실이다. 모델은 코드를 안 본다.
- 교대 규칙(user/assistant 번갈아)은 히스토리를 **가공할 때마다** 깨질 수 있다.
  가공 함수 안에서 병합을 보장한다.

## 같은 day 의 다른 함정

**#24 — DDB Put 이 `undefined` 필드로 실패**

행 종류마다 있는 필드가 다르다. `code` 는 `tool_call` 에만,
`toolUseId` 는 `tool_call`/`tool_result` 에만 있다.
`undefined` 를 그대로 넣으면 DDB 가 거부한다.

`putRow` 가 **저장 직전에 `undefined` 키를 제거**하도록 했다.
한 테이블에 여러 모양의 행을 넣을 때 반복되는 패턴이다.

**#25 · #26 — 루프가 안 끝난다 / timeout 에 걸린다**

모델이 계속 `toolUse` 만 뱉을 수 있다. `MAX_TURN_STEPS`(=5) 로 컷을 걸고,
Worker timeout 을 60초 → 300초로 늘렸다.
→ [결정 06](../decisions/06-도구를-executeCode-하나로.md) 의 "무한루프를 어떻게 막았나"

## 배운 점

**에러 메시지가 원인을 안 가리키는 종류가 있다.**
`ValidationException` 만 보고 `toolUseId` 짝을 의심하기는 어렵다.
이럴 땐 에러를 읽는 대신 **프로토콜 문서의 제약 조건을 하나씩 대조**하는 게 빠르다.

**LLM 과의 인터페이스도 계약이다.** 함수 시그니처처럼, "무엇이 보이고 무엇이 안 보이는지"를
명시해야 한다. `read()` 규칙을 프롬프트에 안 적은 건 문서화 누락이 아니라 **버그**였다.
