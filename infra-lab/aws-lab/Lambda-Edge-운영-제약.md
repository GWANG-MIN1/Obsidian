# Lambda@Edge 운영 제약

> 메모: 일반 Lambda에 제약이 붙은 게 아니라 다른 실행 환경. us-east-1 고정 · 환경변수 불가 · 삭제 지연

---

## 개념

CloudFront에서 요청을 가로채는 방법은 둘이다. **무엇을 바꿀 수 있느냐**가 갈린다.

|  | CloudFront Function | Lambda@Edge |
|---|---|---|
| 실행 | JS 격리 런타임 | Node.js |
| 지연 | µs | ms (콜드스타트 있음) |
| 비용 | 2M/월 무료 | 요청 + 실행시간 과금 |
| **바꿀 수 있는 것** | **URI·헤더·쿠키·쿼리 문자열** | **+ `request.origin` 자체** |
| **AWS API 호출** | **불가** | **가능** (SSM·DDB 등) |
| 이벤트 | viewer-request/response | + origin-request/response |

**origin을 런타임에 고르려면 Lambda@Edge가 필요하다.**
CF Function은 문자열만 만질 수 있어서 "SSM에서 백엔드 주소를 읽어 오리진을 바꾼다"가 안 된다.

반대로 `/api` 접두어를 떼는 정도면 CF Function이 맞다. µs에 무료다.

### 제약 다섯 가지

| 제약 | 결과 | 대응 |
|---|---|---|
| **us-east-1 필수** | 다른 리전에 만들면 CloudFront가 연결 거부 | 스택 리전 고정 |
| **환경변수 불가** | `Lambda@Edge does not support environment variables` | 설정값을 **소스 상수**로 |
| **삭제 지연** | `destroy`가 실패한다 | 정상. CF 내려간 뒤 시간 두고 재시도 |
| **X-Ray active tracing 미지원** | `Tracing.ACTIVE` 주면 배포 실패 | 엣지만 제외 |
| 패키지·실행시간 상한이 더 작음 | viewer 이벤트는 특히 빡빡 | 무거운 로직 금지 |

> **환경변수 불가가 가장 자주 발목을 잡는다.** 내가 env를 안 줬는데도 배포가 실패하는데,
> CDK `NodejsFunction`이 기본으로 `AWS_NODEJS_CONNECTION_REUSE_ENABLED=1`을 넣기 때문이다.
> 일반 Lambda에서는 성능에 좋은 값이라 기본값인데, 여기서는 배포를 막는다.

> **삭제 지연은 사고가 아니다.** 엣지 함수 복제본이 전 리전에 퍼져 있어서
> CloudFront 배포가 완전히 내려가야 정리된다. 실패 메시지만 보면 사고처럼 보인다.

## 실행

```ts
// bin/app.ts — 리전 고정
new EdgeStack(app, 'EdgeStack', { env: { region: 'us-east-1' } });
```

```ts
const edgeFn = new nodejs.NodejsFunction(this, 'EdgeFn', {
  entry: 'lambda/edge-origin-request.mjs',
  runtime: lambda.Runtime.NODEJS_20_X,
  awsSdkConnectionReuse: false,      // ← 기본 env 주입을 끈다
  // Tracing.ACTIVE 금지
});

edgeFn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['ssm:GetParameter'],
  resources: [`arn:aws:ssm:us-east-1:${this.account}:parameter/myapp/backend/url`],
}));
```

엣지 함수 역할의 신뢰 주체에 **`edgelambda.amazonaws.com`이 같이 들어가야** 한다
(`lambda.amazonaws.com`만으로는 안 된다).

### 설정값 주입 — define 대신 소스 상수

env가 막히니 esbuild `--define`으로 빌드 타임 치환을 시도하게 되는데,
Windows 셸에서 안쪽 따옴표가 벗겨져 `Invalid define value`가 난다.

그런데 **엣지 함수는 어차피 설정이 빌드 타임에 고정된다.** define으로 넣으나 소스에 적으나 결과가 같다.

```js
const PROJECT    = "myapp";        // 엣지는 env 불가 → 그냥 상수
const SSM_REGION = "us-east-1";
```

### 같은 스택의 값은 CDK 토큰이라 JS로 못 만진다

```ts
const host = new URL(backendFn.functionUrl.url).hostname;   // ✗ synth 에러
```

같은 스택 안의 Function URL은 synth 시점에 `${Token[TOKEN.123]}` 문자열이다.
실제 값은 deploy 때 정해진다. CloudFormation 내장 함수로 남긴다.

```ts
const host = cdk.Fn.select(2, cdk.Fn.split('/', backendFn.functionUrl.url));
// "https://xxx.lambda-url.us-east-1.on.aws/" → ["https:", "", "xxx.lambda-url...", ""]
```

`new URL(token)`, `token.split()`, `token.includes()` 전부 안 된다.

### origin을 바꾸면 Host도 같이 바꾼다

```js
export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  if (request.uri.startsWith("/api/")) {
    const host = new URL(await getBackendUrl()).hostname;   // 여기선 진짜 문자열이라 OK
    request.origin = { custom: { domainName: host, port: 443, protocol: "https",
                                 sslProtocols: ["TLSv1.2"], readTimeout: 30, keepaliveTimeout: 5,
                                 customHeaders: {} } };
    request.headers.host = [{ key: "Host", value: host }];   // ← 빠뜨리면 403
    request.uri = request.uri.slice(4);                      // /api strip
  }
  return request;
};
```

> **Host를 안 바꾸면 오리진이 403을 낸다.** Function URL·ALB는 자기 도메인 Host만 받는다.
> CloudFront behavior 단에서는 `OriginRequestPolicy.ALL_VIEWER_EXCEPT_HOST_HEADER`가 같은 일을 해 주는데,
> **엣지에서 origin을 직접 교체하면 헤더도 직접 세팅해야 한다.**

SSM 조회는 엣지 인스턴스가 사는 동안 캐시한다.

```js
let cached = null;
async function getBackendUrl() {
  if (cached && cached.expires > Date.now()) return cached.value;
  const out = await ssm.send(new GetParameterCommand({ Name: `/${PROJECT}/backend/url` }));
  cached = { value: out.Parameter?.Value ?? null, expires: Date.now() + 60_000 };
  return cached.value;
}
```

이러면 백엔드를 재배포하고 **SSM 파라미터만 갱신**하면 CloudFront는 손대지 않아도 된다.
CDN 재배포 전파(5~15분)에 묶이지 않는 게 이 패턴의 이유다.

## 배운 점

**Lambda@Edge는 "제약이 붙은 Lambda"가 아니라 다른 실행 환경으로 취급한다.**
프레임워크 기본값이 안 맞는 것부터가 신호다. `NodejsFunction`을 그대로 쓰면 배포가 안 된다.

**"내가 안 썼는데 왜"는 프레임워크가 넣었다는 뜻이다.** `cdk synth` 결과를 열어 보면 바로 보인다.
없는 걸 만드는 문제가 아니라 **있는 걸 끄는 문제**인 경우가 많다.

**우회를 우회하기 전에 원래 문제로 돌아간다.** define 따옴표를 고치느라 시간을 쓰다가,
엣지 함수에 define이 애초에 필요 없다는 걸 뒤늦게 알았다.
우회책의 문제를 푸는 데 빠지면 그 우회가 왜 필요했는지를 잊는다.

**같은 문제가 계층을 바꿔 다시 나온다.** Host 헤더 문제는 CloudFront behavior에서 한 번,
엣지에서 origin을 교체할 때 또 한 번 나왔다. 첫 번째는 managed policy로 풀리고
두 번째는 코드로 풀어야 한다. **"origin을 바꾸면 Host도"를 한 쌍으로 외운다.**

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 09](../../projects/aws-serverless-agent/decisions/09-CF-Function-대신-Lambda-Edge와-SSM.md)
- 배포 실패 진단: [트러블 07](../../projects/aws-serverless-agent/troubleshooting/07-Lambda-Edge는-환경변수를-못-쓴다.md)
- Host 헤더 문제의 첫 등장: [트러블 02](../../projects/aws-serverless-agent/troubleshooting/02-CloudFront를-끼우자-API가-403.md)
