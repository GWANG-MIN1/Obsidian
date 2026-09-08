# 05. IoT Core 로 publish 했는데 아무 데도 안 간다

- **발생**: 2026-06-05 (Day 14) · 저장소 함정 **#27 · #28 · #29**
- **증상 한 줄**: `PublishCommand` 가 에러를 내거나, 성공하는데 콘솔 test client 엔 안 뜬다

---

## 현상

Day 14 에서 Worker 가 Agent Loop 의 각 단계를 IoT Core MQTT 토픽에 publish 하도록 했다.
`sessions/${sessionId}/events`.

세 단계로 막혔다.

1. 패키지를 못 찾는다 (`Cannot find package @aws-sdk/client-iot...`)
2. 그걸 고치니 endpoint 에러
3. 그걸 고치니 `AccessDenied`

## 환경

- AWS IoT Core, us-east-1
- Worker Lambda (NodejsFunction, esbuild 번들, `externalModules` 사용)
- 확인 수단: AWS 콘솔 IoT → MQTT test client

## 진단

### 1) SDK 패키지가 두 개다 (#29)

IoT 는 **control plane 과 data plane 의 SDK 패키지가 다르다.**

| 하려는 일 | 패키지 | 클라이언트 |
|---|---|---|
| 엔드포인트 조회 (`DescribeEndpoint`) | `@aws-sdk/client-iot` | `IoTClient` |
| 메시지 publish (`Publish`) | `@aws-sdk/client-iot-data-plane` | `IoTDataPlaneClient` |

하나만 넣고 둘 다 쓰려다 막혔다.
esbuild `externalModules` 에도 **둘 다** 넣어야 번들에서 빠지고 런타임 SDK 를 쓴다.

### 2) 엔드포인트를 안 주면 기본값이 안 맞는다 (#27)

`IoTDataPlaneClient` 를 endpoint 없이 만들면 SDK 가 리전 기본 엔드포인트를 쓴다.
그런데 publish 는 **계정별 IoT Data 엔드포인트**로 가야 한다.

계정마다 다른 주소이므로 **런타임에 조회해야 한다.**

```js
const { endpointAddress } = await iot.send(
  new DescribeEndpointCommand({ endpointType: "iot:Data-ATS" })
);
const data = new IoTDataPlaneClient({ endpoint: `https://${endpointAddress}` });
```

`iot:Data-ATS` 여야 한다 — ATS(Amazon Trust Services) 인증서 체인을 쓰는 현행 엔드포인트다.

### 3) publish 와 subscribe 의 ARN 형태가 다르다 (#28)

IAM 을 이렇게 줬었다.

```
arn:aws:iot:us-east-1:<acct>:topicfilter/sessions/*/events
```

`AccessDenied`. IoT 의 리소스 ARN 은 **동작마다 형태가 다르다.**

| 동작 | ARN 형태 |
|---|---|
| `iot:Publish` | `topic/...` |
| `iot:Subscribe` | `topicfilter/...` |
| `iot:Receive` | `topic/...` |
| `iot:Connect` | `client/...` |

publish 에 `topicfilter/` 를 주면 안 먹는다. 이름이 비슷해서 눈에 안 띈다.

## 원인

세 가지가 겹쳤다.

1. **control plane 과 data plane 패키지를 구분 안 함** — 엔드포인트 조회와 publish 는 다른 서비스다
2. **계정별 Data-ATS 엔드포인트를 런타임에 조회하지 않음** — 기본값으로는 안 간다
3. **`iot:Publish` 의 리소스 ARN 을 `topicfilter/` 로 줌** — publish 는 `topic/`

## 해결

```ts
// CDK — publish 는 topic/, 토픽 접두어는 코드와 같은 값
workerFn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['iot:Publish'],
  resources: [`arn:aws:iot:${region}:${account}:topic/sessions/*/events`],
}));
workerFn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['iot:DescribeEndpoint'],
  resources: ['*'],                    // DescribeEndpoint 는 리소스 한정이 안 된다
}));
```

```js
// 런타임 — 엔드포인트 조회 후 data plane 클라이언트 생성, 결과는 캐시
const { endpointAddress } = await iot.send(
  new DescribeEndpointCommand({ endpointType: "iot:Data-ATS" })
);
```

그리고 **best-effort 로 감쌌다** (#31).

```js
async function publishEvent(sessionId, payload) {
  try { await data.send(new PublishCommand({ topic, qos: 1, payload: JSON.stringify(payload) })); }
  catch (e) { console.warn("publish failed", e?.message); }    // 던지지 않는다
}
```

처음엔 publish 에러를 그냥 throw 했는데, **그러면 publish 실패가 Agent Loop 를 통째로 멈춘다.**
진실은 DDB 고 MQTT 는 곁가지다 → [결정 07](../decisions/07-실시간을-IoT-MQTT-push로.md)

## 확인이 안 될 때 — 콘솔 쪽 함정 (#30)

publish 는 성공하는데 MQTT test client 에 안 뜨는 경우가 있었다.
원인은 셋 중 하나다.

- **콘솔 리전이 배포 리전과 다르다** (제일 흔하다)
- 토픽 오타
- 정확한 토픽으로 구독했는데 sessionId 가 그때그때 달라진다

`sessions/+/events` 로 **와일드카드 구독**을 먼저 해서 뭐라도 오는지 보고,
그다음 정확한 토픽으로 좁히는 게 빠르다.

## 재발 방지

- **IoT 는 control/data plane 이 갈린다는 걸 전제로 시작한다.** 패키지 두 개.
- **엔드포인트를 코드에 박지 않는다.** 계정마다 다르므로 `DescribeEndpoint` 로 조회하고 캐시한다.
  (Day 16 의 SSM 캐싱과 같은 발상 — Day 14 README 에 "엔드포인트 조회를 SSM 캐싱으로"를
  옵션으로 남겨 뒀는데 안 했다.)
- **IAM 리소스 ARN 은 동작별 형태를 확인한다.** `topic/` vs `topicfilter/` 처럼
  한 글자 차이로 갈리는 게 있다.
- **토픽 접두어를 환경변수 하나로 통일한다.** Day 15 에서 API 와 Worker 의
  `MQTT_TOPIC_PREFIX` 가 달라서 또 헤맸다 (함정 #37).
- **부가 기능은 best-effort 로 감싼다.** 관측/알림 계열이 본 로직을 멈추면 안 된다.

## 배운 점

**"성공했다"가 "도착했다"는 아니다.** `PublishCommand` 가 예외 없이 끝나도
엔드포인트가 틀리면 아무 데도 안 간다. **수신 쪽에서 확인**해야 검증이다.
이 프로젝트에서 매 day 실배포 + 실호출 검증을 고집한 이유가 이거다.

**AWS 서비스 안에서도 API 표면이 갈린다.** IoT 는 control plane 과 data plane 이
패키지·엔드포인트·ARN 형태까지 전부 다르다. 한 서비스 이름 아래 있다고 한 덩어리가 아니다.
