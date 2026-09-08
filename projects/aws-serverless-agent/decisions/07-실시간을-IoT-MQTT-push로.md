# 07. 실시간 진행 표시를 폴링 대신 IoT Core MQTT push 로

> 메모: Day 11 에서 async 로 바꾸면서 결과 볼 방법이 폴링밖에 없어짐. 원본이 IoT Core WSS 를 쓰길래 따라감. publish 는 best-effort — 진실은 DDB

---

## 상황

[결정 04](04-API와-Worker를-async-invoke로-분리.md) 로 API 가 202 를 즉시 돌려주게 되면서
**결과를 볼 방법이 사라졌다.** Day 11~13 에서는 `GET /sessions/:id/messages` 를 손으로 폴링했다.

Agent Loop 가 들어온 Day 13 이후로는 이게 더 아쉬워졌다.
루프가 5스텝을 돌면 중간에 tool_call → tool_result → text 가 차례로 쌓이는데,
폴링은 그 **진행 과정**을 못 보여준다. 다 끝난 뒤의 결과만 본다.

## 결정

Worker 가 **행을 저장하는 그 순간** 같은 내용을 IoT Core MQTT 토픽에 publish 한다.
토픽은 세션별로 `sessions/${sessionId}/events`.

```js
// putRow 마다 한 줄 더
await putRow(row);
await publishEvent(sessionId, { type: "entity_update", row });   // best-effort
```

**publish 실패는 턴을 죽이지 않는다.** try/catch 로 감싸고 warn 만 남긴다.

## 왜

- **진실은 DDB 고 MQTT 는 곁가지다.** 이 한 줄이 이 결정의 핵심이다.
  publish 가 실패해도 데이터는 DDB 에 남아 있고 `GET /messages` 로 다 볼 수 있다.
  그래서 publish 에러를 throw 하지 않는다 (함정 #31 — 처음엔 throw 해서 루프가 통째로 멈췄다).
- **저장 = publish 라 페이로드가 같다.** DDB 행과 MQTT 메시지가 같은 모양이라
  브라우저의 렌더 코드를 **히스토리 조회와 실시간 구독이 공유**한다. 코드가 두 벌이 안 된다.
- **폴링은 진행을 못 보여준다.** 1초 폴링을 걸어도 "지금 뭘 하는 중"이 아니라
  "지금까지 뭐가 쌓였나"만 보인다. tool_call 이 뜨고 몇 초 뒤 tool_result 가 뜨는 리듬이
  에이전트가 무엇을 하고 있는지 보여주는 정보다.
- **원본이 IoT Core WSS 를 쓴다.** 그리고 이유가 납득됐다 —
  브라우저에 AWS 자격증명 없이 push 를 꽂는 방법으로 IoT Core 는 **SigV4 presigned WSS** 라는
  깔끔한 경로가 있다. → [결정 08](08-브라우저-권한을-세션정책으로.md)
- **세션별 토픽이 권한 경계가 된다.** `sessions/${id}/events` 로 나눠 두면
  브라우저에게 "그 세션만 구독" 권한을 주는 게 자연스럽다.

## 왜 다른 건 안 썼나

**폴링 유지 (`GET /messages` 를 1초마다)**

가장 단순하고 추가 인프라가 0이다. 실제로 Day 11~13 을 이걸로 버텼다.

안 쓴 이유: **진행 과정이 안 보이고**, 유휴 상태에서도 요청이 계속 나간다.
그리고 무엇보다 Phase 2 회고가 남긴 질문 —
*"브라우저가 AWS IoT 에 어떻게 직접 인증해?"* — 에 답하는 게 Phase 3 의 목표 중 하나였다.

**API Gateway WebSocket API**

정공법이고 관리형이다. 안 쓴 이유는 [결정 02](02-API-Gateway-대신-Function-URL.md) 와 같은 줄에 있다 —
**원본이 API Gateway 를 아예 안 쓴다.** 그리고 WebSocket API 를 쓰면 연결 상태를
DDB 에 따로 관리해야 한다(`connectionId` 테이블). IoT Core 는 브로커가 그걸 대신한다.

**Function URL 응답 스트리밍으로 되돌리기**

Day 6~7 구조. 그러면 [결정 04](04-API와-Worker를-async-invoke로-분리.md) 가 통째로 무효가 된다.
클라이언트가 끊기면 작업이 날아가는 문제로 되돌아간다.

**SSE (Server-Sent Events)**

Function URL 스트리밍 위에 SSE 를 얹는 방법. 같은 문제 — HTTP 연결이 살아 있어야 한다.

## 결과

- Day 14 에서 AWS 콘솔 MQTT test client 로 이벤트가 뜨는 걸 확인했다.
  같은 내용이 DDB 에도 있는지 교차 확인했다.
- Day 15 에서 브라우저가 직접 구독하게 만들면서, Agent Loop 의 tool_call → tool_result 가
  **폴링 없이 화면에 흐르는** 데모가 완성됐다. 저장소 루트 README 의 첫 데모가 이거다.
- Worker 의 publish 는 끝까지 best-effort 로 남겨 뒀다. **한 번도 이 결정을 후회하지 않았다.**

## 확인하지 못한 것

**턴 종료 신호가 없다.** 구독자는 이벤트가 더 안 오면 끝난 줄 안다.
`type: "turn_done"` 같은 명시적 종료 이벤트를 Day 14 README 에 "옵션"으로 남겼는데 안 넣었다.
지금 UI 는 "언제 끝났는지"를 모른다.

**user 메시지는 publish 되지 않는다.** API 가 쓰는 행이라 Worker 의 publish 경로를 안 탄다.
그래서 브라우저 화면에 **내 말풍선이 안 뜬다** — 프론트가 로컬로 그려 넣는다.

## 관련 함정

→ [트러블 05](../troubleshooting/05-IoT-publish가-아무-데도-안-간다.md) (#27 · #28 · #29)

그 외:
- **#30** MQTT test client 에 안 뜸 — 콘솔 리전이 배포 리전과 다르거나 토픽 오타.
  `sessions/+/events` 로 와일드카드 구독한 뒤 좁히는 게 빠르다.
- **#31** publish 실패가 턴 전체를 죽임 — best-effort 로 바꾼 계기.
