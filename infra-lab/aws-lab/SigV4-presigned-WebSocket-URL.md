# SigV4 presigned WebSocket URL (IoT)

> 메모: 키 없는 브라우저를 IAM만으로 IoT Core에 붙이는 방법. 보안 토큰을 서명에서 빼는 IoT 특례가 함정

---

## 개념

브라우저에는 AWS 자격증명이 없다. 그런데 IoT Core에 WebSocket으로 붙어야 한다면,
**서버가 미리 서명한 URL**을 발급해 주는 방식이 정공법이다.

```
브라우저 ── GET /realtime ──▶ 서버(Lambda)
   ▲                            1) STS AssumeRole (+ 세션정책으로 좁히기)
   │  { url: "wss://...-ats.iot.../mqtt?X-Amz-..." }
   │                            2) DescribeEndpoint(iot:Data-ATS) → host
   │◀───────────────────────────3) SigV4 서명
   │
   └─ mqtt.connect(url) ──▶ IoT Core
```

서명에 권한이 박혀 있어서 **추가 인증 핸드셰이크가 없다.**
X.509 디바이스 인증서 없이 IAM만으로 붙는다.

### IoT 특례 — 보안 토큰을 서명에서 뺀다

일반 SigV4 presigned URL(S3 등)은 `X-Amz-Security-Token`을 canonical query에 **포함해서** 서명한다.
**IoT는 반대다.** 서명 계산에서 빼고 URL 끝에만 붙인다.

```js
// service = iotdevicegateway, path = /mqtt, signed header = host 하나
const canonicalRequest = [
  "GET", "/mqtt", canonicalQuery, `host:${host}\n`, "host", sha256hex(""),
].join("\n");

// ... 서명 계산 (canonicalQuery 에 X-Amz-Security-Token 없음) ...

let url = `wss://${host}/mqtt?${canonicalQuery}&X-Amz-Signature=${signature}`;
if (creds.sessionToken) {
  url += `&X-Amz-Security-Token=${encodeURIComponent(creds.sessionToken)}`;  // ★ 서명 후 추가
}
```

이걸 모르면 **다른 게 다 맞아도 `SignatureDoesNotMatch`가 난다.**

### 이름이 안 맞는 것 두 가지

| 쓰는 곳 | 값 |
|---|---|
| IAM action | `iot:Connect` / `iot:Subscribe` / `iot:Receive` |
| **SigV4 서비스명** | **`iotdevicegateway`** (`iot` 아님) |
| 경로 | `/mqtt` |

그리고 IAM 리소스 ARN이 **동작마다 형태가 다르다.**

| 동작 | ARN |
|---|---|
| `iot:Publish` / `iot:Receive` | `topic/...` |
| `iot:Subscribe` | **`topicfilter/...`** |
| `iot:Connect` | `client/...` |

publish에 `topicfilter/`를 주면 `AccessDenied`. 한 글자 차이라 눈에 안 띈다.

### 권한은 두 정책의 교집합으로 좁힌다

Lambda 자기 자격증명으로 서명하면 **URL을 가진 사람이 그 Lambda 권한을 전부 갖는다.**
AssumeRole의 세션정책(`Policy`)으로 한 번 더 깎는다.

```js
const creds = await sts.send(new AssumeRoleCommand({
  RoleArn: process.env.REALTIME_ROLE_ARN,
  RoleSessionName: `rt-${sessionId}`,
  DurationSeconds: 3600,
  Policy: JSON.stringify({          // 세션정책 — 더 좁힐 수만 있다
    Version: "2012-10-17",
    Statement: [{
      Effect: "Allow",
      Action: ["iot:Connect", "iot:Subscribe", "iot:Receive"],
      Resource: [ /* 이 세션 토픽만 */ ],
    }],
  }),
}));
```

| 계층 | 허용 | 어디서 |
|---|---|---|
| 역할 자체 | `sessions/*/events` | IaC (상한선) |
| 세션정책 | `sessions/<이 id>/events` | 런타임 (좁히기) |
| **실제 권한** | **교집합** | — |

**세션정책은 넓힐 수 없다.** 좁히는 코드에 버그가 나도 역할의 상한을 못 넘는다.

## 실행

브라우저 쪽. 두 가지가 기본값과 다르다.

```js
import mqtt from "https://esm.sh/mqtt";

const { url, channel } = await (await fetch(`/api/sessions/${id}/realtime`)).json();

const client = mqtt.connect(url, { protocolVersion: 4 });   // ← 4 = MQTT 3.1.1
client.on("connect", () => client.subscribe(channel, { qos: 1 }));
client.on("message", (topic, payload) => render(JSON.parse(payload.toString())));
```

> **`protocolVersion: 4` 필수.** mqtt.js는 기본으로 MQTT 5를 시도하는데
> AWS IoT Core는 **MQTT 3.1.1**이다. 안 맞으면 연결 직후 끊긴다.

> **구독 먼저, 작업 나중.** retained 메시지가 아니면 구독 이전 발행분은 안 온다.
> 작업을 던지고 구독하면 첫 이벤트를 놓친다.

엔드포인트는 계정마다 다르므로 런타임 조회 후 캐시한다.

```js
const { endpointAddress } = await iot.send(
  new DescribeEndpointCommand({ endpointType: "iot:Data-ATS" })
);
```

`iot:Data-ATS`여야 한다 — ATS(Amazon Trust Services) 인증서 체인을 쓰는 현행 엔드포인트다.
그리고 조회(control plane)와 발행(data plane)은 **SDK 패키지가 다르다.**

| 일 | 패키지 | 클라이언트 |
|---|---|---|
| `DescribeEndpoint` | `@aws-sdk/client-iot` | `IoTClient` |
| `Publish` | `@aws-sdk/client-iot-data-plane` | `IoTDataPlaneClient` |

## 배운 점

**서명 코드는 직접 쓰기 전에 레퍼런스 구현을 찾는다.** SigV4는 한 글자만 틀려도 안 되고,
`SignatureDoesNotMatch`는 "틀렸다"만 알려주지 "어디가"는 안 알려준다.
**가설을 세워 대조하는 것보다 작동하는 구현과 diff하는 게 압도적으로 빠르다.**

**AWS 서비스마다 SigV4 특례가 있는지 확인한다.** IoT의 보안 토큰 처리처럼
"이 서비스만 이렇게"가 존재한다. 일반 규칙을 알아도 서비스 문서를 봐야 한다.

**한 서비스 이름 아래 있다고 한 덩어리가 아니다.** IoT는 control plane과 data plane이
패키지·엔드포인트·ARN 형태·서명 서비스명까지 전부 갈린다.

**클라이언트 라이브러리의 기본값이 서버보다 최신일 수 있다.** mqtt.js가 MQTT 5를 기본으로
시도하는 건 합리적인 선택이고, 서버가 못 따라올 뿐이다. 연결이 직후에 끊기면 버전을 의심한다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 08](../../projects/aws-serverless-agent/decisions/08-브라우저-권한을-세션정책으로.md)
- 서명 디버깅 과정: [트러블 06](../../projects/aws-serverless-agent/troubleshooting/06-WSS-서명이-계속-거부된다.md)
- publish 쪽 ARN·엔드포인트 함정: [트러블 05](../../projects/aws-serverless-agent/troubleshooting/05-IoT-publish가-아무-데도-안-간다.md)
