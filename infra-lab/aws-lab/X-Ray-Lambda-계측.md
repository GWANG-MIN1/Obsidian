# X-Ray로 Lambda 계측하기

> 메모: active tracing만 켜면 함수 상자만 보인다. 다운스트림 호출을 보려면 클라이언트를 래핑해야 하고, async invoke는 trace가 저절로 안 이어진다

---

## 개념

분산 추적의 일반 개념(Span·컨텍스트 전파·샘플링·OpenTelemetry)은
→ [`../observability-lab/07-tracing/`](../observability-lab/07-tracing/)

여기는 **AWS Lambda + X-Ray에서만 생기는 것**을 적는다.

### 계측은 두 단계다 — 한쪽만 하면 반쪽이 보인다

| 단계 | 무엇을 켜나 | 보이는 것 |
|---|---|---|
| ① active tracing | `Tracing.ACTIVE` (인프라) | **함수 자체**의 지속시간·에러 |
| ② 클라이언트 래핑 | `captureAWSv3Client` (코드) | **다운스트림 호출**이 subsegment로 |

①만 켜면 trace에 함수 상자 하나만 뜬다.
"Bedrock 호출이 몇 초 걸렸나"를 보려면 ②가 필요하다. **둘 다 해야 한다.**

```js
import AWSXRay from "aws-xray-sdk-core";
import { BedrockRuntimeClient } from "@aws-sdk/client-bedrock-runtime";

const bedrock = AWSXRay.captureAWSv3Client(new BedrockRuntimeClient({}));
//               ↑ 이 래핑이 없으면 호출이 trace에 안 나온다
```

### async invoke는 trace가 저절로 안 이어진다

```
API Lambda ──InvocationType: Event──▶ Worker Lambda
```

둘 다 `Tracing.ACTIVE`를 켜도 **서로 다른 trace로 따로 기록된다.**
"요청 하나가 API를 거쳐 Worker까지 갔다"가 한 장으로 안 보인다.

호출하는 쪽의 **Lambda 클라이언트를 래핑**해야 invoke가 subsegment로 잡히고 trace가 이어진다.

```js
const lambda = AWSXRay.captureAWSv3Client(new LambdaClient({}));
//               ↑ API 쪽. 이게 있어야 API→Worker가 한 trace
```

> 계측을 켠 것과 호출이 보이는 것이 다르고, 두 함수를 다 계측한 것과
> 두 함수가 한 trace로 이어지는 것이 또 다르다. **세 번 확인해야 한다.**

### ESM 번들에서 X-Ray SDK는 터진다

`aws-xray-sdk-core`는 내부적으로 동적 `require()`를 호출한다.
**ESM 번들에는 `require`가 없어서 모듈 로드 시점에 크래시**한다.
→ [esbuild ESM 번들의 동적 require](esbuild-ESM-번들의-동적-require.md)

그리고 미사용 경로에서 `require('aws-sdk')`(v2)를 해석하려 들어 번들이 깨진다.

```ts
bundling: { externalModules: ['aws-sdk'] }
```

### Lambda@Edge는 active tracing을 지원하지 않는다

`Tracing.ACTIVE`를 주면 배포가 실패한다. 엣지 함수만 제외해야 한다.
→ [Lambda@Edge 운영 제약](Lambda-Edge-운영-제약.md)

## 실행

```ts
// CDK — 엣지를 뺀 나머지에만
for (const fn of [apiFn, workerFn, discordFn]) {
  fn.addPropertyOverride('TracingConfig.Mode', 'Active');   // 또는 tracing: lambda.Tracing.ACTIVE
}
```

```ts
new nodejs.NodejsFunction(this, 'ApiFunction', {
  bundling: {
    format: nodejs.OutputFormat.ESM,
    banner: 'import { createRequire } from "module"; const require = createRequire(import.meta.url);',
    externalModules: ['aws-sdk'],
  },
  tracing: lambda.Tracing.ACTIVE,
});
```

검증은 **서비스 맵**으로 한다. 상자가 이어져 있으면 통과다.

```
Client → API → Worker → Bedrock
                  ├──▶ DynamoDB
                  ├──▶ STS
                  └──▶ IoT
```

API와 Worker가 **끊어진 두 덩어리로 보이면** 호출 쪽 래핑이 빠진 것이다.

### 알람 두 가지 함정

```ts
new cloudwatch.Alarm(this, 'WorkerErrors', {
  metric: workerFn.metricErrors({ period: cdk.Duration.minutes(5) }),
  threshold: 1,
  evaluationPeriods: 1,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,   // ★
});
```

> **`treatMissingData`를 안 주면 알람이 계속 `INSUFFICIENT_DATA`다.**
> 트래픽이 없으면 에러 지표 자체가 빈값이라 그렇다. 학습·저트래픽 환경에서는 기본 상태가 이것이 된다.

> **SNS 이메일 구독은 확인 링크를 클릭해야 발송된다.** 구독을 만들어 놓고
> "알람이 울리는데 메일이 안 온다"로 헤매기 쉽다. 상태가 `PendingConfirmation`인지 본다.

## 배운 점

**"계측을 켰다"와 "보인다"는 다르다.** active tracing은 함수 경계까지만 그린다.
안에서 무엇을 불렀는지는 클라이언트 래핑이 있어야 나온다.
**설정을 켠 뒤에 실제로 원하는 화면이 나오는지 확인해야 한다.**

**비동기 경계는 추적이 끊기는 기본 지점이다.** HTTP 호출은 헤더로 컨텍스트가 전파되지만
`InvocationType: Event`는 그냥 던지고 끝난다. SDK 래핑이 그 자리를 메운다.
큐·이벤트버스를 끼우면 같은 문제가 또 생긴다.

**관측성 계측은 비즈니스 로직과 분리해서 넣을 수 있다.** 클라이언트를 한 번 감싸고
CDK에 위젯을 선언하는 것으로 끝났다 — 핸들러 코드는 안 건드렸다.
그런데 그 대가로 **번들링 계층에서 사고가 났다.** 의존성을 하나 들이는 비용은 코드가 아니라
빌드에서 나올 수 있다.

**알람은 "울리는가"보다 "안 울릴 때 어떤 상태인가"를 먼저 본다.**
`INSUFFICIENT_DATA`로 앉아 있는 알람은 없는 알람과 같다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) — Day 20
- 번들 크래시로 이어진 사고: [트러블 12](../../projects/aws-serverless-agent/troubleshooting/12-재배포하니-전-요청이-500.md)
- 일반 분산추적 개념: [`../observability-lab/07-tracing/`](../observability-lab/07-tracing/)
