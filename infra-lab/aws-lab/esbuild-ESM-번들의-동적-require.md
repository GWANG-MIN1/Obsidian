# esbuild ESM 번들의 동적 `require`

> 메모: ESM 번들엔 require가 없다. 의존성이 동적 require를 부르면 모듈 로드 시점에 터진다 — banner로 createRequire 주입

---

## 개념

esbuild로 ESM 출력을 만들면 번들 안에 **`require`가 존재하지 않는다.**
그런데 의존성 중 하나가 동적 `require()`를 부르면 이런 에러가 난다.

```
Dynamic require of "util" is not supported
```

### esbuild `__require` shim의 동작

esbuild는 이럴 때 `__require` shim을 넣는다. 동작이 이렇다.

```
전역 require 가 있으면  →  그걸 쓴다
없으면                  →  throw
```

Node의 ESM 실행 컨텍스트에는 전역 `require`가 없다. 그래서 던진다.
CJS 번들에는 `require`가 있으니 **같은 의존성이 CJS에서는 멀쩡히 돈다.**

> **"함수 A는 되는데 B는 안 된다"의 답이 출력 포맷인 경우가 있다.**
> 코드가 같아도 ESM/CJS에 따라 다르게 동작한다.

### 왜 로드 시점에 터지나

동적 `require()`가 **실제로 실행되지 않는 코드 경로**에 있어도 터질 수 있다.
모듈 최상단에서 평가되는 자리에 있으면, 그 모듈을 import하는 것만으로 실행된다.

Lambda에서는 이게 **INIT 단계 크래시**가 된다. 증상 셋이 같이 나온다.

| 증상 | 의미 |
|---|---|
| **모든 요청이 실패** | 특정 라우트 문제가 아니다 |
| **`/health`도 실패** | 핸들러 안 로직 문제가 아니다 |
| **요청 처리 로그가 없음** | 핸들러가 아예 등록되지 않았다 |

셋이 같이 보이면 핸들러 안이 아니라 **모듈 로드**를 본다.

## 실행

### 해결 — banner로 `require`를 만들어 준다

```ts
new nodejs.NodejsFunction(this, 'ApiFunction', {
  bundling: {
    format: nodejs.OutputFormat.ESM,
    banner: 'import { createRequire } from "module"; const require = createRequire(import.meta.url);',
  },
});
```

esbuild CLI라면 이렇게 된다.

```bash
esbuild src/index.mjs --bundle --format=esm --platform=node \
  --banner:js='import { createRequire } from "module"; const require = createRequire(import.meta.url);'
```

이 한 줄이 번들 최상단에 전역 `require`를 만들고,
`__require` shim이 그걸 발견해서 쓴다. **ESM을 유지하면서 CJS로 전환하지 않아도 된다.**

### 다른 선택지

| 방법 | 언제 |
|---|---|
| **banner로 `createRequire`** | ESM을 유지하고 싶을 때. 가장 가볍다 |
| CJS로 출력 | ESM일 이유가 없을 때 (top-level await 안 쓰면) |
| `externalModules`로 제외 | 그 패키지를 런타임이 이미 갖고 있을 때 (Lambda의 AWS SDK 등) |
| 의존성 교체 | 근본적이지만 대개 과하다 |

미사용인데 해석만 되어도 번들이 깨지는 경우는 `externalModules`가 답이다.

```ts
externalModules: ['aws-sdk']    // v2를 안 쓰는데 코드 경로에 require('aws-sdk')가 있다
```

### 확인 — 번들이 로드되는지 테스트한다

`cdk synth`는 CloudFormation 템플릿만 본다. **번들 결과물이 Node에서 로드되는지는 안 본다.**
CI에 넣을 최소 테스트는 이것이다.

```bash
node -e "import('./dist/index.mjs').then(m => {
  if (typeof m.handler !== 'function') { console.error('handler 없음'); process.exit(1); }
  console.log('로드 OK');
})"
```

INIT 크래시는 이 한 줄로 잡힌다.

## 배운 점

**번들 포맷은 런타임 동작을 바꾼다.** ESM/CJS는 문법 차이가 아니라 실행 환경 차이다.
같은 의존성이 한쪽에서만 터지면 포맷을 먼저 의심한다.

**의존성의 미사용 코드 경로도 로드된다.** "그 기능 안 쓰는데?"는 방어가 안 된다.
모듈 최상단에서 평가되는 것은 import만으로 실행된다.

**INIT 크래시는 애플리케이션 에러와 증상이 다르다.**
전부 실패 + `/health`도 실패 + 요청 로그 없음. 이 조합을 기억하면 진단이 몇 분으로 줄어든다.

**"동작한다"는 환경에 대한 진술이지 코드에 대한 진술이 아니다.**
한 환경에서 배포가 됐다는 사실이 다른 환경을 보장하지 않는다.
그래서 **"번들이 로드되는가"는 자동 테스트로 고정**해 둬야 한다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [트러블 12](../../projects/aws-serverless-agent/troubleshooting/12-재배포하니-전-요청이-500.md)
  (프로젝트를 닫고 8일 뒤 재배포에서 발견)
- 원인이 된 의존성: [X-Ray로 Lambda 계측하기](X-Ray-Lambda-계측.md)
