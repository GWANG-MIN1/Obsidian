# 03. API 와 Worker 를 분리했는데 API 가 여전히 Worker 를 기다린다

- **발생**: 2026-06-02 (Day 11) · 저장소 함정 **#13 · #15**
- **증상 한 줄**: 202 를 즉시 돌려주려고 분리했는데 응답이 30초 걸린다

---

## 현상

Day 11 에서 API Lambda 와 Worker Lambda 를 쪼갰다.
의도는 **API 가 Worker 를 부르고 즉시 202 를 반환**하는 것이다.

```
POST /chat  →  API: user 메시지 Put → Worker 호출 → 202 즉시
```

그런데 `curl` 이 30초쯤 매달렸다가 응답한다.
Worker 가 Bedrock 을 다 부르고 끝난 뒤에야 202 가 온다.

**쪼개긴 했는데 동기로 붙어 있다.**

## 환경

- AWS SDK v3 `@aws-sdk/client-lambda`
- API Lambda timeout 30s, Worker Lambda timeout 60s
- `lambda.Alias('live')` 를 두 함수 모두에 붙여 둔 상태 ([결정 02](../decisions/02-API-Gateway-대신-Function-URL.md) 에서 자리 잡아 둔 것)

## 진단

### 1) 호출 코드를 본다

```js
await lambda.send(new InvokeCommand({
  FunctionName: process.env.AGENT_WORKER_FUNCTION_NAME,
  Payload: JSON.stringify({ type: "run_chat", ... }),
}));
```

`InvocationType` 이 없다.

### 2) SDK 기본값을 확인한다 → 여기다

`InvocationType` 을 안 주면 기본값은 **`RequestResponse`** — 동기 호출이다.
즉 Worker 가 끝날 때까지 API 가 기다린다.

"async invoke 를 쓴다"고 결정해 놓고 **명시를 안 했더니 동기로 돌았다.**

## 원인

**`InvocationType` 을 명시하지 않아 SDK 기본값(`RequestResponse`)으로 동기 호출됐다.**

이건 에러가 아니다. 정상 동작이고, **의도만 다르다.**
그래서 로그에도 아무 흔적이 없다. 응답이 느린 것만이 유일한 증상이다.

## 해결

```js
import { InvokeCommand, InvocationType } from "@aws-sdk/client-lambda";

await lambda.send(new InvokeCommand({
  FunctionName: process.env.AGENT_WORKER_FUNCTION_NAME,
  InvocationType: InvocationType.Event,        // ← 항상 명시
  Payload: JSON.stringify({ type: "run_chat", ... }),
}));
return c.json({ sessionId, status: "queued", userSk }, 202);
```

## 이어서 나온 것 — alias 에 권한을 안 주면 거부된다 (#15)

`Event` 로 바꾸고 나니 이번엔 **AccessDenied** 가 났다.

권한은 이렇게 줬었다.

```ts
workerFn.grantInvoke(apiFn);      // 함수 자체에 grant
```

그런데 env 로 넘긴 건 **alias ARN** 이다.

```ts
environment: { AGENT_WORKER_FUNCTION_NAME: workerAlias.functionArn }
```

**Lambda 권한은 qualifier(버전/alias) 단위**다.
함수 자체에 grant 하면 `$LATEST` 만 부를 수 있고, alias ARN 으로 부르면 거부된다.

```ts
workerAlias.grantInvoke(apiFn);   // ← alias 에 grant
```

grant 대상과 호출 대상이 **같은 qualifier** 여야 한다.

## 재발 방지

- **비동기 호출은 `InvocationType` 을 항상 명시한다.** 기본값이 동기라는 사실을
  코드에서 읽을 수 있어야 한다. 생략하면 "왜 동기지?"를 나중에 다시 추적하게 된다.
- **grant 대상과 호출 대상의 qualifier 를 맞춘다.** alias 로 부를 거면 alias 에 grant.
  이건 Lambda 권한 모델의 특성이지 버그가 아니다.
- **응답 시간을 검증 항목에 넣는다.** "202 가 왔다"만 보면 통과한다.
  `curl -w "%{time_total}"` 로 **얼마나 걸렸는지**까지 봐야 이 종류가 잡힌다.

## 같은 day 의 다른 함정 두 개

**#14 — Worker 가 `messages[0].role` 이 user 임을 보장 못 함**

Bedrock Converse 는 첫 메시지가 `user` 여야 한다. 빈 세션이거나
API 의 user Put 이 실패한 채로 Worker 를 부르면 `ValidationException` 이 난다.

두 겹으로 막았다 — Worker 안에 `head === 'user'` 가드, 그리고
**항상 "user Put → invoke" 순서** 유지. 순서가 계약이다.

**#16 — API 에 Bedrock 권한이 남아 있었다**

Day 7 스택을 복사해 오면서 `bedrock:InvokeModel` 이 API 쪽에 그대로 딸려 왔다.
동작에는 문제가 없지만, **"API 는 모델을 모른다"는 책임 분리가 무너진다.**
Day 11 에서 명시적으로 제거했다.

복붙한 IAM 은 눈에 안 띈다. 스택을 옮길 때 권한부터 훑어야 한다.

## 배운 점

**기본값이 조용히 틀린 값인 경우가 가장 찾기 어렵다.** 에러가 없고, 로그가 없고,
동작은 하는데 의도만 다르다. 이 프로젝트에서 같은 유형을 세 번 더 만났다 —
`ScanIndexForward` 기본 `true`(함정 #5), Cost Explorer `End` exclusive
([트러블 08](08-오늘-비용이-0으로-나온다.md)), CloudFront 의 Host 전달
([트러블 02](02-CloudFront를-끼우자-API가-403.md)).

**"분리했다"는 구조 얘기지 동작 얘기가 아니다.** Lambda 를 둘로 나눈 것과
둘이 비동기로 도는 것은 별개다. 구조를 바꿨으면 **동작이 바뀌었는지**를 따로 재야 한다.
