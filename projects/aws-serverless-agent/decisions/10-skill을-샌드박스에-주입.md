# 10. skill 은 도구가 아니라 샌드박스에 주입하고, 호출기록은 LLM 에 숨긴다

> 메모: Day 13 에서 원본의 skill 주입을 일부러 들어냈다가 Day 17 에 되살림. `awsCost` 를 샌드박스 글로벌로 넣음. skillCalls 는 저장/MQTT 에만, LLM 엔 안 보냄

---

## 상황

[결정 06](06-도구를-executeCode-하나로.md) 으로 도구는 `executeCode` 하나다.
샌드박스 글로벌은 안전 내장(JSON/Date/Math/…)뿐이고 **네트워크가 없다** — 순수 계산기다.

Day 17 에서 "에이전트가 진짜 AWS 비용을 조회한다"를 하려면 외부 호출이 필요하다.
그러면 두 가지를 정해야 한다.

1. 외부 호출을 **어디에** 붙일 것인가 — 도구를 하나 더 만드나, 샌드박스 안에 넣나
2. 호출 기록(`skillCalls`)을 **누가** 볼 것인가

## 결정

**(1) skill 은 샌드박스 글로벌로 주입한다.** 도구는 여전히 `executeCode` 하나다.

```js
// 샌드박스 밖(모듈 스코프)에 실제 구현
const ce = new CostExplorerClient({ region: "us-east-1" });
async function getAwsCost({ start, end, granularity = "MONTHLY", ... } = {}) { ... }

// runSandbox 안: 호출 흔적을 남기는 래퍼로 감싸 글로벌에 주입
async function awsCost(args = {}) {
  const t0 = Date.now();
  try   { const r = await getAwsCost(args);
          skillCalls.push({ name: "awsCost", input: args, ok: true, ms: Date.now() - t0 });
          return r; }
  catch (e) { skillCalls.push({ name: "awsCost", input: args, ok: false, error: e?.message }); throw e; }
}
const sandbox = { read, awsCost, JSON, Date, Math, /* ... 안전 글로벌 ... */ };
```

**(2) `skillCalls` 는 LLM 에게 보내지 않는다.** 저장(DDB `tool_result` 행)과 MQTT 이벤트에만 싣는다.
모델이 받는 `toolResult` 에는 `reads` 와 `error` 만 들어간다.

## 왜

### skill 을 샌드박스에 넣은 이유

- **도구 1개로 임의의 행동을 조합할 수 있다.** 모델은 코드를 쓰고, 코드가 `awsCost()` 를 부른다.
  "이번 달 비용을 서비스별로 뽑아서 상위 3개만" 이 한 번의 `executeCode` 로 끝난다.
  도구로 분리했으면 `getCost` 호출 → 결과를 컨텍스트에 들고 → 정렬을 또 시키는 왕복이 된다.
- **네트워크를 **딱 그 함수 하나로만** 연다.** 샌드박스에 `fetch` 나 `require` 를 준 게 아니다.
  클로저로 감싼 `awsCost` 함수 하나가 노출될 뿐이라, 모델은 **Cost Explorer 말고는 아무것도 못 부른다.**
  "네트워크를 열었다"가 아니라 "이 호출 하나를 열었다"이다.
- **도구 스펙이 안 늘어난다.** 도구 정의는 매 Converse 호출의 input 토큰에 실린다.
  skill 을 아무리 추가해도 그 비용이 0이다. (대신 시스템 프롬프트에 한 줄이 는다.)
- **원본이 이 구조다.** `agent-runtime/code-executor.ts` 가 `buildSkills()` 결과를
  `CodeExecutor` 바인딩으로 넣는다. **서버가 허용한 skill 만 샌드박스에 노출**된다는 원칙이 같다.

### `skillCalls` 를 LLM 에 숨긴 이유

- **토큰 낭비.** `{name, input, ok, ms}` 가 호출마다 쌓여서 매 턴 되먹여지면 컨텍스트가 부푼다.
- **모델이 헷갈린다.** 모델에게 필요한 건 **결과값**이지 "내가 뭘 몇 ms 만에 불렀는지"가 아니다.
  메타데이터가 섞이면 그걸 답변에 옮겨 적으려는 경향이 생긴다.
- **관측용과 추론용은 다른 데이터다.** 호출 내역은 **운영자와 UI** 가 볼 것이다.
  DDB 행과 MQTT 이벤트에 실어서, 브라우저 실시간 화면에 "awsCost 호출 ✓ 900ms" 로 뜬다.
- 원본도 `skillCalls` 는 UI 전용(`realtime-events.ts` 의 `AssistantMessageContent.skillCalls`)이다.

## 왜 다른 건 안 썼나

**skill 마다 Converse 도구를 하나씩**

`getAwsCost`, `getCalendar` 를 각각 도구로 정의하는 방식. 입력 스키마가 계약이라 안전하다.
안 쓴 이유는 [결정 06](06-도구를-executeCode-하나로.md) 과 같다 — **조합이 안 된다.**
그리고 도구가 늘 때마다 매 호출 토큰이 는다.

**샌드박스에 `fetch` 를 통째로 주기**

가장 유연하다. 그리고 **모델이 임의의 URL 을 부를 수 있게 된다.**
프롬프트 인젝션이 들어오면 데이터를 외부로 실어 보낼 수 있고,
내부 메타데이터 엔드포인트(`169.254.169.254`)를 칠 수도 있다. 선택지가 아니었다.

**`skillCalls` 도 LLM 에 주기**

모델이 "비용 조회를 두 번 했네, 한 번만 해도 되겠다" 같은 자기 교정을 할 수 있다는 논리는 있다.
실제로는 토큰만 늘고 답변에 잡음이 섞인다. 그리고 **호출 횟수 제어는 모델이 아니라
`MAX_TURN_STEPS` 가 할 일**이다.

## 이 결정으로 열린 것

Day 19 의 `calendar()` skill 이 **같은 자리에 함수 하나 더 넣는 것**으로 끝났다.
도구 스펙, Agent Loop, DDB 스키마, MQTT 페이로드가 전부 그대로다.

```
Day 17  sandbox = { read, awsCost, ...안전 글로벌 }
Day 19  sandbox = { read, awsCost, calendar, ...안전 글로벌 }
```

Day 19 README 의 diff 표에서 "권한: 추가 IAM 없음" 이 그 증거다 (캘린더는 공개 HTTPS 라서).

## 결과 · 그리고 이 skill 이 만든 아이러니

- `awsCost` 는 **자기참조적 데모**다 — "이 프로젝트가 돈을 얼마 쓰는지 에이전트가 안다."
- 그런데 정작 **Phase 3~4 비용을 이걸로 정리하지 않았다.** 집계된 실비용은 Phase 2 의 $0.07 뿐이다.
  도구를 만들어 놓고 안 쓴 셈이다. → [프로젝트 README](../README.md#아직-안-한-것과-이유)

## 비용 주의

Cost Explorer 는 **API 호출당 ~$0.01** 이다. 비용을 물어보는 것 자체가 비용이다.
`MAX_TURN_STEPS` 가 컷을 걸어 질문당 보통 1~2회로 끝나지만, 이건 우연히 안전한 것이지
설계로 막은 게 아니다. skill 호출 횟수에 별도 상한을 두지 않았다.

## 관련 함정

→ [트러블 08](../troubleshooting/08-오늘-비용이-0으로-나온다.md) (#47 · #48 · #52)

그 외:
- **#49** 샌드박스 timeout — 네트워크가 들어오면서 5s 가 빠듯해져 10s 로 (Day 19 에 15s)
- **#50** 토큰 폭증 — `skillCalls` 를 LLM 에 되먹였을 때. 이 결정의 계기
- **#51** CE 호출당 비용 — 위
