# 06. 도구를 여러 개 만드는 대신 `executeCode` 하나로

> 메모: 원본의 single-tool 설계. 도구를 N개 늘리는 대신 "코드 실행" 하나로 일반화. 단 `node:vm` 은 진짜 격리가 아니라는 걸 결정 안에 박아둠

---

## 상황

Day 13 에서 Worker 를 "Bedrock 한 번 호출"에서 **루프**로 키웠다.
모델이 도구를 쓰겠다고 하면 실행하고, 결과를 다시 먹이고, 도구를 더 안 쓸 때까지 반복한다.

그러면 **도구를 무엇으로 줄 것인가**를 정해야 한다.
보통은 `getWeather`, `searchWeb`, `calculate` 처럼 기능별로 도구를 나열한다.

## 결정

**도구는 `executeCode` 하나.** 샌드박스에서 JavaScript 를 실행하고,
코드가 `read(value)` 로 표시한 값만 모델에게 돌려준다.

```js
const EXECUTE_CODE_TOOL = {
  toolSpec: {
    name: "executeCode",
    description: "Execute JavaScript in a sandbox. Call read(value) to surface a value back…",
    inputSchema: { json: { type: "object", properties: {
      description: { type: "string" },   // 유저 언어로 된 한 줄 라벨
      code:        { type: "string" },   // 실행할 JS
    }, required: ["code"] } },
  },
};
```

```js
function read(value) { reads.push(JSON.parse(JSON.stringify(value))); }
const sandbox = { read, console, JSON, Date, Math, Array, Object, String,
                  Number, Boolean, RegExp, Map, Set, Promise, parseInt, ... };
// 네트워크/파일시스템 글로벌은 안 준다 — 순수 계산용.
```

## 왜

- **LLM 이 머리로 계산하면 틀린다.** 산술, 정렬, 날짜 계산, 데이터 가공을 모델이 직접 하면
  그럴듯하지만 틀린 값이 나온다. 코드로 실행해서 **진짜 값**을 되먹이면 그 부류가 통째로 사라진다.
- **도구를 안 늘려도 조합이 무한하다.** "이번 달 비용을 서비스별로 정렬해서 상위 3개"는
  `sortBy` 도구와 `top-n` 도구를 만들 일이 아니라 코드 세 줄이다.
- **도구가 늘면 프롬프트도 늘고 모델도 헷갈린다.** 도구 스펙은 매 호출 input 토큰에 들어간다.
  10개면 10개어치가 매번 실린다.
- **원본이 택한 설계다.** `agent-runtime/tools.ts` 의 `executeCodeTool` 하나.
  그리고 **확장 지점을 도구가 아니라 skill 로 열어 뒀다** —
  샌드박스 안에 함수를 주입하는 방식. → [결정 10](10-skill을-샌드박스에-주입.md)
- **`read()` 강제가 출력 경계를 만든다.** `console.log` 는 버려지고 `read()` 한 값만 모델에게 간다.
  샌드박스가 무엇을 하든 **모델이 보는 것은 명시적으로 표시한 값뿐**이다.

## 왜 다른 건 안 썼나

**기능별 도구 여러 개 (`getCost`, `getCalendar`, `calculate`, …)**

각 도구의 입력 스키마가 곧 계약이라 안전하고 예측 가능하다. 실무에서는 대체로 이쪽이 맞다.

안 쓴 이유: **조합을 못 한다.** "비용을 조회해서 서비스별로 정렬하고 상위 3개만"을 하려면
모델이 도구를 세 번 부르고 중간 결과를 자기 컨텍스트에 들고 있어야 한다.
토큰도 많이 쓰고, 중간에 값을 옮겨 적다가 틀린다.
`executeCode` 면 한 번에 끝난다.

**LLM 에게 계산도 시키기 (도구 없이)**

가장 싸다. 그리고 틀린다. 이건 선택지가 아니었다.

**Python 샌드박스 / 컨테이너 실행 (Lambda 안에서 별도 프로세스)**

진짜 격리가 된다. 그런데 Lambda 콜드스타트가 커지고, 런타임을 하나 더 들여야 한다.
학습 범위에서 과했다.

## ⚠️ 한계 — `node:vm` 은 진짜 격리가 아니다

이 결정에 딸린 위험을 결정 안에 같이 적어 둔다.

```js
// 이건 못 막는다
while (true) {}
```

동기 무한루프는 **이벤트 루프 자체를 멈춘다.** `Promise.race` 로 걸어 둔
`SANDBOX_TIMEOUT_MS` 는 타이머가 돌아야 발동하는데, 타이머가 안 돈다.
Lambda timeout(5분)까지 매달렸다가 죽는다.

`node:vm` 이 막는 것은 **글로벌 접근**이지 실행 자체가 아니다.
`require`, `process`, `fetch` 를 안 주면 그건 못 쓰지만, CPU 를 태우는 건 못 막는다.

**원본도 같은 한계를 갖는다.** 진짜 격리는 `worker_threads` 나 isolate(V8 isolate) 가 필요하다.
이 프로젝트에서는 **학습용 경계로만 쓴다**고 못박아 두었다.

여기에 더해 원본은 sandbox 코드를 **TypeScript 컴파일러로 타입 체크한 뒤** 실행한다.
우리는 그 TypeChecker 를 일부러 들어냈다 — 루프 흐름 학습이 목적이라 타입 게이트가 과했다.

## 무한루프를 어떻게 막았나 (막을 수 있는 것만)

| 컷 | 값 | 무엇을 막나 |
|---|---|---|
| `MAX_TURN_STEPS` | 5 | 모델이 도구만 계속 부르는 것 |
| `SANDBOX_TIMEOUT_MS` | 5s → 10s(Day 17) → 15s(Day 19) | **비동기** 코드가 오래 도는 것 |
| Worker timeout | 300s | 그 위의 모든 것 |

`SANDBOX_TIMEOUT_MS` 가 day 마다 늘어난 건 skill 이 네트워크를 타기 시작해서다.
Cost Explorer 호출에 ~900ms, 캘린더 ICS 다운로드에 최대 8초가 든다.

## 결과

- 여기서 이 프로젝트가 "텍스트 생성기"에서 "도구를 실행하는 에이전트"가 됐다.
  캡스톤에서 가장 큰 설계 전환점 중 하나로 꼽았다.
- 도구를 하나로 둔 덕에 **Day 17·19 의 skill 추가가 도구 스펙을 안 건드렸다.**
  `worker.mjs` 의 샌드박스 글로벌에 함수 하나를 더 넣는 게 전부였다.
- 루프의 각 단계를 `MessagesTable` 에 `kind`(text / tool_call / tool_result)로 풀어 저장해서,
  `GET /sessions/:id/messages` 로 **전 과정을 들여다볼 수 있게** 했다.
  이 저장 형태가 Day 14 의 MQTT 이벤트 페이로드와 같은 모양이라 렌더 코드를 공유한다.

## 관련 함정

→ [트러블 04](../troubleshooting/04-toolResult를-넣으면-ValidationException.md) (#21 · #22 · #23)

그 외:
- **#24** DDB Put 이 `undefined` 필드로 실패 — 행 종류마다 `code`/`toolUseId` 유무가 갈린다.
  `putRow` 가 저장 전 `undefined` 키를 제거하도록 했다.
- **#25 · #26** 루프 컷과 Worker timeout — 위 표.
