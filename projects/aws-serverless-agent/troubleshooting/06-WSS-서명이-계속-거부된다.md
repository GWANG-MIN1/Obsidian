# 06. 브라우저 WSS 연결이 계속 서명 불일치로 거부된다

- **발생**: 2026-06-06 (Day 15) · 저장소 함정 **#32 · #33 · #36 · #38**
- **증상 한 줄**: presigned WSS URL 로 붙으면 `SignatureDoesNotMatch`, 고치면 연결 직후 끊김

---

## 현상

Day 15 에서 브라우저가 IoT Core 에 직접 붙게 만들었다.
API 가 SigV4 로 서명한 `wss://...-ats.iot.us-east-1.amazonaws.com/mqtt?X-Amz-...` URL 을 발급하고,
브라우저는 `mqtt.js` 로 그 URL 에 `connect()` 한다.

네 단계로 막혔다.

1. `SignatureDoesNotMatch`
2. 서명은 맞는데 여전히 거부
3. 연결은 되는데 **직후에 끊김**
4. 구독은 되는데 **이벤트가 안 옴**

## 환경

- 브라우저: `mqtt.js` (esm.sh 로 로드, 빌드 없음)
- API Lambda 가 STS AssumeRole → 세션정책 → SigV4 서명
- IoT Core Data-ATS 엔드포인트, us-east-1

## 진단

### 1) 보안 토큰을 서명에 포함시켰다 (#32)

일반 SigV4 presigned URL 은 `X-Amz-Security-Token` 을 **canonical query 에 포함**해서 서명한다.
S3 presigned URL 이 그렇다.

**IoT 는 다르다.** 토큰을 서명 계산에서 **빼고** URL 끝에 붙여야 한다.

```js
const canonicalRequest = ["GET", "/mqtt", canonicalQuery, `host:${host}\n`, "host", sha256hex("")].join("\n");
// ... 서명 계산 (canonicalQuery 에 X-Amz-Security-Token 없음) ...
let url = `wss://${host}/mqtt?${canonicalQuery}&X-Amz-Signature=${signature}`;
if (creds.sessionToken) url += `&X-Amz-Security-Token=${encodeURIComponent(creds.sessionToken)}`;
//                                    ↑ 서명 후 추가
```

이걸 모르면 **다른 모든 게 맞아도 안 된다.** 그리고 에러 메시지는 그냥 서명 불일치다.

### 2) 서비스명과 경로가 틀렸다 (#33)

| 항목 | 값 |
|---|---|
| service | **`iotdevicegateway`** (`iot` 아님) |
| path | **`/mqtt`** |
| signed header | `host` 하나만 |

서비스명이 `iot` 가 아니라는 게 함정이다. IAM 액션은 `iot:Connect` 인데
SigV4 서명 스코프는 `iotdevicegateway` 다. 이름이 안 맞는다.

### 3) MQTT 프로토콜 버전이 안 맞았다 (#36)

서명이 통과하고 연결이 열렸는데 **직후에 끊긴다.**

`mqtt.js` 는 기본으로 MQTT 5 를 시도한다. **AWS IoT Core 는 MQTT 3.1.1** 이다.

```js
mqtt.connect(url, { protocolVersion: 4 });   // 4 = MQTT 3.1.1
```

### 4) 구독하기 전에 발행된 이벤트는 안 온다 (#38)

연결·구독이 다 되는데 첫 턴 이벤트가 안 보인다.

MQTT 는 **retained 메시지가 아니면 구독 이전 발행분을 안 준다.**
`POST /chat` 을 먼저 던지고 그다음 구독하면, 이미 지나간 이벤트는 못 받는다.

**구독 먼저, chat 나중.**

## 원인

네 겹이었고 **성격이 다 다르다.**

| # | 계층 | 원인 |
|---|---|---|
| 32 | 서명 | IoT 는 보안 토큰을 서명에서 제외 (AWS 서비스 중 특례) |
| 33 | 서명 | 서비스명 `iotdevicegateway`, 경로 `/mqtt` |
| 36 | 프로토콜 | IoT Core 는 MQTT 3.1.1 — 클라이언트 기본값이 5 |
| 38 | 사용 순서 | retained 가 아니므로 구독 전 발행분은 유실 |

앞의 둘은 **문서를 읽어야 알고**, 셋째는 **클라이언트 기본값**, 넷째는 **프로토콜 특성**이다.
전부 "코드가 틀렸다"가 아니라 "전제를 몰랐다"에 가깝다.

## 해결

원본의 `backend/src/lib/iot-sigv4.ts` 를 그대로 포팅했다.
직접 서명 코드를 쓰기 전에 **레퍼런스 구현을 확인한 게 결국 가장 빨랐다.**

브라우저 쪽은 순서를 고정했다.

```
1) GET /sessions/:id/realtime   → { url, channel }
2) mqtt.connect(url, { protocolVersion: 4 })
3) subscribe(channel, { qos: 1 })
4) ← 여기까지 끝난 뒤에 POST /chat
```

## 함께 있던 다른 함정

**#34 — 브라우저에 과한 권한**

Lambda 자기 자격증명으로 서명하면 브라우저가 그 권한을 전부 갖는다.
AssumeRole + 세션정책으로 좁혔다 → [결정 08](../decisions/08-브라우저-권한을-세션정책으로.md)

**#35 — CDK 순환 의존 (Role ↔ Lambda)**

역할의 trust 에 Lambda 의 role 을 넣고, Lambda 의 env 에 역할 ARN 을 넣으면
CDK 가 서로를 참조해서 합성이 안 된다.

```ts
// env 는 함수 생성 후에 주입
apiFn.addEnvironment('REALTIME_ROLE_ARN', realtimeRole.roleArn);
// trust 는 ArnPrincipal 로
new iam.Role(this, 'RealtimeRole', { assumedBy: new iam.ArnPrincipal(apiFn.role.roleArn) });
```

**#37 — API 와 Worker 의 토픽 접두어가 다름**

API 가 발급한 `channel` 과 Worker 가 publish 하는 토픽이 안 맞았다.
`MQTT_TOPIC_PREFIX` 를 양쪽 다 `sessions` 로 통일했다.
→ [트러블 05](05-IoT-publish가-아무-데도-안-간다.md) 의 재발 방지와 같은 항목

## 재발 방지

- **서명 코드를 직접 쓰기 전에 레퍼런스 구현을 찾는다.** SigV4 는 한 글자만 틀려도 안 되고,
  에러 메시지가 어디가 틀렸는지 안 알려준다. 디버깅 비용이 검색 비용보다 훨씬 크다.
- **서비스마다 SigV4 특례가 있는지 확인한다.** IoT 의 보안 토큰 처리처럼
  "이 서비스만 이렇게"가 존재한다.
- **클라이언트 라이브러리의 기본 프로토콜 버전을 확인한다.** `mqtt.js` 의 기본값이 5 인 건
  라이브러리 입장에선 최신을 쓰는 합리적 선택이고, 서버가 못 따라올 뿐이다.
- **push 기반은 "구독 먼저" 를 순서로 고정한다.** 이건 코드가 아니라 절차의 문제라
  README 검증 절차에 단계 번호로 적어 뒀다.

## 배운 점

**"안 된다"의 원인이 네 겹일 수 있다.** 하나 고칠 때마다 증상이 바뀌면서
각각 다른 문제처럼 보였다. 증상이 바뀌면 **앞 문제는 해결된 것**이라고 세는 게
진행 상황을 잃지 않는 방법이었다.

**서명 실패는 정보가 없는 에러다.** `SignatureDoesNotMatch` 는
"틀렸다"만 알려주고 "어디가"는 안 알려준다. 이런 종류는 **가설을 세워 하나씩 대조**하는 것보다
**작동하는 구현과 내 것을 diff 하는 게** 압도적으로 빠르다.
