# 07. Lambda@Edge 가 환경변수 때문에 배포부터 실패한다

- **발생**: 2026-06-07 (Day 16) · 저장소 함정 **#39 · #40 · #41**
- **증상 한 줄**: `Lambda@Edge does not support environment variables` — 내가 env 를 준 적이 없는데

---

## 현상

Day 16 에서 CloudFront Function 을 origin-request Lambda@Edge 로 올렸다.
`cdk deploy` 가 이 메시지로 실패한다.

```
Lambda@Edge does not support environment variables
```

**환경변수를 준 적이 없다.** 스택 코드에 `environment:` 가 없다.

## 환경

- AWS CDK v2, `aws-cdk-lib/aws-lambda-nodejs` 의 `NodejsFunction`
- Lambda@Edge (us-east-1 고정)
- 개발 환경: Windows + PowerShell

## 진단

### 1) 합성된 템플릿을 본다 → env 가 있다

`cdk synth` 결과를 열어 보면 Lambda 정의에 환경변수가 하나 들어 있다.

```json
"Environment": { "Variables": { "AWS_NODEJS_CONNECTION_REUSE_ENABLED": "1" } }
```

**`NodejsFunction` 이 기본으로 넣는다.** AWS SDK 의 TCP 커넥션 재사용을 켜는 값으로,
일반 Lambda 에선 성능에 좋다. Lambda@Edge 에선 배포를 막는다.

내가 안 쓴 것 때문에 실패한 것이다.

### 2) 설정값은 어떻게 주입하나 → env 를 못 쓰면 방법이 하나뿐

`PROJECT` 같은 값을 넘길 방법이 필요한데 env 가 막혔다.
esbuild 의 `--define` 으로 빌드 타임 치환을 시도했다.

### 3) Windows 에서 define 이 깨진다 (#40)

```
esbuild: Invalid define value
```

`--define:process.env.PROJECT="serverless-agent"` 의 **안쪽 따옴표가 셸에서 벗겨져서**
`=serverless-agent` 로 전달된다. PowerShell 의 인자 파싱 문제다.

이스케이프를 이리저리 바꿔 보다가, **애초에 define 이 필요한지**를 다시 물었다.

### 4) Function URL 을 URL 로 파싱하려니 synth 가 깨진다 (#41)

엣지 함수에 백엔드 host 를 넘기려고 스택에서 이렇게 썼다.

```ts
const host = new URL(backendFn.functionUrl.url).hostname;   // synth 에러
```

같은 스택 안의 Function URL 은 **CDK 토큰**이다.
synth 시점에는 `${Token[TOKEN.123]}` 같은 문자열이라 URL 파싱이 안 된다.
실제 값은 **deploy 시점**에 결정된다.

## 원인

세 가지가 Lambda@Edge 의 제약 하나에서 파생됐다.

**#39 — `NodejsFunction` 의 기본 env 가 Lambda@Edge 제약과 충돌.**
내가 안 준 값이라 코드를 봐서는 안 보인다.

**#40 — Windows 셸에서 esbuild `define` 의 따옴표가 벗겨짐.**
#39 를 우회하려다 만난 2차 문제.

**#41 — 같은 스택의 Function URL 은 deploy-time 토큰이라 synth 때 파싱 불가.**

## 해결

### #39 — 기본 env 를 끈다

```ts
new nodejs.NodejsFunction(this, 'EdgeFn', {
  awsSdkConnectionReuse: false,     // ← AWS_NODEJS_CONNECTION_REUSE_ENABLED 를 안 넣는다
  ...
});
```

### #40 — define 을 안 쓰고 소스 상수로

```js
const PROJECT = "serverless-agent";   // 엣지는 어차피 빌드타임 고정
const SSM_REGION = "us-east-1";
```

**엣지 함수는 환경변수를 못 쓰므로 설정값이 어차피 빌드 타임에 고정된다.**
그러면 define 으로 주입하나 소스에 적나 결과가 같다. 소스에 적는 쪽이 셸 인용 문제도 없고 읽기도 쉽다.

우회를 우회하려다 **원래 문제가 우회할 가치가 없다는 걸** 알았다.

### #41 — CloudFormation 내장 함수로 deploy-time 에 자른다

```ts
const host = cdk.Fn.select(2, cdk.Fn.split('/', backendFn.functionUrl.url));
// "https://xxx.lambda-url.us-east-1.on.aws/" → ["https:", "", "xxx.lambda-url...", ""]
```

`Fn.select`/`Fn.split` 은 **CloudFormation 이 deploy 때 평가**한다.
JS 로 파싱하지 않고 템플릿에 표현식으로 남긴다.

## Lambda@Edge 제약 정리

이 day 에서 실배포로 확인한 것들.

| 제약 | 결과 |
|---|---|
| **us-east-1 필수** | 다른 리전에 만들면 CloudFront 가 연결을 거부 (#42) |
| **환경변수 불가** | 설정은 소스 상수 (#39) |
| **삭제 지연** | `destroy` 가 실패한다 — 복제본이 전 리전에 남아 있어서. **정상**이고, CloudFront 가 내려간 뒤 시간 두고 재시도 (#46) |
| **X-Ray active tracing 미지원** | Day 20 에서 `Tracing.ACTIVE` 를 엣지만 제외해야 했다 (#65) |
| 패키지 크기 / 실행시간 상한이 일반 Lambda보다 작음 | 이번엔 안 걸렸다 |

**#46 이 특히 헷갈렸다.** `destroy` 실패가 사고처럼 보이는데 정상 동작이다.
CloudFront 배포가 완전히 내려가야 복제본이 정리된다.

## 재발 방지

- **Lambda@Edge 는 "일반 Lambda 에 제약이 붙은 것"이 아니라 다른 실행 환경으로 취급한다.**
  `NodejsFunction` 의 기본값이 안 맞는 것부터가 그 신호다.
- **`cdk synth` 결과를 읽는다.** 내가 안 쓴 설정이 들어가 있을 수 있다.
  이 문제는 템플릿을 열자마자 보였다.
- **CDK 토큰을 JS 로 조작하지 않는다.** deploy-time 값이 필요하면 `Fn.*` 계열을 쓴다.
  `new URL(token)`, `token.split()`, `token.includes()` 는 전부 안 된다.
- **셸 인용 문제를 만나면 그 인자가 꼭 필요한지 먼저 묻는다.** 이번엔 아니었다.

## 배운 점

**우회를 우회하기 전에 원래 문제로 돌아간다.** define 따옴표를 고치느라 시간을 썼는데,
**엣지 함수에 define 이 애초에 필요 없었다.** 우회책의 문제를 푸는 데 빠지면
그 우회가 왜 필요했는지를 잊는다.

**"내가 안 썼는데 왜"는 프레임워크가 넣었다는 뜻이다.** CDK 는 편의를 위해 기본값을 넣는다.
편의가 제약과 충돌하면 그 기본값을 끄는 옵션이 대개 있다 —
없는 걸 만드는 게 아니라 있는 걸 끄는 문제였다.
