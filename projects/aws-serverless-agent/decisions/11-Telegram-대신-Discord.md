# 11. 원본의 Telegram 대신 Discord Interactions

> 메모: Phase 4 는 "원본에서 갈라지기". 원본 Telegram 미러링 대신 Discord Interactions 웹훅. Ed25519 서명검증 + deferred 응답, 의존성 0

---

## 상황

원본은 Telegram 채널을 붙여 에이전트 대화를 미러링한다.
Phase 4 의 주제는 **"원본에서 갈라지기"** — 코어 재현이 끝났으니 여기서부턴 내 사용처로 간다.

채널을 하나 붙이되, **원본과 다른 것**으로 붙이기로 했다.

## 결정

**Discord Interactions 엔드포인트**를 붙인다. 전용 Lambda + Function URL.
프레임워크 없이 `node:crypto` 만 쓴다 — **의존성 0**.

```
Discord ──POST /interactions──▶ Discord Lambda (Function URL)
                                  1) Ed25519 서명검증 (raw body)
                                  2) type 1 (PING) → { type: 1 } PONG
                                  3) /ask 슬래시 → { type: 5 } DEFERRED 즉시 반환
                                  4) Worker async invoke { channel: "discord", ... }
                                                    │
                                                    ▼
                              Worker (Day 13~17 그대로) → Agent Loop → awsCost skill
                                                    │
                                  followup webhook PATCH ─┘
```

## 왜

- **Discord 를 실제로 쓴다.** Telegram 은 안 쓴다. 채널을 붙이는 목적이
  "봇이 동작한다"를 보이는 게 아니라 **내가 쓰는 곳에서 에이전트를 부르는 것**이라면,
  안 쓰는 메신저에 붙이는 건 의미가 없다.
- **채널 디커플링을 검증할 수 있다.** Day 11 에서 만든
  `{type: "run_chat", ...}` discriminated union payload 에 `channel: "discord"` 한 칸만 더했다.
  Agent Loop, skill, DDB, MQTT 는 **하나도 안 건드렸다.**
  "채널 람다는 수신·검증·전달만 하고, 에이전트는 Worker 가 재사용한다"가 증명된 셈이다.
- **Interactions 웹훅은 서버리스와 잘 맞는다.** 봇을 게이트웨이에 상주 연결시키는 방식
  (discord.js 등)은 WebSocket 을 계속 물고 있어야 해서 Lambda 와 안 맞는다.
  Interactions 는 HTTP 요청/응답이라 Function URL 하나면 된다.
- **bot token 을 런타임에 안 쓴다.** followup 응답은 interaction token 으로 보낸다.
  bot token 은 슬래시 명령 등록 **1회**에만 쓴다. 유출 표면이 줄어든다.
- **의존성 0.** Ed25519 검증은 Node 20 의 `node:crypto` 로 된다.
  라이브러리를 하나 안 들이면 번들도, 취약점 추적도, 콜드스타트도 그만큼 가볍다.

## 왜 다른 건 안 썼나

**Telegram (원본 그대로)**

원본을 따라가면 매핑이 깔끔하고 참고할 코드가 있다.
안 쓴 이유는 하나 — **Phase 4 의 주제가 "갈라지기"** 이고, 안 쓰는 메신저를 붙일 이유가 없다.
Phase 1~3 에서 원본을 따라간 것과 여기서 갈라지는 것은 같은 원칙의 앞뒤다.

**discord.js 로 게이트웨이 봇**

기능이 훨씬 많다(메시지 수신, 리액션, 음성). 그런데 **상주 연결이 필요하다.**
서버리스 프로젝트에서 상주 프로세스를 띄우면 그 자체로 주제와 어긋난다.
그리고 이 프로젝트에서 필요한 건 슬래시 명령 하나다.

**Slack**

Discord 와 구조가 비슷하다(서명검증 + 3초 제한 + deferred). 고르는 데 큰 차이가 없었고,
개인적으로 Discord 를 더 쓴다는 게 이유의 전부다. 이건 근거라기보다 선호다.

**웹 UI 만 두고 채널 안 붙이기**

Phase 4 에 채널을 안 넣으면 "Worker 가 채널에 무관한가"를 검증할 기회가 없다.
Day 11~17 내내 "채널 디커플링" 을 주장했는데, 채널이 하나뿐이면 그건 주장일 뿐이다.

## 3초의 벽 — 이 결정의 핵심 제약

Discord 는 Interactions 응답을 **3초 안에** 요구한다.
Agent Loop 는 Bedrock 을 여러 번 왕복하므로 **절대 3초 안에 안 끝난다.**

그래서 **두 단계로 나눈다.**

```
1) 즉시   { type: 5 }  DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE  → 디스코드가 "생각 중…" 표시
2) 나중   PATCH /webhooks/{application_id}/{interaction_token}/messages/@original
```

Worker 가 답을 다 만들면 followup webhook 을 PATCH 해서 자리를 채운다.
**bot token 불필요** — interaction token 만으로 된다.

이 구조는 [결정 04](04-API와-Worker를-async-invoke로-분리.md) 의 async 분리와 정확히 같은 모양이다.
HTTP 응답 시간과 실제 작업 시간을 분리하는 것. Discord 는 그걸 프로토콜로 강제한다.

## 결과

- Day 18 에서 Discord `/ask` 로 물어보면 `awsCost` skill 까지 동작하는 걸 확인했다.
  웹 UI 의 변화는 0이다 (커밋에 `chore(day-18): web/index.html — day-17 페이지 그대로 동봉` 이 있다).
- Day 19 의 캘린더 skill 검증도 **Discord 로 했다.** 채널이 하나 더 있으니
  검증 경로가 하나 더 생긴 셈이다.

## 확인하지 못한 것

**중간 단계를 안 보여준다.** 웹은 MQTT 로 tool_call → tool_result 가 흐르는데,
Discord 는 "생각 중…" 하나만 뜨고 최종 답만 채워진다.
followup 을 여러 번 PATCH 하면 스트리밍처럼 만들 수 있는데 안 했다.

**멀티 채널 추상화가 없다.** 지금은 Worker 안에 `if (channel === "discord")` 분기가 있다.
채널이 셋이 되면 이 분기를 정리해야 한다. Day 18 README 에 "옵션"으로 남겼다.

## 관련 함정

→ [트러블 09](../troubleshooting/09-Discord가-Endpoint-URL을-저장-못-함.md) (#53 · #54 · #58)
→ [트러블 10](../troubleshooting/10-봇이-응답-실패만-띄운다.md) (#55 · #56 · #57)
